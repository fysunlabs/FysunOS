# 05 · Runtime ABI：最小生命周期 RPC、控制点与 A2A

- 状态：设计稿 v0.7.0（2026-09-08）
- 变更依据：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md)；ADR-0001 ~ ADR-0006（v0.7.0 修订）与 ADR-0007 ~ ADR-0012
- 前置：[02 对象模型](02-object-model.md)、[03 架构](03-architecture.md)、[04 Capability API](04-capability-api.md)、[00 术语表](00-terminology.md)
- 决策依据：[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md)（协议优先 Runtime 边界、三块结构）、[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md)（`describe()` 五字段、大声失败、投递路径）、[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)（来源标签、A2A Artifact 标签）、[ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md)（`approval_result` 载荷、审批面）、[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md)（`conflict.idempotency`）
- 来源：拆分自 v0.6.0 `04-abi.md` §2.2–2.7；§2.1（MCP 守卫）独立为 [06](06-mcp-guard.md)

Runtime ABI 是 Runtime ↔ OS 的公共接口。本文定义其中的最小生命周期 RPC、可选控制点、Runtime → OS 调用、不得调用清单、会话账本关联义务与
A2A 外部 Handoff 形态；MCP 守卫（所有 Runtime 必经的工具面）的规范在 [06](06-mcp-guard.md)；Runtime 的登记、资格化、MCP-only Runtime 的
投递路径与 hooks 阻断路径在 [07](07-runtime-onboarding.md)；DSH 参考适配器在 [08](08-dsh-adapter.md)。

## 1. 三块结构

| 块 | 谁必须实现 | 一句话 | 规范位置 |
|---|---|---|---|
| MCP 守卫 | 所有 Runtime 必经 | `fysund` 以 MCP 中间层身份位于 harness 与工具之间；工具面与工具调用在此受控；harness 不直接持有工具连接 | [06](06-mcp-guard.md) |
| 最小生命周期 RPC（§2） | 所有 Runtime 必须实现 | OS → Runtime 的描述、启动、恢复、取消、输入投递、检查点、健康 | 本文 |
| 可选控制点（§3） | 按 Runtime 声明 | `context_injection` / `checkpoint` / `fork` / `compaction_hook` / `session_link` / `subagent_delegation` | 本文 |

补充路径（不属于三块，不计入 conformance 主路径）：hooks.json PreToolUse 阻断路径（`fysun hook pre-tool-use`，[06 §9](06-mcp-guard.md)）；
守卫系统工具 `fysun.await_approval` / `fysun.pending_inputs`（[06 §7](06-mcp-guard.md)）。

速记说明：叙述中把只实现前两块的 Runtime 速记为 Level-1（MCP-only Runtime），把至少声明一个控制点的 Runtime 速记为 Level-2，
把附带 hooks.json 阻断路径的 MCP-only Runtime 速记为 Level-1b。三者只是文档速记，不是等级制度：账本记录的是
runtime/qualified 携带的控制点清单、原生工具清单与两个 digest，环境匹配依据控制点清单与原生工具清单，不依据 Level 标签；
速记不得出现在任何对象名、字段名、组件名、事件名、错误码、Cedar 实体与属性名中（禁令全文见 00 §1.3）。

线协议：最小生命周期 RPC 为语言无关的 **JSON-RPC over stdio**（MVP）；MCP 守卫跟随 MCP 规范（stdio + Streamable HTTP）；A2A 跟随 A2A 规范；
所有跨边界对象带 `digest`。最小 RPC 可由 Runtime 本体实现，也可由**薄包装进程**代为实现（MCP-only harness 的常态，[07 §3](07-runtime-onboarding.md)）；
资格化针对"Runtime + 包装进程"这一 Resource 整体，`describe_digest` 覆盖二者。

## 2. 最小生命周期 RPC（OS → Runtime，所有 Runtime 必须实现）

