# 01 · 定位与原则

- 状态：提案草案 v0.8.0-draft（2026-09-11）
- 决策依据：[ADR-0013 草案](adr-0013-draft-repositioning.md)；继承 [ADR-0001](../adr/0001-founding-decisions.md) §D1 原点与 §D5–D8
- 术语以 [00 术语表](00-terminology.md) 为准

## 1. 原点回顾

ADR-0001 §D1："管理'决策核心 × 资源 × 环境'的绑定，并可靠地运行这种绑定。"

这句话在 v0.7.0 里被理解为"`fysund` 亲自坐在每次工具调用与模型请求的路径上"。重定义把它读回本义：**管理绑定**是能力管理，
**可靠运行绑定**是 Runtime 控制；两者都不要求 `fysund` 成为数据面。诊断细节见 [10 §1](10-roadmap.md)。

## 2. 定义

### 2.1 一句话定义

> **FysunOS 是 Agent 能力管理（提示词、Skills、工具、MCP、Harness、Loop）和 Runtime 控制的运行基座。**

这是全仓库唯一的正式定义句；其他文档引用时不改写、不缩略（00 §8）。

### 2.2 完整定义

FysunOS 把提示词、Skills、工具、MCP server、Harness、Loop 策略，以及模型、记忆提供者与知识，统一为带 manifest 的**能力资产**：可登记、可版本化、
可组合、可资格化、可计量、可追溯。资产组合为 **Agent Definition**，Agent Definition 声明它实现的 **Capability 契约**（做什么、怎样算做好、依靠什么、
在什么边界内）。基座把 Agent Definition 绑定到经资格化的 **Runtime**（Harness 的安装实例），以 **Activity / Execution** 管理其生命周期、预算、
取消、恢复与执行边界；治理事实（Grant、Policy 决定、Approval、Effect、Verification）进入唯一权威账本。在此之上，Agent 作为 **Service** 发布：
有租户、配额、计量、版本、Agent Card 与证据，经 A2A / MCP / HTTP 被调用。

可判定限定：

- "任意 Runtime"指任何实现 Runtime ABI 最小面（`describe` / `start` / `resume` / `cancel` / `health` + 事件回报）并**如实声明** `control_points[]`
  与 `native_effect_tools[]` 的 Harness 实例；承诺是否成立由 `runtime/qualified` 判定，不由文字判定（继承 v0.7.0 01 §1.1）。
- "可靠运行"的范围由执行点的资格化决定：F3 / F7 / F9 只对经资格化执行点的工具面成立；没有执行点的 Runtime 只得到生命周期控制与账本，
  这在 conformance 表中如实分列（[04 §4.5](04-runtime-substrate.md)）。

### 2.3 "基座"指什么

基座只做四件事，其余留给生态：

| 动作 | 内容 | 不做 |
|---|---|---|
| **登记** | 资产 manifest、版本、digest、来源、许可证、效果声明、兼容与依赖、证据 | 不写 prompt、不写 skill、不实现工具 |
| **组合** | Agent Definition：资产引用 + 覆盖 + 契约 + 需求 → 解析为锁定包 | 不做 prompt 装配算法、不选工具（那是 Harness / Loop 的事） |
| **承载** | Runtime 资格化与绑定、Activity / Execution、预算 / 取消 / 恢复、执行边界、账本 | 不写 loop、不做沙箱、不持有 Loop 状态 |
| **服务化** | Service / Tenant / Plan / Meter / Endpoint、Agent Card、入口执行点 | 不做计费结算、不做聊天 UI（审批面除外） |

## 3. 平面划分

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ 消费者 / 上层应用   A2A client · MCP client · HTTP · Agent Shell · Personal      │
├──────────────────────── 服务平面（AaaS）──────────────────────────────────────┤
│ Service · ServiceVersion · Endpoint · Tenant · Plan · Meter · Agent Card        │
│ 入口执行点 fysun-gateway（鉴权 · 配额 · 计量 · 调用 → Activity/Execution）        │
├──────────────────────── 资产 / 组合平面（能力管理）────────────────────────────┤
│ Registry：prompt · skill · tool · mcp_server · harness · loop · model ·         │
│           memory_provider · knowledge · runtime · agent                          │
│ Agent Definition · Capability 契约 · 解析与锁定 · 证据 · Adjustment              │
├──────────────────────── 运行平面（Runtime 控制）──────────────────────────────┤
│ Runtime 目录与资格化纪元 · 调度与触发 · Activity/Execution · 检查点与恢复 ·        │
│ 执行点资格化与回报 · 治理账本（event-sourced）· Node                             │
├──────────────────────── 治理（横切）──────────────────────────────────────────┤
│ Principal/Scope · Grant · Policy（Cedar 子集，版本化下发）· Approval ·            │
│ 来源标签 · Verification · 供应链（签名 / 隔离 / 撤销）                            │
├────────── Runtime ABI · 执行点契约 · MCP · A2A · Agent Skills · OCI ───────────┤
│ 生态供给：Harness（DSH / Claude Code / Codex / 边缘 Runtime）· 工具与 MCP server · │
│           模型 Provider · 记忆引擎 · 外部 A2A Agent · 外部网关                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

