# ADR-0011 · Approval 结果集、事件配对与审批面来源

- 状态：accepted
- 日期：2026-09-08
- 决策者：owner
- 背景来源：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md) §I.1 决策 11、12，§I.2 H-1、H-3、H-4、H-7，§0 C1-6、C2-4；[ADR-0005](0005-approval-and-evidence-gated-evolution.md) §D1–§D4、§D7；DeepSeek Harness `docs/subsystems/approval.md`（`approval/request` waterfall、`never` 策略执行位置）；DSH goal 恢复后需人工 re-arm 的设计
- supersedes：无（新增）。承接并细化 ADR-0005 §D1（状态集与事件）、§D2（派生条件）、§D3（等待态）、§D4（恢复水位）、§D7（载体）；ADR-0005 正文已于 v0.7.0 同步修订。

## 背景

ADR-0005 建立了 Approval 一级对象，但 v0.6.0 的细则有几处不闭合：

- **结果集开放**：状态 `pending / approved / denied / modified / expired` 中 `modified` 与 `approved` 都表示放行、`expired` 本身不是决定而是
  触发 `on_expiry` 的条件；派生 Grant / Adjustment 的条件写成"`approve_for_*`"（一个 option）而不是结果，导致"哪些结果可以派生"要靠读 options
  推断。五个事件（请求、批准、拒绝、修改、过期各一，见归档 `docs/archive/v0.6.0/design/00-terminology.md` §4）与状态逐一对应，
  但"过期"事件后还会有第二个决定事件，同一 Approval 会出现两条终态事件。
- **事件不配对**：ADR-0003 §D2 的负例只要求"`outcome = approve` 必有请求事件"，没有要求"每个决定必有请求"；`fysund` 在请求事件落账失败时
  的行为未定义。
- **参数复制**：`presentation` 含"参数（含来源标签）"，等于把工具参数明文复制进 Approval 对象与事件，`untrusted_external` 内容随之进入审批面
  与账本正文（T1 二次注入面、账本体量）。
- **审批面来源未定**：ADR-0005 §D7 列出四种载体，但没有说明 Runtime 自绘的审批 UI（DSH 审批卡片）是否有效；蓝图 H-4 新增 T11 "审批面伪造"
  ——聊天通道里的"批准"文本、Runtime 自己渲染的卡片，都不应产生 `approval/decided`。
- **等待态歧义**（C1-6）：Execution `waiting` 同时表达等触发、等 Approval、等输入。
- **`handoff_accept` 无触发条件**（C2-4）。
- **审批轰炸**（H-4）：ADR-0005 §D3 的 `max_pending_approvals` 限制并发未决数，但不限制速率；被注入的 Runtime 可以在每个 Approval 被拒后立刻再
  发一个，OPC 用户被迫连续点"拒绝"。
- **恢复后武装**（H-7）：02 §2.2 "进程重启后按 `policy_binding` 决定是否自动恢复，不强制人工 re-arm"与 DSH goal "恢复后需人工 re-arm"的对照
  被写成 FysunOS 的优点，但对含外部效果的 Activity，重启后无条件继续等于让崩溃前的模型状态在无人确认下继续产生外部效果。

DSH 事实（外部调研）：DSH 的 `approval/request` 是 waterfall 服务，任一监听器可应答而不 `next()`；但 approval policy 为 `never`（一律拒绝）时
在服务内部先于 waterfall 执行，`prepend` 的监听器也绕不过。这决定了 `fysun-dsh` 拦截路径的前置条件。

## 决策

### D1 · 结果集封闭：`approved_once | rejected | cancelled | unavailable`；只有 `approved_once` 派生

- Approval 状态集：**`pending / approved_once / rejected / cancelled / unavailable`**。`pending` 是唯一非终态。
- `Approval.result ∈ approved_once | rejected | cancelled | unavailable`：
  - `approved_once`：本次 subject 放行。**所有放行类决定的结果都是 `approved_once`**，包括 `modify(params)`（改写后放行，记新 `params_digest`）
    与 `approve_for_activity` / `approve_for_capability`（放行本次 + 派生）；
  - `rejected`：Principal 拒绝，或 `expiry` 到期且 `on_expiry = deny`（缺省）；
  - `cancelled`：subject 已不存在（Execution 取消、Activity 取消、候选被 Runtime 撤回）或 batch 被整体取消；
  - `unavailable`：审批面不可用或事件无法配对提交（D2）。
