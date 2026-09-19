# 04 · Capability API 与错误语义

- 状态：设计稿 v0.7.0（2026-09-08）
- 变更依据：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md)；ADR-0001 ~ ADR-0006（v0.7.0 修订）与 ADR-0007 ~ ADR-0012
- 前置：[02 对象模型](02-object-model.md)、[03 架构](03-architecture.md)、[00 术语表](00-terminology.md)
- 决策依据：[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md)（协议优先 Runtime 边界；两组公共接口）、[ADR-0003](../adr/0003-policy-object-and-language.md)（Policy API）、[ADR-0005](../adr/0005-approval-and-evidence-gated-evolution.md) 与 [ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md)（Approval API）、[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md)（Runtime 资格化 API）、[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md)（`tool_effects[]` 校验、`conflict.idempotency`）
- 来源：拆分自 v0.6.0 `04-abi.md` §1（Capability API）、§2.8（Agent Skills）、§3（错误与拒绝语义）

## 0. 接口总览

FysunOS 对外有**两组**接口。它们约定**外部行为**，不统一内部思考步骤；只有需要机器执行或跨
组件交接的部分才必须结构化，其余可以保留为自然语言与可引用资源。

| 接口 | 方向 | 使用者 | 内容 | 规范位置 |
|---|---|---|---|---|
| **Capability API** | 应用 ↔ OS | 产品、Agent Shell、SDK、CLI、UI | 按对象分组的操作面与管理面 | 本文 §1 |
| **Runtime ABI** | Runtime ↔ OS | 任何 MCP 兼容 harness、`fysun-dsh`、边缘 Runtime | MCP 守卫（必经）/ 最小生命周期 RPC（必须实现）/ 可选控制点（能力位） | [05 Runtime ABI](05-runtime-abi.md)（生命周期 RPC、控制点、`os.*`、A2A）；[06 MCP 守卫规范](06-mcp-guard.md)（工具面与工具调用） |

有标准的部分直接用标准表达，FysunOS 不另定义：外部 Handoff 走 **A2A**（[05 §7](05-runtime-abi.md)），实现包走
**Agent Skills**（本文 §2），工具面走 **MCP**（[06](06-mcp-guard.md)）。v0.5.0 的第三组接口（组件图 ↔ OS）已降为内部适配器生命周期，
文档位置为 [08 附录 A](08-dsh-adapter.md)。

线协议：

- Capability API 由 `fysund` 以本地 RPC 暴露（CLI `fysun`、Web UI、SDK 都是它的客户端；传输形态为开放问题 Q1，[12 §7](12-mvp-plan.md)）；
- MCP 守卫的传输跟随 MCP 规范（stdio + Streamable HTTP）；
- 最小生命周期 RPC 为语言无关的 JSON-RPC over stdio（MVP）；
- A2A 跟随 A2A 规范；
- 所有跨边界对象带 `digest`。

Capability API 的调用者必须解析为 Principal；Runtime 进程不是 Capability API 的调用者（它只经 Runtime ABI 与 OS 交互），
这是 F1 与 T9 的断言面之一。错误码全表见 §3。方法索引（累积）另见 [00 §10](00-terminology.md)。

## 1. Capability API

面向应用与人机入口，按对象分组：

### 1.1 Capability

```text
capability.register(contract, implementation: skills_dir_ref | ..., requirements) -> capability_id
capability.revise(id, contract?, implementation?, requirements?)                 -> new version / derived id
capability.derive(from_id, reason, changes)                                       -> new capability_id (lineage)
capability.find(query: natural language | structured filter)                      -> [summary + evidence 摘要 + 最低 verifier 等级]
capability.describe(id)                                                           -> full record incl. evidence, tool_effects[], health
capability.match(id, environment)                                                 -> runnable | adaptable(bindings) | not_runnable(missing[])
capability.health(id)                                                             -> 从 evidence 与 Adjustment 历史折叠的健康摘要（含"未分类工具"）
capability.withdraw(id)
```

- `implementation` 首选引用一个 Agent Skills 目录（`skills_dir_ref`），登记时 **digest 固定**；
  目录内 `SKILL.md` 的 `allowed-tools` 成为该 Capability 下 Grant 请求的**初始上界**（只能再收窄），
  `compatibility` 映射到 `requirements`（§2）。其他实现形态（代码包、工具组合、决策核心配置）
  仍允许，但同样以 digest 登记。
