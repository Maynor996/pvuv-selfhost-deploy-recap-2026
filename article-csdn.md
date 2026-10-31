# 自建流量监控系统部署复盘：Cloudflare Workers 上的 pvuv（analytics.momoai.asia）

> 开源仓库：github.com/qiayue/pvuv.ai（MIT）  
> 自建控制台：`analytics.momoai.asia` · 采集端：`in.momoai.asia` · 埋点脚本：`f.js`  
> 部署账号：自有 Cloudflare（QQ 账号侧）· 登录：Google OAuth · 已接入约 76 个站点  
> 时间：2026-10-31（Asia/Shanghai）

这是一次把开源 **pvuv.ai** 完整自建到自家域名上的部署复盘。目标很简单：流量数据留在自己的 Cloudflare 账号里，同时把「流量质量」和「广告防护」当成一等公民，而不是事后再补一层 bot 过滤。

---

## 1. 为何自建

我们站点矩阵不小，视频梗站、AI 工具站混在一起。如果继续把 PV/UV 交给第三方托管统计，会有几件事长期别扭：

1. **隐私与数据归属**  
   访客行为、来源、UTM、会话链路都落在别人的库里。自建后，D1 / KV / Queues 全在自己的 Cloudflare 账号，不经过第三方分析服务。

2. **流量质量要当一等公民**  
   多数自托管脚本只「数请求」。真人、可疑、机器人、已识别爬虫搅在一起，看总数很容易自我安慰。pvuv 从设计上就把真实性打分和判定（正常 / 可疑 / 机器人 / 爬虫）做成主面板能力。

3. **广告防护**  
   靠广告变现的站，无效流量不只是「脏数据」，还可能牵连广告账号。pvuv 可以按档位估算拦截影响，再决定是否只给可信流量注入广告代码（默认 fail-open，误伤收入宁可少拦）。

4. **运维模型匹配**  
   我们已经在 Cloudflare Workers 上跑站点；再加一套 Workers + D1 的统计，不需要自建机房或外部数据库。

托管版 pvuv.ai 也很好用；我们选自建，主要是为了数据和策略都捏在自己手里。

---

## 2. 架构（对照开源 README，域名换成自家）

开源 README 里的链路，我们按自有域名落成了这样：

```
浏览器 (f.js)
  → in.momoai.asia   ingest Worker：校验 + 服务端补全 + 实时打分
  → Cloudflare Queue
  → consumer Worker：批量写 D1 events，更新 sessions / 画像
  → Cron（按小时）  rollup 预聚合
  → Cron（按天）    群体分析 → 改判 + KV 黑名单
  analytics.momoai.asia   控制台（看板、流量质量、CWV、广告防护）
```

技术栈与上游一致：**Workers**（ingest / consumer / api / console / cron）· **D1**（按月分表事件）· **Queues**（削峰）· **KV**（黑名单与配置缓存）· Cron Triggers。无外部数据库。

域名职责也按上游「暴露子域 / 内部子域」思路拆开：

| 角色 | 我们的域名 |
|------|------------|
| 控制台 | `analytics.momoai.asia` |
| 采集 / 判定 | `in.momoai.asia` |
| 埋点 | `f.js`（由控制台/静态资源下发；嵌入时 `data-api` 必须指向自家 ingest） |

自部署时，站点嵌入代码里的 **`data-api` 必须写成 `https://in.momoai.asia`**。SDK 内置默认值仍是参考域名，不改就会把数据打到别人家。

控制台侧我们日常用到的能力包括：

- 「你的站点」网格，**按浏览量（PV）排序**的站点卡，并直接标出真人比例  
- **浅色 / 深色**主题切换  
- 单站概览（访客、会话、浏览量、跳出、真人占比、异常色带）  
- **Core Web Vitals（CWV）**：LCP / INP / CLS / FCP / TTFB 的 p75  
- **流量质量**环图 + **广告防护拦截档位**表（平衡 / 激进 / 最严格）

---

## 3. 踩坑

### 3.1 Google OAuth 与 `user_id` / `email` UNIQUE

控制台是 OAuth-only（我们用的是 Google）。D1 里 `users` 表大致是：

- `user_id` 主键  
- `email` **UNIQUE**

自建版是**单租户**：所有写在 `ADMIN_EMAILS` 里的管理员邮箱，登录后都映射到同一个内部 owner id（源码里固定为 `admin`），共享一份站点列表。

踩坑现象：如果早期种子数据 / 批处理建用户时，把 **邮箱字符串当成了 `user_id`**，库里会先留下一行「email 既当主键又当 UNIQUE 字段」的记录。之后真正走 OAuth 回调时，逻辑要插入 `user_id = admin` 且带上同一邮箱——就会撞上 **`email` UNIQUE**，回调直接失败（表现成登录失败 / 服务端 500，具体看 `wrangler tail`）。

修法方向（已在开源 `resolveUser` 里体现，方向如下，勿照抄未核实的临时 SQL）：

