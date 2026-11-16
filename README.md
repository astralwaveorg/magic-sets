# magic-sets

自维护的规则集合，供 MagicPanel 下发，也供 Stash / Mihomo 客户端直接订阅。

仓库只放**自己维护**的内容。规模大、更新频繁的通用规则集（geosite / geoip 等）不在这里，
它们由客户端从上游项目拉取，本仓库只维护增量与私有部分。

## 目录结构

| 目录 | 内容 | 用途 |
| --- | --- | --- |
| `rule/` | 规则集合 | 域名、IP 段、进程名等经典规则的批量定义，对应 `rule-providers` |

`script/` 与 `rewrite/` 已于 2026-11-17 移除：这些 Stash 专属片段（UA 改写、请求头去跟踪、
强制 HTTP 引擎）默认停用且面板从未引用，保留只会增加维护面；需要时从 git 历史找回。

## rule/

本仓库的集合**一律用 `classical` 行为**（`magic-us` 除外，用 `domain`）：条目少，但类型可以混着写，
域名、IP 段、进程名放在一个文件里，配一个规则就能表达一整类分流意图。

```yaml
# 名称: jp
# behavior: classical
# 条目: 8
# 说明: 日本线路的域名
payload:
  - DOMAIN-SUFFIX,rsshub.in
  - IP-CIDR,158.179.167.131/32
```

`classical` 里能写的类型，与 [Stash 官方文档](https://stash.wiki/rules/rule-types) 一致：

| 写法 | 含义 |
| --- | --- |
| `DOMAIN,example.com` | 精确匹配这一个域名 |
| `DOMAIN-SUFFIX,example.com` | 该域及其全部子域 |
| `DOMAIN-KEYWORD,abema` | 域名里含这个关键词 |
| `IP-CIDR,158.179.167.131/32` | IPv4 / IPv6 网段 |
| `PROCESS-NAME,qbittorrent` | 进程名 |

注意 `DOMAIN` 与 `DOMAIN-SUFFIX` 的区别：原来 `domain` 行为里不带 `+.` 的是精确匹配，
换成 `classical` 时要写成 `DOMAIN,`，写成 `DOMAIN-SUFFIX,` 会把整个域和子域都算进去。

## 2026-11-17 审计与瘦身

按产物规则顺序做「首个命中」仿真，把被更靠前规则截杀的死条目全部移除：

- `magic-chat`：Discord×6、WhatsApp×2、LINE×4、Kakao×2、Slack×2 被
  blackmatrix7 同目标专集覆盖，`zoom.us` 误分类（被 `magic-dev` 截走）；补充
  Element / Matrix 家族 3 条。43 → 29
- `magic-dev`：sentry / huggingface / datadoghq 被 `global-ai` 覆盖（AI 组），
  `zoom.us` 归位到即时通讯。34 → 30
- `magic-jp`：panel/bot.eoysky.com 被 `magic-direct` 的 eoysky.com 后缀直连
  （自建域名直连更安全），死条目移除。10 → 8
- `magic-adult`：pornhub.com 已由 `proxy-bm7` 覆盖（同目标）。9 → 8
- `magic-proxy`：补充 ip.sb / ifconfig.me / icanhazip.com（IP 查询，配合面板
  解锁看板与测速脚本）。18 → 21

删除默认停用且零引用的 `script/` 与 `rewrite/` 目录（6 个 Stash 专属片段）。

## 规则清单

| 文件 | 说明 | 条目 | 规则指向 |
| --- | --- | --- | --- |
| `rule/magic-proxy.yaml` | IP 查询、云服务、自建与金融站点 | 21 | `🚀 默认代理` |
| `rule/magic-direct.yaml` | 自建服务、国内直连站点与广告 SDK | 19 | `DIRECT` |
| `rule/magic-reject.yaml` | 最小广告域名集 | 4 | `REJECT` / `🛑 广告过滤` |
| `rule/magic-jp.yaml` | 日本线路域名 + 韩国固定 IP 段 | 8 | `🇯🇵日本` |
| `rule/magic-process.yaml` | 本地传输、同步与远控类进程 | 15 | `DIRECT` |
| `rule/magic-chat.yaml` | 小众聊天应用补充（Element/Matrix 等） | 29 | `💬 即时通讯` |
| `rule/magic-social.yaml` | 小众社区平台补充 | 16 | `👥 海外社交` |
| `rule/magic-game.yaml` | Steam 下载 CDN | 2 | `🎮 海外游戏` |
| `rule/magic-us.yaml` | 美国区域域名 | 2 | `🧠 人工智能` |
| `rule/magic-google.yaml` | 谷歌服务补充域名 | 1 | `🔍 谷歌服务` |
| `rule/magic-microsoft.yaml` | 微软服务补充域名 | 1 | `Ⓜ️ 微软服务` |
| `rule/magic-adult.yaml` | 成人内容站点域名 | 8 | `🚀 默认代理` |
| `rule/magic-dev.yaml` | 开发者平台、工具与包源域名 | 30 | `🚀 默认代理` |

**一个目标组一个文件**：想加一条「这个域名走默认代理」「这个域名直连」「这个域名拦截」，
加到对应文件里就行，不必为每个服务单独建集合。`magic-proxy` / `magic-direct` / `magic-reject`
是三种通用意图的固定落点，其余按分组各一个文件。

通用规则集（geosite / geoip / blackmatrix7 的分类集合）一律不进本仓库 —— 面板里直接引用
上游 URL 即可，避免同一份内容存两处、走两套更新。新增条目前先按产物顺序确认它没有被
更靠前的规则截杀（死条目等于维护噪声）。

## 命名约定

- 文件名用小写英文加短横线，**一律带 `magic-` 前缀**（表示自维护）；文件即引用键，改名要同步改引用它的规则
- 中文展示名写在文件头的注释里，不进文件名，避免 URL 里的百分号编码
- 新增条目追加到对应文件末尾，保持原有排序习惯，不要重排既有内容

## 与 MagicPanel 的关系

本仓库由 MagicPanel 后台的「自主维护」页托管：面板把它 clone 到容器里，在页面上直接浏览、
编辑、提交、推送。`rule/` 下的文件会按「维护仓库」这一类规则来源同步进面板，内容以仓库为准。

面板里的规则来源有三种，边界要分清：

| 类型 | 含义 | 内容来源 |
| --- | --- | --- |
| 远程 URL | 直接引用上游（geosite、blackmatrix7 等） | 上游 URL，面板定期抓取 |
| 内联内容 | 直接写在面板里 | 面板数据库，改完保存即刻生效 |
| 维护仓库 | 由本仓库托管 | `rule/*.yaml`，同步时以仓库为准 |

规则来源改成「远程 URL」或「内联内容」后，它就不再消费本仓库的文件，对应的仓库文件可以删掉；
反之要把一个集合收回本仓库，就在仓库里放好文件、在面板上把来源改成「维护仓库」并填上仓库路径。