- **`contract.bounds.tool_effects[]` 必填校验**（[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md) §D2）：
  `capability.register` 与 `capability.revise` 检查 `contract.bounds.tool_effects[]: [{tool, effect_category}]` 存在且 `effect_category`
  取 [00 §5](00-terminology.md) 六值之一；**缺整个字段 → 拒绝**（`denied.policy`，附缺失定位）。`tool_effects[]` 应覆盖 `allowed-tools`
  中每个工具；未覆盖的工具**不阻止注册**，运行时按 `external_mutation` 处理，并由 `capability.health` 标为"未分类工具"信号
  ——这是契约缺陷信号，不是运行时错误。Capability 单方不能把 Resource 目录已登记为更高严重度的工具声明为更低类别（合成规则见 [06 §3](06-mcp-guard.md)）。
- `requirements` 可声明 Runtime 控制点（`control_points[]`）、Resource 类型与条件、环境要求，以及**拒绝含特定 `native_effect_tools` 的 Runtime**
  （例如"不接受含无沙箱 shell 的 Runtime"）。
- `capability.find` 的每条结果附 evidence 摘要与该 Capability 契约要求的最低 verifier 等级
  （`verifier_kind` 的最低要求，如"至少 `deterministic` + `human`"），便于应用层选择。
- `capability.match` 的环境匹配依据 Runtime 已资格化的 `control_points[]`、如实声明的 `native_effect_tools[]`（[05 §3](05-runtime-abi.md)、
  [07 §1](07-runtime-onboarding.md)）与实际 Resource；未资格化或已 `runtime/disqualified` 的 Runtime 不参与匹配；
  不设兼容等级，缺项**明确报告**于 `not_runnable(missing[])`。
- `capability.health` 折叠冗余（契约近似的其他 Capability）、过期（evidence 全部来自已退役
  Runtime、已 `runtime/disqualified` 且未重新资格化的 Runtime，或已撤回 Memory）、反复失败的 checklist 项、**未分类工具**；只读，不改变 `status`。

### 1.2 Activity / Execution

```text
activity.create(capability_id, target, constraints, triggers?, policy_binding?) -> activity_id
activity.start(id) / activity.pause(id) / activity.resume(id) / activity.cancel(id)
activity.trigger(id, event)
activity.observe(id)               -> state (paused.reason / blocked reason), current executions, blockers, pending approvals, budget, artifacts
activity.wait(id, condition, timeout)
execution.observe(id)              -> state (含 waiting.reason), checkpoints, effects summary, handoffs, pending approvals, runtime_binding
execution.cancel(id)
execution.fork(id, boundary?)      -> new execution_id
```

- `target` 创建后不可变；修订走 Adjustment。
- `constraints` 可含 `max_pending_approvals`、`approval_sla`、`approval_rate_limit`、`consecutive_rejections_alert`；前两项与第三项超限时 Activity
  进入 `input_required`，连续拒绝达阈值时进入 `blocked{reason: consecutive_rejections}`（[02 §2.2](02-object-model.md)）。
- `activity.pause` 产生 `activity/paused{reason: user}`。
- **`activity.resume`**：解除 `paused`——包括 `paused{reason: user}` 与崩溃恢复后的 **`paused{reason: rearm_required}`**（重新武装，
  [03 §6](03-architecture.md)）——产生 `activity/resumed`，Activity 恢复为 `waiting{reason: trigger}` 或继续现有 Execution；也用于解除
  `blocked{reason: consecutive_rejections}`。调用者必须是 Principal；Policy `auto_resume` 允许的 Activity 由 `fysund` 自动产生同一事件，
  不需要本调用。
- `activity.observe` 返回 `paused.reason`（`user` / `rearm_required` / …）与 `blocked` 的原因及恢复动作，使 UI 能区分"用户暂停"、
  "等待重新武装"、"连续拒绝阻塞"、"离线撤销阻塞"。
- **`execution.observe` 返回 `waiting.reason`**（`trigger` / `approval` / `input`），以及 `runtime_binding`（含 `qualification_ref`、
  `native_effect_tools[]`）与 `error_report`（`failed` 时，≤ 4096 字节脱敏 `message` + `diagnostics_ref`）。
- `execution.fork` 要求绑定的 Runtime 声明并已资格化 `fork` 控制点，否则返回 `not_runnable.environment{missing: [fork]}`。

### 1.3 Handoff / Artifact / Verification（观察面）

```text
handoff.list(execution_id) / handoff.observe(id)
artifact.get(ref) / artifact.put(bytes, media_type) -> ref
verification.report(id)
```

