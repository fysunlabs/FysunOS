# 04 · 接口与 ABI

- 状态：设计稿 v0.6.0（2026-09-07）
- 变更依据：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) 与 ADR-0002 ~ ADR-0006
- 前置：[02 对象模型](02-object-model.md)、[03 架构](03-architecture.md)
- 决策依据：[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md)（协议优先 Runtime 边界）

FysunOS 对外有**两组**接口。它们约定**外部行为**，不统一内部思考步骤；只有需要机器执行或跨
组件交接的部分才必须结构化，其余可以保留为自然语言与可引用资源。

| 接口 | 方向 | 使用者 | 内容 |
|---|---|---|---|
| **Capability API** | 应用 ↔ OS | 产品、Agent Shell、SDK、CLI、UI | 按对象分组的操作面与管理面（§1） |
| **Runtime ABI** | Runtime ↔ OS | 任何 MCP 兼容 harness、`fysun-dsh`、边缘 Runtime | MCP 守卫（必经）/ 最小生命周期 RPC（必须实现）/ 可选控制点（能力位）（§2） |

有标准的部分直接用标准表达，FysunOS 不另定义：外部 Handoff 走 **A2A**（§2.7），实现包走
**Agent Skills**（§2.8）。v0.5.0 的第三组接口（组件图 ↔ OS）降为内部适配器生命周期（附录 A）。

线协议：

- MCP 守卫的传输跟随 MCP 规范（stdio + Streamable HTTP）；
- 最小生命周期 RPC 为语言无关的 JSON-RPC over stdio（MVP）；
- A2A 跟随 A2A 规范；
- 所有跨边界对象带 `digest`。

## 1. Capability API

面向应用与人机入口，按对象分组：

### 1.1 Capability

```text
capability.register(contract, implementation: skills_dir_ref | ..., requirements) -> capability_id
capability.revise(id, contract?, implementation?, requirements?)                 -> new version / derived id
capability.derive(from_id, reason, changes)                                       -> new capability_id (lineage)
capability.find(query: natural language | structured filter)                      -> [summary + evidence 摘要 + 最低 verifier 等级]
capability.describe(id)                                                           -> full record incl. evidence
capability.match(id, environment)                                                 -> runnable | adaptable(bindings) | not_runnable(missing[])
capability.health(id)                                                             -> 从 evidence 与 Adjustment 历史折叠的健康摘要
capability.withdraw(id)
```

- `implementation` 首选引用一个 Agent Skills 目录（`skills_dir_ref`），登记时 **digest 固定**；
  目录内 `SKILL.md` 的 `allowed-tools` 成为该 Capability 下 Grant 请求的**初始上界**（只能再收窄），
  `compatibility` 映射到 `requirements`（§2.8）。其他实现形态（代码包、工具组合、决策核心配置）
  仍允许，但同样以 digest 登记。
- `capability.find` 的每条结果附 evidence 摘要与该 Capability 契约要求的最低 verifier 等级
  （`verifier_kind` 的最低要求，如"至少 `deterministic` + `human`"），便于应用层选择。
- `capability.match` 的环境匹配依据 Runtime 声明的 `control_points[]`（§2.3）与实际 Resource；
  不设兼容等级，缺项**明确报告**。
- `capability.health` 折叠冗余（契约近似的其他 Capability）、过期（evidence 全部来自已退役
  Runtime 或已撤回 Memory）、反复失败的 checklist 项；只读，不改变 `status`。

### 1.2 Activity / Execution

```text
activity.create(capability_id, target, constraints, triggers?, policy_binding?) -> activity_id
activity.start(id) / pause(id) / resume(id) / cancel(id)
activity.trigger(id, event)
activity.observe(id)               -> state, current executions, blockers, pending approvals, budget, artifacts
activity.wait(id, condition, timeout)
execution.observe(id)              -> state, checkpoints, effects summary, handoffs, pending approvals
execution.cancel(id)
execution.fork(id, boundary?)      -> new execution_id
```

- `target` 创建后不可变；修订走 Adjustment。
- `constraints` 可含 `max_pending_approvals`、`approval_sla`；超限时 Activity 进入 `input_required`。
- `execution.fork` 要求绑定的 Runtime 声明了 `fork` 控制点，否则返回 `not_runnable.environment`。

### 1.3 Handoff / Artifact / Verification（观察面）

