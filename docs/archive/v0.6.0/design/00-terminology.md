# 00 · 术语与对象命名对照表

- 状态：设计稿 v0.6.0（2026-09-07）
- 变更依据：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) 与 ADR-0002 ~ ADR-0006
- 用途：全仓库术语、对象名、状态名、事件名、字段名的唯一基准。其他设计文档与本表冲突时，先修正冲突方；本表变更须同步 CHANGELOG。

## 1. v0.5.0 → v0.6.0 改名对照

| v0.5.0 用法 | v0.6.0 用法 | 依据 |
|---|---|---|
| `Intent`（效果级请求记录） | `ActionIntent` | ADR-0006；避免与 AOS / MCP / A2A 中"顶层目标"义的 intent 冲突 |
| 事件族 `intent/*` | `action-intent/recorded` / `action-intent/rejected` / `action-intent/replayed` | 同上 |
| Harness ABI | **Runtime ABI** | ADR-0002 |
| Composition ABI（公共接口） | 降为内部接口："内部适配器生命周期"（04 附录 A） | ADR-0002 |
| DSH = 默认参考 Runtime；`fysun` bundle | DSH = **第一个 Level-2 参考适配器**；适配器名 `fysun-dsh` | ADR-0002 |
| `fysun-tool-gate`（DSH tools 流水线订阅） | 取消；由 MCP 守卫取代 | ADR-0002 |
| 部署策略 = 本地文件 | **Policy** 版本化账本对象（Cedar 子集） | ADR-0003 |
| `needs_human` 仅为错误码 | `Approval` 一级对象；`needs_human(approval_id)` 保留为错误码 | ADR-0005 |
| Handoff 状态 `issued / accepted / in_progress / delivered` | `submitted / working / completed`（对齐 A2A v1.0），`verified` 为 FysunOS 叠加态 | 02 §2.8 |
| 事件 `handoff/issued`、`handoff/accepted`、`handoff/delivered` | `handoff/submitted`、`handoff/working`、`handoff/completed` | 同上 |
| Activity 等待态只有 `waiting` | `waiting`（等触发）与 `input_required`（等人）分开 | ADR-0005 |
| 安全包络作为 Effect 字段 | 安全包络为绑定到设备 Resource 的 Policy 对象；Effect 只引用 | 07 §2.3 |
| 里程碑 M2 | M2a（MCP 守卫 + Level-1 接入）/ M2b（DSH Level-2 适配） | 08 §3 |

"Harness"一词保留为工程术语（01 §7：一次 Execution 中围绕决策核心装配出的支撑组合），不再出现在任何接口名中。

## 2. 一级对象

| 对象 | 归属子系统 | 一句话 | 定义位置 |
|---|---|---|---|
| Capability | Capability | 可复用、可配置、可演进的行为能力；契约与实现分离 | 02 §2.1 |
| Activity | Activity | 有终点或持续的工作目标；状态而非调度器；`target` 不可变 | 02 §2.2 |
| Execution | Activity | Activity 的一次运行；绑定 Capability 快照、Runtime、Resource | 02 §2.3 |
| Principal / Scope | Authority | 授权主体与范围；Execution 是可被授权的主体 | 02 §2.4 |
| Grant | Authority | 只能衰减的授权，带预算与撤销 epoch | 02 §2.5 |
| **ActionIntent** → Effect → Receipt → Verification | Authority | 外部效果协议：先记录再执行，独立验证 | 02 §2.6、§2.7 |
| Handoff | Activity | 委派与交接事实；外部形态为 A2A Task | 02 §2.8 |
| Resource | Resource | 模型、Secret、上下文源、记忆/知识、工具/适配器、设备、其他 Agent | 02 §2.9 |
| Adjustment | Capability | 轻量演进载体；带预测与评估对照 | 02 §2.10 |
| **Approval**（新增） | Authority | 人在环审批；权限扩张的唯一合法路径 | 02 §2.11 |
| **Policy**（新增） | Authority | 版本化部署策略对象；每个准入决定引用其版本 | 02 §2.12 |
| **Memory**（新增） | Resource | 经准入的 durable 知识记录；带来源、作用域、撤回 | 02 §2.13 |
| Event / Projection / Artifact | Authority | 账本真源 / 投影 / 内容寻址产物 | 02 §2.14 |
| 来源/完整性标签（横切） | Resource 赋予 | 每个上下文片段、工具结果、Artifact 携带的 `label` | 02 §2.15 |

不进入对象模型：角色、团队、数字员工、Project、Routine、业务流程、Agent Shell、聊天消息、Prompt 正文、Skill 正文、工具 schema、模型流式 chunk、**Plan**（决策核心内部规划）。