- `handoff.observe(id)` 返回 Handoff 状态（对齐 A2A TaskState，含 `working.phase`）与 `deliverables[]`（Artifact 引用、Verification 引用、
  ≤ 4096 字节脱敏 `summary`）；roster / mailbox / task DAG 不在返回中（它们属 Runtime 会话账本，[02 §2.8](02-object-model.md)）。
- `artifact.put` 写入内容寻址产物并赋予 `label`（调用者为 Principal 时 `integrity = trusted_user`；应用代模型写入时须声明来源，无法归因则
  `untrusted_external`）；`artifact.get` 返回内容与 `label`。
- `verification.report(id)` 返回 [02 §2.7](02-object-model.md) 的扩展字段：`verifier_id`、`verifier_kind`、`independence`、
  `rubric_ref`、`items[]`、`calibration_ref`、`verdict`、`assurance?`。报告不可变；`verdict = inconclusive`
  不得被应用解释为通过。

### 1.4 Adjustment

```text
adjustment.apply(target, scope, change, effective_from?, predicted_effect?)  -> adjustment_id (auto risk_class)
adjustment.revert(id, reason)
adjustment.promote(id, to_scope)    -> e.g. this_activity -> capability_default
adjustment.history(target)          -> 含 predicted_effect、evaluated_against 与 evidence_gate 结果
```

- **`predicted_effect` 可选**：为空时允许，此时 `evaluated_against.result` 恒为 `inconclusive`；给出后 `fysund` 在后续 Verification
  到达时自动填充 `evaluated_against`；**给出但缺 `statement` → 拒绝**（`denied.policy`，附定位）。由 `approve_for_capability` 派生的 Adjustment
  自动 `statement = approval.reason`（[ADR-0005](../adr/0005-approval-and-evidence-gated-evolution.md) §D5 修订）。
- `risk_class` 由 `fysund` 判定；触及 Grant 扩权、新外部动作、预算上调、设备安全时走 Policy 检查，可产生 `subject = adjustment` 的 Approval
  （返回 `needs_human(approval_id)`）。
- `promote` 在 `evidence_gate.enabled` 时要求回归集评估结果存在且非 `fail`，否则返回 `denied.policy`。
- `adjustment.revert` 产生 `adjustment/reverted{reason}`，用于统计回滚率。

### 1.5 Approval

```text
approval.list(filter)                          -> [approval summary]   （filter: activity / subject / state / batch_ref）
approval.observe(id)                           -> subject, subject_ref, reason, presentation, options, expiry, on_expiry, state, result?, derived[]
approval.decide(id, option, params?)           -> {approval_id, result, option, params_digest?, derived[]}   （option 取自 Approval.options；modify 需 params）
approval.batch_decide(batch_ref, option)       -> [{approval_id, result, …}]
```

- 决定者**必须是 Principal**（用户或被授权的外部系统），不能是 Runtime；Runtime 侧只能观察到
  `needs_human(approval_id)` / `auth_required(approval_id)` 并等待 `approval_result`（[05 §2](05-runtime-abi.md)、[07 §3](07-runtime-onboarding.md)）。
- **`approval.decide` 返回 `result`**：`approved_once` / `rejected` / `cancelled` / `unavailable`（封闭集，[ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md) §D1）。
  `option` 决定 `derived` 与 `params_digest`，不决定 `result`：`approve_once` / `approve_for_activity` / `approve_for_capability` / `modify(params)`
  的结果一律 `approved_once`；`deny` → `rejected`；`defer(until)` 保持 `pending` 并更新 `expiry`（返回 `state: pending`，无 `result`）。
- `approve_for_activity` 派生衰减 Grant；`approve_for_capability` 派生 `scope = capability_default` 的 Adjustment（`statement = approval.reason`）；
  两者均落账本事件（`grant/issued`、`adjustment/applied`，带 `approval_id` 与 `policy_decision`），且**只有 `result = approved_once` 可派生**。
- `modify(params)` 改写候选参数后放行；`approval/decided.params_digest` 记改写后的 canonical digest，ActionIntent 记改写后的参数。
- 到期未决按 `on_expiry` 处理，缺省 `deny` → `rejected`；`keep_waiting` / `escalate` 保持 `pending`。不存在 `expired` 结果。
- **决定只能经本组接口或 `fysund` 发起的 elicitation 落账**：Web UI 与 CLI `fysun` 是 `approval.decide` / `approval.batch_decide` 的呈现形态；
  `fysund` 发起的 MCP `elicitation/create` 应答绑定该 session 的 Principal 后同样经本组接口落账（[06 §6](06-mcp-guard.md)）。
  Runtime 的文本输出、模型生成的确认控件、Runtime 自绘的审批卡片（含 DSH 原生卡片）都不产生 `approval/decided`（T11，[13](13-threat-model.md)）。