```text
handoff.list(execution_id) / handoff.observe(id)
artifact.get(ref) / artifact.put(bytes, media_type) -> ref
verification.report(id)
```

`verification.report(id)` 返回 02 §2.7 的扩展字段：`verifier_id`、`verifier_kind`、`independence`、
`rubric_ref`、`items[]`、`calibration_ref`、`verdict`、`assurance?`。报告不可变；`verdict = inconclusive`
不得被应用解释为通过。

### 1.4 Adjustment

```text
adjustment.apply(target, scope, change, effective_from?, predicted_effect?)  -> adjustment_id (auto risk_class)
adjustment.revert(id, reason)
adjustment.promote(id, to_scope)    -> e.g. this_activity -> capability_default
adjustment.history(target)          -> 含 evaluated_against 与 evidence_gate 结果
```

- `predicted_effect` 可选；给出后 `fysund` 在后续 Verification 到达时自动填充 `evaluated_against`。
- `promote` 在 `evidence_gate.enabled` 时要求回归集评估结果存在，否则返回 `denied.policy`。

### 1.5 Approval

```text
approval.list(filter)                          -> [approval summary]   （filter: activity / subject / state / batch_ref）
approval.observe(id)                           -> subject, reason, presentation, options, expiry, state
approval.decide(id, option, params?)           -> decision record      （option 取自 Approval.options；modify 需 params）
approval.batch_decide(batch_ref, option)       -> [decision record]
```

- 决定者**必须是 Principal**（用户或被授权的外部系统），不能是 Runtime；Runtime 侧只能观察到
  `needs_human(approval_id)` 并等待 `deliver_input(approval_result)`（§2.2）。
- `approve_for_activity` 派生衰减 Grant；`approve_for_capability` 派生 `scope = capability_default`
  的 Adjustment；两者均落账本事件（`grant/issued`、`adjustment/applied`），且带 `policy_decision`。
- 到期未决按 `on_expiry` 处理，缺省 `deny`。
- MCP elicitation 是 Approval 的传输载体之一（§2.1），最终决定仍经本组接口落账。

### 1.6 Resource / Grant / Policy / Memory（管理面）

```text
resource.register(descriptor) / resource.list(filter) / resource.health(id)
secret.import(source, name)                    -> secret_ref   （只经批准的非日志路径；永不返回明文）
grant.list(principal, scope) / grant.revoke(id)
policy.load(source)                            -> {policy_id, version, digest}
policy.describe()                              -> {version, digest, rules_summary}
policy.explain(event_ref)                      -> policy_decision
memory.list(filter)                            -> [memory summary]  （filter: scope / state / provenance）
memory.retract(id, reason)
```

- `policy.load` 产生 `policy/loaded` 或 `policy/revised`；版本只增不改；`source` 为 Cedar 子集文本或其引用。
- `policy.explain` 回答"这个准入事件当时为什么被放行/拒绝/要求审批"，返回该事件携带的
  `policy_decision{policy_version, rule_id, outcome, obligations[]}`。
- `memory.retract` 产生 `memory/retracted`；已装配进上下文的引用在下一次 `context/assembled` 时消失。

### 1.7 系统

```text
system.status() / system.events(filter, cursor) / system.replay_check()
```

## 2. Runtime ABI

Runtime ABI 由三块组成；前两块是任何 Runtime 的最小面，第三块按 Runtime 声明。

| 块 | 谁必须实现 | 一句话 |
|---|---|---|
| MCP 守卫（§2.1） | 所有 Runtime 必经 | `fysund` 以 MCP 中间层身份位于 harness 与工具之间；工具面与工具调用在此受控 |
| 最小生命周期 RPC（§2.2） | 所有 Runtime 必须实现 | OS → Runtime 的启动、恢复、取消、输入投递、检查点、健康 |
| 可选控制点（§2.3） | 按 Runtime 声明 | `context_injection` / `checkpoint` / `fork` / `compaction_hook` / `session_link` / `subagent_delegation` |

**Level-1 / Level-2 只是文档速记，不是等级制度**：Level-1 = MCP 客户端 + 最小生命周期 RPC；
Level-2 = Level-1 + 至少一个控制点。账本记录的是 `runtime/qualified{control_points[]}`，环境匹配
依据 `control_points[]`，不依据 Level 标签。