## 3. 状态机

| 对象 | 状态集 | 说明 |
|---|---|---|
| Capability.status | `draft` / `active` / `withdrawn` | 不变 |
| Activity.state | `created` / `active` / `waiting` / `input_required` / `paused` / `blocked` / `completed` / `cancelled` | `waiting` = 等触发；`input_required` = 等人（Approval 未决或需要用户输入）；`blocked` 携带原因与恢复动作 |
| Execution.state | `starting` / `running` / `waiting` / `checkpointed` / `finished` / `failed` / `cancelled` | 不变；Execution 等 Approval 时为 `waiting`，其所属 Activity 为 `input_required` |
| Handoff.state | `submitted` / `working` / `input_required` / `auth_required` / `completed` / `verified` / `failed` / `rejected` / `cancelled` | 对齐 A2A v1.0 TaskState；`verified` 为 FysunOS 叠加态；`working` 可带 `phase: accepted \| in_progress` 作为属性而非状态 |
| Effect.state | `authorized` / `dispatch_started` / `receipt` / `outcome_unknown` / `reconciled` / `quarantined` / `compensated` | 不变 |
| Approval.state | `pending` / `approved` / `denied` / `modified` / `expired` | `expired` 按 `on_expiry` 决定后续动作，缺省 deny |
| Policy.status | `active` / `retired` | 版本只增不改 |
| Memory.state | `proposed` / `admitted` / `rejected` / `superseded` / `retracted` | 矛盾记录不合并，标记 `conflict_with` |
| Adjustment | `applied` / `reverted` / `promoted`，附 `evaluated_against.result` | 见 §5 |
| Verification.verdict | `pass` / `pass_with_warning` / `fail` / `inconclusive` | `inconclusive` 不得解释为通过 |

### Handoff ↔ A2A v1.0 TaskState 对照

| A2A TaskState | Handoff.state |
|---|---|
| `TASK_STATE_SUBMITTED` | `submitted` |
| `TASK_STATE_WORKING` | `working` |
| `TASK_STATE_INPUT_REQUIRED` | `input_required` |
| `TASK_STATE_AUTH_REQUIRED` | `auth_required` |
| `TASK_STATE_COMPLETED` | `completed` |
| （无） | `verified`（FysunOS 叠加：父 Execution 或独立 Verifier 对 deliverables 的验证） |
| `TASK_STATE_FAILED` | `failed` |
| `TASK_STATE_REJECTED` | `rejected` |
| `TASK_STATE_CANCELED` | `cancelled`（FysunOS 全仓库统一用英式拼写） |

## 4. 事件族（v0.6.0 完整集）

```text
principal/created | principal/revoked
scope/defined | scope/revised

capability/registered | capability/revised | capability/derived
capability/evidence-recorded | capability/withdrawn

activity/created | activity/activated | activity/waiting | activity/input-required
activity/paused | activity/resumed | activity/blocked | activity/completed | activity/cancelled
activity/policy-bound | activity/triggered

execution/started | execution/checkpointed | execution/resumed | execution/forked
execution/finished | execution/failed | execution/cancelled

handoff/submitted | handoff/working | handoff/input-required | handoff/auth-required
handoff/completed | handoff/verified | handoff/failed | handoff/rejected | handoff/cancelled

grant/issued | grant/attenuated | grant/consumed | grant/revoked

action-intent/recorded | action-intent/rejected | action-intent/replayed
effect/authorized | effect/dispatch-started | effect/receipt
effect/outcome-unknown | effect/reconciled | effect/quarantined | effect/compensated

verification/requested | verification/reported | verification/inconclusive

approval/requested | approval/granted | approval/denied | approval/modified | approval/expired

policy/loaded | policy/revised | policy/retired

memory/proposed | memory/admitted | memory/rejected | memory/superseded | memory/retracted

adjustment/applied | adjustment/predicted | adjustment/evaluated
adjustment/reverted | adjustment/promoted

context/assembled

resource/registered | resource/bound | resource/metered | resource/unavailable | resource/quarantined
runtime/attached | runtime/qualified | runtime/health | runtime/detached
```

通用字段：`seq`、`time`、`principal`、`causation_id`、`correlation_id`；与 Execution 相关的事件另带 `execution_id`、`fencing_epoch`。
准入类事件（`effect/authorized`、`action-intent/rejected`、`approval/requested`、`grant/issued`、`memory/admitted`、`memory/rejected`）
另带 `policy_decision`。`adjustment/reverted` 带 `reason`。`context/assembled` 带 `labels_digest`。`runtime/qualified` 带 `control_points[]`。
由 Approval 派生的 `grant/issued` 与 `adjustment/applied` 另带 `approval_id`。

