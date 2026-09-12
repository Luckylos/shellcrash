# shellcrashyaml

基于当前 ShellCrash 实际需求整理的一份 **Mihomo / 订阅转换后端模板 YAML**。

## 文件

- `subconverter-shellcrash-needs.yaml`：唯一模板。分流 `🤖 AI 服务`、`📲 Telegram`、`🎯 国内流量`，其余流量交给 `🐟 漏网之鱼`，并提供 `🔗 链式落地` 两跳链式代理。

## 设计目标

- 适配 **订阅转换后端** 使用，不直接内置具体节点
- 保留当前 ShellCrash 的核心路由需求
- 风格参考 `666OS/YYDS` 的 `Pro_cn.yaml`
- 地区节点支持“手动指定优先，失效后同地区自动回退”
- 未匹配到已知地区关键字的节点统一归入 `其余地区`
- 中国大陆流量默认直连
- 支持链式代理（两跳），且不产生 dialer 环路

## 当前策略

- `🤖 AI 服务`：美国优先
- `📲 Telegram`：新加坡优先
- `🎯 国内流量`：默认 `DIRECT`，保留代理作为手动备选
- `🐟 漏网之鱼`：其余全部流量默认跟随 `🚀 节点选择`
- `🔗 链式落地`：两跳链式节点，可在 `🚀 节点选择`、`🤖 AI 服务`、`📲 Telegram`、`🐟 漏网之鱼` 中直接选用
- 未命中港/日/新/美筛选规则的节点：归入 `其余地区`
- 私网/回环/IPv6 本地链路：强制直连
- DNS：`fake-ip + 0.0.0.0:1053 + 阿里/腾讯 DoH`，国内域名按 `rule-set:cn` 走国内 DoH

结构规模：22 个 `proxy-group`、1 个 `proxy-provider`、5 个 `rule-provider`、16 条 rules、0 个 listener。

## 策略组顺序

`proxy-groups` 的书写顺序就是面板里的显示顺序（已实测：运行态 `/proxies` 返回顺序与模板一致）。按“操作频率从高到低”分四段，常用的在上，实现细节在下：

```text
一、入口策略组   🚀 节点选择 / 🤖 AI 服务 / 📲 Telegram / 🎯 国内流量 / 🐟 漏网之鱼
二、链式代理     🔗 链式前置 / 🔗 链式落地
三、地区主组     🇭🇰 香港 / 🇯🇵 日本 / 🇸🇬 新加坡 / 🇺🇸 美国 / 其余地区
四、地区实现组   <地区>-手动 / <地区>-自动（按地区成对排列）
```

- 第一段是规则命中后的落点，日常切换主要在这里。
- 第二段紧随其后：链式要在「前置」「落地」各选一次，放在一起避免来回翻。
- 第三段是入口策略组的实际落点。
- 第四段是 `fallback` 的内部实现，成对排列，改某个地区时同地区两组相邻；一般不需要直接操作。

Mihomo 允许前向引用（第一段引用后面才定义的地区组），所以这个顺序只影响可读性和面板体验，不影响解析。

规则表在 AI / Telegram 之后、`MATCH` 之前插入：

```yaml
- RULE-SET,cn,🎯 国内流量
- RULE-SET,cnip,🎯 国内流量,no-resolve
```

- `cn`：`666OS/rules` 的 `domain/China.mrs`
- `cnip`：`666OS/rules` 的 `ip/China.mrs`
- 不使用 `GEOIP,CN`：`cnip` 已覆盖，同时避免依赖 geoip 数据库下载
- `cnip` 带 `no-resolve`，避免 fake-ip 模式下为 IP 规则强制触发 DNS 解析

`rule-provider` 名称**必须是 `cn`**：`dns.nameserver-policy` 以 `rule-set:cn` 引用它，改名会变成悬空引用。

AI 规则刻意排在国内规则之前，避免国内域名规则集误收 AI 域名。

## 链式代理

Mihomo 已弃用 `relay` 策略组，链式能力由 `dialer-proxy` 提供，而 `dialer-proxy` 不能写在 `proxy-groups` 上，只能写在具体节点或 `proxy-providers.override` 上。本模板不内置 `proxies`，所以走 provider override：

```yaml
proxy-providers:
  chainpool:
    type: file
    path: ./yamls/config.yaml
    override:
      additional-prefix: "🔗 "
      dialer-proxy: 🔗 链式前置
```

把 ShellCrash 已生成的订阅配置当节点源再读一次，复制出一份带 `🔗 ` 前缀的副本，每个副本的出站前先经 `🔗 链式前置`，形成两跳：

```text
客户端 → 🔗 链式前置（第一跳，手动选） → 🔗 落地节点（第二跳） → 目标
```

- `🔗 链式前置`：候选为订阅本体节点 + 5 个地区组，手动选择
- `🔗 链式落地`：候选为 chainpool 复制出的全部 `🔗 ` 节点