`fysund` 承担资产 / 组合平面、运行平面的控制部分、服务平面的目录部分与全部治理决策；**不承担任何数据面转发**。数据面由执行点承担：
`fysun-gateway`（入口）、`fysun-mcp-guard`（工具侧，可选）、harness hooks、harness 适配器、外部网关（[04 §4](04-runtime-substrate.md)）。

## 4. 对外坐标（AOS 两平面）

| AOS 平面 / 职责 | FysunOS 对应 | 供给声明 |
|---|---|---|
| Control & Governance Plane — intent、policy、trust、authority、confidence、auditability、observability、human oversight | 治理横切 + 治理账本 + Approval；Trust 由服务平面联邦需要引入（[05 §11](05-agent-as-a-service.md)） | **供给**（Trust 在 R4 引入前为"供给（Trust 除外）"） |
| Runtime & Coordination Plane — lifecycle、workflow coordination、model/tool routing、context & memory coordination、scheduling、traffic management、runtime assurance | 运行平面（生命周期、调度、交接、资格化）；服务平面（流量入口）；组合平面（模型 / 工具 / 记忆的**绑定**而非路由） | **供给控制面，不供给数据面**：决策核心 loop、prompt 装配、沙箱、请求转发留给 Harness 与执行点 |
| 平台服务 — 宿主 OS、容器、物理设施 | 不做 | 不供给 |

与 v0.7.0 的差别：Runtime & Coordination Plane 从"部分供给"改为"供给控制面"——不是少做了，而是把"做什么"与"在哪做"分开说清。

## 5. Capability 与 Agent 的关系

- **Capability** 仍是契约：行为、质量目标、边界（含 `tool_effects[]`）、验证方式。契约可移植，过程不承诺（P3）。
- **Agent Definition** 是契约的实现：一组资产引用与覆盖（prompt、skills、tools / mcp、harness + loop、model、memory、knowledge）+ 需求 + 环境变体。
  一个 Agent Definition 可实现多个 Capability；一个 Capability 可由多个 Agent Definition 实现，证据分别归属。
- **Service** 是 Agent Definition 某版本的对外发布形态。

v0.7.0 里 `Capability.implementation` 是"一个 Skills 目录引用"；重定义后它是 Agent Definition 引用，Skills 目录只是其中一种资产。

## 6. 设计原则

| # | 原则 | 含义 |
|---|---|---|
| P1 | 标准化必要边界，让能力自由变化 | 只结构化机器必须检查或跨组件交接的部分（manifest、契约、事件） |
| P2 | 可靠性由系统承担，不变成用户流程 | 权威写入、资格化、效果回报核对、独立验证在系统内自动发生 |
| P3 | 契约可移植，过程不承诺 | 换 Harness / Loop / Model / Runtime 不改契约；证据按资产版本归属 |
| P4 | 长期活动是一级生命周期 | Activity 可无终点；Execution 是一次运行；均不依附会话或进程 |
| P5 | 协作是普通能力行为 | 子 Agent 与外部服务经 Handoff / A2A 引用；无专门多 Agent 子系统 |
| P6 | 调整默认轻量，检查只在风险边界 | Adjustment 直接生效、留痕、可回退；扩权 / 新外部动作 / 预算上调 / 设备安全走 Policy |
| P7 | 部署环境声明策略，基座执行边界 | Policy 版本化下发；能力不自扩权 |
| P8 | **不规定决策核心，但登记并承载它** | 不写 loop / prompt 装配 / 沙箱；Harness 与 Loop 作为资产被登记、版本化、绑定、度量 |
| P9 | **一切可复用之物是资产** | 无 manifest、无 digest 的东西不能进入 Agent Definition |
| P10 | **Agent 的对外形态是服务** | 发布即有租户、配额、计量、版本、Agent Card；接口只用 A2A / MCP / HTTP |
| P11 | **基座不在热路径** | `fysund` 只做管理与决策；拦截、转发、渲染在执行点或 Harness 内 |

