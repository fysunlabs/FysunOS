# 01 · 定义与设计原则

- 状态：设计稿 v0.6.0（2026-09-07）
- 变更依据：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) 与 ADR-0002 ~ ADR-0006
- 决策依据：[ADR-0001](../adr/0001-founding-decisions.md)、[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md)、[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)

## 1. FysunOS 的定义

### 1.1 一句话定义

> **FysunOS 是 Agent 能力的操作系统：它定义描述能力、访问资源、运行与交接工作的
> 公共契约，并提供让同一能力在任意 Agent 实现、Runtime 与环境中持续、受控、可恢复地
> 运行并从反馈中改进的运行服务。**

可判定限定：这里的"任意 Agent 实现、Runtime 与环境"指**任何实现 Runtime ABI 最小面
（MCP 客户端 + 最小生命周期 RPC）的 Runtime**（见 [04 接口与 ABI](04-abi.md)、
[00 术语 §7](00-terminology.md#7-runtime-abi)）。不满足最小面的 harness 不在承诺范围内；
承诺是否成立由该 Runtime 是否通过 Runtime ABI 资格化判定，不由文字判定。

### 1.2 完整定义

FysunOS 以 **Capability（可复用的行为能力）** 为应用单元，以 **统一的能力契约、资源接口与
执行协议** 为系统边界。它把 Agent 的决策核心（模型、提示、决策策略）与其所需的认知资源和
环境接口绑定为一次次可持续、可恢复、受控的执行，并由系统承担执行生命周期、资源供给与计量、
权限与预算、外部效果、结果验证、协作交接、观测与恢复。由此，同一能力可以在不同的 Agent
实现、Runtime 和部署环境之间被快速配置、持续运行、按环境适配，并在用户校准与运行反馈中
不断改进。

它的目的不是让 Agent 遵守更多流程，而是**提高 Agent 完成实际工作的有效性**——完成率、
成本、耗时与人工介入——并把可信与可靠作为系统承担的底线，而非使用者的负担。

用最短的话概括系统职责：

> **管理"决策核心 × 资源 × 环境"的绑定，并可靠地运行这种绑定。**

### 1.3 "操作系统"在这里指什么

不是替代 Windows / Linux / 设备控制系统，也不是模仿传统 OS 的全部子系统。保留的是 OS 类比
中真正有价值的三件事：

1. **应用声明需求**：能力声明它需要什么资源、运行条件和边界；
2. **系统提供稳定接口**：隐藏 Runtime 与环境差异，同时明确能力限制；
3. **运行结果有约定语义**：启动、暂停、取消、恢复、失败、交接、升级不能各说各话。

价值在于：能力开发者不必反复解决上下文装配、资源访问、执行恢复、协作交接、授权与审计的
问题；这些由系统一次性提供。

### 1.4 对外坐标

FysunOS 采用 AOS 参考架构（*The Agent Operating System*，arXiv:2608.03214）的两平面坐标做
**自我声明**，以便 Runtime 实现者与外部读者判断本系统供给哪些职责、不供给哪些。只借用其
平面坐标，不采用其命名作为 FysunOS 术语。

| AOS 平面 / 职责 | FysunOS 对应 | 供给声明 |
|---|---|---|
| **Control & Governance Plane** — intent、policy、trust、authority、confidence、auditability、observability、human oversight | Authority 子系统（Principal / Scope、Grant、ActionIntent → Effect → Receipt → Verification、Approval、Policy）、治理账本、可选 `assurance` | **全量供给**（含 Approval 与可选 assurance） |
| **Runtime & Coordination Plane** — agent lifecycle、workflow coordination、model/tool routing、context & memory coordination、scheduling、traffic management、runtime assurance | Activity 子系统（生命周期、触发与调度、检查点、Handoff）、Resource 子系统（模型代理、上下文装配、Memory 准入）、Verifier | **部分供给**：生命周期、调度、交接、上下文/记忆协调、验证；**不供给**：决策核心 loop、沙箱、traffic management（留给 Runtime 与网关） |
| **平台服务** — 宿主 OS、容器、物理基础设施 | §6 明确不做 | 不供给 |

对象级映射（AOS 概念 ↔ FysunOS 对象）见 [02 对象模型 §4](02-object-model.md)；conformance
声明（MVP 满足哪些不变量、哪些控制点可选、哪些威胁不在范围）见
[09 威胁模型](09-threat-model.md)。

## 2. Capability 的定义

> **Capability 是可复用、可配置、可演进的 Agent 行为能力：描述在一定条件与约束下能完成
> 什么工作或持续提供什么行为，以及实现它所需的资源、方法与运行支持。**

一项 Capability 由两部分组成，二者必须分开：

- **行为契约**（对外承诺，迁移时保持）：输入、输出或持续行为；怎样算做好（质量目标，机器
  可判定表示法见 [07 §1.5](07-evolution-and-edge.md#15-质量目标的表示法)）；成本、时间、
  权限与风险边界；验证方式。功能、质量、资源/风险都属于契约，但由不同机制检查（功能与
  质量由 Verifier；资源与风险由 Grant / 预算 / Policy）。
- **实现与运行描述**（可整体替换）：Prompt、Skills、知识与记忆配置、工具、执行方法，以及
  对决策核心、Runtime 的要求。实现包采用 Agent Skills 规范（`SKILL.md`），FysunOS 不定义
  自有格式。

规则：

- 不区分"基础能力"与"高级能力"。能力可以调整、组合、分化甚至演变为另一种能力。
- 系统自动记录来源与变化并支持回退；只在扩大权限、增加外部动作、提高预算或影响设备安全
  时，才按部署环境 Policy 增加检查。
- 一项能力可以独立闭环，也可以在执行中调用其他能力、委派工作、传递产物。**协作是能力面中
  的普通一项，不是独立子系统。**
- 配置了一套资源只说明"准备如何工作"；运行结果与反馈才是能力证据。
- 面向使用者，一项能力只需回答四个问题：做什么、怎样算做好、依靠什么做、在什么边界内做。
  这些可由用户与 Agent 对话生成，不要求手写协议。

## 3. 八条设计原则

| # | 原则 | 含义 |
|---|---|---|
| P1 | **标准化必要边界，让能力自由变化** | 只结构化机器必须检查或跨组件交接的部分；其余保留自然语言与可引用资源 |
| P2 | **可靠性由系统承担，不变成用户流程** | 权威写入、效果记录、独立验证、secret 隔离在系统内自动发生 |
| P3 | **契约可移植，过程不承诺** | 迁移保持外部行为契约；模型、Prompt、Loop、Agent 数量、内部步骤都可换 |
| P4 | **长期活动是一级生命周期** | Activity 可以没有终点；Execution 是它的一次运行；两者都不依附于会话或进程 |
| P5 | **协作是普通能力行为** | 没有多 Agent 子系统、没有特殊 Manager 身份；委派发生时系统自动记录交接事实 |
| P6 | **调整默认轻量，检查只在风险边界** | 扩权、新增外部动作、提高预算、影响设备安全时执行部署 Policy；其余直接生效、自动留痕、可回退 |
| P7 | **部署环境声明策略，OS 执行边界** | 不内置统一风险等级；能力不得自扩权、不得隐瞒不满足运行条件；策略以版本化 Policy 对象承载 |
| P8 | **不重造决策核心** | 凡 Runtime 已具备且不涉及权威的能力，一律通过 Runtime ABI 复用（凡有业界标准的边界用标准表达：MCP / A2A / Agent Skills；DSH 为第一个 Level-2 参考适配器） |

## 4. 系统承担的可靠性底线

以下不是设计目的，而是定义得以成立的底座。它们由系统而非用户承担，且不因实现方便、
Runtime 偏好或"自校正"而放松：

| ID | 底线 | 可判定表述 |
|---|---|---|
| F1 | 唯一权威写入者 | 只有 `fysund` 可以签发 Grant、推进 Activity/Execution/Effect/Verification 状态、签发与关闭 Approval、记账预算、写权威账本。Runtime、UI、CLI、SDK、第三方 Agent 都是客户端 |
| F2 | 概率组件只产候选 | 模型与任何第三方 Agent 的输出（含工具调用）都是候选；准入、调度、Effect 提交、对账、最终验收是确定性的 daemon 操作 |
| F3 | 外部效果先记录再执行 | 每个外部或不可逆 mutation 都先持久化 ActionIntent 与 Effect、带幂等键、在 fencing 下分发与对账；工具成功不等于完成 |
| F4 | 完成由独立验证判定 | Activity 阶段完成需要独立 Verifier 对照契约与已对账 Effect 判定；进程退出、模型说"完成"、Provider 返回、外部回执都不够 |
| F5 | secret 隔离 | Provider 与用户 secret 只进 Secret Store 与批准的非日志输入路径；Runtime 进程只见 `SecretRef`，请求经 daemon 代理 |
| F6 | 契约与负例不为实现让路 | 公共契约、状态迁移、错误、负例向量不因某个实现不通过而删改；变更走正式流程。范围含 Policy：策略本身版本化、可追溯，不得为放行某次实现而静默改写 |
| F7 | 部署策略声明、能力不自扩权 | 环境决定什么动作可自动执行；能力不得自行扩大 Grant、不得隐瞒不满足的运行条件 |
| F8 | 未知外部改动受保护 | 自主代理不覆盖、回退、混入非本任务的并发改动 |
| F9 | Execution 内有效权限单调收窄 | 一次 Execution 生命周期内，有效权限集（Grant ∩ Policy 允许集 ∩ 工具面）只能因两类事件变化：收窄自动生效；任何扩张必须经 Approval，且扩张后仍 ⊆ 父 Grant。可写成负例向量：构造试图扩权的候选，断言账本中不出现无 Approval 的 `grant/issued` |

**来源标签作为 F2 的配套机制。** 候选不仅要被当作候选，其参数值的**来源**也要可见：每个进入
上下文的片段、每个工具结果、每个 Artifact 携带 `label{source_principal, integrity, origin_ref}`，
`integrity` 取 `trusted_user` / `trusted_system` / `verified_data` / `untrusted_external`；标签由
Resource 子系统在装配时赋予，Runtime 不能改写；Policy 可引用标签（例如"外部发送类 Effect 的
收件人参数含 `untrusted_external` 来源 → `approve`"）。这不是完整的 CaMeL 级信息流控制，而是
在协议边界即可实现的"标签 + 引用监视器"；完整信息流控制列为后续研究（见
[09 威胁模型 §7](09-threat-model.md#7-开放问题)）。

F1–F5 与 F8 直接迁移自 CognitiveOS 公理 A1–A5、A8；F6 对应 A6；F7 是首轮讨论新增；F9 依据
Progent 单调限制（arXiv:2504.11703）与 AOS 委派单调性（arXiv:2608.03214），决策见
[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)。
证据晋升纪律（原 A7）在 FysunOS 中作为工程流程规则而非系统底线保留，见 [08 MVP 计划](08-mvp-plan.md)。

## 5. 设计目标与优势

| 目标 | 使用者获得什么 |
|---|---|
| 能力可复用 | 为 X 建立的内容能力，重绑定适配层后即可用于小红书，不必重建整套 Agent |
| 工作可持续 | 会话结束、进程重启、引擎替换，任务和长期活动都不从头开始 |
| 资源可适配 | 同一能力在桌面用远端模型与浏览器工具，在边缘用本地模型与传感器；不满足条件时明确报告，不静默降质 |
| 协作有连续性 | 委派、产物、依赖、进展是系统事实，不只活在某个主 Agent 的聊天记录里 |
| 改进可积累 | 校准与反馈影响后续行为，自动留痕、可回退，不要求走重型晋升流水线；每次调整带预测，系统自动对照后续验证结果 |
| 轻量易用 | 复杂性由系统吸收；不要求使用者维护审批、版本晋升或迁移流程；审批是一级对象，等人与等触发在 UI 上区分 |

任务完成率、成本、耗时、人工介入率、回滚率、验证独立性比例是**需要实测**的收益指标（定义见
[08 §4](08-mvp-plan.md#4-效率与性能的度量)）；"符合 FysunOS 标准"本身不证明 Agent 更有能力。
后续应在同模型、同预算或明确披露成本差异的条件下，与直接使用 Agent 的效果比较。

## 6. 明确不做的事

- 不规定模型、Prompt、Loop 算法或多 Agent 拓扑；
- 不内置角色、团队、数字员工、业务流程或 Agent Shell（它们是上层应用与入口）；
- 不替代宿主 OS 或设备控制系统；运动控制、急停等硬实时职责留在宿主控制器；
- 不承诺模型内部状态、进程或凭据的透明迁移；
- 不区分能力等级，不设复杂的兼容等级和升降级制度；
- 不定义自己的实现包格式（采用 Agent Skills）；
- 不自造策略语言（采用 Cedar 子集）；
- 不自建记忆引擎（记忆引擎为可插拔 Resource 提供者，OS 只拥有准入、作用域、撤回）；
- 不自建 loop、prompt 装配、工具注册流水线、会话日志、compaction、subagent 传输、sandbox、
  聊天 UI、skills 加载、终端与 LSP——这些由 Runtime 提供，经 Runtime ABI 接入。

## 7. 五个正交概念（速览）

| 概念 | 回答的问题 | 属于谁 |
|---|---|---|
| **Capability** | 提供什么行为、怎样算做好、需要什么、在什么边界内 | 应用层声明，OS 读取其契约 |
| **决策核心（Agent Core）** | 下一步做什么：模型调用、Prompt、规划、Skill 选择、继续/等待/重试/求助/停止 | Runtime 实现；OS 视其为可替换的候选产生者 |
| **Resource** | 执行消耗什么：认知资源（模型、上下文、记忆、知识、Skill）与操作资源（计算、存储、网络、工具、传感器、执行器、其他 Agent） | OS 统一发现、授权、装配、计量 |
| **Activity / Execution** | 工作是什么、现在到哪了 | OS 的核心管理对象 |
| **Runtime** | 在哪里、由什么引擎跑 | 通过 Runtime ABI（MCP 守卫 + 最小生命周期 RPC + 可选控制点）向 OS 暴露控制点 |

四个曾经重叠的词的归位：**认知资源**是 Resource 的子类；**Loop** 是决策核心的控制策略
（OS 只握预算、取消、恢复边界）；**Harness** 降为工程术语，指 OS 在一次 Execution 中围绕
决策核心装配出的支撑组合，不是规范对象；**Runtime** 收窄为承载与适配。详见
[02 对象模型](02-object-model.md)。