- `options` 保留（`approve_once / approve_for_activity / approve_for_capability / deny / modify(params) / defer(until)`），它决定
  **`derived`** 与 `params_digest`，不决定 `result`：`approve_for_activity` → `derived` 含衰减 Grant；`approve_for_capability` → `derived` 含
  `Adjustment{scope: capability_default}`（自动 `statement = approval.reason`，ADR-0005 §D5 修订）；`defer(until)` 保持 `pending` 并更新 `expiry`。
- `on_expiry`：`deny` → `rejected`；`keep_waiting` → 保持 `pending`；`escalate` → 保持 `pending` 并按升级目标重新呈现（升级目标的声明方式继续开放）。
  **不存在 `expired` 结果**。
- **只有 `result = approved_once` 可以派生** `grant/issued` / `adjustment/applied`；由 Approval 派生的这两类事件的 `approval_id` 只可指向
  `result = approved_once` 的 Approval。
- 负例：`result = rejected` 的 Approval 派生出 `grant/issued` → 失败；账本中出现 `approved`、`denied`、`modified`、`expired` 作为 Approval 状态
  或结果 → 失败；同一 Approval 出现两条 `approval/decided` → 失败。

### D2 · `approval/asked` / `approval/decided` 配对；任一无法提交 → `unavailable.approval`

- 事件只有两个：**`approval/asked{approval_id, subject, subject_ref, policy_decision, expiry}`**（准入类）与
  **`approval/decided{approval_id, result, option?, params_digest?, decided_by, derived[]}`**。
- 每个 `approval/decided` 必有且仅有一个配对的 `approval/asked`（同 `approval_id`，`causation_id` 指向它）；每个 `approval/asked` 最终有且仅有
  一个 `approval/decided`（`pending` 是等待配对的状态，不是终态）。
- **fail-closed**：`fysund` 在创建 Approval 时若 `approval/asked` 无法落账（账本写入失败、审批面完全不可用且 Policy 未允许纯带外等待）→ 以
  `unavailable.approval` 拒绝整个请求（候选 `action-intent/rejected{reason: unavailable.approval}`），不创建"半个" Approval；
  决定到达时若 `approval/decided` 无法落账 → 决定不生效，Approval 仍 `pending`，向决定者返回错误；`fysund` 主动关闭时以 `result = unavailable`
  写出 `approval/decided`。
- `subject_ref` 见 D3。`policy_decision` 只在 `approval/asked` 上（它是准入决定）；`approval/decided` 不再携带。
- 负例：`approval/decided` 无配对 `approval/asked` → 失败；`approval/asked` 落账失败后候选仍 `effect/authorized` → 失败；
  `approval/asked` 缺 `policy_decision` → 失败。

### D3 · 请求不复制参数；引用 `action_intent_id` + `params_digest`

- `Approval.presentation` 与 `approval/asked` **不复制工具参数**。它们通过 **`subject_ref`** 引用被审对象：
  形状为 `subject_ref{action_intent_id?, params_digest?, object_ref?}`——`subject = effect_candidate` 时填 `action_intent_id` + `params_digest`；
  其他 subject（`grant_expansion` / `adjustment` / `handoff_accept` / `memory_admit` / `verification_override`）填 `object_ref` 指向对应对象。
- 审批面渲染时由 `fysund` 按 `subject_ref` 从账本 / Artifact 读取当前参数并附来源标签呈现；`untrusted_external` 参数值在呈现时按 Policy 可脱敏或折叠。
- `modify(params)` 后 `approval/decided.params_digest` 记改写后的 canonical digest；ActionIntent 记改写后的参数（ADR-0005 §D2 不变）。
- 负例：`approval/asked` 载荷含参数明文 → 失败；`presentation` 内联 Artifact 正文 → 失败（沿 ADR-0005 §D1）。

### D4 · 审批事实不进入 Runtime 会话账本的模型可见面

- Approval 的 `presentation`、`decided_by`、`reason`、`options` 不投递给 Runtime。Runtime 只收到
  `deliver_input(approval_result{approval_id, result, params_digest, grants_summary?})`（05 §2）或经守卫 `tools/call` 阻塞路径得到同样内容
  （[ADR-0007](0007-runtime-native-tools-and-conformance-scope.md) §D5）。
- 理由：决定者身份与理由进入模型上下文会成为新的注入面（"用户已批准所有后续动作"）；模型只需要知道结果与参数 digest 以继续。
- 负例：`deliver_input(approval_result)` 载荷含 `decided_by` 或 `reason` → 失败。

### D5 · 审批面只由 `fysund` 提供

