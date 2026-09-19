# 06 · MCP 守卫规范（`fysun-mcp-guard`）

- 状态：设计稿 v0.7.0（2026-09-08）
- 变更依据：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md)；ADR-0001 ~ ADR-0006（v0.7.0 修订）与 ADR-0007 ~ ADR-0012
- 前置：[02 对象模型](02-object-model.md)（§2.6 准入流程、§2.9 Resource、§2.15 标签）、[03 架构](03-architecture.md)、[05 Runtime ABI](05-runtime-abi.md)、[00 术语表](00-terminology.md)
- 决策依据：[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md) §D2（守卫语义与不可区分性）、[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)（来源标签与 `flags`）、[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D3/§D5（hooks 路径、投递路径、守卫系统工具）、[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md)（Effect 类别三源合一、审计快路径、幂等冲突、补偿）、[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md)（评估输入构造）、[ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md)（elicitation 作为审批载体、T11）、[ADR-0012](../adr/0012-mcp-guard-self-implementation.md)（自实现，关闭 agentgateway 复用评估）
- 来源：拆分自 v0.6.0 `04-abi.md` §2.1（MCP 守卫）、`02-object-model.md` §2.6 准入流程、`03-architecture.md` §3 守卫段；§3、§5、§7、§8、§9 为 v0.7.0 新建

MCP 守卫是 TCB（`fysund` 进程）的**唯一外部工具边界**：每个 Runtime 的每次工具发现与工具调用都经它；F3（先记录再执行）、F7（能力不自扩权）、
F9（有效权限单调收窄）对 MCP 工具面的机械化全部在此实现。本文是守卫行为的唯一规范；[13](13-threat-model.md) 与 [12](12-mvp-plan.md) 只引用本文。

## 1. 定位

- **`fysund` 为唯一 MCP server 或前置**：harness 把 `fysund` 配置为**唯一** MCP server，或 `fysund` 前置在 harness 配置的所有 MCP server 之前。
  两种形态下 harness 看到的工具面都只来自守卫。
- **harness 不直接持有工具连接**：下游工具 / MCP server 由 `fysund` 作为 MCP client 代为连接（§8）；harness 与下游之间没有直接通道，
  也拿不到下游凭据或会话标识。
- **守卫是 `fysund` 内的组件**（`fysun-mcp-guard`，Rust），与账本写入在同一进程、同一事务边界内完成判定与记账；F1（唯一权威写入者）不因守卫而分裂。
- **自实现，借用 agentgateway 语义**（[ADR-0012](../adr/0012-mcp-guard-self-implementation.md)）：守卫结果语义 `Pass` / `Reject` / `Mutated` 三值与
  Cedar 策略格式借用自 agentgateway `mcpGuardrails`，便于对照测试；**不以 agentgateway 为库依赖、不 fork、不 vendor、不作为独立进程前置**，
  `fysund` 依赖清单中不出现 agentgateway crate。守卫结果与 `policy_decision.outcome` 的对应固定为：`allow → Pass`、`deny → Reject`、
  `modify → Mutated`、`approve → 阻塞后按 Approval 结果 Pass / Reject`；不存在表外映射、不存在三值以外的结果。
- **与 DSH `guard/` 包无关**：DSH 的 `guard/` 包（`packages/guard`）是 loop 卫生插件（检测重复工具调用、空转、超长 turn），作用于 DSH 进程内的
  loop 决策，不是安全守卫、不在 TCB 内，其判定对 OS 只是 Runtime 内部意见（[05 §5](05-runtime-abi.md)）。全仓库以"MCP 守卫"/ `fysun-mcp-guard`
  指本文组件。
- **覆盖范围如实声明**：守卫只覆盖经 MCP 的工具面。Runtime 内置、不经 MCP 的原生工具（`describe().native_effect_tools[]`）不经守卫；
  对 MCP-only Runtime，F3 / F7 / F9 的范围限于守卫工具面（[13 §6](13-threat-model.md) 按 Runtime 类型分列）；hooks.json 阻断路径（§9）只能阻断，
  不使原生工具进入守卫范围。`fysun-dsh` 的 DSH 原生工具另经 `approval/request` 拦截 + `tools/post-execute` receipt 路径治理（[08 §2](08-dsh-adapter.md)），
  不属本文。

守卫覆盖的 MCP 方法一览（各节展开）：

| MCP 方法 | 守卫行为 | 对应 OS 语义 | 节 |
|---|---|---|---|
| `tools/list` | 按 Grant ∩ Policy ∩ `allowed-tools` **过滤并改写**工具面；被过滤工具不可区分；附守卫系统工具 | F9 的机械化：看不见就调不了；工具面 = 有效权限集的一个因子 | §2、§7 |
| `tools/call` pre | = `submit_candidate`：ActionIntent 持久化（幂等键）→ 类别合成 → Policy → Grant → 来源标签 | `action-intent/recorded` 或 `action-intent/rejected`；`effect/authorized`；`approve` 时 `approval/asked` | §3、§4 |
| `tools/call` 执行 | Effect dispatch：在 `fencing_epoch` 下转发给下游 server | `effect/dispatch-started` | §4 |
| `tools/call` post | Receipt 回填；结果打 `label`（一律 `untrusted_external`）并分类 `flags`；Policy 可 `modify` / `deny` 结果 | `effect/receipt`；无结果则 `effect/outcome-unknown` | §4 |
| `tools/call`（`local_read`） | 审计快路径：Scope / Grant / 标签检查 + `audit/observed`，不持久化 ActionIntent / Effect | `audit/observed` | §5 |
| `tasks/*` | 长任务与 outcome-unknown 的标准表达；`needs_human` 的挂起载体（client 支持时） | Effect 状态机；`waiting{reason: approval}` | §6 |
| `elicitation/create` | `fysund` 发起的审批载体；应答绑定 session Principal | `approval/asked` → `approval/decided` | §6 |
| `resources/read` / `prompts/get` | 暴露 `fysun://context/<execution_id>` 上下文视图（MCP-only Runtime 的渲染路径） | `context/assembled` | [07 §3](07-runtime-onboarding.md) |
| `notifications/tools/list_changed`（下游 → 守卫） | 重新拉取下游 `tools/list`，重新过滤与重分类，再向 harness 发出同名通知 | 目录更新；T2 供应链检查点 | §2、§8 |

## 2. `tools/list`：过滤与不可区分性

### 2.1 过滤规则

对当前 Execution，守卫返回的工具面 = **Grant ∩ Policy ∩ `allowed-tools`**：

- **Grant**：Execution 有效 Grant 覆盖的工具（含下游 server 级授权）；`grant/revoked` 或 `grant/attenuated` 后下一次 `tools/list` 即收窄；
- **Policy**：当前 Scope 的 `active` Policy 对该 principal / action / resource 不产出 `deny` 的工具（`approve` 与 `modify` 的工具**保留在列表中**，
  调用时再触发 Approval 或改写）；对整个下游 server 的 `forbid` 使其全部工具消失；
- **`allowed-tools`**：Capability 实现包 `SKILL.md` 声明的初始上界（[04 §2](04-capability-api.md)）；无 Agent Skills 实现的 Capability 以
  `contract.bounds.tool_effects[]` 中列出的工具为上界。

三者都只能收窄，任何一方变化只会使工具面变小或不变；扩大只能经 Approval `approve_for_activity` 派生新 Grant（F9）。
守卫在返回前**改写**工具描述符：去掉下游的会话 / 凭据相关字段、保留下游 tool annotations 原样透传（供模型参考，守卫自身不读取它们做任何决定，
[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md) §D4）、附加守卫系统工具（§7）。
描述符中**不存在可放明文的字段**（SecretRef 三规则之"描述面无值槽"）。

工具名以 `<server>/<tool>` 限定（与 Cedar action 实体 `Action::"<server>/<tool>"` 一致，[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md) §D1）；
harness 看到的名字由守卫决定，下游 server 名冲突由守卫消歧。

### 2.2 不可区分性

被过滤的工具对 harness 必须**不存在**（借鉴 DeepSeek Harness scope 过滤）：

- 被过滤工具不出现在 `tools/list`；
- 对被过滤工具的 `tools/call` → 守卫 `Reject`，**对外返回的 MCP 错误与调用一个不存在的工具完全相同**：错误码、消息形状、耗时特征都不泄露
  "该工具存在但被过滤"；
- 账本内记 **`action-intent/rejected{reason: denied.grant | denied.policy}`** 保留真实原因（准入类，带 `policy_decision`），供 `policy.explain` 与审计；
- 不产生 `effect/authorized`，不转发到下游。

理由：注入（T1）与被投毒实现包（T2）常先探测"有哪些工具"再构造调用；可区分的错误等于向模型泄露工具面之外的能力清单。

### 2.3 下游 `notifications/tools/list_changed`

下游 server 发出 `notifications/tools/list_changed` 后，守卫：

1. 重新拉取该 server 的 `tools/list`；
2. 对每个工具**重新合成 `effect_category`**（§3）——工具语义可能已变（T2 供应链威胁，§8）；
3. 重新按 2.1 过滤；
4. 向 harness 发出 `notifications/tools/list_changed`，harness 下一次 `tools/list` 得到新工具面；
5. 在途 Effect 不受影响（它们绑定的是提交时的类别与描述符 digest）。

工具描述符 digest 变化但类别未变 → 只更新目录；类别变化 → 更新目录并记 `resource/registered`（新描述符）；类别从低变高的工具若在
Capability `tool_effects[]` 中被声明为低类别，`capability.health` 立即出现"未分类 / 类别不一致"信号。

## 3. Effect 类别：三源合一

`effect_category` 是封闭枚举，按严重度升序（[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md) §D1）：

```text
local_read < external_read < local_mutation < external_mutation < credential_use < physical_action
```

| 类别 | 含义 | 例子 |
|---|---|---|
| `local_read` | 读取本机、本 Scope 内、无副作用、结果不出本机的观察 | 读文件、列目录、查询本地索引、读取账本投影 |
| `external_read` | 读取外部系统，可能留下访问痕迹或消耗外部配额，但不改变外部状态 | HTTP GET、搜索 API、读取远端邮箱 |
| `local_mutation` | 改变本机 / 本 Scope 内状态，可由本机手段回退 | 写文件、修改本地配置、本地 git commit |
| `external_mutation` | 改变外部系统状态或向外部发送内容 | 发帖、发邮件、写远端数据库、git push |
| `credential_use` | 使用、导出、轮换凭据，或以凭据换取更高权限 | OAuth 授权、`secret.import`、调用需要额外授权的接口 |
| `physical_action` | 驱动执行器产生物理世界效果 | 阀门、电机、锁具 |

序只表达"更保守"的方向，不表达风险量化，不引入分数。一个工具只有一个类别；同一工具在不同参数下严重度不同（如 `git` 既能 `status` 也能 `push`）
时，按其**最严重**用法分类，或要求 MCP server 拆为多个工具名（如 `git_status` 与 `git_push`）。

### 3.1 三个来源

| 来源 | 谁写 | 约束 |
|---|---|---|
| (a) Capability `contract.bounds.tool_effects[]: [{tool, effect_category}]` | Capability 作者（[02 §2.1](02-object-model.md)） | **必填**；应覆盖 `allowed-tools` 中每个工具；`capability.register` 缺整个字段 → 拒绝 |
| (b) Resource 目录登记的缺省类别 | 部署者登记 MCP server / 工具时（[02 §2.9](02-object-model.md)、`resource.register`） | 可选；MCP tool annotations（`readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint`）只作为登记时呈现给部署者的**建议输入** |
| (c) Policy 覆盖 | 部署者，Cedar 策略 | 可升可降；是**唯一**能降低类别的来源（[03 §7](03-architecture.md)） |

### 3.2 合成规则

```text
effective = (c) if present
          else max_severity((a), (b))      // (a)(b) 任一缺失取另一方
(a)(b) 均缺 -> external_mutation            // fail-closed
```

- 合成在 `tools/call` pre 阶段对每次调用执行，结果写入 ActionIntent 的 `action`（`effect_category` + 工具名）与 Policy context
  `effect_category`，并决定该调用走审计快路径（§5）还是完整协议（§4）。
- (a) 中未列出但出现在 `allowed-tools` 的工具：按 `external_mutation` 处理，`capability.health(id)` 标为"未分类工具"（契约缺陷信号，不是运行时错误）。
- **Capability 单方不能降低 (b) 已登记的类别**（`max_severity` 保证）：防止实现包（T2）把 `external_mutation` 工具声明成 `local_read` 走快路径。
  只有 Policy (c) 能降低。
- 未分类兜底取 `external_mutation` 而非 `physical_action`：后者要求包络 Policy 与物理扩展字段，会使普通工具 `not_runnable`；`external_mutation`
  是"走完整协议、缺省需 Policy 明示才能自动执行"的最低保守点。
- 守卫**不读取** tool annotations 做任何判定；annotations 在 `tools/list` 改写时原样透传给 harness。守卫代码路径中出现以 annotations 为依据的分支是负例。
- 下游 `notifications/tools/list_changed` 后全部重新合成（§2.3）。
- Cedar 侧：`fysund` 按合成结果为每个工具级 action 建立 `in Action::"category:<effect_category>"` 关系，Policy 可只对类别组写规则
  （[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md) §D5）。

## 4. `tools/call`：pre / 执行 / post

`tools/call` 在 ABI 层的定义就是 **Effect 候选**（[05 §4](05-runtime-abi.md)）。守卫对每次调用执行下列流程（[02 §2.6](02-object-model.md) 的守卫侧展开）：

```text
tools/call(name, arguments, _meta?)
  pre ─┬─ 工具面检查：name 不在当前 tools/list → Reject（与未知工具同形；账本 action-intent/rejected{denied.grant | denied.policy}）
       ├─ 幂等键：_meta 携带显式 idempotency_key 则用之，否则 = canonical(arguments) digest
       │     同键同参数 → 命中 action-intent/replayed，返回已有 Effect 状态，流程结束
       │     显式同键不同 canonical 参数 → conflict.idempotency 硬拒绝（action-intent/rejected），不去重，流程结束
       ├─ 类别合成（§3）→ effect_category
       │     local_read → 审计快路径（§5），流程结束
       ├─ 来源标签：按参数值与已装配片段 / 先前工具结果匹配，尽力标注 label；无法匹配 → untrusted_external
       ├─ Policy 评估（context 由 fysund 构造：effect_category、params_digest、min_integrity、untrusted_params、budget_remaining、
       │     reversibility、runtime_type、control_points、native_effect_tools、assurance_level、offline、pending_approvals）
       │     deny    → Reject；action-intent/rejected{denied.policy}
       │     modify  → Mutated；按 obligations 改写参数，记新 canonical digest，继续
       │     approve → 创建 Approval；approval/asked（带 policy_decision）；Execution waiting{reason: approval}；
       │               投递路径：MCP Tasks 挂起 / 阻塞至决定或 approval_wait_timeout / needs_human(approval_id)（§6、07 §3）
       │               approved_once → 继续（params_digest 以 approval/decided 为准）；rejected / cancelled / unavailable → action-intent/rejected
       │     allow   → 继续
       ├─ Grant 检查：无覆盖 → denied.grant；预算 / 审批容量 / 审批速率耗尽 → denied.budget；escrow 越界（边缘）→ denied.budget
       ├─ 持久化 ActionIntent（action-intent/recorded）→ 持久化 Effect（effect/authorized，带 policy_decision）
  执行 ─ 在 fencing_epoch 下经 fysund 持有的下游会话转发（effect/dispatch-started）；陈旧 epoch → denied.fencing
  post ─┬─ 下游返回 → Receipt 回填（effect/receipt）；超时 / 会话失效 / 无回执 → effect/outcome-unknown（对账或 quarantine，不假定成功）
       ├─ 结果标签：label{source_principal: <server>, integrity: untrusted_external, origin_ref: effect_id, flags?}
       ├─ flags 分类：authority_claim / secret_shaped（形状分类，不改变 integrity）
       ├─ Policy 结果评估（context.result_flags）：modify → 脱敏后回传；deny → 不回传 harness，返回与工具执行失败同形的结果并留痕；allow → 回传
       └─ 回传 harness：结果内容 + label（Runtime 不能改写）
```

### 4.1 pre 阶段规则

- **persist-before-dispatch（F3）**：ActionIntent 与 Effect 在转发给下游之前落账；一次非快路径 `tools/call` 无配对的 `action-intent/recorded` 是负例。
- **幂等键**：缺省 = canonical 参数 digest（[ADR-0006](../adr/0006-terminology-action-intent.md) §D1）；Runtime 可在 MCP 请求 `_meta` 中显式携带
  `idempotency_key`。**`conflict.idempotency` 只在显式携带且同键不同 canonical 参数时触发**；未显式携带时键即 digest，同键必同参数，只可能重放。
  冲突时不去重、不以任一方为准、不覆盖（CognitiveOS `effects.rs` 经验：同键不同参数意味着 Runtime bug 或重放篡改，静默去重会让账本参数与实际执行参数不一致）。
- **来源标签**：声明 `context_injection` 的 Runtime 经 `os.submit_candidate` 自行标注（守卫路径下同样以 `fysund` 记录为准）；MCP-only Runtime 由守卫
  按参数值与 context_view 片段 / 先前工具结果匹配**尽力**标注，无法归因的值记 `untrusted_external`（fail-closed）。Runtime 传回的任何标签字段被视为声明，
  高于 `fysund` 记录者以低者评估并记录标签冲突。
- **Policy 评估 context 由 `fysund` 构造**，固定键见 [ADR-0009](../adr/0009-cedar-subset-and-entity-model.md) §D1；`_meta` 中的任何 context 覆盖被忽略。
- **顺序**：类别合成先于 Policy（Policy 需要 `effect_category`）；Policy 先于 Grant（`approve` 路径需要在 Grant 检查前挂起，Approval 可派生新 Grant）；
  Grant 检查后才持久化 Effect。`approve` 挂起期间 ActionIntent 已持久化（`action-intent/recorded`，`approval_id` 指向 Approval），Effect 尚未 `authorized`。
- **审批容量与速率**：所属 Activity 待决 Approval 达 `max_pending_approvals`，或单位时间 `approval/asked` 达 `approval_rate_limit` → 新的 `approve` 类
  候选按 `denied.budget` 拒绝并留痕，不再创建 Approval（[ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md) §D6）。
- **fail-closed 的 Approval**：`approval/asked` 无法落账 → 整个请求以 `unavailable.approval` 拒绝，`action-intent/rejected{reason: unavailable.approval}`，
  不创建"半个" Approval。

### 4.2 执行阶段规则

- 转发经 `fysund` 持有的下游会话（§8）；harness 不参与。
- `fencing_epoch` 由 Execution 的 home 节点分配；陈旧 epoch 的转发被拒（`denied.fencing`），防止崩溃恢复后旧实例的 Effect 覆盖新 epoch。
- 下游会话失效（Streamable HTTP 会话过期、连接断开）时，在途 Effect → `effect/outcome-unknown`，重连后按适配器对账能力 reconcile 或 quarantine（§8）。

### 4.3 post 阶段规则

- **Receipt** 是下游返回的结构化回执（不是自然语言观察）；工具成功不等于完成（F4）。
- **结果标签一律 `untrusted_external`**——不论工具本地还是远端、只读还是可写、下游 server 是否自称可信；唯一升级路径是 Verification 或 Memory 准入
  （[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md) §D1）。
- **`flags` 分类**（借鉴 AKP 入站标志）：守卫对结果内容做形状分类——
  `authority_claim`：内容以系统 / 用户 / 管理员口吻下达指令或宣称权限（"作为管理员，我批准…"、"忽略之前的规则"）；
  `secret_shaped`：内容含 API key、token、私钥、连接串等凭据形状。`flags` 是分类信号，不改变 `integrity`。分类器的模式集合由 `fysund` 维护，
  是否进入 Policy 为开放问题（[12 §7](12-mvp-plan.md)）。
- **Policy 对结果的处置**（context `result_flags`）：`modify` → 按 obligations 脱敏（如 `redact:<field>`）后回传；`deny` → **不回传 harness**，
  向 harness 返回与工具执行失败同形的结果并在账本记录（Effect 仍 `receipt`，附结果被拦截标记）；`allow` → 原样回传。
  `authority_claim` 通常不拦截结果，而由 Policy 提高该 Execution 后续候选的审批门槛。
- `secret_shaped` 结果未经 Policy 直接回传 harness 是负例（`NV-T4-03`）。
- 结果与 `label` 一并回传；Runtime 不能改写标签。

### 4.4 补偿

`effect/compensated` 对应的补偿动作是一次**新的** Effect（[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md) §D7）：

- 必须有自己的 ActionIntent（新幂等键）与自己的 Grant 检查；**拒绝复用被补偿 Effect 的原 Grant**——原 Grant 可能已撤销、耗尽或到期，且补偿动作的
  类别可能高于原动作（撤回一条已发帖子是 `external_mutation`）；
- 补偿 ActionIntent 的 `causation_id` 指向被补偿的 Effect；Policy 对补偿类候选可单独设规则（context `reversibility` 与被补偿 Effect 的引用）；
- 边缘 `revoked_offline` 冲突**不自动补偿**（补偿需 Principal 决定，[11 §3](11-edge-and-offline-ledger.md)）。

## 5. 审计快路径与 `audit/observed`

- **仅 `effective effect_category = local_read` 的调用走审计快路径**（[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md) §D5）。
  快路径仍执行：工具面检查（§2）、Scope 检查、Grant 检查、来源标签检查与结果打标签；**不持久化 ActionIntent / Effect**，
  而是产生事件 **`audit/observed{execution_id, tool, effect_category: local_read, params_digest, result_label, path: guard | hook | adapter, verdict?}`**。
- `audit/observed` **不是准入类事件**，不带 `policy_decision`；它与 `tools/call` 配对一起构成 MCP-only Runtime 的可审计关联面（[03 §4](03-architecture.md)、
  [05 §6](05-runtime-abi.md)）。`path: guard` 为本节路径；`path: hook` 为 §9 的 hooks 路径（此时另带 `verdict`）；`path: adapter` 为声明控制点的 Runtime 经
  适配器拦截点 / `os.submit_candidate` 提交的只读候选（如 `fysun-dsh` 对 DSH 原生只读工具的处理，[08 §2](08-dsh-adapter.md)），不经本文守卫。
  适配器拦截路径的非 `local_read` Effect 经 **`os.report_receipt`** 回填 receipt（[05 §4](05-runtime-abi.md)、[08 §2](08-dsh-adapter.md)）；
  本文守卫路径的 receipt 由 §4.3 post 阶段自行回填，不经该调用。
- **`external_read` 走完整协议**（ActionIntent → Effect → Receipt），不走快路径。Policy 通常对其 `allow`，实际开销主要是账本写入；理由：外部读取留下访问
  痕迹、消耗外部配额、可能触发外部速率限制，且其结果是 T1 注入的主要载体，账本必须能回答"这条 `untrusted_external` 内容来自哪次外部读取"。
- 快路径不能被 Capability 单方启用：`local_read` 必须是合成结果——(b) 或 (c) 登记为更高类别的工具，Capability (a) 声明 `local_read` 也不走快路径（§3.2）。
- Memory 写入不走快路径也不走 Effect 状态机（`memory/admitted` 是准入事件，[02 §2.13](02-object-model.md)）。
- 度量项"审计快路径占比" = `audit/observed` / (`audit/observed` + `action-intent/recorded`)（[12 §4](12-mvp-plan.md)）。
- `audit/observed` 的保留策略（账本体量）是否需要独立压缩规则为开放问题（[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md) 待后续）。

## 6. `tasks/*` 与 `elicitation/create`

### 6.1 `tasks/*`（长任务与 outcome-unknown）

- `tasks/*` 是长任务与 `outcome_unknown` 的标准表达（MCP Tasks，实验特性，正移入扩展；FysunOS 对其支持为**可选**，`outcome_unknown` 的账本语义不依赖它）：
  `tasks/get` 读取 Effect 状态；`tasks/result` 对应 `effect/reconciled`；`tasks/cancel` 不撤销已进入世界的效果，只使 Effect 进入对账。
- **作为 `needs_human` 的挂起载体**：harness 支持 `tasks/*` 时，`outcome = approve` 的 `tools/call` 以 task 形式挂起，Approval 决定到达后经
  `tasks/result` 返回（`approved_once` → 继续执行并返回结果；其他结果 → 返回拒绝）。这是 MCP-only Runtime 下 `deliver_input(approval_result)` 的优先落地路径
  （[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D5、[07 §3](07-runtime-onboarding.md)）。
- harness 不支持 `tasks/*` 时：守卫**阻塞** `tools/call` 直到决定到达或 Policy `approval_wait_timeout` 超时；超时返回 `needs_human(approval_id)`，
  harness 可调用 `fysun.await_approval(approval_id)`（§7）继续等待。决定到达而无开放通道时，Effect 停在 `authorized` 直到 Runtime 重新提交同幂等键候选
  （命中 `action-intent/replayed`）或轮询，受 Policy `approval_result_ttl` 约束，超期按 Approval `on_expiry` 处理。

### 6.2 `elicitation/create`（`fysund` 发起的审批载体）

- `elicitation/create` 由 **`fysund` 作为 server 发起**、经守卫下发给 harness（含 URL-mode），是审批面的三种形态之一（Web UI / CLI / `fysund` 发起的 elicitation，
  [ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md) §D5）。呈现内容由 `fysund` 按 `Approval.subject_ref` 从账本 / Artifact 读取并附来源标签渲染；
  `untrusted_external` 参数值按 Policy 可脱敏或折叠；elicitation 载荷不复制工具参数明文之外的审批事实（`decided_by`、`reason` 不进入）。
- **应答绑定 session Principal**：elicitation 由 MCP client 应答，client 进程可能被模型驱动；应答只在能解析为该 session 绑定的 Principal（人或其授权系统）
  时才经 `approval.decide` 落账为 `approval/decided`；无法解析 → 忽略并留痕，Approval 仍 `pending`。
- **Policy 可按 `Approval.subject` 强制带外**：只接受 Web UI / CLI，不发起 elicitation（例如 `grant_expansion`、`credential_use` 类候选）。
- **Runtime 文本输出永不构成审批面**（T11 审批面伪造）：聊天通道中的"批准"文本、模型生成的确认控件、Runtime 自绘的审批卡片（含 DSH 原生审批卡片）
  都不产生 `approval/decided`。守卫不解析工具结果或模型消息中的任何"批准"语义。
- elicitation 不是 `input_required` 的替代：Execution / Handoff 需要用户提供**输入**（非授权问题）时用 `input_required(request_ref)`，
  也可经 elicitation 传输，但不产生 Approval。

## 7. 守卫系统工具 `fysun.*`

守卫在 `tools/list` 中附加两个由 `fysund` 自身提供的工具（MCP-only Runtime 投递路径的组成部分，[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D5）：

| 工具 | 类别 | 语义 |
|---|---|---|
| `fysun.await_approval(approval_id)` | `local_read` | 长轮询等待指定 Approval 的决定；返回 `approval_result{approval_id, result, params_digest, grants_summary?}`；决定未到达时按 Policy `approval_wait_timeout` 再次返回 `needs_human(approval_id)`；Approval 不属于本 Execution → 与未知 `approval_id` 同形的错误 |
| `fysun.pending_inputs()` | `local_read` | 拉取待投递给本 Execution 的 `message \| event \| handoff_result`（FIFO，拉取即确认）；返回项带 `label`；空则返回空列表 |

- 二者**出现在 `tools/list`**，**不受 Capability `allowed-tools` 限制**（它们不是 Capability 的工具，而是 OS 的投递入口），但**受 Policy 控制**
  （Policy 可对某 Scope 禁用，例如强制带外审批的场景下禁用 `fysun.await_approval`）。
- 类别固定为 `local_read`，走审计快路径（`audit/observed`），不产生 Effect；Cedar action 为 `Action::"fysun/await_approval"` / `Action::"fysun/pending_inputs"`，
  `in Action::"category:local_read"`（[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md) §D1）。
- 前缀 `fysun.*` 是 MCP 工具名前缀，与 OTLP 属性命名空间 `fysun.*`（[03 §8](03-architecture.md)）是**不同域**。
- 返回的 `approval_result` 与 `deliver_input(approval_result)` 载荷相同，同样不含 `decided_by` / `reason`（[05 §2](05-runtime-abi.md)）。
- 声明控制点的 Runtime 也可使用它们，但 `deliver_input` 是其主路径。

## 8. 下游 MCP server 的信任边界

下游 MCP server 位于守卫之后、TCB 之外；`fysund` 作为 MCP client 连接它们。

- **凭据**：下游 server 需要的凭据（API key、OAuth token）以 **`SecretRef`** 登记在 Resource 目录（`resource.register`），**只在 `fysund` 建立到该 server
  的连接时注入**（Authorization 头、环境变量等）；遵守 SecretRef 三规则——每次连接重解析、描述面无值槽、空值即未配置。harness 与模型永不见明文；
  `tools/list` 改写后的描述符不含任何凭据字段。
- **会话**：每个下游 server 一条由 `fysund` 持有的 **Streamable HTTP 会话**（`Mcp-Session-Id` 只在 `fysund` 内，不暴露给 harness）或 stdio 子进程；
  harness 与下游之间没有直接通道。多个 Execution 共用同一下游会话时，守卫以 `execution_id` + `fencing_epoch` 区分在途调用。
- **会话失效**：下游返回会话过期、连接断开或进程退出 → `fysund` 重连并**重新 `tools/list`**（走 §2.3 的重过滤与重分类）；在途 Effect →
  `effect/outcome-unknown`，按适配器对账能力 reconcile 或 quarantine，不假定成功。重连失败 → `resource/unavailable`，关联 Grant `grant/revoked`，
  未决 Effect 对账（[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md) §D6 规则）。
- **结果**：下游返回内容一律 `untrusted_external` 并分类 `flags`（§4.3）；下游自报的 tool annotations 只作 (b) 登记的建议输入（§3）。
- **工具语义变更 = T2 供应链威胁**：下游 server 更新后同名工具的行为可能变化（描述符 digest 变化、annotations 变化、参数 schema 变化）；
  守卫在 `list_changed` 或重连后重新合成类别并更新目录；类别从低变高时 Capability `tool_effects[]` 中的旧声明由 `max_severity` 自动覆盖；
  变更后未重分类是负例（`NV-T2-04`）。可疑 server → `resource/quarantined`，其工具不出现在 `tools/list`。
- **server 级 Policy**：Cedar resource 实体 `McpServer` 承载整 server 规则（禁用、只允许 `external_read`、要求 Approval），[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md) §D1。
- 守卫不审计下游 server 代码（[13 §4](13-threat-model.md) 不防项）；只做来源登记、工具面限制、类别合成、结果标签。

## 9. hooks.json 阻断路径（Level-1b）

对支持 hooks.json 的 harness（Claude Code / Codex / DSH 兼容的 `PreToolUse` command hook），部署者可把 hook 命令指向 CLI 子命令
**`fysun hook pre-tool-use`**，为 harness **原生（非 MCP）工具**增加一条 Policy 阻断路径（[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D3）：

```text
harness PreToolUse(tool_name, tool_input) ──stdin──▶ fysun hook pre-tool-use
  → 调 fysund 本地 RPC（携带 execution_id、tool_name、canonical(tool_input) digest）
  → fysund：类别合成（native_effect_tools[] 缺省类别 / Policy 覆盖；无登记 → external_mutation）→ Policy 评估
      deny        → 退出码 2（harness 阻断本次调用）；账本 action-intent/rejected{reason: denied.policy, path: hook}（准入类，带 policy_decision）
      其他 outcome → 退出码 0（放行 = abstain）；账本 audit/observed{execution_id, tool, params_digest, path: hook, verdict: abstain}
  → fysund 不可达 / RPC 超时 → 退出码 2（fail-closed）
```

能力边界（这条路径**只能 deny 或 abstain**）：

- 不持久化 ActionIntent、不产生 Effect、没有 receipt、不打来源标签（hook 看不到工具结果，参数也不做来源匹配，[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md) §D2）；
- 不能 `modify` 参数（hook 协议无改写通道）；不能触发 Approval（harness 的 hook 是同步阻断，无等待语义）——Policy 对该候选产出 `approve` 或 `modify`
  时 hook 只能 abstain 并留痕（`verdict: abstain`，附 `outcome`），不能放行"等审批后再执行"的语义；
- Policy 在 hook 路径中只能基于工具名、Runtime 类型、`native_effect_tools[]` 缺省类别与 Scope 判定，无参数来源标签；
- 除 `action-intent/rejected{path: hook}` 之外，hooks 路径**不得**产生任何其他准入类事件（不得写 `effect/authorized`）。

定位：

- 速记 Level-1b = MCP 客户端 + 最小生命周期 RPC + hooks.json PreToolUse 阻断，只是叙述速记，禁止进入任何标识符（00 §1.3）。
- **不计入 conformance 主路径**：hook 由 harness 进程调用、harness 可被配置绕过、hook 失败时 harness 的缺省行为不受 `fysund` 控制、hook 看不到结果。
  [13 §6](13-threat-model.md) 的"MCP-only + hooks"列只能把它描述为"对原生工具的 Policy 阻断（尽力）"，不得据此把 F3 / F7 / F9 的范围扩大到原生工具；
  设计文档或 conformance 声明把 hooks 路径描述为"安全边界"是负例。
- hooks 判定对 OS 只是 Runtime 内部意见（[05 §5](05-runtime-abi.md)）；它不使原生工具进入守卫范围，也不替代 `describe()` 对 `native_effect_tools[]` 的如实声明。
- 部署与 harness 侧配置见 [07 §4](07-runtime-onboarding.md)；多个 hooks 路径并存（harness 自带 + `fysun hook`）时的先后与冲突处置为开放问题。

## 10. 负例

以下负例进入 [13 §5](13-threat-model.md) 索引与 [12 §3](12-mvp-plan.md) 对应里程碑验收：

- **被过滤工具可区分**：被 `tools/list` 过滤的工具其 `tools/call` 返回的 MCP 错误在错误码、消息形状或耗时特征上与调用不存在的工具可区分 → 失败（`NV-T1-04`，M2a）；
  账本出现对应的 `effect/authorized` → 失败。
- **未分类工具走快路径**：`tool_effects[]` 与目录均未登记的工具产生 `audit/observed` 而无 `action-intent/recorded` → 失败（M1）；
  该工具被按低于 `external_mutation` 的类别评估 → 失败。
- **以 annotations 为依据**：`readOnlyHint = true` 且 (a)(b) 均缺的工具被按 `local_read` 评估 → 失败；守卫代码路径出现以 annotations 为依据的分支 → 失败。
- **Capability 单方降级**：目录登记为 `external_mutation` 的工具被 Capability 声明为 `local_read` 后走了快路径 → 失败。
- **同幂等键不同参数被去重**：显式同幂等键、不同 canonical 参数的两次 `tools/call` 都返回成功，或后者被静默视为重放 → 失败（`NV-T5-04`，M1）。
- **`secret_shaped` 直接回传**：结果被分类为 `secret_shaped` 且 Policy 未评估 `result_flags` 即回传 harness → 失败（`NV-T4-03`，M1）。
- **`fysund` 依赖 agentgateway**：`fysund` 依赖清单出现 agentgateway crate、fork 或 vendor 副本 → 失败；守卫结果出现 `Pass` / `Reject` / `Mutated` 三值以外 → 失败。
- **`external_read` 走快路径**：`external_read` 工具产生 `audit/observed` 而无 `action-intent/recorded` → 失败（M1）。
- **`local_read` 无审计**：一次 `local_read` 快路径调用无 `audit/observed` → 失败（M2a）。
- **persist-before-dispatch 破坏**：一次非快路径 `tools/call` 无配对的 `action-intent/recorded`，或 `effect/dispatch-started` 早于 `effect/authorized` → 失败（M1）。
- **补偿复用原 Grant**：`effect/compensated` 的补偿 Effect 引用原 Grant 且原 Grant 已 `grant/revoked` 仍执行，或补偿 Effect 无独立 `action-intent/recorded` → 失败（M1）。
- **下游变更未重分类**：下游 `notifications/tools/list_changed` 后同名工具类别变化而目录与合成结果未更新 → 失败（`NV-T2-04`，M2a）。
- **凭据泄露到描述面**：`tools/list` 返回的描述符含下游凭据字段或值槽，或 `Mcp-Session-Id` 出现在 harness 可见面 → 失败（M1）。
- **elicitation 应答未绑定 Principal**：经 elicitation 返回的决定未解析出 session Principal 即写 `approval/decided` → 失败（M3）。
- **审批面伪造**：工具结果或模型消息中的"批准"文本产生 `approval/decided` → 失败（`NV-T11-01`，M3）。
- **hooks 越界**：hooks 路径的 deny 被记为 `action-intent/rejected` 以外的准入类事件 → 失败；`fysun hook pre-tool-use` 在 `fysund` 不可达时退出码为 0 → 失败；
  hooks 路径产生任何 `label` → 失败；hooks 判定被当作守卫放行而对原生工具宣称 F3 → 失败（`NV-T9-03`，M2a）。
- **守卫系统工具越权**：`fysun.await_approval` 返回不属于本 Execution 的 Approval 结果，或其返回载荷含 `decided_by` / `reason` → 失败。
- **`approval_wait_timeout` 后返回成功**：超时后守卫返回成功而非 `needs_human(approval_id)` → 失败（M2a）。
- **`approval/asked` 落账失败仍放行**：`approval/asked` 无法落账后候选仍 `effect/authorized` → 失败（M1）。
