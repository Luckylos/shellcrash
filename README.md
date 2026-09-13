# shellcrashyaml

基于当前 ShellCrash 实际需求整理的一份 **可直接加载的 Mihomo YAML 配置**，也可由订阅转换后端原样分发。

## 文件

- `shellcrash.yaml`：唯一的 Mihomo YAML 基础模板。分流 `🤖 AI 服务`、`📲 Telegram`、`🎯 国内流量`，其余流量交给 `🐟 漏网之鱼`，并提供 `🔗 链式落地` 两跳链式代理。

## 设计目标

- 可直接作为 Mihomo 配置使用，不直接静态内置节点
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

结构规模：22 个 `proxy-group`、2 个 `proxy-provider`、5 个 `rule-provider`、16 条 rules、0 个 listener。

## 策略组顺序

`proxy-groups` 按面板操作频率从高到低组织，便于支持配置顺序的客户端直接复用；Mihomo `/proxies` 返回的是 JSON 对象，键顺序不构成 UI 排序保证。当前模板分四段，常用的在上，实现细节在下：

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

Mihomo 已弃用 `relay` 策略组，链式能力由 `dialer-proxy` 提供；`dialer-proxy` 不能写在
`proxy-groups` 上，只能写在具体节点或 `proxy-providers.override` 上。

### 当前方案：自包含双 HTTP provider

本模板只依赖 Mihomo 官方机制，不依赖 SubConverter 注入顶层节点，也不使用任何模板变量。
两个 HTTP provider 读取同一个固定的私有 MiSub profile；前置副本使用 `🛰️ 前置` 前缀，
落地副本使用 `🔗` 前缀：

```yaml
proxy-providers:
  frontpool:
    type: http
    url: "<固定 MiSub profile 地址>"
    path: ./providers/frontpool.yaml
    override:
      additional-prefix: "🛰️ 前置 "

  chainpool:
    type: http
    url: "<固定 MiSub profile 地址>"
    path: ./providers/chainpool.yaml
    override:
      additional-prefix: "🔗 "
      dialer-proxy: "🔗 链式前置"
```

对应链路：

```text
固定 MiSub profile
       ├── frontpool  → 🔗 链式前置，可动态选择第一跳
       └── chainpool  → 🔗 链式落地，节点通过 dialer-proxy 使用第一跳
```

`frontpool` 和 `chainpool` 都读取源配置的顶层 `proxies:`，因此每次订阅节点变化后，
前置池和落地池都会同步变化。落地节点由 provider 的 `override.dialer-proxy` 动态附加，
不是静态复制脚本，也不是 `relay`。

使用前必须满足：

- 固定 profile 地址必须返回包含顶层 `proxies:` 的 Mihomo/Clash YAML；当前地址使用 Clash 输出参数，避免拿到 base64 URI 列表。
- 本模板不依赖 `request.url`、`clash_rule_base` 注入或其他非 Mihomo 模板变量；可以直接作为
  Mihomo 配置加载。若通过 SubConverter 分发，后端必须返回这份完整 YAML，而不是另一份旧模板。
- `frontpool` 和 `chainpool` 使用不同的相对缓存路径，避免两个 provider 互相覆盖缓存。
- `exclude-type` 使用 Mihomo provider 的官方字段，排除不适合经 TCP 前置中转的 UDP 类节点。
- `path` 的基准是 Mihomo 的 `-d` 目录；客户端必须能够创建 `./providers/` 目录并支持 HTTP provider。
- 当前仓库为私有仓库；不要公开固定 profile 地址、生成配置、节点认证或 token。

### 动态节点级链式，而不是静态节点清单

落地 provider 的核心字段仍然是官方的节点级配置：

```yaml
override:
  additional-prefix: "🔗 "
  dialer-proxy: "🔗 链式前置"
```