### 两个必须遵守的约束

**1. provider 路径必须是相对路径。** Mihomo 有 safe-path 检查，拒绝 home 目录（`-d` 指定，ShellCrash 为 `/etc/ShellCrash`）以外的绝对路径：

```text
parse proxy provider chainpool error: path is not subpath of home directory or SAFE_PATHS
```

ShellCrash 的启动命令是 `CrashCore -d /etc/ShellCrash -f /tmp/ShellCrash/config.yaml`，配置文件本身在 `/tmp` 下，**不能**作为 provider 路径；`./yamls/config.yaml` 在 home 内，可用。这样也不必把订阅地址写进本模板。

**2. 地区组必须排除 `🔗 ` 节点。** `include-all: true` 会把 provider 复制出的链式节点一并纳入，导致 `落地 → 前置 → 落地` 环路。因此所有 `include-all` 的地区组都带 `exclude-filter: "🔗"`：

| 写法 | 是否纳入 provider 链式节点 |
|---|---|
| `include-all: true` | 纳入（会成环，必须配 `exclude-filter`） |
| `include-all-proxies: true` | 不纳入（只含 `proxies:` 本体节点） |

`🔗 链式前置` 用 `include-all-proxies: true`，候选里只放地区组，不放 `🚀 节点选择` / `🐟 漏网之鱼`，否则会出现 `落地 → 前置 → 节点选择 → 落地` 的环路。

`health-check` 默认关闭：开启会让每个链式节点都经前置节点额外测速一遍。

## 已移除的独立出口 listeners

原模板内置 9 个绑定 `172.17.0.1` 的 mixed listeners（`17891`-`17899` / `🎛️ 出口 1-9`），现已全部删除。

> **下游影响**：`/opt/cliproxyapi/config.yaml` 中有 45 处 `proxy-url` 指向 `http://host.docker.internal:1789x`（`17891`-`17898` 各 5 处，`17899` 4 处）。本模板生效后这些端口不再存在，需要另行调整 cli-proxy-api 配置。

需要固定出口的客户端可改用 `mixed-port: 7890`，或按需自行加回 listeners。

## 私网直连规则

保留的 9 条 `DIRECT` 规则只覆盖私网与本地地址：

```text
127.0.0.0/8  10.0.0.0/8  172.16.0.0/12  192.168.0.0/16
100.64.0.0/10  169.254.0.0/16  ::1/128  fc00::/7  fe80::/10
```

这些**不能删**。当前 ShellCrash 以 TUN/TPROXY + host 网络运行；一旦把私网段也灌进隧道，宿主机、容器互访与 LAN 可达性会一起断掉。

`🐟 漏网之鱼` 组内末位保留 `DIRECT` 作为**手动**开关，仅用于临时排障，默认不选中。

## 验证

以生产同版本核心 **Mihomo Meta v1.19.29**（`/tmp/ShellCrash/CrashCore`）在隔离容器内实测，注入 37 个真实节点：

- 静态：`-t` 通过；22 组无重名、0 悬空组引用、0 孤立 rule-provider、无 `listeners` / `proxies` 残留
- rule-provider 实际加载：`cn` 111035 条、`cnip` 9624 条、`AI` 273 条、`Telegram` 36 条、`TelegramIP` 24 条
- 规则命中（运行态日志）：`RuleSet(cn) using 🎯 国内流量[DIRECT]`、`RuleSet(Telegram) using 📲 Telegram`、`RuleSet(AI) using 🤖 AI 服务`、`Match using 🐟 漏网之鱼`
- 链式差分：同一落地节点下，前置正常 → `HTTP 200`；前置换成不可用节点 → 链路失败（`HTTP 000`/`502`，取决于失败发生在连接还是握手阶段）而同刻直连对照仍 `HTTP 200`；恢复前置 → `HTTP 200`
- 无环：前置候选中 `🔗 ` 节点数为 0，5 个地区组均无 `🔗 ` 泄漏，前置可达闭包不含 `🔗 链式落地`

> 首次运行若看到 rule-provider `ruleCount` 为 0，通常是 mihomo 自身的规则集下载被 `MATCH` 指向的节点拦下（该节点不可用），与模板无关；把 `🐟 漏网之鱼` 临时切到 `DIRECT` 再刷新即可。

## 说明

这份 YAML 主要表达：

1. 策略组结构
2. 区域分组筛选规则
3. 规则集与路由意图
4. DNS 行为
5. 链式代理结构

具体节点应由订阅转换后端注入，或由上游订阅提供。

## 参考

- https://raw.githubusercontent.com/666OS/YYDS/refs/heads/main/mihomo/config/cn/Pro_cn.yaml
- https://wiki.metacubex.one/config/proxies/dialer-proxy/
