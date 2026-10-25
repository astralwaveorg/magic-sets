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

每个文件是一个标准规则集合，文件头注明 behavior、条目数与用途：

```yaml
# 名称: chat-extra
# behavior: domain
# 条目: 43
# 说明: 即时通讯补充域名
payload:
  - +.example.com
```

`behavior` 三种，与 [Stash 官方文档](https://stash.wiki/rules/rule-set) 一致：

| behavior | 支持内容 | 写法 |
| --- | --- | --- |
| `domain` | 域名 / 域名通配符 | `+.example.com` 匹配该域及其子域，`example.com` 精确匹配 |
| `ipcidr` | IPv4 / IPv6 CIDR | `158.179.167.131/32` |
| `classical` | 任意规则类型 | `DOMAIN-KEYWORD,abema`、`DST-PORT,123` |

引用方式：

```yaml
rule-providers:
  chat-extra:
    type: http
    behavior: domain
    format: yaml
    path: ./rules/chat-extra.yaml
    url: https://raw.githubusercontent.com/astralwaveorg/magic-sets/main/rule/chat-extra.yaml
    interval: 86400
```

`format` 用 `yaml`。Stash 对 `domain` / `ipcidr` 的 `mrs` 二进制格式做了专门压缩优化，
条目数很大时内存更省；本仓库条目量级不大，统一用 `yaml` 便于维护与审阅。

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

| 文件 | behavior | 内容 | 条目 |
| --- | --- | --- | --- |
| `rule/chat-extra.yaml` | `domain` | 域名 | 43 |
| `rule/jp.yaml` | `domain` | 域名 | 9 |
| `rule/kr.yaml` | `ipcidr` | IP 段 | 1 |
| `rule/local-process.yaml` | `classical` | 经典规则 | 15 |
| `rule/social-niche.yaml` | `domain` | 域名 | 16 |

本仓库只收**上游与第三方都没有收录、需要自己维护**的集合：

- `chat-extra` / `social-niche`：小众聊天应用与社区平台的补充域名
- `local-process`：本地传输、同步与远控类进程，保持直连
- `jp` / `kr`：自建服务所在地区的域名与 IP 段

通用规则集（geosite / geoip / blackmatrix7 的分类集合）一律不进本仓库 —— 面板里直接引用
上游 URL 即可，避免同一份内容存两处、走两套更新。

## 命名约定

- 文件名用小写英文加短横线；IP 段的加 `-ip` 后缀（`apple-push` / `apple-push-ip`）
- 中文展示名写在文件头的注释里，不进文件名，避免 URL 里的百分号编码
- 新增条目追加到对应文件末尾，保持原有排序习惯，不要重排既有内容

## 与 MagicPanel 的关系

本仓库由 MagicPanel 后台的「自主维护」页托管：面板把它 clone 到容器里，在页面上直接浏览、
编辑、提交、推送。`rule/` 下的文件会按「仓库托管」这一类规则来源同步进面板，内容以仓库为准。

面板里的规则来源有四种，边界要分清：

| 类型 | 含义 | 内容来源 |
| --- | --- | --- |
| 远程 URL | 直接引用上游（geosite、blackmatrix7 等） | 上游 URL，面板定期抓取 |
| 内联内容 | 直接写在面板里 | 面板数据库，改完保存即刻生效 |
| 维护仓库 | 由本仓库托管 | `rule/*.yaml`，同步时以仓库为准 |

规则来源改成「远程 URL」或「内联内容」后，它就不再消费本仓库的文件，对应的仓库文件可以删掉；
反之要把一个集合收回本仓库，就在仓库里放好文件、在面板上把来源改成「维护仓库」并填上仓库路径。