## 5. 关键字段与取值

| 字段 | 所在对象 | 取值 |
|---|---|---|
| `label` | 上下文片段、工具结果、Artifact、Effect 参数 | `{source_principal, integrity, origin_ref}` |
| `label.integrity` | 同上 | `trusted_user` / `trusted_system` / `verified_data` / `untrusted_external` |
| `policy_decision` | 准入类事件 | `{policy_version, rule_id, outcome, obligations[]}` |
| `policy_decision.outcome` | 同上 | `allow` / `deny` / `modify` / `approve`（`approve` = 需要 Approval） |
| `Approval.subject` | Approval | `effect_candidate` / `grant_expansion` / `adjustment` / `handoff_accept` / `memory_admit` / `verification_override` |
| `Approval.options` | Approval | `approve_once` / `approve_for_activity` / `approve_for_capability` / `deny` / `modify(params)` / `defer(until)` |
| `Approval.on_expiry` | Approval | `deny`（缺省）/ `escalate` / `keep_waiting` |
| `Activity.constraints` 新增 | Activity | `max_pending_approvals`、`approval_sla` |
| `Verification.verifier_kind` | Verification | `deterministic` / `model_judge` / `human` / `downstream_outcome` |
| `Verification.independence` | Verification | 集合，元素取自 `different_process` / `different_model` / `different_principal`；空集记为 `not_independent` |
| `Verification.calibration_ref` | Verification | judge 类引用校准集与 Krippendorff α；缺失记 `uncalibrated` |
| `Verification.verdict` | Verification | `pass` / `pass_with_warning` / `fail` / `inconclusive` |
| `assurance`（可选） | Effect 候选、Verification | `{level: high \| medium \| low \| unknown, basis[], dependence}`；`basis` 为证据引用，不是分数 |
| `Adjustment.predicted_effect` | Adjustment | `{statement, metric?, direction?}` |
| `Adjustment.evaluated_against` | Adjustment | `{window, result: supported \| refuted \| inconclusive, verification_refs[]}` |
| `Adjustment.evidence_gate` | Adjustment | `{enabled, regression_set_ref, result}`；由 Policy 或用户开启 |
| `Memory.scope` | Memory | `principal` / `activity` / `capability` / `global` |
| `Handoff.context_id` | Handoff | = Activity id 或 Handoff 链根；对应 A2A `contextId` |
| `Policy` | Policy | `{id, version, digest, source, loaded_at, rules_ref, status}` |
| 安全包络（Policy 子类） | 绑定设备 Resource 的 Policy | `velocity_bounds`、`geofence`、`interface_allowlist`、`rate_limit`、`deadman`、`emergency_stop_ref`；Effect 记录 `envelope_ref` 与 `envelope_check` |
| 质量目标骨架 | Capability.contract.quality | Skill Contract `(P 前置条件, O 操作, A 产物, V 验证器, F 已知失败模式)`；V 为 checklist，每项标注 `verifier_kind` |

## 6. 底线 F1–F9

| ID | 名称 |
|---|---|
| F1 | 唯一权威写入者 |
| F2 | 概率组件只产候选（配套机制：来源标签） |
| F3 | 外部效果先记录再执行 |
| F4 | 完成由独立验证判定 |
| F5 | secret 隔离 |
| F6 | 契约与负例不为实现让路（范围含 Policy） |
| F7 | 部署策略声明、能力不自扩权 |
| F8 | 未知外部改动受保护 |
| **F9** | **Execution 内有效权限单调收窄**：有效权限集 = Grant ∩ Policy 允许集 ∩ 工具面；收窄自动生效；任何扩张必须经 Approval，且扩张后仍 ⊆ 父 Grant |

## 7. Runtime ABI

