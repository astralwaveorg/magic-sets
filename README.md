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
# 名称: ai-overrides
# behavior: domain
# 条目: 47
# 说明: AI 服务补充域名
payload:
  - +.copilot.microsoft.com
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
  ai-overrides:
    type: http
    behavior: domain
    format: yaml
    path: ./rules/ai-overrides.yaml
    url: https://raw.githubusercontent.com/astralwaveorg/magic-sets/main/rule/ai-overrides.yaml
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
| `rule/ads-minimal.yaml` | `domain` | 域名 | 4 |
| `rule/ai-overrides.yaml` | `domain` | 域名 | 47 |
| `rule/apple-intelligence.yaml` | `domain` | 域名 | 4 |
| `rule/apple-push-ip.yaml` | `ipcidr` | IP 段 | 9 |
| `rule/apple-push.yaml` | `domain` | 域名 | 3 |
| `rule/chat-extra.yaml` | `domain` | 域名 | 43 |
| `rule/claude-extra.yaml` | `domain` | 域名 | 11 |
| `rule/cn-services.yaml` | `domain` | 域名 | 236 |
| `rule/connectivity-check.yaml` | `domain` | 域名 | 6 |
| `rule/dev-extra.yaml` | `domain` | 域名 | 71 |
| `rule/direct-ports.yaml` | `classical` | 经典规则 | 3 |
| `rule/dns-direct.yaml` | `domain` | 域名 | 4 |
| `rule/gemini.yaml` | `domain` | 域名 | 13 |
| `rule/jp.yaml` | `domain` | 域名 | 8 |
| `rule/kr.yaml` | `ipcidr` | IP 段 | 1 |
| `rule/local-process.yaml` | `classical` | 经典规则 | 15 |
| `rule/lvsp-direct.yaml` | `domain` | 域名 | 7 |
| `rule/lvsp-ip-check.yaml` | `domain` | 域名 | 6 |
| `rule/mail-password.yaml` | `domain` | 域名 | 4 |
| `rule/mirror-accelerate.yaml` | `domain` | 域名 | 2 |
| `rule/openai-extra.yaml` | `domain` | 域名 | 13 |
| `rule/payment-finance.yaml` | `domain` | 域名 | 8 |
| `rule/proxy-domain.yaml` | `domain` | 域名 | 6207 |
| `rule/proxy.yaml` | `classical` | 经典规则 | 100 |
| `rule/search.yaml` | `domain` | 域名 | 17 |
| `rule/self-hosted.yaml` | `domain` | 域名 | 5 |
| `rule/social-extra.yaml` | `domain` | 域名 | 14 |
| `rule/social-niche.yaml` | `domain` | 域名 | 16 |

`rule/proxy-domain.yaml` 是「必须走代理」的域名全集，已剔除与现有分流规则重叠的部分 ——
上游的 Proxy 规则集排在业务规则之前，会把本该走专用线路的域名抢走。

## 命名约定

- 文件名用小写英文加短横线；IP 段的加 `-ip` 后缀（`apple-push` / `apple-push-ip`）
- 中文展示名写在文件头的注释里，不进文件名，避免 URL 里的百分号编码
- 新增条目追加到对应文件末尾，保持原有排序习惯，不要重排既有内容

## 与 MagicPanel 的关系

本仓库以 submodule 形式挂在 MagicPanel 的 `magic-sets/` 目录下：

```bash
git clone --recurse-submodules https://github.com/astralwaveorg/MagicPanel.git
# 已经 clone 过主仓库时：
git submodule update --init --recursive
```

面板里的规则集内容目前仍存在数据库里，产物中的 `rule-providers` URL 指向面板自己的
`/res/rule/<name>` 端点，客户端从面板拉取。要让规则内容以本仓库文件为准，
还需要在面板侧加一条读取链路的改动。
