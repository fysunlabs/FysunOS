# ADR-0005 · Approval 一级对象与证据门控演进

- 状态：accepted
- 日期：2026-09-07
- 决策者：owner
- 背景来源：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) §2.5、§3.1、§3.4、§3.6、§5.4、§6.1、§7.1、§7.2、§8.2
- supersedes：无（新增）。ADR-0001 §D8"默认直接生效、自动留痕；不设候选→评估→灰度→晋升的强制流水线"保留有效；
  本 ADR 补充可选门控与预测对照。ADR-0001 §D7"何时需人工"由部署环境声明保留有效；本 ADR 补充其对象载体。

## 背景

v0.5.0 中人在环（HITL）只有一个错误码 `needs_human` 加一个 `request_id`；Activity 的 `waiting` / `blocked`
不区分"等触发"与"等人"。审查报告 §2.5 引用 CAP（arXiv:2512.04123）的生产数据：68% 的生产 Agent 在 ≤10 步后
即人工介入，47% 在 <5 步；HITL 是永久性架构控制而非过渡阶段。在 OPC 场景中"人"是最稀缺的资源，AOS 6.14 把
human-approval capacity 列为语义资源。OPC 用户不会区分"Activity 暂停等待触发"与"Activity 等我批准"，两种等待
在 UI 与通知语义上完全不同。

Progent 的"扩张需外部批准"（[ADR-0004](0004-provenance-labels-and-monotonic-privilege.md) F9）需要一个
承接批准的对象；A2A 的 `INPUT_REQUIRED` / `AUTH_REQUIRED` 与 MCP elicitation 提供了标准传输载体。

演进侧，报告 §3.6 与 §6.1 指出：Adjustment 默认直接生效且无评估钩子；技能库生命周期研究
（arXiv:2607.10113）审计 124 篇文献，结论是"准入与修复反复被证明重要"、"单次成功不等于新配置更好"；
AHE（arXiv:2604.25850）的核心发现是把每次编辑变成"预测 → 下一轮任务级结果回验"的可证伪契约，是自动演进
不退化为试错的关键。Forrester / Anaconda 面板（二手，仅作趋势）显示有全量自动评估的 Agent 年回滚率 9%，
无评估者 47%。

验证侧，报告 §3.4 指出 Verification 报告没有可比较的字段：74% 的生产团队依赖人工评估，LLM judge 在 agentic
工具使用 rubric 上噪声最大（RuVerBench，arXiv:2606.29920）；"人工复核"应是一级 verifier 类型而非兜底。

## 决策

### D1 · Approval 为 Authority 子系统一级对象

- `Approval` 归 Authority 子系统，由 `fysund` 签发与关闭。字段按 [00-terminology](../design/00-terminology.md) §5：

| 字段 | 内容 |
|---|---|
| `id`, `execution_id`, `activity_id` | 归属 |
| `subject` | `effect_candidate` / `grant_expansion` / `adjustment` / `handoff_accept` / `memory_admit` / `verification_override` |
| `reason` | 触发规则：`policy_decision` 引用（`policy_version` + `rule_id`）、来源标签冲突、F9 扩张判定 |
| `presentation` | 呈现给人的最小上下文：候选摘要、参数（含来源标签）、预算影响、可逆性、Verifier 预检结果；引用 Artifact，不复制正文 |
| `options` | `approve_once` / `approve_for_activity` / `approve_for_capability` / `deny` / `modify(params)` / `defer(until)` |
| `decision`, `decided_by`, `decided_at` | 决定者必须是 Principal |
| `expiry`, `on_expiry` | `deny`（缺省，fail-closed）/ `escalate` / `keep_waiting` |
| `derived` | 决定派生出的对象引用（见 D2） |
| `batch_ref` | 同类候选合并为一次审批 |

- 状态：`pending` / `approved` / `denied` / `modified` / `expired`。事件：`approval/requested`、`approval/granted`、
  `approval/denied`、`approval/modified`、`approval/expired`。`approval/requested` 携带 `policy_decision`。
- Runtime 只能观察到 `needs_human(approval_id)`（Policy 要求人工确认）或 `auth_required(approval_id)`
  （`subject = grant_expansion`）；ABI 中不存在 `decide_approval`。
- Capability API：`approval.list(filter)`、`approval.observe(id)`、`approval.decide(id, option, params?)`、
  `approval.batch_decide(batch_ref, option)`。`decide` 的调用者必须解析为 Principal。
- 负例：一个 `approval/granted` 事件的 `principal` 解析为 Execution 或 Runtime 而非人类 / 系统 Principal → 失败；
  `presentation` 中内联了 Artifact 正文而非引用 → 失败；`expiry` 到期且 `on_expiry` 缺省时状态不是 `expired`
  且后续动作不是拒绝 → 失败。

