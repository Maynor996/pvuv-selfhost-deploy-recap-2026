# HANDOFF：pvuv 自建部署复盘 → CSDN

**状态（Asia/Shanghai 2026-10-31）：GitHub Pages 已上线（HTTP 200）；CSDN 待父代理浏览器发到 xianyu120（勿重复发）。用户（上站工具3）已授权发文，无需再确认草稿。**

---

## 已完成：本地成稿

| 项 | 路径 |
| --- | --- |
| 工作目录 | `/workspace/pvuv-selfhost-deploy-recap-2026/` |
| 正文（GitHub） | `article.md` |
| 正文（CSDN，软链纯文本） | `article-csdn.md` |
| 可读 HTML | `index.html` |
| 本地五图 | `assets/` |
| 标题 | `csdn-title.txt` |
| 元信息 | `csdn-meta.txt` |
| 图床列表 | `upload-urls.json` |

---

## GitHub / Pages（Maynor996）

| 项 | 值 |
| --- | --- |
| 仓库 | https://github.com/Maynor996/pvuv-selfhost-deploy-recap-2026 |
| Pages | https://maynor996.github.io/pvuv-selfhost-deploy-recap-2026/ |
| About homepage | Pages URL |
| About description | 自建 pvuv 流量分析部署复盘 · analytics.example.com · 开源 qiayue/pvuv.ai |
| 源 | `main` + `/` |

Pages 已确认 HTTP 200（`/` 与 `/index.html`）；About homepage/description 已设。

---

## CSDN 待父代理 computerUse 发布（xianyu120）

executor **无浏览器**，请父代理用浏览器登录 CSDN 账号 **xianyu120** 发文。

| 项 | 内容 |
| --- | --- |
| 标题 | 自建流量监控系统部署复盘：Cloudflare Workers 上的 pvuv |
| 可贴正文 | `/workspace/pvuv-selfhost-deploy-recap-2026/article-csdn.md` |
| 类型 | **原创** |
| 可见性 | **全部可见** |
| 建议标签 | `Cloudflare`,`Workers`,`D1`,`流量统计`,`pvuv`,`Web分析`,`Core Web Vitals`,`反爬` |
| 建议专栏/分类 | 云计算 / 后端 / 经验分享（以后台可选为准） |
| 摘要 | 把开源 pvuv.ai 自建到 analytics.example.com / in.example.com 的部署复盘：为何自建、Workers+D1+Queue 架构、Google OAuth 与 users.email UNIQUE 坑、自检误用 ingest 主机名、约 76 站接入与 站点 A CWV（LCP ~7.5s→3.15s）旁证。含控制台实机截图。 |

### 发文注意

- 图片优先用正文内图床 URL（upload.maynor1024.live）；外链转存失败再本地上传 `assets/` 五图
- 文末 **analytics.example.com / in.example.com / github.com/qiayue/pvuv.ai / example.com** 保持**纯文本**，勿改成 markdown 蓝链
- 不要写入任何 token、OAuth secret、cookie、wrangler account id
- 发完核对审核状态，回报文章链接；父代理再回 上站工具3（GitHub Pages + CSDN 两链）

### 图床 URL（5）

1. https://upload.maynor1024.live/file/1793440519091_01-site-a-cwv-quality-tiers.png  
   （CWV LCP 3.15s + 流量质量环 + 拦截档位）
2. https://upload.maynor1024.live/file/1793440521163_02-time-filter-site-cards.png  
   （最近 7 天筛选 + 站点卡真人%）
3. https://upload.maynor1024.live/file/1793440531493_03-sites-grid-by-pv.png  
   （你的站点网格，按 PV）
4. https://upload.maynor1024.live/file/1793440528620_04-site-b-overview-alert.png  
   （站点 B 单站概览 + 无效流量告警）
5. https://upload.maynor1024.live/file/1793440535351_05-site-a-cwv-before.png  
   （站点 A 早期 CWV LCP 7.74s）

### 本地 fallback

`/workspace/pvuv-selfhost-deploy-recap-2026/assets/`

- `01-remixmeme-cwv-quality-tiers.png`
- `02-time-filter-site-cards.png`
- `03-sites-grid-by-pv.png`
- `04-findaiforme-overview-alert.png`
- `05-remixmeme-cwv-before.png`

---

## 标题（最终）

**自建流量监控系统部署复盘：Cloudflare Workers 上的 pvuv**