`frontpool` 不设置 `dialer-proxy`，只给前置副本增加 `🛰️ 前置` 前缀；`chainpool` 给落地副本
增加 `🔗` 前缀并统一指向 `🔗 链式前置`。因此模板不写死服务器、端口、协议或认证参数。

普通地区组通过 `use: [frontpool]` 读取动态前置 provider，链式前置组也只读取 `frontpool`；
链式落地组只读取 `chainpool`。这样即使配置中没有顶层 `proxies:`，也能形成完整的两跳链路。

客户端若只显示顶层 `proxies`、不支持 provider 节点，或面板错误地只请求
`/proxies/<provider-node>` 而不读取 `/providers/proxies`，仍可能看到 provider 节点为空。
这是客户端/provider 命名空间兼容性边界；需要使用支持 `proxy-providers` 的 Mihomo 客户端。

### 防环约束

- `🔗 链式前置` 只使用 `frontpool`，不读取 `chainpool`，因此落地副本不会成为第一跳候选。
- `🔗 链式落地` 只使用 `chainpool`，并通过 `empty-fallback: REJECT` 防止 provider 尚未加载时回退到
  `COMPATIBLE` 或直连。
- 普通地区组也只使用 `frontpool`，不会把带 `🔗` 前缀的落地节点吸入地区候选。
- 链式前置不引用 `🚀 节点选择`、`🐟 漏网之鱼` 或 `🔗 链式落地`，避免
  `落地 → 前置 → 节点选择 → 落地` 环路。
- `health-check` 默认关闭：开启会让每个链式节点都经前置节点额外测速一遍。

---

## 已移除的独立出口 listeners

原模板内置 9 个绑定 `172.17.0.1` 的 mixed listeners（`17891`-`17899` / `🎛️ 出口 1-9`），现已全部删除。

> **下游影响**：模板不再提供 `17891`-`17899`。历史审计曾发现 cli-proxy-api 有 45 处 `proxy-url` 依赖这些端口；本轮对当前 `/opt/cliproxyapi/config.yaml` 的只读复核已观察到 `proxy-url` 仅 1 处、`1789x` 依赖为 0。部署到其他环境前仍必须重新扫描目标 cli-proxy-api 配置，不能按历史计数推断现状。

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

当前版本已完成静态和源地址验证：

- YAML 可解析；包含 22 个 `proxy-group`、5 个 `rule-provider`、无 `listeners`、无顶层 `proxies`、无重复组名。
- `frontpool` 与 `chainpool` 均为官方 `type: http` provider，使用固定真实 URL 和不同缓存路径。
- 两个 provider 源地址都解析为具体 HTTP URL；源响应为合法 YAML，并包含 34 个顶层 `proxies`。
- `🔗 链式前置` 仅使用 `frontpool`；`🔗 链式落地` 仅使用 `chainpool`，后者保留 `dialer-proxy: 🔗 链式前置`。
- 当前环境没有可用于本版本最终运行态测试的 Mihomo/CrashCore 可执行文件，因此尚未宣称真实两跳连通性已经验证。

运行态验收应检查：

1. `frontpool` 和 `chainpool` 两个 provider 都成功加载；
2. `🔗 链式前置` 出现带 `🛰️ 前置` 前缀的动态节点；
3. `🔗 链式落地` 出现带 `🔗` 前缀的动态节点；
4. `/providers/proxies/chainpool` 中的节点包含 `dialer-proxy: 🔗 链式前置`；
5. 切换前置节点后，链式落地流量随之切换或失败。

## 说明

这份 YAML 主要表达：

1. 策略组结构
2. 区域分组筛选规则
3. 规则集与路由意图
4. DNS 行为
5. 链式代理结构

具体节点由 `frontpool` 和 `chainpool` 在运行时从固定 MiSub profile 读取；模板本身不静态保存节点参数。

## 参考

- <https://raw.githubusercontent.com/666OS/YYDS/refs/heads/main/mihomo/config/cn/Pro_cn.yaml>
- <https://wiki.metacubex.one/config/proxies/dialer-proxy/>