## 7. 底线 F1–F9（重述，含由谁执行）

| ID | 底线 | 可判定表述 | 由谁执行 |
|---|---|---|---|
| F1 | 唯一权威写入者 | 只有 `fysund` 可以登记 / 发布资产与 Service、签发 Grant、推进 Activity / Execution / Effect / Verification 状态、签发与关闭 Approval、记账预算、写权威账本 | `fysund` |
| F2 | 概率组件只产候选 | 模型与任何第三方 Agent 的输出都是候选；准入、调度、Effect 状态推进、验收是确定性操作 | `fysund` + 执行点 |
| F3 | 外部效果先记录再执行 | 经资格化执行点的每个外部 / 不可逆 mutation 先持久化 ActionIntent 与 Effect（带幂等键、`policy_version`）再分发，并回报 receipt；无回报的 Effect 不得进入 `verified` | 执行点持久化与回报；`fysund` 核对 |
| F4 | 完成由独立验证判定 | Activity 阶段完成需独立 Verifier 对照契约与已回报 Effect 判定 | `fysund` |
| F5 | secret 隔离 | OS 管理的 secret 只以 `SecretRef` 出现，只向资格化执行点按 Grant 解析；Harness 与 Runtime 进程不见明文。Harness 自持凭据不在范围，列入不防 | `fysund`（解析）+ 执行点（注入） |
| F6 | 契约与负例不为实现让路 | 公共契约、manifest schema、状态迁移、错误、负例不因某个实现不通过而删改 | 流程 |
| F7 | 部署策略声明、能力不自扩权 | Policy 决定什么可自动执行；Agent Definition 与 Harness 不得自行扩大 Grant、不得隐瞒未满足条件 | 执行点强制；`fysund` 签发 |
| F8 | 未知外部改动受保护 | 自主代理不覆盖、回退、混入非本任务的并发改动 | Harness + 执行点 |
| F9 | Execution 内有效权限单调收窄 | 有效权限集 = Grant ∩ Policy 允许集 ∩ 工具面；收窄自动生效，扩张只经 Approval（`approved_once`）且仍 ⊆ 父 Grant | `fysund` 签发；执行点过滤 |

来源标签（`label{source_principal, integrity, origin_ref, flags?}`）继续作为 F2 配套机制；赋予责任在执行点与 Harness 控制点，`fysund` 记录 `labels_digest`。

## 8. 明确不做的事

- 不实现 loop、prompt 装配、工具选择、compaction、subagent 传输、会话日志、沙箱、终端、LSP、聊天 UI——由 Harness 资产提供；
- 不做数据面转发：`fysund` 不作为 MCP server 转发工具调用、不代理模型请求、不渲染上下文进 harness 输入；
- 不做计费结算：只产生 `meter/recorded` 事件与导出，账单由外部系统生成；
- 不自造包格式（OCI artifact + Agent Skills）、策略语言（Cedar 子集）、传输协议（MCP / A2A / HTTP）；
- 不内置角色、团队、数字员工、业务流程；
- 不替代宿主 OS、容器编排或设备控制系统；
- 不区分能力等级、不设兼容等级制度；
- 不持有 Runtime 的 Loop 状态；不承诺模型内部状态或进程的透明迁移。

## 9. 与 v0.7.0 的定位差异（摘要）

| 维度 | v0.7.0 | 重定义 |
|---|---|---|
| 定义句 | Agent 能力的操作系统；提供运行服务 | Agent 能力管理（提示词、Skills、工具、MCP、Harness、Loop）和 Runtime 控制的运行基座 |
| Harness / Loop | 工程术语 / 不做 | 一级资产种类；登记、版本、绑定、度量 |
| Capability 实现 | 单一 Skills 目录 | Agent Definition（多种资产组合） |
| `fysund` 与数据面 | 唯一 MCP server、Provider 代理、渲染上下文、投递管道 | 不在热路径；执行点承担 |
| 守卫 | `fysund` 内组件、必经之路 | 独立可选执行点之一 |
| 多主体 | 开放问题 Q5 | 核心：Tenant = Principal + Scope + Plan |
| 服务化 | 无 | 服务平面：Service / Endpoint / Plan / Meter / Agent Card |
| F3 / F7 / F9 | 限经守卫的 MCP 工具面（脚注） | 由资格化执行点执行（架构） |