- `approval.observe` 的 `presentation` 由 `fysund` 按 `subject_ref` 从账本 / Artifact 读取当前参数并附来源标签渲染；`untrusted_external`
  参数值按 Policy 可脱敏或折叠；Approval 对象本身不复制工具参数。
- fail-closed：`approval/asked` 与 `approval/decided` 必须配对；`approval.decide` 落账失败时决定不生效、Approval 仍 `pending`、向调用者返回错误。

### 1.6 Resource / Grant / Policy / Memory / Runtime（管理面）

```text
resource.register(descriptor) / resource.list(filter) / resource.health(id)
secret.import(source, name)                    -> secret_ref   （只经批准的非日志路径；永不返回明文）
grant.list(principal, scope) / grant.revoke(id)
policy.load(source)                            -> {policy_id, version, digest}
policy.describe(policy_id?)                    -> [{policy_id, version, digest, rules_summary}] | {policy_id, version, digest, rules_summary}
policy.explain(event_ref)                      -> policy_decision
memory.list(filter)                            -> [memory summary]  （filter: scope / state / provenance）
memory.retract(id, reason)
runtime.list()                                 -> [{resource_id, type, version, source, bundle_digest, describe_digest, control_points[], native_effect_tools[], qualification}]
runtime.qualify(resource_id)                   -> qualified{control_points[], native_effect_tools[]} | rejected(reason)
```

- `resource.register(descriptor)`：登记 MCP server 时可为其工具登记**缺省 `effect_category`**（类别来源 (b)；MCP tool annotations 只作建议输入），
  下游凭据以 `SecretRef` 登记（[06 §8](06-mcp-guard.md)）；登记已安装 Runtime 时记录类型、版本、来源、`bundle_digest`（[07 §1](07-runtime-onboarding.md)）。
  `resource/registered` 不隐含任何 Grant。
- `secret.import` 遵守 **SecretRef 三规则**：每次操作重解析（不缓存明文）、描述面无值槽（`resource.describe` / `tools/list` 中不存在可放明文的字段）、
  空值即未配置（不以空串或占位符表示）。
- `grant.revoke` 产生 `grant/revoked`，立即使后续 ActionIntent 按 `denied.grant` 拒绝；撤销 escrow 父 Grant 时对离线边缘的后果见 [11 §3](11-edge-and-offline-ledger.md)。
- `policy.load` 产生 `policy/loaded` 或 `policy/revised`；版本只增不改；`source` 为 Cedar 子集文本或其引用；解析后以 Cedar schema 校验，
  引用未知实体 / 属性、使用子集外语法 → 拒绝并返回定位（[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md) §D4）。
- **`policy.describe(policy_id?)`**：无参返回当前 Scope 全部 `active` Policy 的列表 `[{policy_id, version, digest, rules_summary}]`；
  有参返回单个 Policy 的同形记录（[ADR-0003](../adr/0003-policy-object-and-language.md) §D1）。
- `policy.explain` 回答"这个准入事件当时为什么被放行/拒绝/要求审批"，返回该事件携带的
  `policy_decision{policy_version, rule_id, outcome, obligations[]}`；对 `audit/observed` 与 `approval/decided`（非准入类）返回其
  `causation_id` 指向的准入事件的决定。
- `memory.retract` 产生 `memory/retracted`；已装配进上下文的引用在下一次 `context/assembled` 时消失。
- **`runtime.list()`** 返回已登记的 Runtime Resource 及其资格化状态：生效的 `runtime/qualified` 引用，或最近的 `runtime/disqualified{reason}`，
  或从未资格化。
