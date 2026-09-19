# ADR-0013（草案）· 重定位：Agent 能力管理（提示词、Skills、工具、MCP、Harness、Loop）和 Runtime 控制的运行基座

- 状态：**draft**（2026-09-11）；采纳后移入 `docs/adr/0013-*.md`，并触发 `docs/design` 的 v0.8.0 修订
- 决策者：owner
- 背景来源：2026-09-11 对 v0.7.0 的定位审查；[ADR-0001](../adr/0001-founding-decisions.md) §D1 原点；[v0.7.0 蓝图](../research/20260908-v0.7.0-refactor-blueprint.md)
- 相关：[01 定位与原则](01-positioning.md)、[10 路线图](10-roadmap.md)

## 背景

ADR-0001 §D1 把 FysunOS 的职责定为"管理'决策核心 × 资源 × 环境'的绑定，并可靠地运行这种绑定"。v0.5.0 → v0.7.0 的演进中，
"可靠地运行"逐步吞掉了"管理"：`fysund` 成为 harness 的唯一 MCP server、持有下游会话、代理模型请求、渲染上下文进 harness 启动输入、
为审批结果发明三条投递管道；06 / 07 / 08 / 13 四篇约 160 KB 全部围绕内联拦截。与此同时 Capability 的注册、组合、谱系、演进只占不到守卫规范一半的篇幅，
Harness 与 Loop 被排除在对象模型之外（"不做"），多主体 Scope 停在开放问题。

内联模型在异构 Runtime 面前的结构性极限已被 v0.7.0 自己承认：ADR-0007 把 F3 / F7 / F9 收窄到"经守卫的 MCP 工具面"，hooks 路径"只能拦不能记"，
F5 收窄到"经 `fysund` 代理的 Provider"。这些是脚注形式的架构结论。

同时，owner 对目标的表述变为：FysunOS 应成为 **Agent 能力（提示词、Skills、工具、MCP、Harness、Loop）管理与 Runtime 控制的运行基座**，
服务未来的 Agent as a Service 生态。

## 决策

### D1 · 定义

FysunOS 是 **Agent 能力管理（提示词、Skills、工具、MCP、Harness、Loop）和 Runtime 控制的运行基座**。这是唯一正式定义句。两项职责：(1) 把提示词、Skills、工具、MCP server、Harness、Loop 策略（以及模型、记忆、知识）
统一为可登记、可版本化、可组合、可计量的**能力资产**并管理其全生命周期；(2) 控制 Runtime 的接入、资格化、绑定、Activity / Execution 生命周期与执行边界。
一个目标：在此基座上 Agent 作为服务被发布、发现、调用、治理与演进。

### D2 · 一切可复用之物是资产；Harness 与 Loop 升为一级资产种类

资产种类封闭枚举：`prompt / skill / tool / mcp_server / harness / loop / model / memory_provider / knowledge / runtime / agent`（[02](02-asset-model.md)）。
Harness 是可执行的 Agent 程序资产（loop 实现、prompt 装配、工具分发、会话日志）；Loop 是 Harness 消费的策略配置资产（步数预算、继续 / 停止、重试、委派策略）。
基座**不实现** loop 与 prompt 装配（延续 ADR-0001 §D2 "不自建"清单），但**登记、版本化、绑定并度量**它们。

### D3 · Runtime = Harness 的资格化实例；Runtime 控制是基座职责

Runtime 是某个 Harness 资产在某个 Node 上安装后、经 `runtime.qualify` 取得资格化纪元的实例（继承 v0.7.0 07 §1）。基座控制：接入与 `describe()`、
资格化与失格、与 Agent Definition 的绑定、Activity / Execution 状态机、预算与取消、检查点与恢复、重新武装、有效权限集。基座不持有 Loop 状态、不做沙箱。

### D4 · 三平面 + 横切治理；`fysund` 不在数据面热路径

- **资产 / 组合平面**（能力管理）：Registry、Agent Definition、解析与锁定、证据。
- **运行平面**（Runtime 控制）：Runtime 目录与资格化、调度、Activity / Execution、执行点、账本。
- **服务平面**（AaaS）：Service / Endpoint / Tenant / Plan / Meter、Agent Card、`fysun-gateway`。
- **治理**（横切）：Principal / Scope、Grant、Policy、Approval、来源标签、Verification、供应链。