1. 查出「同一邮箱、但 `user_id` 不是 `admin`」的旧行；  
2. 把该旧 id 名下的 `sites.owner_id`、`api_tokens.owner_id` 迁到 `admin`；  
3. 删掉旧用户行，再对 `admin` 做 `ON CONFLICT(user_id)` 的 upsert，更新 email / name。

部署时还要把 Google OAuth 回调指到：

`https://analytics.momoai.asia/api/auth/google/callback`

并把真实可登录邮箱写进控制台 Worker 的 `ADMIN_EMAILS`（不要把 client secret 写进仓库）。

### 3.2 自检误用错误的 ingest 主机名

控制台有「自检」：一边访问自己的页面，一边确认事件是否进库。我们一度把采集端想成类似 `in.analytics.…` 的拼接——**错**。

正确采集主机是部署时绑定的：

**`in.momoai.asia`**

自检、嵌入代码、`data-api`、文档里的 ingest 示例，都应对齐这个主机。控制台域名 `analytics.momoai.asia` 只负责看板与登录，不负责收 beacon。混用会表现为「页面能开、自检永远没数据」。

---

## 4. 效果

- **接入规模**：站点注册表与控制台侧约 **76** 个站点已嵌入。  
- **站点卡**：按 PV 排列，卡上直接看会话数与真人比例，矩阵一眼可扫。  
- **流量质量**：例如 RemixMeme 某一窗口下，正常 / 可疑 / 机器人 / 爬虫可以拆开看；拦截档位表会估算「平衡 / 激进 / 最严格」各自会挡掉多少浏览量、以及预计误伤上限。  
- **CWV 旁证（RemixMeme）**：干净流量上的 LCP p75 从早期约 **7.5s 量级（截图里曾见 7.74s，标「较差」）** 收敛到约 **3.15s（需改进）**——统计系统本身不「优化」页面，但 CWV 面板让我们能用同一套自建数据盯性能，而不是另开一套第三方 RUM。

下面五张图都是控制台实机截图（深色主题；浅色模式控制台已支持，本篇未另截）。

![RemixMeme：CWV（LCP p75=3.15s 需改进）+ 流量质量环图（正常11 / 可疑26 / 机器人3 / 爬虫76）+ 拦截档位表](https://upload.maynor1024.live/file/1793436245685_01-remixmeme-cwv-quality-tiers.png)

*图 1 · RemixMeme 核心网页指标与流量质量：LCP p75 3.15s；质量环为正常 11 / 可疑 26 / 机器人 3 / 已验证爬虫 76；下方为平衡 / 激进 / 最严格三档拦截估算（当时广告防护模式为关闭）。*

![时间筛选「最近 7 天」与站点卡上的真人比例](https://upload.maynor1024.live/file/1793436240187_02-time-filter-site-cards.png)

*图 2 · 顶部时间筛选（滚动：最近 24 小时 / 今天 / 最近 7 天…）；站点卡片段可见 RemixMeme 等卡的会话数与「真人%」标签。*

![「你的站点」网格按浏览量排列](https://upload.maynor1024.live/file/1793436246632_03-sites-grid-by-pv.png)

*图 3 · 「你的站点」网格：FindAIForMe、RemixMeme 等按浏览量排序，卡上同时展示访客 / 会话与真人比例。*

![FindAIForMe 单站概览：无效流量占比告警](https://upload.maynor1024.live/file/1793436243965_04-findaiforme-overview-alert.png)

*图 4 · FindAIForMe 单站概览：顶部黄色告警「无效流量占比过高」（示例窗口内机器人/可疑约占 43.1%，告警线 30%）；KPI 含真人占比、跳出率与浏览量时序上的异常色带。*

![RemixMeme 早期 CWV：LCP p75 约 7.74s（较差）](https://upload.maynor1024.live/file/1793436247773_05-remixmeme-cwv-before.png)

*图 5 · RemixMeme 较早窗口的 CWV：LCP p75 7.74s、FCP 7.52s 标「较差」（样本较少）；可与图 1 的 3.15s 对照，作为性能收敛的旁证。*

---

## 5. 小结与可复用清单

1. 先确认 Cloudflare 域名 DNS 已托管，且 Workers **付费套餐**（Queues 硬依赖）。  
2. 按上游 `DEPLOY.zh-CN.md` / `npm run setup` 拉齐 Worker、D1、KV、Queue 与自定义域。  
3. OAuth 回调与 `ADMIN_EMAILS` 配齐；留意 `users.email` UNIQUE 与单租户 `admin` id 的迁移。  
4. 嵌入与自检一律指向 **`in.momoai.asia`**，不要臆造 `in.analytics…`。  
5. 先观察流量质量与广告防护「影子/估算」，再改拦截档位。

开源上游仍可参考：github.com/qiayue/pvuv.ai

（本文不记录任何 token、OAuth client secret、cookie 或账号 ID。）