### D2 · `approve_for_*` 是权限扩张的唯一合法路径

- `approve_once`：仅放行本次候选；不派生对象。
- `approve_for_activity`：`fysund` 自动签发一条衰减 Grant（`grant/issued`，`derived` 记录其 id），范围限于本
  Activity，且 ⊆ 父 Grant。
- `approve_for_capability`：`fysund` 自动产生一条 Adjustment（`scope = capability_default`，`derived` 记录其 id）
  并留痕；该 Adjustment 仍受 D5 的预测与评估规则约束。
- `modify(params)`：改写候选参数后放行，对应 `policy_decision.outcome = modify`；改写后的参数记入 ActionIntent。
- 这是 F9（[ADR-0004](0004-provenance-labels-and-monotonic-privilege.md) §D4）中"扩张必须经 Approval"的
  唯一实现路径；除此之外账本中不得出现扩张性的 `grant/issued`。
- 负例：一条 `grant/issued` 的范围大于同 Execution 先前的有效权限集，且其 `causation_id` 不指向一个
  `approval/granted` → 失败。

### D3 · 审批容量进入预算模型

- `Activity.constraints` 新增 `max_pending_approvals`（同时未决 Approval 上限）与 `approval_sla`（期望决定时限）。
- 超过 `max_pending_approvals` 时，Activity 进入 `input_required` 而不是继续堆积候选；`approval_sla` 超时由
  `on_expiry` 决定后续动作。
- Activity 状态机新增 `input_required`（等人：Approval 未决或需要用户输入），与 `waiting`（等触发）分开；
  事件 `activity/input-required`。Execution 等 Approval 时自身为 `waiting`，其所属 Activity 为 `input_required`。
- 对应 A2A `TASK_STATE_INPUT_REQUIRED`；Handoff 也有 `input_required` 与 `auth_required` 两态。
- 负例：一个 Activity 有未决 Approval 却处于 `waiting` 而非 `input_required` → 失败；未决 Approval 数超过
  `max_pending_approvals` 后仍出现新的 `approval/requested` → 失败。

### D4 · 崩溃恢复中的 Approval 水位

- 崩溃前未决的 Approval 在恢复后仍为 `pending`，不得被 Runtime 或 `fysund` 视为已批。
- `resume_execution` 在 `effect_watermark` 之外携带 **Approval 水位**（恢复点之前已关闭的 Approval 集合摘要）；
  Runtime 据此知道哪些候选仍在等待。
- 恢复顺序不变（03 §6：读投影 → 校验 epoch → 对账 outcome-unknown → resume），在对账步骤中同时列出未决 Approval。
- 负例：崩溃恢复后，一条 `pending` Approval 对应的 Effect 出现 `effect/authorized` 而无 `approval/granted` → 失败。
  此负例进入 M3 验收。

### D5 · Adjustment 携带预测与评估；证据门控为可选

- Adjustment 新增字段：
  - `predicted_effect{statement, metric?, direction?}`：申请方（用户、Agent、评估器）对本次调整预期改善什么的
    声明；`statement` 为自然语言（P1），`metric` 与 `direction` 可选；
  - `evaluated_against{window, result: supported | refuted | inconclusive, verification_refs[]}`：生效后 `window`
    次 Execution 的 Verification 汇总与 `predicted_effect` 的对照，由 `fysund` 在后续 Verification 到达时自动填充；
  - `evidence_gate{enabled, regression_set_ref, result}`：可选，由 Policy 或用户开启；开启时 `promote` 到
    `capability_default` 前必须对回归集跑一次评估。
- 回归集 = 该 Capability 历史上已 `verified` 的 Execution 输入与验收 checklist，自然形成，不需人工构造。
- 事件：`adjustment/predicted`（记录预测）、`adjustment/evaluated`（填充对照）；`adjustment/reverted` 带
  `reason`（用于统计回滚原因）。
- ADR-0001 §D8 不变：`evidence_gate` 缺省关闭，Adjustment 默认直接生效并自动留痕；本 ADR 只让"是否更好"可被
  系统而非用户回答。
- `capability.health(id)` 从 evidence 与 Adjustment 历史折叠出冗余、过期、反复失败的 checklist 项。
- 负例：`evidence_gate.enabled = true` 时未跑回归集的 `adjustment/promoted` → 失败（进入 M4 验收）；一条
  Adjustment 生效后 `window` 内有 Verification 到达但 `evaluated_against` 未填充 → 失败；`predicted_effect` 缺
  `statement` → `propose_adjustment` 拒绝。

### D6 · Verification 扩展

