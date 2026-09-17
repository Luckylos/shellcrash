# shellcrashyaml

基于当前 ShellCrash 实际需求整理的一份 **Mihomo / Clash Meta / 订阅转换后端通用 YAML 模板**。

## 文件

- `shellcrash.yaml`：唯一模板。分流 `🤖 AI 服务`、`📲 Telegram`、`🎯 国内流量`，其余流量交给 `🐟 漏网之鱼`。

当前仓库已经改为私有仓库并完成重命名。若订阅转换后端需要读取模板，GitHub 私有仓库的 raw 地址不能默认被公共后端访问；应使用后端已有的授权访问能力，或将模板部署到后端可访问的受控地址。

## 设计目标

- 适配 **Mihomo / Clash Meta / 订阅转换后端** 使用，不直接内置具体节点
- 没有 ShellCrash 覆盖时，模板自身也提供完整的 Route 等价 DNS 行为
- 通用客户端控制器仅绑定本机回环；宿主 ShellCrash 继续使用自身控制器配置
- 保留当前 ShellCrash 的核心路由需求
- 风格参考 `666OS/YYDS` 的 `Pro_cn.yaml`
- 地区节点支持“手动指定优先，失效后同地区自动回退”
- 未匹配到已知地区关键字的节点统一归入 `其余地区`
- 中国大陆流量默认直连
- 不提供链式代理、`dialer-proxy` 或链式节点 provider

## 当前策略

- `🤖 AI 服务`：美国优先
- `📲 Telegram`：新加坡优先
- `🎯 国内流量`：默认 `DIRECT`，保留代理作为手动备选
- `🐟 漏网之鱼`：其余全部流量默认跟随 `🚀 节点选择`
- 未命中港/日/新/美筛选规则的节点：归入 `其余地区`
- 私网/回环/IPv6 本地链路：强制直连
- DNS：通用客户端使用 `redir-host + respect-rules`；国内域名按 `rule-set:cn` 走国内 DoH，境外域名使用境外 DoH；宿主 ShellCrash 可覆盖为自身的 Route 生成形态

结构规模：20 个 `proxy-group`、0 个 `proxy-provider`、5 个 `rule-provider`、22 条 rules、0 个 listener。

## 策略组顺序

`proxy-groups` 按面板操作频率从高到低组织，便于支持配置顺序的客户端直接复用；Mihomo `/proxies` 返回的是 JSON 对象，键顺序不构成 UI 排序保证。当前模板分三段，常用的在上，实现细节在下：

```text
一、入口策略组   🚀 节点选择 / 🤖 AI 服务 / 📲 Telegram / 🎯 国内流量 / 🐟 漏网之鱼
二、地区主组     🇭🇰 香港 / 🇯🇵 日本 / 🇸🇬 新加坡 / 🇺🇸 美国 / 其余地区
三、地区实现组   <地区>-手动 / <地区>-自动（按地区成对排列）
```

- 第一段是规则命中后的落点，日常切换主要在这里。
- 第二段是入口策略组的实际落点。
- 第三段是 `fallback` 的内部实现，成对排列，改某个地区时同地区两组相邻；一般不需要直接操作。

Mihomo 允许前向引用（第一段引用后面才定义的地区组），所以这个顺序只影响可读性和面板体验，不影响解析。

规则表在 AI / Telegram 之后、`MATCH` 之前插入：

```yaml
- RULE-SET,cn,🎯 国内流量
- RULE-SET,cnip,🎯 国内流量,no-resolve
```

- `cn`：`666OS/rules` 的 `domain/China.mrs`
- `cnip`：`666OS/rules` 的 `ip/China.mrs`
- 不使用 `GEOIP,CN`：`cnip` 已覆盖，同时避免依赖 geoip 数据库下载
- `cnip` 带 `no-resolve`，避免对裸 IP 规则强制触发 DNS 解析
- 所有远程规则集通过 `🚀 节点选择` 下载，避免手机直连 GitHub 失败后规则集为空

`rule-provider` 名称**必须是 `cn`**：`dns.nameserver-policy` 以 `rule-set:cn` 引用它，改名会变成悬空引用。

AI 规则刻意排在国内规则之前，避免国内域名规则集误收 AI 域名。

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

这些**不能删**。宿主 ShellCrash 以 TUN/TPROXY + host 网络运行；手机等第三方客户端也必须在自身设置中启用 VPN/TUN，YAML 只能定义规则和 DNS，不能替客户端创建系统 VPN。

`🐟 漏网之鱼` 组内末位保留 `DIRECT` 作为**手动**开关，仅用于临时排障，默认不选中。

## 验证要点

- YAML 可解析；包含 20 个 `proxy-group`、0 个 `proxy-provider`、5 个 `rule-provider`、无 `listeners`、无顶层 `proxies`、无重复组名。
- 普通地区组使用 `include-all: true`；`其余地区` 仅排除已知地区名称。
- 配置不包含链式组、链式 provider、`dialer-proxy` 或相关模板变量。
- `cn` 与 DNS 中的 `rule-set:cn` 引用保持一致。

## 说明

这份 YAML 主要表达：

1. 策略组结构
2. 区域分组筛选规则
3. 规则集与路由意图
4. DNS 行为

具体节点应由订阅转换后端注入，或由上游订阅提供。

## 参考

- <https://raw.githubusercontent.com/666OS/YYDS/refs/heads/main/mihomo/config/cn/Pro_cn.yaml>
