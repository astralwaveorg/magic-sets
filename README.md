# magic-sets

自维护的规则与脚本集合，供 MagicPanel 下发，也供 Stash / Mihomo 客户端直接订阅。

仓库只放**自己维护**的内容。规模大、更新频繁的通用规则集（geosite / geoip 等）不在这里，
它们由客户端从上游项目拉取，本仓库只维护增量与私有部分。

## 目录结构

| 目录 | 内容 | 用途 |
| --- | --- | --- |
| `rule/` | 规则集合 | 域名、IP 段、经典规则的批量定义，对应 `rule-providers` |
| `script/` | Stash JavaScript 脚本 | 需要逻辑判断的请求改写，对应 `script-providers` + `script` |
| `rewrite/` | Stash HTTP 重写片段 | URL 重写、请求头增删、强制 HTTP 引擎，对应 `http` 段 |

## rule/

本仓库的集合**一律用 `classical` 行为**（`magic-us` 除外，用 `domain`）：条目少，但类型可以混着写，
域名、IP 段、进程名放在一个文件里，配一个规则就能表达一整类分流意图。

```yaml
# 名称: jp
# behavior: classical
# 条目: 10
# 说明: 日本线路的域名与韩国的固定 IP 段
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

引用方式：

```yaml
rule-providers:
  magic-jp:
    type: http
    behavior: classical
    format: yaml
    path: ./rules/magic-jp.yaml
    url: https://raw.githubusercontent.com/astralwaveorg/magic-sets/main/rule/magic-jp.yaml
    interval: 86400
```

## script/ 与 rewrite/

都是 Stash 的功能片段，**默认停用**，按需启用。

`script/` 每个文件包含两段：`script-providers` 声明脚本体，`script` 声明它作用在哪些请求上：

```yaml
script-providers:
  ua-mobile:
    interval: 86400
    payload: |
      $request.headers["User-Agent"] = "...";
      $done();

script:
  - type: request
    name: ua-mobile
    match: "^https?://"
```

`rewrite/` 每个文件是 `http` 段的一个片段：

```yaml
http:
  url-rewrite:
    - '^https?://[a-z0-9-]+\.doubleclick\.net/.* - reject'
```

`http` 段的 URL 重写与请求头改写依赖 HTTP 引擎，HTTPS 请求需要设备信任 Stash 的 CA 证书。

## 规则清单

| 文件 | 说明 | 条目 | 规则指向 |
| --- | --- | --- | --- |
| `rule/magic-proxy.yaml` | 自维护的「走默认代理」条目（IP 查询、云服务、自建、金融） | 16 | `🚀 默认代理` |
| `rule/magic-direct.yaml` | 自维护的「直连」条目 | 7 | `DIRECT` |
| `rule/magic-reject.yaml` | 自维护的「拦截」条目 | 4 | `REJECT` / `🛑 广告过滤` |
| `rule/magic-jp.yaml` | 日本线路域名 + 韩国的固定 IP 段 | 10 | `🇯🇵日本` |
| `rule/magic-process.yaml` | 本地传输、同步与远控类进程 | 15 | `DIRECT` |
| `rule/magic-chat.yaml` | 即时通讯补充域名 | 43 | `💬 即时通讯` |
| `rule/magic-social.yaml` | 海外社交补充域名 | 16 | `👥 海外社交` |
| `rule/magic-game.yaml` | Steam 下载 CDN（服务器在境外） | 2 | `🎮 海外游戏` |
| `rule/magic-us.yaml` | 美国区域域名 | 2 | `🇺🇸美国` |
| `rule/magic-google.yaml` | 谷歌服务补充域名 | 1 | `🔍 谷歌服务` |
| `rule/magic-microsoft.yaml` | 微软服务补充域名 | 1 | `Ⓜ️ 微软服务` |
| `rule/magic-adult.yaml` | 成人内容站点域名 | 9 | `🚀 默认代理` |
| `rule/magic-dev.yaml` | 开发者平台、工具与包源域名 | 34 | `🚀 默认代理` |

**一个目标组一个文件**：想加一条「这个域名走默认代理」「这个域名直连」「这个域名拦截」，
加到对应文件里就行，不必为每个服务单独建集合。`magic-proxy` / `magic-direct` / `magic-reject`
是三种通用意图的固定落点，其余按分组各一个文件。

通用规则集（geosite / geoip / blackmatrix7 的分类集合）一律不进本仓库 —— 面板里直接引用
上游 URL 即可，避免同一份内容存两处、走两套更新。

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
