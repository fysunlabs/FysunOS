# 02 · 对象模型

- 状态：设计稿 v0.6.0（2026-09-07）
- 变更依据：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) 与 ADR-0002 ~ ADR-0006
- 前置：[01 定义与设计原则](01-definition-and-principles.md)
- 决策依据：[ADR-0003](../adr/0003-policy-object-and-language.md)、[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)、[ADR-0005](../adr/0005-approval-and-evidence-gated-evolution.md)、[ADR-0006](../adr/0006-terminology-action-intent.md)

本文定义 FysunOS 权威账本中的一级对象。字段列表是**设计意图**，不是 schema；正式 schema 在
实现阶段以机器可读形式登记，并配负例向量。对象名、状态名、事件名、字段名以
[00 术语与对象命名对照表](00-terminology.md) 为唯一基准。

## 1. 对象总览

```text
Principal ──owns──▶ Scope
                      │
   Capability ◀───── declares/uses ───── Application
      │ contract · implementation（可为 Agent Skills 目录）· requirements · evidence · lineage · health
      │
   Activity ──runs as──▶ Execution ──binds──▶ { Capability snapshot, Runtime(control_points[]), Resource bindings, Agent Core config }
      │  policy_binding ──▶ Policy      │
      │                     ├── ActionIntent ──▶ Effect ──▶ Receipt ──▶ Verification
      │                     │      └── 准入决定引用 Policy(version) ──▶ policy_decision
      │                     ├── Approval ──derives──▶ { Grant（衰减签发） | Adjustment(scope=capability_default) }
      │                     ├── Handoff ──▶ (Capability | Execution | 外部 A2A Agent)
      │                     └── Checkpoint（Runtime 检查点引用 + Effect 水位 + Approval 水位）
      │
   Adjustment ──scoped to──▶ { Execution | Activity | Capability default }
      │ predicted_effect · evaluated_against · evidence_gate

   Grant  ──attenuates──▶ Resource / Capability use, per Principal+Scope, with Budget & epoch
   Policy ──referenced by──▶ Activity.policy_binding · 每个准入决定 · 设备 Resource（安全包络）
   Memory ──Resource 子系统──▶ 经准入的 durable 知识；读取经上下文装配，写入是内部 Effect
   label{source_principal, integrity, origin_ref} ──标注在──▶ context_view 片段 · 工具结果 · Artifact · Effect 参数
   Event ──folds into──▶ Projection   （账本唯一真源；状态是投影）
   Artifact  内容寻址产物（带 label），被 Execution / Handoff / Verification / Memory 引用
```

## 2. 一级对象

### 2.1 Capability

| 字段组 | 内容 |
|---|---|
| `id`, `contract_version` | 稳定身份；契约兼容变化递增版本，契约不兼容变化默认产生新 Capability（见谱系） |
| **contract** | `behavior`（输入/输出 schema 或持续行为描述）、`quality`（质量目标与判定方式；骨架为 Skill Contract `(P 前置条件, O 操作, A 产物, V 验证器, F 已知失败模式)`，其中 V 为 checklist，每项标注 `verifier_kind`，详见 [07 演进 §1.5](07-evolution-and-edge.md)）、`bounds`（成本、时间、权限、风险边界）、`verification`（Verifier 类型与参数） |
| **implementation** | 当前实现引用：Prompt/Skill/代码包/工具组合/决策核心配置，带 `digest`；可引用 Agent Skills 目录（`SKILL.md`，digest 固定），此时 `allowed-tools` 为 Grant 请求的初始上界（只能再收窄）、`compatibility` 映射到 `requirements`；一项 Capability 可有多个实现版本；FysunOS 不定义自己的实现包格式 |
| **requirements** | 需要的 Resource 类型与最低条件、Runtime 控制点（Runtime ABI 控制点能力位）、环境要求（延迟、离线、能耗等） |
| **evidence** | 运行证据引用：在什么环境、什么输入下的验证结果（含 Verification 的 `verifier_kind` 与 `independence`）；配置声明不算证据 |
| **lineage** | `derived_from` / `forked_from` / `evolved_from` 关系与触发原因 |
| `health` | 从 evidence 与 Adjustment 历史折叠的健康摘要：冗余（契约近似的其他 Capability）、过期（evidence 全部来自已退役的 Runtime 或实现版本）、反复失败的 checklist 项；是投影，由 `capability.health(id)` 读取 |
| `status` | `draft` / `active` / `withdrawn`；不设 candidate→proven 的强制阶梯 |