### 2.1 MCP 守卫（所有 Runtime 必经）

`fysund`（组件 `fysun-mcp-guard`）以 MCP 中间层身份位于 harness 与工具之间。harness 把 `fysund`
配置为**唯一** MCP server，或把 `fysund` 前置在其配置的所有 MCP server 之前；工具 server 由
`fysund` 代为连接，harness 不直接持有工具连接。

| MCP 方法 | 守卫行为 | 对应 OS 语义 |
|---|---|---|
| `tools/list` | 按当前 Execution 的 Grant 与 Policy **过滤并改写**返回的工具面；被过滤的工具对模型不可见 | F9 的机械化：看不见就调不了；工具面 = 有效权限集的一个因子 |
| `tools/call` pre | `submit_candidate`：ActionIntent 持久化（幂等键 = canonical 参数 digest）→ Policy 评估 → Grant 检查 → 来源标签检查 | `action-intent/recorded` 或 `action-intent/rejected`；`effect/authorized` |
| `tools/call` 执行 | Effect dispatch：在 `fencing_epoch` 下转发给实际工具 server | `effect/dispatch-started` |
| `tools/call` post | Receipt 回填；结果作为工具结果返回 harness，并携带 `label` | `effect/receipt`；无结果则 `effect/outcome-unknown` |
| `tasks/*` | 长任务与 outcome-unknown 的标准表达：`tasks/get` 读取 Effect 状态；`tasks/result` 对应 `effect/reconciled`；`tasks/cancel` 不撤销已进入世界的效果，只使 Effect 进入对账 | Effect 状态机 |
| `elicitation/create` | Approval 的传输载体之一：`fysund` 作为 server 向 harness 发起 elicitation，呈现 `Approval.presentation` 与 `options`；应答归属于该 session 绑定的 Principal | `approval/requested` → `approval/granted` 等 |

规则：

- **只读工具走审计快路径**：仍产生描述符、Scope 与审计事件，不持久化 ActionIntent/Effect。
  是否只读由 Policy 判定；工具 server 自报的 annotation（如 `readOnlyHint`）只是输入，不是依据。
- 来源标签：Level-2 Runtime 在候选参数上标注 `label`；Level-1 由 OS 按参数值匹配上下文片段与
  工具结果**尽力标注**，无法匹配的参数值记为 `untrusted_external`（fail-closed）。Policy 可引用标签
  （如 `untrusted_external` 来源的收件人参数 → `approve`）。
- 守卫结果语义 `Pass / Reject / Mutated` 与 `policy_decision.outcome` 对应：`allow` → Pass，
  `deny` → Reject（返回 §3 的拒绝码），`modify` → Mutated（改写后的参数进入 ActionIntent），
  `approve` → 若 harness 支持 elicitation 则发起 `elicitation/create`，否则返回 `needs_human(approval_id)`
  并由 Runtime 等待 `deliver_input(approval_result)`。Policy 可对特定 `Approval.subject` 要求只接受
  带外载体（Web UI / CLI 的 `approval.decide`），此时不发起 elicitation。
