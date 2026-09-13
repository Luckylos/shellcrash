# shellcrashyaml

基于当前 ShellCrash 实际需求整理的一份 **Mihomo / 订阅转换后端模板 YAML**。

## 文件

- `shellcrash.yaml`：唯一模板。分流 `🤖 AI 服务`、`📲 Telegram`、`🎯 国内流量`，其余流量交给 `🐟 漏网之鱼`，并提供 `🔗 链式落地` 两跳链式代理。

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

### 当前方案：HTTP provider + `request.url`

本模板不内置具体节点，也不依赖 ShellCrash 专用目录。Subconverter 渲染模板时，
`request.url` 取当前请求的订阅源，并写入 HTTP provider：

```yaml
proxy-providers:
  chainpool:
    type: http
    url: "{{ request.url }}"
    path: ./providers/chainpool.yaml
    override:
      additional-prefix: "🔗 "
      dialer-proxy: 🔗 链式前置
```

provider 会读取订阅源当前返回的 `proxies` 段，动态复制每个真实节点，并给每个副本加上
`🔗` 前缀和节点级 `dialer-proxy: 🔗 链式前置`；模板不写死任何入口或出口节点参数。
因此每次订阅内容变化后，链式入口池和链式落地池都随真实节点变化：

```text
客户端 → 当前订阅中的入口节点 → 当前订阅复制出的出口节点 → 目标
```

使用前必须满足：

- 只能传入**单个**订阅 URL；当前模板不负责把多个 URL 自动拆分后再提供给 Mihomo。
- 该 URL 必须返回 Mihomo/Clash YAML，并包含 `proxies:`；纯 URI 列表不适用这个 provider。
- `exclude-type` 使用 Mihomo provider 的官方字段，排除不适合经 TCP 前置中转的 UDP 类节点。
- `request.url` 只会在兼容 Subconverter 模板语法的转换后端渲染。直接把仓库里的原始模板交给 Mihomo，`{{ request.url }}` 不会自动求值。
- 生成配置会包含订阅源地址，这是运行时拉取 provider 所必需的；不要公开分享生成配置或把真实订阅地址提交到仓库。
- `path` 使用相对路径，基准是 Mihomo 的 `-d` 目录；`./providers/chainpool.yaml` 可用于普通客户端，避免绑定 `/etc/ShellCrash/yamls/` 等宿主机布局。

### 动态节点级链式，而不是静态节点清单

用户示例中的核心字段是出口节点上的：

```yaml
- name: "🔗 当前订阅中的出口节点"
  # 当前订阅节点的真实参数由 provider 动态提供
  dialer-proxy: "🔗 链式前置"
```

本模板通过 HTTP `proxy-provider` 的 `override` 动态注入这个字段：

```yaml
override:
  additional-prefix: "🔗 "
  dialer-proxy: 🔗 链式前置
```

因此不会把入口节点、出口节点、服务器、端口或认证参数静态写进模板。`🔗 链式前置`
使用 `include-all-proxies: true` 动态吸收当前订阅的原始节点；`🔗 链式落地` 动态使用
provider 复制出的节点。

客户端若只显示顶层 `proxies`、不支持 provider 节点，或面板错误地只请求
`/proxies/<provider-node>` 而不读取 `/providers/proxies`，则可能看到「链式落地为空」。
这是客户端/provider 命名空间兼容性边界，不是把节点静态化或改成 `type: file` 就能解决的问题；
此时需要使用客户端支持 provider UI，或由上游另行生成顶层链式节点。

### 防环约束

- `🔗 链式前置` 使用官方策略组字段 `include-all-proxies` 与 `exclude-filter: "🔗"`，动态纳入
  当前订阅本体节点，不写死 `proxies` 节点清单，也不吸收 `chainpool` 复制出的链式节点。
- `🔗 链式落地` 使用官方策略组字段 `empty-fallback: REJECT`，provider 尚未加载时不回退到
  `COMPATIBLE`/直连。
- 所有使用 `include-all: true` 的地区组都带 `exclude-filter: "🔗"`，避免链式副本进入普通地区候选。
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

验证核心为生产同版本 **Mihomo Meta v1.19.29**（从生产容器读取二进制，仅在隔离容器执行）。原始模板通过 YAML/引用探针：22 个 `proxy-group`、5 个 `rule-provider`、无 `listeners`、无顶层 `proxies`、无重复组名，`cn` 与 DNS 的 `rule-set:cn` 引用一致。

使用去除旧链式副本的 38 个临时夹具节点做最终配置验证；节点字段只用于隔离测试，未写回仓库：

- 生成配置执行 `CrashCore -t -d /home -f /home/config.yaml` 成功。
- HTTP provider 运行态：`vehicleType=HTTP`、`chainpool` 加载 38 个节点，38 个均带 `🔗` 前缀，38 个均保留 `dialer-proxy: 🔗 链式前置`。
- 防环：`🔗 链式前置` 的候选由同一份运行态 `/proxies` 动态读回，`🔗` 链式节点数必须为 0；`🔗 链式落地` 的成员必须全部来自当前 `chainpool`，带前缀并保留 `dialer-proxy: 🔗 链式前置`。
- 规则 provider 运行态全部加载：`cn` 111035 条、`cnip` 9624 条、`AI` 273 条、`Telegram` 36 条、`TelegramIP` 24 条；`/rules` 共 16 条，AI/Telegram 位于 cn/cnip 之前。
- 实际分流：通过隔离核心访问 `http://www.baidu.com/` 返回 `HTTP 200`，日志出现 1 次 `RuleSet(cn)` 命中 `🎯 国内流量`；运行态 `🎯 国内流量.now` 为 `DIRECT`。
- 两跳功能差分：正常前置 + 链式落地 `HTTP 204`；将 `🐟 漏网之鱼` 切为 `DIRECT` 的对照 `HTTP 204`；前置切换到死节点后链式 `HTTP 502`；恢复前置后链式 `HTTP 204`。
- provider 可见性边界：链式节点在 `/providers/proxies` 中可读，但 Mihomo 1.19.29 的单节点 `/proxies/<provider-node>` 返回 `404`。客户端必须支持 provider 命名空间；只按顶层 `/proxies` 渲染的面板会显示为空，需要上游生成顶层克隆节点。

## 说明

这份 YAML 主要表达：

1. 策略组结构
2. 区域分组筛选规则
3. 规则集与路由意图
4. DNS 行为
5. 链式代理结构

具体节点应由订阅转换后端注入，或由上游订阅提供。

## 参考

- <https://raw.githubusercontent.com/666OS/YYDS/refs/heads/main/mihomo/config/cn/Pro_cn.yaml>
- <https://wiki.metacubex.one/config/proxies/dialer-proxy/>