- 有效的审批面（能产生 `approval/decided` 的入口）只有三种：`fysund` 静态服务的 **Web UI**、**CLI `fysun`**（`approval.decide` 的命令行形态）、
  **`fysund` 发起的 MCP `elicitation/create`**（应答绑定该 session 的 Principal，[06](../design/06-mcp-guard.md) §6）。
  三者最终都经 Capability API `approval.decide` / `approval.batch_decide` 落账；`deliver_input(approval_result)` 是 OS → Runtime 的投递，
  不是审批面。
- **Runtime 的文本输出永不构成审批面**：聊天通道中的"批准"、"同意"文本、模型生成的确认按钮、Runtime 自绘的审批卡片，都不能产生
  `approval/decided`（T11 审批面伪造）。
- **DSH**：`fysun-dsh` 的 `fysun-approval` 组件在 DSH `approval/request` waterfall 中**拦截、不 `next()`、全部转给 `fysund` 的 Approval 对象**；
  `fysund` 是唯一 answerer；DSH 自身的审批卡片失效（用户在卡片上的点击不落账、不解除阻塞），审批 UI 由 `fysund` 提供。
  此拦截点同时是 DSH 原生工具的 ActionIntent 持久化点，`fysun-dsh` 订阅 `tools/post-execute` 采集 receipt（蓝图 §I.2 H-3 选项 B；
  [ADR-0002](0002-protocol-first-runtime-boundary.md) §D5 修订、[08](../design/08-dsh-adapter.md) §2）。
- **前置条件**：`fysun` profile 必须禁止 DSH approval policy 为 `never`——DSH 的 `never` 在服务内部先于 waterfall 执行，会使拦截失效且原生工具
  调用无 ActionIntent。检测到 `never` 时 `fysun-dsh` 拒绝启动或写 `runtime/disqualified{reason: adapter_precondition_failed}`。
- Policy 可对特定 `Approval.subject` 强制**带外**（只接受 Web UI / CLI，不发起 elicitation）。
- 负例：DSH 原生审批卡片的应答落账为 `approval/decided` → 失败；聊天文本"批准"产生 `approval/decided` → 失败；DSH approval policy 为 `never`
  时 `fysun-dsh` 仍正常启动 → 失败；`fysun-approval` 对 `approval/request` 调用 `next()` → 失败。

### D6 · 审批轰炸：`approval_rate_limit` 与 `consecutive_rejections_alert`

- `Activity.constraints` 新增 **`approval_rate_limit`**（单位时间内该 Activity 可产生的 `approval/asked` 上限）与
  **`consecutive_rejections_alert`**（连续 `rejected` 次数阈值）。二者与 ADR-0005 §D3 的 `max_pending_approvals`、`approval_sla` 并列，
  缺省由 Policy 给出。
- 超过 `approval_rate_limit` → 新的 `approve` 类候选按 `denied.budget` 拒绝并留痕（与 `max_pending_approvals` 超限同处置），Activity 进入
  `input_required`。
- 连续 `rejected` 达到 `consecutive_rejections_alert` → Activity 进入 **`blocked{reason: consecutive_rejections}`**，需 Principal 显式
  `activity.resume`；理由：连续被拒表明 Runtime 当前状态与 Principal 意图持续偏离，继续运行只会继续消耗审批容量（T10）。
- 负例（`NV-T10-03`）：超过 `approval_rate_limit` 后仍出现新 `approval/asked` → 失败；连续拒绝达阈值后 Activity 未 `blocked` → 失败。

### D7 · `handoff_accept` 的触发条件

`subject = handoff_accept` 的 Approval 在以下任一条件下创建，否则 Handoff 直接 `submitted → working`：

- Handoff 目标为**外部 A2A Agent**且 Policy 要求对外委派需人工确认；
- 目标 Capability 的 `requirements`（含 `control_points`、Resource 类型、`native_effect_tools` 限制）**超出父 Grant 可衰减出的范围**——
  即子 Grant 无法在 F9 三条不等式内签发，需 Principal 决定是否以新 Activity 重新授权（沿 ADR-0004 §D4 "跨越父 Grant 的需求只能通过新 Activity"）。

负例：目标为本地 Capability、子 Grant 可在父 Grant 内签发、Policy 无要求，却产生 `handoff_accept` Approval → 失败（过度审批计入 T10）；
上述任一条件成立却无 Approval 直接 `handoff/working` → 失败。

### D8 · 恢复后重新武装