```text
runtime.describe()                                                                   -> {type, version, control_points[], native_effect_tools[], describe_digest}
runtime.start_execution(execution, context_view, agent_core_config, grants_summary)  -> session_id | not_runnable.environment{missing[]}
runtime.resume_execution(execution, checkpoint_ref, effect_watermark, approval_watermark) -> session_id | not_runnable.environment{missing[]}
runtime.cancel(execution, reason)
runtime.deliver_input(execution, message | event | handoff_result | approval_result)
runtime.checkpoint(execution)                                                        -> checkpoint_ref | unsupported
runtime.health()
```

### 2.1 `describe()`

- 返回形状固定为 **`describe() -> {type, version, control_points[], native_effect_tools[], describe_digest}`**，与 [00 §7](00-terminology.md)、
  [07 §1](07-runtime-onboarding.md)、[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md) §D1、[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D1 字面一致。
- `type` / `version`：Runtime 类型与版本（如 `dsh` / 某 revision）；进入 Policy context `runtime_type`。
- `control_points[]`：声明支持的可选控制点（§3）子集；可为空数组。
- **`native_effect_tools[]`**：Runtime **内置、不经 MCP 守卫**、可产生外部效果（文件写入、shell、网络、凭据使用等）的工具清单；
  每项至少含工具名与缺省 `effect_category`。只读的原生工具可以不列，但列出不算错。**字段必须存在**（空数组合法，缺字段不合法）。
  `fysun-dsh` 的清单不为空（DSH 原生 fs / shell 工具，[08 §2](08-dsh-adapter.md)）。声明责任在 Runtime 或其包装进程 / 适配器；
  **声明不实的 Runtime 不在承诺范围**（[01 §1.1](01-definition-and-principles.md)）。
- **`describe_digest`**：对 `{type, version, control_points[], native_effect_tools[]}` canonical 编码的 digest，是资格化纪元的基准：
  `runtime.qualify` 记录它；任何变化 → `runtime/disqualified{reason: describe_changed}`（[07 §1](07-runtime-onboarding.md)）。
- `describe()` 是纯函数：不依赖会话状态，多次调用结果一致；OS 在 `runtime.qualify` 与每次 `start_execution` 前调用并比对 digest。

### 2.2 `start_execution` —— 大声失败，不得降级运行

以下任一条件成立时，`start_execution` 必须返回 **`not_runnable.environment{missing[]}`** 并**不启动** Runtime 会话
（[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D6，借鉴 DeepSeek Harness 能力位预检）：

- Capability `requirements.control_points` ⊄ Runtime 已资格化的 `control_points[]`；
- Runtime 没有当前有效的 `runtime/qualified`（从未资格化，或已 `runtime/disqualified` 且未重新资格化）；重新 `describe()` 的
  `describe_digest` 与资格化记录不一致亦按此处理；
- Policy 在本 Scope 下拒绝该 Runtime 的 `native_effect_tools[]`（或 Capability `requirements` 声明拒绝其中某项）；
- 适配器前置条件不满足（如 `fysun-dsh` 检测到 DSH approval policy 为 `never`、bundle 的 `dsh_version` / `schema_digest` 不匹配）。

`missing[]` 列出未满足项（控制点名、被拒绝的原生工具名、失格原因、前置条件名）。预检失败即终止，不"尽力运行再看"，不以"部分支持"运行。

其余参数：

- `context_view`：Resource 子系统装配的受控上下文视图（authorize-before-rank、显式 `losses[]`、`digest`、每片段 `label`）；
  声明 `context_injection` 的 Runtime 直接渲染进模型输入；MCP-only Runtime 由薄包装进程在此刻渲染进 harness 启动输入，同时 `fysund` 经
  MCP `resources/read`（`fysun://context/<execution_id>`）与 `prompts/get` 暴露同一视图，conformance 声明"仅初始投递，不保证每步刷新"
  （[07 §3](07-runtime-onboarding.md)）。
- `agent_core_config`：决策核心的有效配置（Prompt 装配策略、Loop 策略参数、Skill 集），OS 只以摘要与 digest 记录。
- `grants_summary`：本 Execution 有效权限集的摘要（Grant ∩ Policy 允许集 ∩ 工具面），只读；实际工具面以守卫 `tools/list` 为准。

### 2.3 `resume_execution` —— 携带两个水位

- `effect_watermark`：截至检查点已对账的 Effect 水位；崩溃前 `dispatch-started` 但无 `receipt` 的 Effect 已由 OS 对账或 quarantine（[03 §6](03-architecture.md)）。
- **`approval_watermark`**：截至检查点已 `approval/decided` 的 Approval 集合摘要（含各自 `result`）。**崩溃前 `pending` 的 Approval 恢复后仍为
  `pending`**，不在水位内，不得被 Runtime 视为 `approved_once`；Runtime 据此知道哪些候选仍在等待，重新提交时命中 `needs_human(approval_id)`。
- 只有 OS 完成对账与**重新武装**（Activity 已 `activity/resumed`，[03 §6](03-architecture.md) 步骤 7）后才会调用；Runtime 收到本调用即可
  认为 Activity 已武装。失败条件与 2.2 相同（`not_runnable.environment`）。

### 2.4 `cancel`、`deliver_input`、`checkpoint`、`health`

- `cancel(execution, reason)`：Runtime 停止会话并尽力停止进行中的原生动作；已进入世界的效果不撤销，由 OS 对账。
- `deliver_input(execution, message | event | handoff_result | approval_result)`：
  - `approval_result` 载荷**限定为 `{approval_id, result, params_digest, grants_summary?}`**：`result ∈ approved_once | rejected | cancelled | unavailable`；
    `params_digest` 为放行参数的 canonical digest（`modify(params)` 后为改写后的 digest）；`grants_summary?` 为派生 Grant 的摘要。
    **不含** `decided_by`、`reason`、`presentation`、`options`——审批事实不进入 Runtime 会话账本的模型可见面
    （[ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md) §D4）。
  - `deliver_input(approval_result)` 是决定**送达** Runtime 的投递路径（OS → Runtime），不是审批面；对 MCP-only Runtime 的落地形态
    （MCP Tasks / 守卫阻塞 `tools/call` / `fysun.await_approval`）见 [07 §3](07-runtime-onboarding.md)。
  - `message | event | handoff_result` 对 MCP-only Runtime 在 turn 边界投递或经 `fysun.pending_inputs` 拉取，conformance 声明"尽力、下一 turn 边界"。
- `checkpoint(execution)`：对未声明 `checkpoint` 控制点的 Runtime 返回 **`unsupported`**；OS 此时以 Effect 水位与 Artifact 为恢复单位，
  不承诺 Runtime 内部状态恢复。声明了 `checkpoint` 却返回 `unsupported`，或未声明却返回非 `unsupported` 的成功，都是负例。
- `health()`：进程活性与会话计数；结果进入 `runtime/health`，不进入准入判定。

## 3. 可选控制点（能力位）

| 控制点 | 语义 | 义务 | 扩展调用 |
|---|---|---|---|
| `context_injection` | Runtime 在每步前接受 OS 装配的受控上下文视图（authorize-before-rank、显式损失、digest 绑定），并渲染进模型输入 | **每次注入必须在 Runtime 会话账本产生一条带 `context_view.digest` 的事件**，否则跨账本 replay 无法定位该步模型输入（M2b 验收）；候选参数由 Runtime 标注来源 `label` | `os.request_context`（Runtime → OS） |
| `checkpoint` | Runtime 能在 OS 请求时产出可恢复点，并在恢复时从该点续接 | `runtime.checkpoint` 返回 `checkpoint_ref`；恢复时从该点续接而不重放已对账 Effect | `runtime.checkpoint`；`os.report_checkpoint` |
| `fork` | Runtime 能在给定边界处 fork 会话，产生新的 session 供 OS 绑定新 Execution | 新 session 继承 `fencing_epoch` 规则，由 OS 分配新 Execution 与新 Grant | `runtime.fork_execution(execution, boundary) -> session_id` |
| `compaction_hook` | Runtime 在会话压缩前后向 OS 报告会话水位与摘要 digest，使压缩不破坏跨账本定位 | 压缩前后各一次报告 | `os.report_checkpoint` / `os.report_progress` |
| `session_link` | Runtime 履行 §6 的会话账本关联义务 | 会话账本记录 `execution_id`、`fencing_epoch`，每次 `report_checkpoint` 提交会话水位 | `os.report_checkpoint`（携带会话水位） |
| `subagent_delegation` | Runtime 内部的 subagent 派生转为 Handoff 与子 Execution，协作事实落 OS 账本 | **只有 Handoff 状态迁移与 `deliverables` 入治理账本**；roster / mailbox / task DAG 留在会话账本，经 `session_link` 关联；`deliverables[].summary` ≤ 4096 字节脱敏 | `os.request_handoff` |

- Capability `requirements` 可要求特定控制点；Capability 子系统据此做环境匹配；缺项在 `start_execution` 大声失败（§2.2）。
- 账本记 `runtime/qualified{resource_id, control_points[], native_effect_tools[], describe_digest, bundle_digest}`；资格化按 Runtime Resource
  独立进行，证据不可转移；`describe_digest` 或 `bundle_digest` 变化 → `runtime/disqualified`（[07 §1](07-runtime-onboarding.md)）。
- **对不可控制的内部机制，Runtime 必须如实声明**（不声明控制点即视为不可控；不列入 `native_effect_tools[]` 的原生效果工具即为声明不实），
  OS 不宣称全面管理。

## 4. Runtime → OS（控制点适配器的内部语义）

MCP-only Runtime 不直接调用本组接口：工具调用经 MCP 守卫隐式完成 `submit_candidate` 与 `observe_effect`（[06 §4](06-mcp-guard.md)）。
声明控制点的 Runtime 在其控制点适配器中显式调用：

```text
os.request_context(execution, need)               -> context_view (authorized, digest-bound, losses[], labels)
os.submit_candidate(execution, candidate)         -> admitted(effect_id) | rejected(reason) | needs_human(approval_id) | input_required(request_ref) | conflict(idempotency)
os.observe_effect(effect_id)                      -> state, receipt?
os.request_resource(execution, resource_request)  -> binding | denied
os.report_checkpoint(execution, checkpoint_ref, session_watermark)
os.report_progress(execution, summary)
os.request_handoff(execution, to, objective, inputs, expected) -> handoff_id | needs_human(approval_id)
os.report_failure(execution, error, recoverable)
os.propose_memory(execution, candidate)           -> queued for admission
os.propose_adjustment(execution, change, scope, predicted_effect?)   -> adjustment_id | needs_human(approval_id)
os.report_receipt(execution, effect_id, receipt)  -> acknowledged | rejected(reason)   （适配器拦截路径专用，见下）
```

- **候选（candidate）**是统一的提交单元：工具调用、外部动作、记忆写入、能力调整、委派请求
  都以候选进入 OS，由 OS 决定准入。候选参数携带 `label`，可选携带 `assurance` 与显式 `idempotency_key`。
  **工具调用在 ABI 层的定义就是 Effect 候选。**
- `os.submit_candidate` 的准入顺序与守卫 `tools/call` pre 相同（[06 §4](06-mcp-guard.md)）：类别合成 → Policy → Grant → 标签 → 持久化。
  返回 **`conflict(idempotency)`** 当且仅当候选显式携带的 `idempotency_key` 已存在且 canonical 参数与已记录者不同（错误码 `conflict.idempotency`，
  硬拒绝，不去重）；同键同参数返回已有 Effect 状态（`action-intent/replayed`）。`effect_category = local_read` 的候选走审计快路径，
  返回 `admitted` 但不产生 `effect_id`（只产生 `audit/observed`）。
- `os.request_handoff`：目标为外部 A2A Agent 且 Policy 要求，或目标 Capability `requirements` 超出父 Grant 可衰减范围时，返回
  `needs_human(approval_id)`（`subject = handoff_accept`，[02 §2.8](02-object-model.md)）。
- **`os.report_failure(error)` 脱敏规范**：`error.message` **≤ 4096 字节**，剥离栈迹、环境变量、宿主绝对路径；完整诊断以 Artifact 写入并经
  `diagnostics_ref` 引用；OS 据此填充 `Execution.error_report{message, diagnostics_ref}`（[02 §2.3](02-object-model.md)）。超长或含未剥离内容的
  `message` 由 OS 截断并记录违规，不拒绝失败报告本身（失败报告必须落账）。
- `os.propose_adjustment` 的 `predicted_effect` 可选；给出但缺 `statement` → 拒绝（[04 §1.4](04-capability-api.md)）。
- `os.propose_memory` 的候选携带来源 `label`；准入由 Policy 决定（`memory/admitted` / `memory/rejected`，或 `subject = memory_admit` 的 Approval），
  写入不是 Effect。
- **`os.report_receipt(execution, effect_id, receipt)`**（适配器拦截路径专用，[00 §7](00-terminology.md)、[08 §2](08-dsh-adapter.md)）：对经 `os.submit_candidate`
  准入、但工具体由 Runtime 自行执行（不经守卫转发）的 Effect 回填 receipt——如 `fysun-dsh` 在 `tools/post-execute` 采集 DSH 原生工具的结果。配套语义：`fysund`
  在返回 `admitted(effect_id)` 时即写 `effect/dispatch-started`（控制权交给 Runtime 执行工具体视为分发开始，带 `fencing_epoch`）；Execution 终止、适配器报错或
  Policy `receipt_timeout` 内无 receipt 到达 → `effect/outcome-unknown`，按适配器对账能力 reconcile 或 quarantine。`effect_id` 不属于本 Execution 或该 Effect
  不处于 `dispatch_started` → 返回 `rejected(reason)` 并留痕，不改变 Effect 状态。`local_read` 快路径候选（`admitted` 无 `effect_id`）不报 receipt，
  其 `audit/observed{path: adapter}` 已在拦截点产生。MCP 守卫路径的 Effect 由守卫 post 阶段自行回填 receipt，不经本调用。
- 上述调用的 Policy 评估 context 由 `fysund` 构造；Runtime 在候选中携带的任何 context 覆盖或标签声明都只是声明，`fysund` 以自己的记录为准
  （[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md) §D1）。

## 5. Runtime 不得调用（不存在于 ABI 中）

```text
mint_grant / commit_effect / complete_activity / write_ledger / mark_verified / decide_approval
```

- 这些操作在 MCP 守卫与最小 RPC 中均无对应方法；"Runtime 越权推进状态"（T9）的负例向量以此为断言面。未知方法一律拒绝并记录。
- **hooks 判定与 DSH 进程内 `guard/` 包判定对 OS 只是 Runtime 内部意见，不替代守卫**：`fysun hook pre-tool-use` 的 deny / abstain 由 harness
  执行，OS 只据 `fysund` 自己的 Policy 评估记账（[06 §9](06-mcp-guard.md)）；DSH `guard/` 包（loop 卫生插件）的判定不进入任何准入事件
  （[ADR-0012](../adr/0012-mcp-guard-self-implementation.md) §D3）。二者都不能使一次原生工具调用被视为"已受守卫"。
- Runtime 也不能提供或改写 Policy 评估的 context、改写已装配片段的 `label`、或自行决定 `effect_category`（[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md) 否决"由 Runtime 自报类别"）。

## 6. 会话账本关联义务（控制点 `session_link`）

Runtime 在自己的会话账本中记录 OS 提供的 `execution_id`、`fencing_epoch`，并在每次
`report_checkpoint` 时提交会话水位；OS 在治理账本中记录 `session_id`。任何一方都可以从
自己的账本出发定位另一方（见 [03 §4](03-architecture.md)）。声明 `context_injection` 的 Runtime 另在每次注入时写一条带 `context_view.digest`
的会话事件（§3）。

未声明 `session_link` 的 MCP-only Runtime：OS 只能以 MCP 守卫记录的**关联面 = `tools/call` 配对 + `audit/observed`** 作为关联面——
每个非快路径 `tools/call` 对应一个 Effect（`action-intent/recorded` → `effect/*`），每个 `local_read` 快路径调用对应一条
`audit/observed{execution_id, tool, params_digest, result_label, path: guard}`；跨账本 replay 只对 Effect 与工具结果配对，不对模型消息定位。
Runtime 自绘的审批呈现（若有）不属于关联面，视为无效（[03 §4](03-architecture.md)）。

## 7. A2A · 外部 Handoff 形态

- Handoff 目标为外部 Agent（非本 `fysund` 管理的 Execution）时，`fysund` 作为 **A2A client** 发起
  Task；`Handoff.context_id` 对应 A2A `contextId`，同一 Activity 下的多次委派共享。
- Handoff 状态直接映射 A2A TaskState（对照表见 [02 §2.8](02-object-model.md)，此处不重复）；`verified` 为 FysunOS 叠加态，
  由父 Execution 或独立 Verifier 产生，A2A 无对应；`completed` 不含验证语义。
- 外部 Task 返回的 Artifact 以内容寻址登记，**一律携带 `label.integrity = untrusted_external`**——不论该外部 Agent 的 Agent Card 是否有签名、
  是否被登记为 Resource；**升级只经 Verification**（`verdict = pass`）或 Memory 准入（[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md) §D1 修订）。
- 外部 Agent 不获得 Grant，只获得输入 Artifact（同样带 `label`）；子 Execution 才有衰减 Grant。
- `input_required` / `auth_required` 分别映射为 `deliver_input` 请求（Execution `waiting{reason: input}`）与 `subject = grant_expansion` 的 Approval。
- 委派给外部 Agent 是否需要 `subject = handoff_accept` 的 Approval 由 Policy 决定（[02 §2.8](02-object-model.md) 触发条件）。
- 对外暴露 A2A server（签名 Agent Card）属多主体场景，**非 MVP**（开放问题 Q5，[12 §7](12-mvp-plan.md)）。

## 8. 负例

- `describe()` 返回缺 `native_effect_tools` 字段（而非空数组）或缺 `describe_digest` → 不得 `runtime/qualified`（M2a）。
- 缺 required 控制点、未资格化、已 `runtime/disqualified` 或 Policy 拒绝其 `native_effect_tools[]` 的 Runtime 被 `start_execution` 成功启动 → 失败（M2a）。
- `describe_digest` 变化后账本无 `runtime/disqualified` 而 `start_execution` 成功 → 失败（M2a）。
- 未声明 `checkpoint` 的 Runtime 收到 `checkpoint()` 后返回非 `unsupported` 的成功 → 失败。
- `deliver_input(approval_result)` 载荷含 `decided_by` 或 `reason` → 失败。
- `resume_execution` 在 Activity 未 `activity/resumed` 前被调用 → 失败（M3）。
- 声明 `context_injection` 的 Runtime 某次注入后会话账本无带 `context_view.digest` 的事件 → 失败（M2b）。
- `subagent_delegation` 把 roster / mailbox / task DAG 写入治理账本 → 失败。
- `os.submit_candidate` 对同显式幂等键、不同 canonical 参数返回 `admitted` → 失败（M1）。
- `os.report_failure` 的 `message` 超过 4096 字节或含宿主绝对路径被原样落账 → 失败（M3）。
- Runtime 调用任一 §5 清单中的方法 → 必须被拒且 Execution 状态不变（`NV-T9-01`，M2a）；声明控制点的 Runtime 经控制点推进 Execution 状态 → 失败（`NV-T9-02`，M2b）。
- 外部 A2A Task 的 `TASK_STATE_COMPLETED` 直接导致 `handoff/verified` → 失败；外部 Artifact 的 `integrity` 高于 `untrusted_external` 且无 Verification / Memory 准入事件为因 → 失败。