### 2.2 Activity

| 字段 | 内容 |
|---|---|
| `id`, `principal`, `scope` | 归属 |
| `target` | 有终点的目标（含验收条件）**或**持续服务目标（响应要求、停止条件、观察指标）；**不可变**——修订走 `Adjustment{scope: this_activity}` 或派生新 Activity |
| `constraints` | 预算、时间窗口、允许的资源与外部动作范围（映射为 Grant 请求）；审批容量 `max_pending_approvals`、`approval_sla` |
| `capability_ref` | 主 Capability 及固定的实现快照 |
| `triggers` | 人工、事件、周期、外部 webhook；触发产生新 Execution 或唤醒现有 Execution |
| `policy_binding` | 引用 Policy 对象（`id` + `version`，见 2.12）；绑定与换版产生 `activity/policy-bound` |
| `state` | `created` / `active` / `waiting` / `input_required` / `paused` / `blocked` / `completed` / `cancelled` |
| `continuity` | 跨 Execution 必须保留的产物引用与摘要（不是聊天记录） |

状态说明：

- `waiting` = 等触发（周期、事件、webhook）；`input_required` = 等人（Approval 未决，或 Execution / Handoff 需要用户输入）。二者在 UI 与通知语义上不同，不合并。
- `blocked` 携带原因与恢复动作（例如 quarantine 的 Effect 待人工对账）。
- 待决 Approval 数达到 `max_pending_approvals` 时，Activity 进入 `input_required`、相关 Execution 进入 `waiting`，daemon 不再为该 Activity 签发新的 Approval（新的需审批候选按 `denied.budget` 拒绝并留痕），直到待决数回落。审批容量是预算的一部分。
- `approval_sla` 是该 Activity 下 Approval 的缺省 `expiry`；到期动作由 Approval 的 `on_expiry` 决定。

Activity 是**状态，不是调度器**（吸收 DSH goal 的定位），但与 DSH goal 不同：它由 `fysund`
持久拥有，进程重启后按 `policy_binding` 决定是否自动恢复，不强制人工 re-arm。

### 2.3 Execution

| 字段 | 内容 |
|---|---|
| `id`, `activity_id`, `parent_execution_id?` | 归属与 fork/委派谱系 |
| `capability_snapshot` | 本次生效的契约版本与实现 digest |
| `runtime_binding` | Runtime 类型与实例（如 `dsh@<version>#<pid>`）、Runtime 侧会话标识（`session_id`）、`control_points[]`（该 Runtime 已资格化的控制点，来自 `runtime/qualified`） |
| `resource_bindings` | 模型、SecretRef、上下文源、工具/适配器、设备的实际绑定与 Grant 引用 |
| `agent_core_config` | 决策核心的有效配置（Prompt 装配策略、Loop 策略参数、Skill 集）——以摘要与 digest 记录，正文由 Runtime 持有 |
| `fencing_epoch` | 写入围栏；旧 Execution 恢复后不能覆盖新 epoch 的 Effect |
| `checkpoints[]` | 可恢复点：Runtime 侧检查点引用 + OS 侧水位，水位同时含**已对账 Effect 水位**与**Approval 水位**（截至该点已关闭的 Approval；未关闭者恢复后仍为 `pending`） |
| `state` | `starting` / `running` / `waiting` / `checkpointed` / `finished` / `failed` / `cancelled`；等待 Approval 决定时为 `waiting`，其所属 Activity 为 `input_required` |