- agentgateway 复用形态待评估（作为库嵌入 / 独立进程前置 / 仅借用语义自实现，见
  [08 开放问题 10](08-mvp-plan.md#7-开放问题待后续-adr)）；无论哪种形态，采用相同的
  Pass / Reject / Mutated 语义与 Cedar 策略格式。

### 2.2 最小生命周期 RPC（OS → Runtime，所有 Runtime 必须实现）

```text
runtime.describe()                                                                   -> {type, version, control_points[]}
runtime.start_execution(execution, context_view, agent_core_config, grants_summary)  -> session_id
runtime.resume_execution(execution, checkpoint_ref, effect_watermark, approval_watermark) -> session_id
runtime.cancel(execution, reason)
runtime.deliver_input(execution, message | event | handoff_result | approval_result)
runtime.checkpoint(execution)                                                        -> checkpoint_ref | unsupported
runtime.health()
```

- `resume_execution` 同时携带 Effect 水位与 **Approval 水位**：崩溃前未决的 Approval 在恢复后
  不得被 Runtime 视为已批准。
- `deliver_input(approval_result)` 是 Approval 决定到达 Runtime 的唯一路径；结果只含决定与派生
  Grant 摘要，不含决定者的凭据。
- `checkpoint()` 对未声明 `checkpoint` 控制点的 Runtime 返回 `unsupported`；OS 此时以 Effect 水位
  与 Artifact 为恢复单位，不承诺 Runtime 内部状态恢复。
- 最小 RPC 可由 Runtime 本体实现，也可由薄包装进程代为实现（见 [05 §6](05-runtime-and-dsh-integration.md)）。

### 2.3 可选控制点（能力位）

| 控制点 | 语义 | 扩展调用 |
|---|---|---|
| `context_injection` | Runtime 在每步前接受 OS 装配的受控上下文视图（authorize-before-rank、显式损失、digest 绑定），并渲染进模型输入 | `os.request_context`（Runtime → OS） |
| `checkpoint` | Runtime 能在 OS 请求时产出可恢复点，并在恢复时从该点续接 | `runtime.checkpoint` 返回 `checkpoint_ref`；`os.report_checkpoint` |
| `fork` | Runtime 能在给定边界处 fork 会话，产生新的 session 供 OS 绑定新 Execution | `runtime.fork_execution(execution, boundary) -> session_id` |
| `compaction_hook` | Runtime 在会话压缩前后向 OS 报告会话水位与摘要 digest，使压缩不破坏跨账本定位 | `os.report_checkpoint` / `os.report_progress` |
| `session_link` | Runtime 履行 §2.6 的会话账本关联义务 | `os.report_checkpoint`（携带会话水位） |
| `subagent_delegation` | Runtime 内部的 subagent 派生转为 Handoff 与子 Execution，协作事实落 OS 账本 | `os.request_handoff` |

- Capability.requirements 可要求特定控制点；Capability 子系统据此做环境匹配。
- 账本记 `runtime/qualified{control_points[]}`；资格化按 Runtime 独立进行，证据不可转移。
- **对不可控制的内部机制，Runtime 必须如实声明**（不声明控制点即视为不可控），OS 不宣称全面管理。

### 2.4 Runtime → OS（Level-2 控制点的内部语义）

Level-1 Runtime 不直接调用本组接口：工具调用经 MCP 守卫隐式完成 `submit_candidate` 与 `observe_effect`。
Level-2 Runtime 在其控制点适配器中显式调用：

```text
os.request_context(execution, need)               -> context_view (authorized, digest-bound, losses[], labels)
os.submit_candidate(execution, candidate)         -> admitted(effect_id) | rejected(reason) | needs_human(approval_id) | input_required(request_ref)
os.observe_effect(effect_id)                      -> state, receipt?
os.request_resource(execution, resource_request)  -> binding | denied
os.report_checkpoint(execution, checkpoint_ref, session_watermark)
os.report_progress(execution, summary)
os.request_handoff(execution, to, objective, inputs, expected) -> handoff_id
os.report_failure(execution, error, recoverable)
os.propose_memory(execution, candidate)           -> queued for admission
os.propose_adjustment(execution, change, scope)   -> adjustment_id | needs_human(approval_id)
```

**候选（candidate）**是统一的提交单元：工具调用、外部动作、记忆写入、能力调整、委派请求
都以候选进入 OS，由 OS 决定准入。候选参数携带 `label`，可选携带 `assurance`。
**工具调用在 ABI 层的定义就是 Effect 候选。**

### 2.5 Runtime 不得调用（不存在于 ABI 中）

```text
mint_grant / commit_effect / complete_activity / write_ledger / mark_verified / decide_approval
```

这些操作在 MCP 守卫与最小 RPC 中均无对应方法；"Runtime 越权推进状态"的负例向量以此为断言面。

### 2.6 会话账本关联义务（控制点 `session_link`）

Runtime 在自己的会话账本中记录 OS 提供的 `execution_id`、`fencing_epoch`，并在每次
`report_checkpoint` 时提交会话水位；OS 在治理账本中记录 `session_id`。任何一方都可以从
自己的账本出发定位另一方（见 [03 §4](03-architecture.md#4-双账本)）。

未声明 `session_link` 的 Level-1 Runtime：OS 只能以 MCP 守卫记录的 `tools/call` 序列作为关联面；
跨账本 replay 只对 Effect 与工具结果配对，不对模型消息定位。

### 2.7 A2A · 外部 Handoff 形态

- Handoff 目标为外部 Agent（非本 `fysund` 管理的 Execution）时，`fysund` 作为 **A2A client** 发起
  Task；`Handoff.context_id` 对应 A2A `contextId`，同一 Activity 下的多次委派共享。
- Handoff 状态直接映射 A2A TaskState（对照表见 02 §2.8，此处不重复）；`verified` 为 FysunOS 叠加态，
  由父 Execution 或独立 Verifier 产生，A2A 无对应。
- 外部 Task 返回的 Artifact 以内容寻址登记，携带 `label.integrity = untrusted_external`，除非该
  外部 Agent 的 Agent Card 签名被 Policy 列为可信来源。
- `input_required` / `auth_required` 分别映射为 `deliver_input` 请求与 `subject = grant_expansion` 的 Approval。
- 对外暴露 A2A server（签名 Agent Card）属多主体场景，**非 MVP**。

### 2.8 Agent Skills · 实现包

FysunOS **不定义**自己的实现包格式。`Capability.implementation` 引用一个 Skills 目录，digest 固定。

| `SKILL.md` frontmatter | FysunOS 映射 |
|---|---|
| `name` / `description` | Capability 摘要的一部分；不替代 `contract.behavior` |
| `license` | 登记进 Capability 记录；与供应链审查关联 |
| `compatibility` | 映射到 `requirements`（Runtime 类型、控制点、Resource 类型） |
| `metadata` | 原样登记，OS 不解释 |
| `allowed-tools` | 该 Capability 下 Grant 请求的**初始上界**；`tools/list` 过滤时与 Grant ∩ Policy 再交 |

渐进披露三层（元数据 → 正文 → 按需资源）与上下文预算模型兼容：元数据层进入 `capability.find`，
正文与资源经上下文装配按预算与 Grant 进入 context_view，并携带 `label`。

## 3. 错误与拒绝语义

所有接口的拒绝都带稳定错误码与可判定原因：

| 错误码 | 可判定说明 |
|---|---|
| `denied.policy` | Policy 评估结果为 `deny`；附 `policy_decision` |
| `denied.grant` | 无覆盖该动作的有效 Grant，或 Grant 已撤销 |
| `denied.budget` | Grant 预算或 Activity 约束耗尽；附耗尽的预算维度 |
| `denied.fencing` | 写入携带的 `fencing_epoch` 陈旧 |
| `not_runnable.environment` | 环境不满足 `requirements`；附缺失清单（含缺失的控制点） |
| `needs_human(approval_id)` | Policy 结果为 `approve`，`fysund` 已创建 Approval；等待 `deliver_input(approval_result)` |
| `input_required(request_ref)` | Execution 或 Handoff 需要父方或用户提供输入（非授权问题）；对应 A2A `INPUT_REQUIRED`、MCP elicitation |
| `auth_required(approval_id)` | 需要额外授权；已创建 `subject = grant_expansion` 的 Approval；对应 A2A `AUTH_REQUIRED` |
| `unknown_outcome` | Effect 结果未知，进入对账；不得假定成功 |
| `inconclusive` | Verification 无法判定；不是通过 |

拒绝不是失败测试；`not-run` 永不等于通过。

## 附录 A · 内部适配器生命周期（原 Composition ABI，ADR-0002 降级）

原 Composition ABI 抽取自 Cordis 的可撤销效果与反应式依赖语义，解决的是进程内动态组合；
FysunOS 的边界是进程外、权威与非权威之间。它从公共接口降为：

- (a) `fysund` 内部适配器 trait；
- (b) `fysun-dsh` 适配器内部对 Cordis 生命周期的映射；
- (c) 账本只保留 `resource/registered | bound | unavailable | quarantined` 事件。

内部 trait 参考（不是公共接口，不承诺跨版本稳定）：

```text
mount(component, scope, dependencies, config)   -> revision
unmount(component, revision)
replace(component, new_revision)
inspect(scope)                                  -> component graph
dependency_status(component)                    -> satisfied | waiting(missing[]) | broken
health(component)
dispose(component)
quarantine(component, reason)
```

后端：Rust 原生模块生命周期；DSH 适配器内的 Cordis 映射。WASM 后端无需求，不预留。

保留为 OS 语义的两条规则（由 Resource 生命周期事件与 Grant 撤销规则表达）：

- 组件消失（例如 Cordis service 卸载）**不得绕过** OS 的 Grant 撤销与 Effect 对账；
- 组件图是依赖图，不是信任/委派图。