- 进程重启后，凡 Capability `contract.bounds.tool_effects[]` 含**非 `local_read`** 类别的 Activity，缺省恢复为
  **`paused{reason: rearm_required}`**（事件 `activity/paused{reason: rearm_required}`）；须由 Principal `activity.resume`（产生
  `activity/resumed`）或 Policy 预授权 **`auto_resume`** 才继续（自动恢复为 `waiting{reason: trigger}`，同样产生 `activity/resumed`）。
- 纯只读 Activity（全部工具为 `local_read`）直接恢复为 `waiting{reason: trigger}`。
- 武装状态由账本事件 `activity/resumed` 承载，**不是**进程内存标志——这是与 DSH goal "armed 不持久化、恢复后需人工 re-arm" 的实质差异：
  FysunOS 也要求重新武装，但武装事实入账本、可由 Policy 预授权、可审计。
- 恢复顺序（03 §6）中"重新武装"步骤位于 Approval 水位之后、`resume_execution` 之前。
- 负例：恢复后未经 `activity/resumed` 出现新的 `effect/authorized` → 失败（M3）；含 `external_mutation` 工具的 Activity 在无 `auto_resume`
  的情况下重启后直接 `waiting` → 失败。

### D9 · Execution `waiting` 带 `reason`

- Execution `waiting` 状态携带 **`reason: trigger | approval | input`**；事件 `execution/waiting{reason}`。
- `approval` = 等 Approval 决定（所属 Activity `input_required`）；`input` = 等 `deliver_input(message | handoff_result)`（所属 Activity
  `input_required`）；`trigger` = 等 Activity 触发（所属 Activity `waiting`）。
- 负例：Execution 因 `pending` Approval 等待却 `waiting{reason: trigger}` → 失败。

## 否决的替代方案

- **保留 `approved / denied / modified / expired` 开放状态集。**`modified` 与 `approved` 派生条件相同却是两个状态；`expired` 不是决定；
  五事件让同一 Approval 可能出现两条终态事件。封闭四值结果 + `options` 决定 `derived` 更简单且 fail-closed。
- **允许 Runtime 渲染审批面（DSH 卡片、模型生成按钮）。**审批面在非权威进程中即可被模型输出伪造（T11）；决定者身份无法解析为 Principal。
- **elicitation 应答不绑定 Principal。**MCP elicitation 由 client 应答，client 进程可能被模型驱动；不绑定 session Principal 等于让 Runtime
  批准自己。
- **重启后无条件自动继续（v0.6.0 表述）。**崩溃前的模型状态可能已被注入；含外部效果的 Activity 需要一次显式（人工或 Policy 预授权）的重新武装。
- **在 `approval/asked` 中复制参数以便审批面离线渲染。**参数明文进入账本与审批面是 T1 二次注入面；`subject_ref` + 渲染时读取即可。
- **只限并发不限速率。**被注入的 Runtime 可以串行轰炸；速率与连续拒绝是不同的信号。

## 后果

- Approval 状态集、事件、字段全部更新（[00 §3/§4/§5](../design/00-terminology.md)、[02 §2.11](../design/02-object-model.md)）；
  旧五事件从事件族删除。
- 错误码新增 `unavailable.approval`；`action-intent/rejected.reason` 取值增 `unavailable.approval`。
- `Activity.constraints` 增两项；Activity `blocked` 新增原因 `consecutive_rejections`；`activity/paused` 增 `reason`；Execution `waiting` 增
  `reason`；新增事件 `execution/waiting{reason}`。
- 度量新增"审批不可用率"（`result = unavailable` / `approval/asked`）；人工介入率的分子改为 `approval/asked`。
- `fysun-dsh` 的 `fysun-approval` 语义从"呈现转发"进一步收窄为"拦截、不 `next()`、全部转 `fysund`"；DSH 卡片失效；`fysun` profile 禁止
  `never`。M2b 验收增对应负例。
- 13 新增 T11 审批面伪造；T10 补审批轰炸机制。
- M3 验收增：`waiting.reason` 正确；恢复缺省 `paused{rearm_required}`；`NV-T10-03`；`handoff_accept` 按触发条件产生 Approval。

## 待后续 ADR 的问题

- `on_expiry = escalate` 的升级目标（另一 Principal、另一载体）如何声明（承 ADR-0005）。
- 多主体下 Approval 决定权分配（承 ADR-0005、ADR-0010）。
- `auto_resume` 的粒度：按 Activity、按 Capability 还是按 `effect_category` 集合预授权。

其余见 [12 MVP 计划 §7](../design/12-mvp-plan.md)。