- Verification 新增字段：`verifier_id`、`verifier_kind ∈ {deterministic, model_judge, human, downstream_outcome}`、
  `independence`（集合，元素取自 `different_process` / `different_model` / `different_principal`；空集记为
  `not_independent`）、`rubric_ref`、`items[]`、`calibration_ref`（judge 类引用校准集与 Krippendorff α；缺失记
  `uncalibrated`）、`verdict ∈ {pass, pass_with_warning, fail, inconclusive}`、可选 `assurance`。
- `human` 是一级 verifier 类型，不是"机械检查不存在时的兜底"。OPC 用户"看过再发"是一次 Verification
  （`verifier_kind = human`），计入 evidence，同时计入人工介入指标。
- `inconclusive` 是合法结果，不得被 Runtime 或应用解释为通过；事件 `verification/inconclusive`。
- Policy 可为 Effect 类别或 Capability 设定最低 verifier 要求（[ADR-0003](0003-policy-object-and-language.md) §D5）。
- `Capability.contract.quality` 采用 Skill Contract 骨架 `(P, O, A, V, F)`，V 为 checklist，每项标注 `verifier_kind`
  （08 开放问题 4 的解法）。
- 负例：`verdict = inconclusive` 的 Verification 触发 `handoff/verified` 或 `activity/completed` → 失败；
  同模型自评的 Verification `independence` 非 `not_independent` → 失败。

### D7 · Approval 的传输载体

Approval 对象与传输载体分离；同一 Approval 可经以下任一载体呈现与决定：

| 载体 | 条件 |
|---|---|
| MCP `elicitation/create` | MCP client 支持 elicitation 时，经 MCP 守卫下发（含 URL-mode） |
| Web UI | Approval 呈现面是 Web UI 的第一优先级 |
| CLI `fysun` | `approval.list / decide` 的命令行形态 |
| `deliver_input(approval_result)` | 父 Execution 或外部 A2A 侧把决定送回；决定者仍必须解析为 Principal |

- 负例：经 MCP elicitation 返回的决定被直接写为 `approval/granted` 而未解析出 Principal → 失败。

## 否决的替代方案

- **HITL 仅为错误码。**无法表达选项、到期、批量、派生；无法把审批容量纳入预算；UI 无法区分等触发与等人。
- **强制候选→评估→灰度→晋升流水线。**违背 ADR-0001 §D8"演进轻量"；OPC 场景中多数 Adjustment 是改一句
  Prompt，强制流水线的成本高于收益。本 ADR 以可选 `evidence_gate` 替代。
- **把审批交给 Runtime 内部。**DSH 的 `ctx.approval` 是一次性 turn 级审批，决定不入治理账本、不能派生 Grant、
  Runtime 可自行"批准"（T9）；与 F1 冲突。`fysun-approval` 保留为 Level-2 控制点适配器，只做呈现转发。
- **Confidence 数值向量作为审批触发。**AOS 自己警告"数值会制造误导性精度"；FysunOS 只取动作映射
  （`assurance{level, basis[], dependence}` 为可选字段），`ESCALATE` = Approval。

## 后果

- Web UI 的 Approval 呈现面成为第一优先级（08 开放问题 3 不变，但优先级明确）。
- M3 验收新增 Approval 闭环：闭环第 2 步"先看过再发布"通过 Approval 对象实现；崩溃恢复后未决 Approval 不被
  视为已批；Handoff 状态可无损映射到 A2A TaskState。
- M4 验收新增预测→评估：一次 Adjustment 带 `predicted_effect`，下一 Execution 的 Verification 自动填充
  `evaluated_against`；`evidence_gate` 开启时未跑回归集的 promote 被拒。
- 度量新增三项：人工介入率（`approval/requested` / Execution）、回滚率（`adjustment/reverted` / `adjustment/applied`）、
  验证独立性比例（`independence != not_independent` 的 Verification 占比）。
- `fysun-dsh` 的 `fysun-approval` 子组件语义收窄为转发与呈现，不再持有决定。
- 每个 Approval 增加账本事件与 Web UI / CLI 的呈现成本；`batch_ref` 缓解 OPC 场景中的审批洪泛（T10）。

## 待后续 ADR 的问题

- `on_expiry = escalate` 的升级目标（另一 Principal、另一载体）如何声明。
- 多主体下 Approval 的决定权分配与分布式一致性（与 08 开放问题 5、9 合并考虑）。
- `downstream_outcome` verifier 的时间窗与归因规则。
- judge 类 verifier 的校准阈值是否写入 Policy（参考 arXiv:2608.00794 的 α≥0.67 / 0.70 / 0.80 分级）。

其余见 [08 MVP 计划 §7](../design/08-mvp-plan.md#7-开放问题待后续-adr)。