**Execution 与 Runtime Session 的关系**：一个 Execution 对应 Runtime 中一个（或 fork 出的多个）
Session；Session 日志是 Runtime 的会话账本，Execution 是 OS 的治理账本条目。两者通过
`execution_id / session_id / causation_id / fencing_epoch` 关联，**不合并**（见 [03 架构 §4](03-architecture.md#4-双账本)）。

### 2.4 Principal / Scope

- **Principal**：用户、Runtime 实例、某个 Execution、外部系统（含外部 A2A Agent）。Execution 是可被授权的主体，
  这样委派出去的工作拥有独立身份与预算。Approval 的决定者必须是 Principal 中的人或其授权系统，不能是 Runtime。
- **Scope**：资源可见与可写的范围（工作区路径、设备组、账号、数据域）。Grant 总是绑定
  `(principal, scope)`。

### 2.5 Grant

| 字段 | 内容 |
|---|---|
| `principal`, `scope`, `resource_or_capability` | 授权对象 |
| `attenuation` | 相对上游 Grant 只能收窄：允许的操作、路径前缀、速率、单次上限 |
| `budget` | 金额/Token/调用次数/时长/审批容量预算及消耗记账 |
| `expiry`, `revocation_epoch` | 到期与撤销；撤销是事件，立即使后续 ActionIntent 拒绝 |
| `issued_by` | 总是 `fysund`；Runtime 只能 `request_resource`，不能签发 |
| `derived_from_approval?` | 由 `approve_for_activity` 派生的 Grant 记录来源 Approval |

委派（Handoff）时子 Execution 的 Grant 由 daemon 从父 Grant 衰减签发，**不会**因转交而扩大：
scope ⊆ 父、expiry ≤ 父、budget ≤ 父剩余，三条不等式均为负例向量。Execution 内有效权限集 =
Grant ∩ Policy 允许集 ∩ 工具面，只能自动收窄；任何扩张必须经 Approval 且扩张后仍 ⊆ 父 Grant（F9）。

### 2.6 ActionIntent → Effect → Receipt → Verification

直接迁移 CognitiveOS 的外部效果协议；`ActionIntent` 是持久化的效果级请求记录，不是顶层目标（顶层目标是
`Activity.target`）：

```text
候选（Runtime 提交的 tool call / 动作请求；Level-1 经 MCP 守卫 tools/call pre 阶段，Level-2 经 os.submit_candidate）
  -> 准入：Policy 评估（输出 policy_decision）+ Grant 检查 + 来源标签检查
       allow   -> 继续
       deny    -> action-intent/rejected（带 policy_decision）
       modify  -> 以改写后的参数继续（守卫结果 Mutated）
       approve -> 创建 Approval；Runtime 收到 needs_human(approval_id)；Execution waiting
                  approved / modified 后继续，denied / expired(deny) 后 action-intent/rejected
  -> 持久化 ActionIntent（幂等键 = canonical 参数 digest；重复命中 action-intent/replayed）
  -> 持久化 Effect（状态 authorized）
  -> 在 fencing_epoch 下分发
  -> Receipt（成功/失败）或 outcome-unknown
  -> outcome-unknown 只能 reconcile 或 quarantine，不能假定成功
  -> 固定 post-state
  -> 独立 Verification（按 Capability.contract.verification）
  -> commit / compensate
```

| 字段 | 所在 | 内容 |
|---|---|---|
| `execution_id`, `idempotency_key` | ActionIntent | 归属；幂等键 = canonical 参数 digest |
| `action` | ActionIntent | Effect 类别 + 工具名（MCP `tools/call` 的 name） |
| `params` | ActionIntent | canonical 参数；每个参数值带 `label`（2.15）：Level-2 由 Runtime 标注，Level-1 由 OS 按参数值与 context_view / 工具结果匹配尽力标注 |
| `assurance?` | ActionIntent | `{level, basis[], dependence}`，见下 |
| `policy_decision` | ActionIntent | 准入结果 `{policy_version, rule_id, outcome, obligations[]}`（2.12） |
| `approval_id?` | ActionIntent | 若 `outcome = approve`，指向对应 Approval |
| `state` | Effect | `authorized` / `dispatch_started` / `receipt` / `outcome_unknown` / `reconciled` / `quarantined` / `compensated` |
| `receipt` | Effect | 适配器返回的结构化回执（不是自然语言观察） |
| `envelope_ref`, `envelope_check` | Effect（物理） | 引用绑定设备 Resource 的安全包络 Policy 及校验结果（见 [07 边缘](07-evolution-and-edge.md)） |

- 只读、可逆的本地观察可走 `NoExternalCommitment` 审计快路径：仍需描述符、Scope、来源标签与审计事件，
  不必持久化 ActionIntent/Effect。
- 文件写入、网络 mutation、凭据使用、设备动作必须走完整协议。
- 物理 Effect 额外字段：`deadline`、`sensor_timestamp`、`uncertainty`、`reversibility_class`、
  `emergency_stop_ref`、`actuator_receipt`、`physical_post_state_verifier`；安全包络不再是 Effect 字段，
  Effect 只引用它（见 [07 边缘](07-evolution-and-edge.md)）。
- `assurance`（可选）：`{level: high | medium | low | unknown, basis[], dependence}`。取 AOS Confidence 的
  **动作映射**、不取其数值向量：`basis` 是证据引用（Verifier 报告、来源标签、Receipt、历史 evidence），不是分数；
  `dependence` 说明证据是否来自同一来源。Policy 可按 Effect 类别 × `assurance.level` 映射到控制动作
  `proceed` / `proceed_with_warning` / `retry` / `fallback` / `escalate` / `terminate`：`retry` 与 `fallback`
  由 Activity 子系统的 wait/switch/escalate 策略承担，`escalate` = 创建 Approval，`terminate` = cancel 并保留证据。
  不做跨步骤传播计算。

### 2.7 Verification

| 字段 | 内容 |
|---|---|
| `id`, `subject` | 被验证对象：Effect post-state、Execution 产出、Handoff deliverables、Adjustment 评估窗口 |
| `verifier_id`, `verifier_kind` | `deterministic`（测试 / schema / digest / 外部状态读取）/ `model_judge` / `human` / `downstream_outcome`（后续 Activity 的结果反推） |
| `independence` | 与执行者的独立性集合：`different_process` / `different_model` / `different_principal`；空集记为 `not_independent` |
| `rubric_ref` | 契约 `quality` checklist 的版本（Skill Contract 的 V） |
| `items[]` | 每条 checklist 项的判定、判定者、置信度（judge 给出）、引用的 Artifact |
| `calibration_ref` | judge 类 verifier 引用其校准集与 Krippendorff α；缺失记 `uncalibrated` |
| `verdict` | `pass` / `pass_with_warning` / `fail` / `inconclusive` |
| `assurance?` | 同 2.6；`dependence` 指出哪些 items 来自同一 judge |

规则：

- 由与执行者不同的 Verifier 执行；优先确定性检查；`model_judge` 必须在执行者可写面之外，且 `independence`
  不得为空集才计入 F4 的"独立验证"。
- `human` 是一级 verifier 类型，不是机械检查不存在时的兜底：用户"看过再发"是一次 Verification，记入 evidence，
  同时计入人工介入指标。
- Policy 可为 Effect 类别或 Capability 设定最低 verifier 要求（例如发布类动作至少 `deterministic` + `human`）；
  达不到即不通过。
- `inconclusive` 是合法结果，但不是通过；Runtime 与应用都不得将其解释为 `pass`（延续"not-run 永不等于通过"）。
- 报告不可变；过期由 Verification 生命周期承载，不改写旧报告。
- Verification 结果驱动 Activity 阶段完成，也作为 Capability 的 evidence 记录，并用于填充 Adjustment 的
  `evaluated_against`。

### 2.8 Handoff

当一个 Execution 调用另一个 Capability 或委派给另一个 Agent 时，系统自动记录：

| 字段 | 内容 |
|---|---|
| `id`, `context_id` | `context_id` = Activity id 或 Handoff 链根；对应 A2A `contextId`，外部 Agent 借此关联多次委派 |
| `from_execution`, `to` | 目标是 Capability（由 OS 解析并启动子 Execution）、已有 Execution，或外部 A2A Agent（经签名 Agent Card 发现） |
| `objective`, `inputs[]`, `expected_outputs` | 委派目标、输入产物引用、期望产出与验收关系 |
| `grant` | 衰减后的子 Grant；外部 Agent 不获得 Grant，只获得输入 Artifact |
| `state` | `submitted` / `working` / `input_required` / `auth_required` / `completed` / `verified` / `failed` / `rejected` / `cancelled`；`working` 可带 `phase: accepted \| in_progress` 属性 |
| `deliverables[]` | 不可变 Artifact 引用与子 Execution 的 Verification 引用 |

Handoff ↔ A2A v1.0 TaskState 对照：

| A2A TaskState | Handoff.state | 说明 |
|---|---|---|
| `TASK_STATE_SUBMITTED` | `submitted` | |
| `TASK_STATE_WORKING` | `working` | A2A 无"已接受未开始"态；FysunOS 以 `phase` 属性区分 |
| `TASK_STATE_INPUT_REQUIRED` | `input_required` | 子方需要父方或人提供输入；经 `deliver_input` 或 Approval 解除 |
| `TASK_STATE_AUTH_REQUIRED` | `auth_required` | 子方需要额外授权；映射为 `subject = grant_expansion` 的 Approval |
| `TASK_STATE_COMPLETED` | `completed` | 不含验证语义 |
| （无） | `verified` | FysunOS 叠加态：父 Execution 或独立 Verifier 对 deliverables 的验证 |
| `TASK_STATE_FAILED` | `failed` | 接单后失败；与 `rejected`（拒绝接单）区分 |
| `TASK_STATE_REJECTED` | `rejected` | |
| `TASK_STATE_CANCELED` | `cancelled` | FysunOS 全仓库统一英式拼写 |

规则：

- 目标是外部 Agent 时，`fysund` 作为 A2A client 发起 Task，Handoff 状态直接映射 TaskState；对外以签名 Agent Card
  暴露为 A2A server 属于多主体场景，非 MVP。
- Artifact 不可变；终态（`completed` / `verified` / `failed` / `rejected` / `cancelled`）不重开；细化或追问 =
  同 `context_id` 下的新 Handoff。
- Handoff 不要求预先定义团队或工作流；简单调用只产生一次 Handoff 记录。参考 DSH Agent Teams
  的 roster / mailbox / task DAG 设计，但真源在 OS 账本而非 Lead Session。参与者完成自己的部分，
  不替代父 Execution 对整体产出的检查。

### 2.9 Resource

统一发现、描述、授权、绑定、计量；保留类型语义，不做成同一种对象：

| 类型 | 例子 | 说明 |
|---|---|---|
| 模型服务 | Provider + model | 经 daemon 代理，计量 Token/费用 |
| Secret | `SecretRef` | 只在 Secret Store；Runtime 不见明文 |
| 上下文源 | 文件、URL、会话摘要、检索索引 | 装配前授权（authorize-before-rank），显式损失，digest 绑定；每个片段在装配时由 Resource 子系统赋予 `label`（2.15），`context/assembled` 记录 `labels_digest` |
| 记忆 / 知识 | 持久已准入的知识 | 见 2.13 Memory；与"上下文"严格区分：Memory 是 durable 知识，Context 是一次 Execution 的受控视图 |
| 工具 / 适配器 | 文件系统、浏览器、发布接口、灌溉阀门；MCP server | 工具是能力提供方的一种实现形态；工具调用 = Effect 候选；工具结果返回时由 Resource 子系统赋予 `label`（缺省 `untrusted_external`，只读本地 trusted 源除外） |
| 实现包 | Agent Skills 目录 | 登记 digest、`allowed-tools`、`compatibility`、来源；被 `Capability.implementation` 引用 |
| 设备 | 传感器、执行器、边缘节点 | 带延迟、能耗、离线行为描述；可绑定安全包络 Policy（2.12） |
| 其他 Agent | 可委派的执行体 | 通过 Handoff 使用；外部 Agent 经 A2A 发现（签名 Agent Card），登记为 Resource 并绑定 Principal |

### 2.10 Adjustment

承载"轻量演进"：

| 字段 | 内容 |
|---|---|
| `target` | 变更对象：Capability 实现/契约、Activity 策略、Execution 的决策核心配置或资源绑定 |
| `scope` | `this_execution` / `this_activity` / `capability_default` |
| `source` | 用户校准、Agent 自校正、评估结果、`approve_for_capability` 派生 |
| `effective_from` | 立即 / 下一 Execution / 指定检查点 |
| `rollback_target` | 回退到的前一版本或快照 |
| `risk_class` | 是否触及 Grant 扩权、新外部动作、预算上调、设备安全；决定是否走 Policy 检查（可产生 `subject = adjustment` 的 Approval） |
| `predicted_effect` | `{statement, metric?, direction?}`：申请方声明这次调整预期改善什么；事件 `adjustment/predicted` |
| `evaluated_against` | `{window, result: supported \| refuted \| inconclusive, verification_refs[]}`：生效后 window 内的 Verification 与预测的对照，由 daemon 在后续 Verification 到达时自动填充；事件 `adjustment/evaluated` |
| `evidence_gate` | `{enabled, regression_set_ref, result}`：由 Policy 或用户开启；开启时 promote 到 `capability_default` 前须对回归集（该 Capability 历史上已 `verified` 的 Execution 输入与验收 checklist，自然形成）跑一次评估，未跑或 `fail` 则拒绝 promote |

状态：`applied` / `reverted` / `promoted`，附 `evaluated_against.result`。默认仍**直接生效、自动留痕、可回退**
（不违背 ADR-0001 D8）；预测与评估只是让"是否更好"可由系统而非用户回答。`adjustment/reverted` 携带 `reason`，
用于统计回滚率。

### 2.11 Approval

人在环审批的一级对象；由 Authority 子系统拥有。

| 字段 | 内容 |
|---|---|
| `id`, `execution_id`, `activity_id` | 归属 |
| `subject` | 待批对象：`effect_candidate` / `grant_expansion` / `adjustment` / `handoff_accept` / `memory_admit` / `verification_override` |
| `reason` | 触发规则：`policy_decision` 引用（`policy_version` + `rule_id`）、来源标签冲突、权限扩张判定、审批容量 |
| `presentation` | 呈现给人的最小上下文：候选摘要、参数（含来源标签）、预算影响、可逆性、Verifier 预检结果；引用 Artifact，不复制正文 |
| `options` | `approve_once` / `approve_for_activity` / `approve_for_capability` / `deny` / `modify(params)` / `defer(until)` |
| `decision`, `decided_by`, `decided_at` | 决定者必须是 Principal，不能是 Runtime |
| `expiry`, `on_expiry` | 到期动作：`deny`（缺省，fail-closed）/ `escalate` / `keep_waiting`；`expiry` 缺省取 `Activity.constraints.approval_sla` |
| `derived` | `approve_for_activity` → 自动签发衰减 Grant（`grant/issued` 引用本 Approval）；`approve_for_capability` → 自动产生 `Adjustment{scope: capability_default}` 并留痕 |
| `batch_ref` | 同类候选合并为一次审批（例如一次批 20 条评论回复）；`approval.batch_decide` 按此引用 |

状态：`pending` / `approved` / `denied` / `modified` / `expired`。

规则：

- Approval 由 `fysund` 签发与关闭；Runtime 只能观察到 `needs_human(approval_id)`，不存在 `decide_approval` 调用。
- `approve_for_*` 是权限扩张的**唯一合法路径**（F9）；扩张后的有效权限仍 ⊆ 父 Grant。
- 人工审批容量进入预算模型：`max_pending_approvals`、`approval_sla` 见 2.2。
- 传输载体：Capability API（`approval.*`）、MCP `elicitation/create`（Runtime 的 MCP client 支持时）；载体不改变对象语义。
- 崩溃恢复后 `pending` 的 Approval 仍为 `pending`，不得被视为已批（见 [03 架构 §6](03-architecture.md)）。

### 2.12 Policy

版本化的部署策略对象；每个准入决定引用它的版本。

| 字段 | 内容 |
|---|---|
| `id`, `version`, `digest` | 版本只增不改；`policy/revised` 产生新 version，旧 version 保留以回答历史决定 |
| `source` | 来源（文件路径、产品缺省、管理接口）与加载者 Principal |
| `loaded_at` | 生效时间；事件 `policy/loaded` |
| `rules_ref` | 规则正文的 Artifact 引用（Cedar 子集） |
| `status` | `active` / `retired` |

`policy_decision{policy_version, rule_id, outcome: allow | deny | modify | approve, obligations[]}` 附在每个准入类事件上
（`effect/authorized`、`action-intent/rejected`、`approval/requested`、`grant/issued`、`memory/admitted`、
`memory/rejected`）；`approve` = 需要 Approval；`obligations[]` 是准入附带的义务（例如最低 verifier 要求、记录附加证据）。
`policy.explain(event_ref)` 返回该决定。

规则：

- **策略语言不自造**：MVP 采用 Cedar 子集（与 agentgateway、AgentCore Policy 相同格式，可静态分析"这套策略是否允许某类动作"）；
  语言选择见 [ADR-0003](../adr/0003-policy-object-and-language.md)。
- 评估输入至少包含：principal、action（Effect 类别 + 工具名）、resource（Scope / 设备 / 账号）、context
  （参数摘要、参数与上下文的来源标签、预算余量、可逆性、`assurance.level`、Runtime 类型与 `control_points[]`）。
- Policy 回答的问题见 [03 架构 §7](03-architecture.md)；OS 只执行策略边界，不内置统一风险等级。
- 安全包络是绑定设备 Resource 的 Policy 子类：`velocity_bounds`、`geofence`、`interface_allowlist`、`rate_limit`、
  `deadman`、`emergency_stop_ref`；Effect 记录 `envelope_ref` 与 `envelope_check`（见 [07 边缘](07-evolution-and-edge.md)）。
- Policy 在 F6 范围内：规则与负例不因某个实现不通过而删改。

### 2.13 Memory

经准入的 durable 知识记录；Resource 子系统拥有其准入、作用域、撤回与可审计性。

| 字段 | 内容 |
|---|---|
| `id`, `scope` | `principal` / `activity` / `capability` / `global` |
| `content_ref` | Artifact 引用（内容寻址） |
| `provenance` | 来源 Execution、来源片段的 `label` 集合 |
| `admitted_by`, `admitted_at` | 准入者：Verifier 或 Principal；对应 `memory/admitted`（带 `policy_decision`） |
| `freshness` | `valid_until`、`supersedes`、`superseded_by` |
| `conflict_with` | 与之矛盾的 Memory 引用；矛盾记录不合并、不静默平均 |
| `retraction` | 撤回原因与时间；对应 `memory/retracted` |
| `state` | `proposed` / `admitted` / `rejected` / `superseded` / `retracted` |

事件：`memory/proposed | memory/admitted | memory/rejected | memory/superseded | memory/retracted`。

规则：

- 写入是**内部 Effect**：走审计快路径、不需外部 receipt，但必须有事件；读取经上下文装配并携带 `label`
  （`integrity` 取 `verified_data`，`origin_ref` 指向 Memory）。
- 记忆引擎（MemOS / Mem0 / Letta 等）作为可插拔 Resource 提供者接入；OS 不自建检索与调度算法，只拥有准入、作用域、
  撤回、可审计（P8）。
- 来源含 `untrusted_external` 的候选按 Policy 决定 `approve`（`subject = memory_admit`）或 `deny`；这是记忆投毒的主要控制点。
- 矛盾记录保留双方并标记 `conflict_with`，由 Policy 决定"优先最新 / 请求澄清 / 忽略"。

### 2.14 Event / Projection / Artifact

- **Event**：append-only 治理事实，带 `seq`、`time`、`principal`、`causation_id`、`correlation_id`；与 Execution 相关的
  事件另带 `execution_id`、`fencing_epoch`。准入类事件携带 `policy_decision`；`context/assembled` 携带 `labels_digest`；
  `runtime/qualified` 携带 `control_points[]`；`adjustment/reverted` 携带 `reason`。完整事件族见
  [03 架构 §5.1](03-architecture.md)。
- **Projection**：由事件折叠出的当前状态（Activity 视图、预算余额、Grant 有效集、Capability 目录与 health、
  待决 Approval 列表…），可删除、可重建、带 source watermark 与 digest。
- **Artifact**：内容寻址（digest）产物引用，payload 存于对象存储或文件系统，账本只存引用；携带 `label`（2.15），
  被 Execution / Handoff / Verification / Memory / Approval.presentation 引用；不可变。

### 2.15 来源与完整性标签（横切）

每个进入上下文的片段、每个工具结果、每个 Artifact、每个 Effect 候选参数携带：

```text
label{ source_principal, integrity, origin_ref }
integrity ∈ trusted_user | trusted_system | verified_data | untrusted_external
```

| `integrity` | 含义 | 例子 |
|---|---|---|
| `trusted_user` | 当前 Principal 直接输入 | 用户消息、用户提供的文件 |
| `trusted_system` | `fysund` 自身产生 | 契约、Policy、账本投影、context_view 的结构部分 |
| `verified_data` | 经 Verification 或准入 | 已 `verified` 的 Artifact、已 `admitted` 的 Memory |
| `untrusted_external` | 其余外部内容 | 网页、外部 API 返回、外部 Agent 产出、未准入的模型输出 |

规则：

- 标签在装配时由 Resource 子系统赋予；**Runtime 不能改写**。
- Effect 候选参数的标注责任：Level-2 由 Runtime 经控制点标注；Level-1 由 OS 按参数值与 context_view / 工具结果匹配
  尽力标注。无法归因的参数值记为 `untrusted_external`（fail-closed），不默认为可信。标签只降不升：合并、摘要、
  改写多个片段得到的新片段，其 `integrity` 取输入中最低者；升级到 `verified_data` 的唯一路径是 Verification 或 Memory 准入。
- Policy 可引用标签：例如"外部发送类 Effect 的 `recipient` 参数含 `untrusted_external` 来源 → `approve`"。
  这是 F2"概率组件只产候选"的配套机制：候选不仅要有权限，还要说明参数从哪里来。
- 这不是完整的 CaMeL 解释器（那要求控制 Runtime 的执行语义），而是 FIDES / Progent 一级的"标签 + 引用监视器"，
  在协议边界即可实现。完整信息流控制列为后续研究。外部基准数字（如 Progent 在 AgentDojo 上 39.9%→1.0%）
  仅说明方向，不构成 FysunOS 的任何性能或安全声明。

## 3. 不进入 OS 对象模型的概念

角色、团队、数字员工、Project、Routine、业务流程、Agent Shell、聊天消息、Prompt 正文、
Skill 正文、工具 schema、模型流式 chunk、**Plan**（决策核心的内部规划；与 AOS 参考架构的有意差异，
见 [01 §3 P8](01-definition-and-principles.md)——Handoff 与 `Activity.continuity` 承担"计划留痕"的最小面）。
前者属应用层组合；后者属 Runtime 会话账本。

## 4. 与 AOS 参考架构的对象映射

对外坐标的对象级说明；AOS 指 *The Agent Operating System (AOS): A Reference Operating Architecture for
Distributed Agentic Systems*（arXiv:2608.03214）。

| AOS 概念 | FysunOS 对象 | 备注 |
|---|---|---|
| Intent（顶层目标） | `Activity.target` + `constraints` + `policy_binding` | `target` 不可变 |
| Context | context_view（authorize-before-rank、digest、losses[]、`label`） | |
| Capability | `Capability`（contract / implementation / requirements / evidence / lineage） | 副作用声明由 `bounds` 承载 |
| Provider | `Capability.implementation` + `Execution.runtime_binding` | 未合成为单独对象 |
| Plan | 无（留给 Runtime 决策核心） | 有意差异，见 §3 |
| Policy | `Policy`（2.12）+ `policy_decision` | |
| Identity / Authority | Principal / Grant | |
| Delegation | Grant 衰减 + Handoff | 三条不等式为负例向量 |
| Trust | 无显式对象 | 多 Runtime 资格化时再引入 |
| Confidence | `assurance`（2.6、2.7） | 取动作映射，不取数值向量 |
| Evidence & Provenance | 治理账本 + `label` + Verification | |
| Execution | `Execution` | |
| Semantic Resource（token / 审批容量 / 时间预算） | `Grant.budget` + `Activity.constraints` 审批容量 + Resource 计量 | |
| Human oversight | `Approval`（2.11） | |

AOS 概念不变量与 F1–F9 的对应：

| AOS 不变量 | FysunOS |
|---|---|
| 每次执行源于不可变 intent 或显式系统目标 | `Activity.target` 不可变 |
| 每次 provider 调用实现已声明的 capability 契约 | `Execution.capability_snapshot` |
| 每个 consequential action 被权限链与策略决定覆盖 | F1 + F3 + F7；`policy_decision` |
| 委派权限单调不扩张 | Grant 衰减、F7、F9 |
| 控制决定与执行动作分别可识别、共同可追溯 | 双账本 + `causation_id` |
| Confidence 决定保留证据并映射到显式动作 | `assurance.basis[]` + Policy 动作映射 |
| 审计记录保留版本、谱系、因果 | 治理账本；Policy 版本化 |