- **`runtime.qualify(resource_id)`**（[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D2/§D7）：调用该 Runtime 的
  `describe()` → 校验返回含 `native_effect_tools[]` 与 `describe_digest` → 把 `control_points[]` 与 `native_effect_tools[]` 送入 Policy 评估
  （`action == Action::"runtime/qualify"`）→ `allow` 则写 `runtime/qualified{resource_id, control_points[], native_effect_tools[], describe_digest,
  bundle_digest}` 并返回 `qualified{…}`；`deny` 则返回 `rejected(reason)`，不产生 `runtime/qualified`。**资格化只接受 `allow` / `deny` 两种结果**：
  `Approval.subject` 是封闭集合，没有"Runtime 资格化"一项，因此 Policy 对本动作产出 `approve` 或 `modify` 时一律按 `deny` 处理（fail-closed）并在
  `rejected(reason)` 中注明；人工把关的方式是先不放行、确认后换版 Policy 再 `runtime.qualify`（[07 §1.2](07-runtime-onboarding.md)）。`describe()` 缺
  `native_effect_tools` 字段（而非空数组）或缺 `describe_digest` → `rejected`。多 Runtime 并存按 `resource_id` 区分，资格化证据不跨 id 转移。

### 1.7 系统与节点

```text
system.status() / system.events(filter, cursor) / system.replay_check()
node.*                                         （边缘节点登记与合并观察；非 MVP，签名随 11 §3 定稿）
```

- `system.events` 按 `seq` 游标读取治理账本事件（含 `node_id` / `origin_seq`）；`system.replay_check` 重放并比对投影 digest。
- `node.*` 属边缘阶段（[11 §3](11-edge-and-offline-ledger.md)、[ADR-0010](../adr/0010-edge-offline-ledger-consistency.md)），MVP 只要求事件
  通用字段含 `node_id` / `origin_seq`（中心恒自身）。

## 2. Agent Skills · 实现包

FysunOS **不定义**自己的实现包格式。`Capability.implementation` 引用一个 Skills 目录，digest 固定（[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md) §D4）。

| `SKILL.md` frontmatter | FysunOS 映射 |
|---|---|
| `name` / `description` | Capability 摘要的一部分；不替代 `contract.behavior` |
| `license` | 登记进 Capability 记录；与供应链审查关联 |
| `compatibility` | 映射到 `requirements`（Runtime 类型、控制点、Resource 类型；可附"拒绝含特定 `native_effect_tools` 的 Runtime"） |
| `metadata` | 原样登记，OS 不解释 |
| `allowed-tools` | 该 Capability 下 Grant 请求的**初始上界**；`tools/list` 过滤时与 Grant ∩ Policy 再交；**每个工具须在 `contract.bounds.tool_effects[]` 有类别映射**，未映射的工具按 `external_mutation` 处理并由 `capability.health` 标"未分类工具" |

渐进披露三层（元数据 → 正文 → 按需资源）与上下文预算模型兼容：元数据层进入 `capability.find`，
正文与资源经上下文装配按预算与 Grant 进入 context_view，并携带 `label`（Skill 正文为 `trusted_system` 当且仅当其 digest 与登记一致，
否则 `untrusted_external`）。DSH 的 skills 加载格式若与 Agent Skills 不同，以 Agent Skills 为准（[08 §5](08-dsh-adapter.md)）。

## 3. 错误与拒绝语义

所有接口（Capability API、Runtime ABI、MCP 守卫对外错误）的拒绝都带稳定错误码与可判定原因。错误码全集与 [00 §8](00-terminology.md) 字面一致：

| 错误码 | 可判定说明 |
|---|---|
| `denied.policy` | Policy 评估结果为 `deny`；附 `policy_decision`。被 `tools/list` 过滤的工具其 `tools/call` 在账本内记 `action-intent/rejected{reason: denied.policy}`，但**对外返回与未知工具相同的 MCP 错误**（不可区分性，[06 §2](06-mcp-guard.md)） |
| `denied.grant` | 无覆盖该动作的有效 Grant，或 Grant 已撤销；被过滤工具的对外形状同上 |
| `denied.budget` | Grant 预算或 Activity 约束耗尽；附耗尽的预算维度（金额 / Token / 调用次数 / 时长 / 审批容量 `max_pending_approvals` / 审批速率 `approval_rate_limit` / escrow） |
| `denied.fencing` | 写入携带的 `fencing_epoch` 陈旧 |
| `not_runnable.environment{missing[]}` | 环境不满足 `requirements`；`missing[]` 列出缺失项。触发条件含：缺任一 required 控制点（含 `start_execution` 时 `requirements.control_points ⊄ runtime.control_points`）、Runtime 未资格化或已 `runtime/disqualified`、Policy 拒绝该 Runtime 的 `native_effect_tools[]`、适配器前置条件不满足（如 DSH approval policy 为 `never`、bundle `dsh_version` / `schema_digest` 不匹配）、实现包 digest 不匹配、`execution.fork` 时无 `fork` 控制点。**不得降级运行**（[05 §2](05-runtime-abi.md)） |
| `conflict.idempotency` | Runtime 显式携带的幂等键（MCP `tools/call` 的 `_meta` 或 `os.submit_candidate` 的 `idempotency_key`）已存在、但 canonical 参数与已记录者不同；**硬拒绝，不去重、不以任一方为准**；账本记 `action-intent/rejected{reason: conflict.idempotency}`。未显式携带时幂等键即 canonical 参数 digest，同键同参数命中 `action-intent/replayed` 而非本错误 |
| `unavailable.approval` | `approval/asked` 或 `approval/decided` 无法提交，或审批面完全不可用且 Policy 未允许纯带外等待；**整个请求拒绝**（fail-closed），候选 `action-intent/rejected{reason: unavailable.approval}`，不创建"半个" Approval |
| `needs_human(approval_id)` | Policy 结果为 `approve`，`fysund` 已创建 Approval 并写 `approval/asked`；声明控制点的 Runtime 等待 `deliver_input(approval_result)`；MCP-only Runtime 的投递路径（MCP Tasks → 守卫阻塞 `tools/call` 至决定或 `approval_wait_timeout` → 超时返回本错误 → `fysun.await_approval`）见 [07 §3](07-runtime-onboarding.md) |
| `input_required(request_ref)` | Execution 或 Handoff 需要父方或用户提供输入（非授权问题）；对应 A2A `INPUT_REQUIRED`、MCP elicitation；Execution 为 `waiting{reason: input}` |
| `auth_required(approval_id)` | 需要额外授权；已创建 `subject = grant_expansion` 的 Approval；对应 A2A `AUTH_REQUIRED` |
| `unknown_outcome` | Effect 结果未知，进入对账；不得假定成功 |
| `inconclusive` | Verification 无法判定；不是通过 |

规则：

- 拒绝不是失败测试；`not-run` 永不等于通过。
- 被 `tools/list` 过滤的工具：账本内保留真实原因（`denied.grant` / `denied.policy`），对外 MCP 错误的错误码、消息形状与耗时特征必须与调用不存在的
  工具一致（[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md) §D2）。
- `conflict.idempotency` 与 `action-intent/replayed` 的分界：**同键同参数 = 重放**（返回已有 Effect 状态），**同键不同参数 = 冲突**（硬拒绝）；
  不存在"以后者为准"。
- `unavailable.approval` 是 `approve` 路径的 fail-closed 出口：宁可拒绝一次候选，不留下无法配对的 Approval。
- `not_runnable.environment` 出现在 `capability.match`、`activity.start`、`execution.fork` 与 Runtime ABI 的 `start_execution` / `resume_execution`
  （[05 §2](05-runtime-abi.md)）；`missing[]` 的每项可判定（控制点名、Resource 类型、`native_effect_tools` 中被拒绝的工具名、失格原因）。

## 4. 负例

- `capability.register` 缺 `contract.bounds.tool_effects[]` 却成功 → 失败（M1）。
- `allowed-tools` 中存在未在 `tool_effects[]` 映射的工具，且该工具在运行时被按低于 `external_mutation` 的类别评估 → 失败（M1）。
- `approval.decide` 的调用者解析为 Execution 或 Runtime → 失败。
- `approval.decide` 返回 `result` 为封闭集之外的值（`approved` / `denied` / `modified` / `expired`）→ 失败。
- `result = rejected` 的 Approval 派生出 `grant/issued` 或 `adjustment/applied` → 失败。
- `adjustment.apply` 接受了给出 `predicted_effect` 但缺 `statement` 的提案 → 失败（M4）；`predicted_effect` 为空的 Adjustment 其
  `evaluated_against.result` 出现 `supported` / `refuted` → 失败（M4）。
- `policy.describe()` 无参只返回单个 Policy 而非列表 → 失败（M1）。
- `runtime.qualify` 对 `describe()` 缺 `native_effect_tools` 字段的 Runtime 返回 `qualified` → 失败（M2a）。
- 同一显式幂等键、不同 canonical 参数的两次提交都返回成功 → 失败（M1）。
- 被过滤工具的 `tools/call` 对外错误与未知工具可区分 → 失败（M2a）。
- `activity.resume` 之外（且无 Policy `auto_resume`）出现 `activity/resumed` → 失败（M3）。
- `execution.observe` 对因 `pending` Approval 等待的 Execution 返回 `waiting.reason = trigger` → 失败（M3）。