| 块 | 内容 | 谁必须实现 |
|---|---|---|
| **MCP 守卫** | `fysund`（组件名 `fysun-mcp-guard`）以 MCP 中间层身份位于 harness 与工具之间：`tools/list` 按 Grant 与 Policy 过滤改写；`tools/call` pre = `submit_candidate`（ActionIntent 持久化、Policy、Grant、来源标签检查）、执行 = Effect dispatch、post = Receipt 回填；`tasks/*` 表达长任务与 outcome-unknown；`elicitation/create` 为 Approval 传输载体之一。守卫结果语义 `Pass` / `Reject` / `Mutated`（与 agentgateway `mcpGuardrails` 相同） | 所有 Runtime 必经 |
| **最小生命周期 RPC** | `describe() -> {type, version, control_points[]}`、`start_execution`、`resume_execution`、`cancel`、`deliver_input`、`checkpoint()`（可返回 `unsupported`）、`health()`；语言无关 JSON-RPC over stdio（MVP） | 所有 Runtime 必须实现 |
| **可选控制点**（能力位） | `context_injection`、`checkpoint`、`fork`、`compaction_hook`、`session_link`、`subagent_delegation` | 按 Runtime 声明；账本记录 `runtime/qualified{control_points[]}` |
| **A2A**（外部 Handoff 形态） | `fysund` 作为 A2A client 发起 Task；Handoff 状态直接映射 TaskState；对外暴露 A2A server（签名 Agent Card）为多主体场景，非 MVP | 外部 Agent |
| **Agent Skills**（实现包） | `Capability.implementation` 引用 Skills 目录（digest 固定）；`allowed-tools` 为 Grant 请求初始上界；`compatibility` 映射到 `requirements` | 实现包作者 |

Runtime → OS 调用（Level-2 控制点的内部语义；Level-1 经 MCP 守卫隐式完成）：`os.request_context`、`os.submit_candidate`、
`os.observe_effect`、`os.request_resource`、`os.report_checkpoint`、`os.report_progress`、`os.request_handoff`、
`os.report_failure`、`os.propose_memory`、`os.propose_adjustment`。

Runtime 不得调用（不存在于 ABI 中）：`mint_grant`、`commit_effect`、`complete_activity`、`write_ledger`、`mark_verified`、`decide_approval`。

**Level-1 / Level-2** 只是文档速记，不是等级制度：Level-1 = MCP 客户端 + 最小生命周期 RPC；Level-2 = Level-1 + 至少一个可选控制点。
环境匹配依据的是 `control_points[]`，不是 Level 标签。

## 8. 错误与拒绝码

`denied.policy` · `denied.grant` · `denied.budget` · `denied.fencing` · `not_runnable.environment` ·
`needs_human(approval_id)` · `input_required(request_ref)` · `auth_required(approval_id)` · `unknown_outcome` · `inconclusive`

- `needs_human`：Policy 要求人工确认，`fysund` 已创建 Approval；
- `input_required`：Execution 或 Handoff 需要父方或用户提供输入（非授权问题）；对应 A2A `INPUT_REQUIRED`、MCP elicitation；
- `auth_required`：需要额外授权，映射为 `subject = grant_expansion` 的 Approval；对应 A2A `AUTH_REQUIRED`；
- `inconclusive`：Verification 无法判定；不是通过。

## 9. 组件与命名

| 名称 | 含义 |
|---|---|
| `fysund` | 权威 daemon（Rust），唯一权威写入者 |
| `fysun` | CLI |
| `fysun-mcp-guard` | MCP 守卫组件（`fysund` 内）；是否复用 agentgateway 待评估（08 开放问题 10） |
| `fysun-dsh` | DSH 的 Level-2 参考适配器；技术形态为叠在 `dsh-base` 之上的 bundle + `fysun` profile，子组件：`fysun-credentials`、`fysun-approval`、`fysun-session-link`、`fysun-subagent`、`fysun-context`、`fysun-rpc` |
| `fysund-edge` | 边缘节点上的精简 daemon（同一账本模型），内含确定性包络校验器 |

## 10. Capability API 新增方法（v0.6.0）

`approval.list(filter)` · `approval.observe(id)` · `approval.decide(id, option, params?)` · `approval.batch_decide(batch_ref, option)` ·
`policy.load(source) -> {policy_id, version, digest}` · `policy.describe() -> {version, digest, rules_summary}` · `policy.explain(event_ref) -> policy_decision` ·
`capability.register(contract, implementation: skills_dir_ref | ..., requirements)` · `capability.health(id)` ·
`memory.list(filter)` · `memory.retract(id, reason)` · `verification.report(id)`（返回 02 §2.7 扩展字段）

## 11. 威胁向量编号

T1 间接 prompt injection · T2 恶意/被投毒实现包 · T3 权限扩张 · T4 凭据外泄 · T5 账本篡改/重放/陈旧 epoch · T6 验证欺骗 ·
T7 记忆投毒 · T8 物理动作越界 · T9 Runtime 越权推进状态 · T10 DoS / 预算耗尽。负例向量编号格式 `NV-T<n>-<序号>`（占位，实现阶段登记）。

## 12. 来源缩写

报告中的 A1–A15（学术）、B1–B6（协议）、C1–C5（工业）、D1–D4（需求数据）编号仅在审查报告内使用；设计文档引用时写全名或 arXiv 号，不写编号。