`fysund` 承担管理与决策（PDP、Approval、账本）；数据面拦截由**执行点**承担：`fysun-gateway`（入口）、`fysun-mcp-guard`（工具侧，独立进程、可选）、
harness hooks、harness 适配器（`fysun-dsh`）、外部网关。执行点须经资格化并以事件回报（[04 §4](04-runtime-substrate.md)）。

### D5 · Policy 下发为主，`approve` 回中心

Policy 以版本化快照下发到执行点本地评估（沿 ADR-0010 边缘快照先例），事件回报携带 `policy_version`；`outcome = approve` 与任何 Grant 扩张
必须回 `fysund` 签发 Approval / Grant。执行点不可决定 Approval。

### D6 · 多主体 Scope 进入核心

Tenant = Principal + Scope + Plan。所有资产、Agent Definition、Service、Activity 都有归属 Scope；Grant 绑定 `(principal, scope)`；跨租户不可见是缺省。
v0.7.0 Q5 关闭为"MVP 起即多主体，但单机部署可只有一个 Tenant"。

### D7 · 标准优先不变，增加分发标准

MCP（工具 / 资源 / 提示）、A2A（服务端点与 Agent Card）、Agent Skills（skill 资产格式）保持；资产打包分发采用 OCI artifact；可观测性沿 OTel。
不自造包格式、策略语言、传输协议。

### D8 · 底线 F1–F9 保留，每条标注"由谁执行"

F1 / F2 / F4 / F6 / F8 由 `fysund` 执行；F3 / F7 / F9 由资格化执行点执行、`fysund` 验证回报并拒绝无回报 Effect 进入 `verified`；F5 改为"OS 管理的 secret 只向资格化执行点解析"
（[01 §7](01-positioning.md)）。

## 否决的替代方案

- **维持 v0.7.0 内联守卫为必经路径**：与异构 Runtime 现实冲突（ADR-0007 已证明），且把 Capability 管理挤成附属。
- **只做纯管理平面、不做 Runtime 控制**：Activity / Execution 生命周期、预算、取消、恢复是 AaaS 服务可靠性的基础，无法外包给每个 harness。
- **基座自带参考 Harness / Loop 实现**：重回"重造决策核心"；Harness 由生态供给（DSH、Claude Code、Codex、边缘 Runtime），基座负责让它们可登记、可资格化。
- **把服务平面做成独立产品**：服务平面依赖资产版本、Runtime 资格化、Grant / Approval，拆出去会复制对象模型。

## 后果

- `docs/design` 需要 v0.8.0 修订：01 / 03 / 12 重写，02 增资产与服务对象，04 增 Registry / Service API，06 / 07 §3–5 / 08 降级为执行点与适配器规范，
  13 conformance 表改按执行点类型分列。逐篇关系见 [10 §2](10-roadmap.md)。
- ADR-0002 §D2 "守卫是必经之路"、ADR-0007 §D5 投递路径、ADR-0012 自实现的前提需重开；ADR-0012 的结论（不嵌入 agentgateway）在"执行点"框架下改为
  "agentgateway 可作为外部执行点接受 Policy 下发"，待评估。
- 迁移清单缩小：ActionIntent / Effect 状态机从 OS 主线退到执行点契约与参考执行点实现。
- Trust 对象（v0.7.0 Q11）从远期变为服务平面联邦的前置条件。

## 负例向量（进入 R1 / R3 验收）

1. 出现一种可复用之物（prompt / skill / tool / mcp_server / harness / loop）被 Agent Definition 引用但无资产登记与 digest → 失败。
2. `fysund` 进程出现在任何 `tools/call` 或模型请求的转发路径上 → 失败（数据面越界）。
3. 执行点本地产生 `approval/decided` 或 `grant/issued` → 失败。
4. 两个 Tenant 的资产、Memory、Activity 在无显式 Grant 时互相可见 → 失败。
5. 某 Runtime 未经 `runtime/qualified` 即被 Service 调用绑定 → 失败。
6. 设计文档把基座描述为"实现 loop / prompt 装配 / 沙箱" → 失败（文档一致性检查）。
