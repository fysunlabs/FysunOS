# 12 · MVP 计划

- 状态：设计稿 v0.7.0（2026-09-08）
- 变更依据：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md)；ADR-0001 ~ ADR-0006（v0.7.0 修订）与 ADR-0007 ~ ADR-0012
- 前置：全部设计文档；术语、事件名、错误码、状态集以 [00 术语表](00-terminology.md) 为唯一基准
- 决策依据：[ADR-0001](../adr/0001-founding-decisions.md)、[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md)、[ADR-0003](../adr/0003-policy-object-and-language.md)、[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)、[ADR-0005](../adr/0005-approval-and-evidence-gated-evolution.md)、[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md)、[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md)、[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md)、[ADR-0010](../adr/0010-edge-offline-ledger-consistency.md)、[ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md)、[ADR-0012](../adr/0012-mcp-guard-self-implementation.md)
- 目的：避免再次做成白皮书。第一个可运行闭环必须小、真实、可验证。本文同时是全仓库**统一开放问题清单**的登记处（§7）；[13 威胁模型](13-threat-model.md) 与各 ADR 只引用，不另设清单。

## 1. MVP 边界（MVP-first）

首个可运行路径只实现当前闭环与底线 F1–F9 要求的最小面：

- owner-local、单主体、单机；Scope 模型预留但不预建 RBAC、审批链、通用 capability 市场；`Node` 只作为事件通用字段
  `node_id` / `origin_seq` 存在（中心恒为自身），边缘合并机制留给边缘阶段（§3）；
- MCP 守卫 + **DSH headless 作为 MCP-only 基线 Runtime**（经薄包装进程实现最小生命周期 RPC，[07 §3](07-runtime-onboarding.md)）；
  随后 `fysun-dsh`——声明全部六个控制点的 DSH 参考适配器（[08](08-dsh-adapter.md)）；
- 一个不依赖任何业务对象的 Capability，实现包为 Agent Skills 目录，契约 `bounds.tool_effects[]` 覆盖其 `allowed-tools`；
- Policy 从本地 Cedar 子集文件载入为版本化对象（经 Cedar schema 校验，[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md)）；
- 不可省略：唯一权威写入者、Secret Store、persist-before-dispatch、预算 / fencing、独立 Verifier、审计边界、Policy 版本引用、
  来源标签、Approval 对象、F9 负例、**Effect 类别合成**（三源合一，[06 §3](06-mcp-guard.md)）、**Approval 配对事件**
  （`approval/asked` / `approval/decided`）、**Runtime 资格化登记**（`runtime.qualify` → `runtime/qualified`，[07 §1](07-runtime-onboarding.md)）。

## 2. 首个闭环

以"趋势调研 → 内容包"能力为例（可换成任何不需要外部账号的能力）：

```text
1.  capability.register        声明契约（输入源、输出内容包 schema、质量 checklist（10 §1.5）、预算/时间边界、Verifier=schema+来源校验；
                               bounds.tool_effects[] 覆盖 allowed-tools 中每个工具）；implementation 为 Skills 目录引用
2.  activity.create + start    建立长期 Activity（周期触发）；"先看过再发布"由 Policy 对发布类 Effect 产出 approve → Approval 对象实现
3.  fysund 启动 Runtime         Runtime 已 runtime/qualified（describe() 含 native_effect_tools[]、describe_digest）；harness 把 fysund 配为 MCP server；
                               start_execution 携带受控上下文视图（带来源标签）与 Grant 摘要；context_view 经薄包装渲染进 harness 启动输入，
                               同时经 resources/read（fysun://context/<execution_id>）可再读
4.  Runtime 执行               工具调用经 MCP 守卫 tools/call pre → 类别合成 → submit_candidate → ActionIntent/Effect → 执行 → Receipt；
                               local_read 走审计快路径产生 audit/observed；tools/list 已按 Grant ∩ Policy ∩ allowed-tools 过滤
5.  产物独立验证                Verifier 校验内容包 schema 与来源引用；Verification（含 verifier_kind、independence、verdict）写入账本与 Capability.evidence
6.  一次 Approval               发布候选命中 policy_decision.outcome=approve → approval/asked；用户 approve_once → approval/decided{result: approved_once}
                               后 Effect 才 authorized；崩溃恢复后未决 Approval 仍为 pending，不视为已批
7.  进程重启                    kill fysund 与 Runtime；重启后该 Activity（tool_effects 含非 local_read 类别）缺省 paused{reason: rearm_required}，
                               用户 activity.resume → activity/resumed → waiting（若有未决 Approval 则 input_required）；未知结果 Effect 对账
8.  一次 Adjustment             用户校准"语气更克制"，scope=this_activity，带 predicted_effect；下一 Execution 生效，其 Verification 自动填 evaluated_against；
                               adjustment.revert 可回退
9.  一次 Handoff                内容包中的配图交给另一 Capability（子 Execution）；Handoff 与子 Execution 在账本中可见；Handoff 状态可无损映射 A2A TaskState
10. 一次被过滤工具调用          Runtime 调用一个被 tools/list 过滤掉的工具；守卫返回与未知工具完全相同的 MCP 错误（不可区分）；
                               账本记 action-intent/rejected{reason: denied.grant | denied.policy}，不出现 effect/authorized
```

## 3. 阶段划分与验收

| 阶段 | 交付 | 验收（全部为可判定项） |
|---|---|---|
| **M0 账本与对象** | 治理账本 schema（event-sourced）、投影、replay；对象模型的 Rust 类型（含 `Node` 与事件通用字段） | replay 两次得到相同投影 digest（`NV-T5-02`）；负例：陈旧 epoch 写入被拒（`NV-T5-01`）；**事件通用字段含 `node_id` / `origin_seq`**，MVP 单机下 `node_id` 恒为中心、`origin_seq = seq`（字段缺失 → 失败） |
| **M1 权威协议** | ActionIntent / Effect / Receipt、Grant / 预算、Verifier 框架、Secret Store（`SecretRef` 三规则）、Provider 代理、Policy 载入（Cedar 子集 + schema 校验）、来源标签赋予、**Effect 类别合成**（三源合一）、**Approval 对象与配对事件**、**Runtime 资格化登记**（`runtime.list` / `runtime.qualify`） | 重复请求幂等（同键同参数命中 `action-intent/replayed`）；预算耗尽拒绝（`NV-T10-01`）；outcome-unknown 进入 reconcile / quarantine；明文 secret 不出现在任何日志 / 账本 / argv（`NV-T4-01`、`NV-T4-02`）；`policy_decision`（含 `policy_version`）附在每个准入类事件上，`policy.explain` 可重建；来源标签在 context_view 与 Effect 参数上可见，且 Policy 可基于标签产生 `approve`；负例：`untrusted_external` 来源的收件人参数直接放行 → 失败（`NV-T1-01`）；F9 负例：扩权候选不产生无 `approval/decided{result: approved_once}` 前置的 `grant/issued`（`NV-T3-02`）；子 Grant 三条不等式（`NV-T3-01`）；**未分类工具按 `external_mutation` 评估**（负例：未分类工具走快路径 → 失败）；**`conflict.idempotency`**：同幂等键不同 canonical 参数硬拒绝（负例：两次都成功 → 失败，`NV-T5-04`）；**补偿必须新 Grant + 新 ActionIntent**（负例：复用原 Grant → 失败）；**`approval/asked` / `approval/decided` 配对且缺一拒绝**（负例：`approval/asked` 落账失败后候选仍 `effect/authorized` → 失败；应为 `action-intent/rejected{reason: unavailable.approval}`）；**SecretRef 三规则**（负例：`resource.describe` / `tools/list` 描述面出现可放明文的值槽 → 失败）；**`policy.describe()` 无参返回当前 Scope 全部 `active` Policy 列表**；**`runtime/qualified` 带 `native_effect_tools[]`、`describe_digest`、`bundle_digest`**（缺字段 → 失败）；**`audit/observed` 只对 `local_read`，`external_read` 走完整协议**（负例：`external_read` 产生 `audit/observed` 而无 `action-intent/recorded` → 失败）；`secret_shaped` 结果未经 Policy 直接回传 → 失败（`NV-T4-03`）；验证独立性负例 `NV-T6-01`、`NV-T6-02` |
| **M2a MCP 守卫 + MCP-only 接入** | `fysun-mcp-guard`：`fysund` 作为 MCP 中间层（自实现，借用 agentgateway 三值语义，[ADR-0012](../adr/0012-mcp-guard-self-implementation.md)）；**基线 Runtime = DSH headless**（`dsh --profile headless` 或等价，经薄包装进程实现最小生命周期 RPC；登记查证项 Q16）；守卫系统工具 `fysun.await_approval` / `fysun.pending_inputs`；下游 MCP server 会话与 `SecretRef` 注入；可选 `fysun hook pre-tool-use` | 每个非快路径 `tools/call` 配对一个 Effect，`local_read` 快路径配对 `audit/observed`；`tools/list` 按 Grant ∩ Policy ∩ `allowed-tools` 过滤（负例：被过滤的工具不可调用，`NV-T1-02`）；**被过滤工具错误形状不可区分**（负例：与未知工具的 MCP 错误可区分 → 失败，`NV-T1-04`）；F9 负例：扩权候选不产生无 Approval 的 `grant/issued`；Runtime 无法推进 Execution 状态（负例 `NV-T9-01`）；Runtime 声明的标签高于 `fysund` 记录 → 以低者评估（`NV-T1-03`）；实现包负例 `NV-T2-01`、`NV-T2-02`、`NV-T2-03`；**下游 `notifications/tools/list_changed` 后重新过滤与重分类**（负例 `NV-T2-04`）；**`start_execution` 缺 required 控制点 / 未资格化 / Policy 拒绝 `native_effect_tools` → `not_runnable.environment{missing[]}` 大声失败**（负例：降级运行 → 失败，`NV-T9-04`）；**context_view 初始投递可见**（harness 启动输入含渲染结果，且 `resources/read` 返回同一视图；`context/assembled` 已记录）；**`needs_human` 阻塞 → `approval_wait_timeout` 超时 → `fysun.await_approval` 路径通过**（负例：超时返回成功而非 `needs_human(approval_id)` → 失败）；**`describe_digest` 变化 → `runtime/disqualified{reason: describe_changed}`**（负例：变化后无该事件仍 `start_execution` 成功 → 失败）；**hooks 路径**（可选项，不阻塞 M2a）：Policy `deny` 时 `fysun hook pre-tool-use` 退出码 2，账本记 `action-intent/rejected{reason: denied.policy, path: hook}`；abstain 时记 `audit/observed{path: hook, verdict: abstain}`；`fysund` 不可达时退出码 2；hooks 判定不被当作守卫放行（负例 `NV-T9-03`） |
| **M2b `fysun-dsh` 适配** | `fysun-dsh`：全部六个控制点（上下文注入、检查点、fork、compaction_hook、session_link、subagent → Handoff）；`fysun-approval` 在 DSH `approval/request` waterfall 拦截；`tools/post-execute` receipt 采集；`fysun` profile（`tools.restrict`、禁止 approval policy `never`）；bundle 校验 `dsh_version` / `schema_digest`；CI 每日漂移预警任务 | 跨账本 replay 双向定位；`runtime/qualified{control_points[], native_effect_tools[], describe_digest, bundle_digest}` 事件正确且 `native_effect_tools[]` 不为空；声明的控制点不能推进 Execution 状态（负例 `NV-T9-02`）；**`approval/request` 拦截不 `next()`，DSH 原生审批卡片应答不落账、不解除阻塞**（负例 `NV-T11-02`）；**DSH 原生工具经拦截点产生 ActionIntent、经 `tools/post-execute` 产生 receipt**（正例；负例：无 ActionIntent 的原生工具调用被放行 → 失败）；**`fysun` profile 的 `tools.restrict` 对被过滤原生工具不可区分**（负例）；**DSH approval policy = `never` 时 `fysun-dsh` 拒绝启动或 `runtime/disqualified{reason: adapter_precondition_failed}`**（负例：仍正常启动 → 失败，`NV-T9-05`）；**`context_injection` 每次注入产生一条带 `context_view.digest` 的 DSH 会话事件**（负例：注入后无该事件 → 跨账本 replay 失败）；**bundle `dsh_version` / `schema_digest` 不匹配拒绝启动**（负例）；**CI 漂移预警任务存在**（对上游最新 tag 跑 conformance，非阻塞） |
| **M3 Activity 闭环** | Activity / Execution 状态机（含 `input_required`、`paused{reason}`、`waiting{reason}`）、周期触发、检查点、恢复顺序（含**重新武装**）、Handoff、Approval 闭环（含**审批限流与连续拒绝告警**）、A2A Handoff 对齐 | 上述闭环第 1–7、9、10 步全部通过；崩溃后不重放已 receipt 的 Effect（`NV-T5-03`）；闭环第 2 步"先看过再发布"通过 Approval 对象实现（不再是策略注释）；Handoff 状态可无损映射到 A2A TaskState；崩溃恢复后未决 Approval 仍为 `pending`，不被视为已批（一条 `pending` Approval 对应的 Effect 出现 `effect/authorized` 而无 `approval/decided{result: approved_once}` → 失败）；**Execution `waiting.reason` 正确**（负例：因 `pending` Approval 等待却 `waiting{reason: trigger}` → 失败）；**恢复缺省 `paused{reason: rearm_required}`，未经 `activity/resumed` 不出现新 `effect/authorized`**（负例；纯只读 Activity 直接 `waiting`）；**审批限流与连续拒绝告警**（负例 `NV-T10-03`：超过 `approval_rate_limit` 后仍出现新 `approval/asked` → 失败；连续拒绝达 `consecutive_rejections_alert` 后 Activity 未 `blocked{reason: consecutive_rejections}` → 失败）；`max_pending_approvals` 超限（`NV-T10-02`）；**deliverables 脱敏**（负例：`summary` > 4096 字节或含栈迹 / 环境变量 / 宿主绝对路径 → 失败）；**`handoff_accept` 按触发条件产生 Approval**（负例：条件成立却直接 `handoff/working` → 失败；条件不成立却产生 Approval → 失败）；聊天通道"批准"文本不产生 `approval/decided`（`NV-T11-01`）；Approval 后的 Grant 仍 ⊄ 父 Grant → 拒绝（`NV-T3-03`）；记忆投毒负例 `NV-T7-01`、`NV-T7-02` |
| **M4 演进与入口** | Adjustment（apply / revert / promote、`predicted_effect` / `evaluated_against`、`evidence_gate`）、`fysun` CLI、最小 Web 状态页（Approval 呈现面优先） | 闭环第 8 步；Adjustment 触及扩权时被 Policy 拦截（负例）；一次 Adjustment 带 `predicted_effect`，下一 Execution 的 Verification 自动填充 `evaluated_against`；负例：`evidence_gate` 开启时未跑回归集的 promote 被拒；**`predicted_effect` 为空 → `evaluated_against.result` 恒 `inconclusive`**（负例：出现 `supported` / `refuted` → 失败）；**给出 `predicted_effect` 但缺 `statement` → 拒绝**（负例：被接受 → 失败）；**`approve_for_capability` 派生的 Adjustment 自动 `statement = approval.reason`**；低于 Policy 最低 verifier 要求 → `inconclusive`（`NV-T6-03`） |
| **M5 第一个应用** | Personal 作为应用切回 FysunOS（角色 / 团队 / 业务对象在应用层） | Personal 不再维护第二套执行路径 |
| **边缘阶段（M3 之后，非 MVP）** | `fysund-edge`（同一账本模型，内含确定性包络校验器）；`Node` 登记与 `node/*` 事件；escrow 子 Grant；离线 Policy 评估（`offline = true`）；按 `origin_seq` 推送合并、digest 链校验、`ledger/merged` / `ledger/conflict`；边缘 Runtime（实现最小面，[07 §6](07-runtime-onboarding.md)）；安全包络 Policy 绑定设备 Resource | **escrow 越界拒绝**（负例：消耗超出 escrow `budget` 被放行 → 失败；escrow `budget > parent.remaining` 的 `grant/issued` → 失败）；**离线 `approve` 阻塞**（负例：离线期间 `approve` 类候选被放行 → 失败；edge 本地写出 `approval/decided` → `home_violation`）；**`ledger/merged` 与 `ledger/conflict{kind: revoked_offline}` → Activity `blocked{reason: revoked_offline}`**，不自动补偿（负例：触发 `effect/compensated` → 失败）；合并后 `origin_seq` 不连续未报 `ledger/conflict` → 失败；中心改写 edge 事件 → 失败（[11 §3.6](11-edge-and-offline-ledger.md)）；**T8 全部负例**（`NV-T8-01`、`NV-T8-02`、`NV-T8-03`） |

阶段可并行局部推进，但 M2a 完成前不做 `fysun-dsh` 适配；M3 之前不做边缘。M2a 的基线固定为 DSH headless（[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D8），
Claude Code / Codex / Goose / OpenCode 等只作候选，均含原生工具，接入时须如实声明 `native_effect_tools[]`，不作为 M2a 验收对象。
负例向量的编号与归属见 [13 威胁模型 §5](13-threat-model.md)；里程碑列中直接标注了已分配编号的向量，未标编号者在实现阶段按 `NV-T<n>-<序号>` 登记。

## 4. 效率与性能的度量

"精简高效"必须可测。基线先测、阈值后定，不能仅凭架构宣称性能提升：

- daemon 启动耗时、空闲内存；
- 调度延迟（trigger → `start_execution`）；
- 每次工具调用的账本事务数与 MCP 守卫往返延迟；
- 崩溃恢复时间（含重新武装步骤）；
- 接入一个新 Capability 所需步骤数；
- 人工介入率（`approval/asked` 数 / Execution 数）；
- 回滚率（`adjustment/reverted` 数 / `adjustment/applied` 数）；
- 验证独立性比例（`independence != not_independent` 的 Verification 占比）；
- **审计快路径占比**（`audit/observed` 数 / (`audit/observed` + `action-intent/recorded`)）；
- **审批不可用率**（`approval/decided{result: unavailable}` 数 / `approval/asked` 数）；
- **重新资格化次数**（`runtime/disqualified` 数，按 `reason` 分组）。

自人工介入率起的六项直接从治理账本事件折叠，并经 OTLP 导出（见 [03 §8](03-architecture.md)）；不另建统计表。
`gen_ai.*` 与 `fysun.*` 属性命名空间的长期关系为开放问题（§7 Q15）。

## 5. 证据纪律（迁移自 CognitiveOS A7，作为流程规则）

- 本地、fixture、普通 CI 证据只推进实现进度；任何"支持"、"发布"、"Gate"类声明需要预登记的环境与流程。
- `not-run` 永不等于 pass；被解析器拒绝的命令记 `not-run`，不是测试失败。
- 每个测试 / 验证单元完成后立即追加到该活动唯一的 running report。
- 审查报告引用的所有外部基准数字（AgentDojo、Terminal-Bench、LoCoMo 等）只说明方向，不构成对 FysunOS 的任何性能或安全声明。

**四类状态用语**（借鉴 CognitiveOS conformance 方法论；[13 §6](13-threat-model.md) 与本节一致使用，不再使用"满足 / 部分"）：

| 状态 | 定义 | 推进条件 |
|---|---|---|
| `specified` | 设计文档已给出可判定的契约、事件与负例；尚无实现代码 | 契约与负例在 00 / 02 / 对应设计篇落定 |
| `implementation available` | 存在实现代码，本地 / fixture / 普通 CI 证据可用；对应负例尚未在预登记环境执行 | 实现代码与其负例测试进入仓库 |
| `test executed` | 正例与负例已在预登记的环境与流程中执行，结果（pass / fail / `not-run`）记入 running report | 预登记环境存在且每次执行留痕 |
| `implemented` | `test executed` 且全部相关负例 pass、无 `not-run`，声明进入版本发布记录 | 只有此状态可对外表述为"已实现" |

状态只能逐级推进；任一负例 fail 或 `not-run` 使该项退回 `test executed` 之前的状态。**本设计稿（v0.7.0）期间 13 §6 的全部条目为 `specified`。**

## 6. 非声明

本设计不声明：

- FysunOS 已实现任何代码；
- DSH 已被接入或已通过资格化；
- 任何 Runtime 已产生 `runtime/qualified`（无论是否声明控制点）；
- **DSH headless 原生工具清单已查证**（Q16）；
- **任何 hooks 路径构成安全边界**（它只是对原生工具的 Policy 阻断补充，不计入 conformance 主路径）；
- Cordis 已成为依赖；
- 任何性能、任务完成率、协作收益、边缘可用性数字；
- 13 §6 conformance 表中任何一项处于 `specified` 之外的状态；
- Personal 已迁移。

## 7. 开放问题（统一清单）

本节是全仓库唯一的开放问题登记处（吸收原威胁模型篇的三条开放问题）；13 与各 ADR 只引用本节。编号只增不复用：Q1–Q10 沿用 v0.6.0 编号，
Q7–Q10 的处置见 §7.2，Q11 起为 v0.7.0 新增。各 ADR"待后续 ADR 的问题"段中未升入本表的细项（如 `auto_resume` 的粒度、`flags` 分类规则的归属、
`audit/observed` 的保留策略、escrow 自动续期、同一工具按参数拆类别）以该 ADR 为登记处，需要跨篇决策时再升入本表。

### 7.1 保留（待后续 ADR）

1. **Q1 传输**：`fysund` 与 Runtime 之间的传输，拆为两问：(a) MCP 传输——stdio + Streamable HTTP，跟随 MCP 规范；
   (b) 最小生命周期 RPC——stdio JSON-RPC（MVP），是否同时支持本地 socket / gRPC 待定；`fysun hook pre-tool-use` 与 `fysund` 之间的本地 RPC
   沿用 (b) 的选择。
2. **Q2 Secret Store 后端**：首选实现（OS keyring vs 文件加密）；任何后端都必须满足 `SecretRef` 三规则（[00 §5](00-terminology.md)）。
3. **Q3 Web UI**：技术栈与是否复用 Personal Control Plane 的组件；Approval 呈现面为 UI 第一优先级，且审批面只由 `fysund` 提供
   （[ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md) §D5）。
4. **Q4 质量目标表示法的剩余问题**：已由 [10 §1.5](10-evolution.md) 采纳 Skill Contract 骨架 + checklist 化 V + `verifier_kind` 标注；
   剩余为 `model_judge` 校准集的来源（谁构造、多大、如何更新），以及 judge 校准阈值是否写入 Policy。
5. **Q5 多主体 Scope**：多主体（多人 / 多设备）Scope 模型何时激活；对外暴露 A2A server（签名 Agent Card）是多主体的第一个触发点。
6. **Q6 许可证**：许可证选择；兼容性说明：Agent Skills 规范与 Cedar 为 Apache-2.0，DSH / Cordis 为 MIT。

### 7.2 已关闭

7. **Q7 策略语言子集——关闭**（[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md)）：Cedar 子集的语法范围、实体模型、context 固定键、
   `outcome` 四值的注解表达约定与 schema 校验已定。剩余两项转为 Q17（`modify` 改写表达式形式）与 Q18（`obligations[]` 枚举全集）；
   原子项"与 CEL 的互操作"随 agentgateway 复用关闭而删除（[ADR-0012](../adr/0012-mcp-guard-self-implementation.md) §D4）。
8. **Q8 `Intent` 改名——删除**：已由 [ADR-0006](../adr/0006-terminology-action-intent.md) 决定 `ActionIntent`，无剩余问题；编号保留以备追溯，不复用。
9. **Q9 边缘账本离线一致性——关闭**（[ADR-0010](../adr/0010-edge-offline-ledger-consistency.md)；正文 [11 §3](11-edge-and-offline-ledger.md)）：
   节点归属唯一、`node_id` / `origin_seq`、escrow 预留预算、离线行为、合并与封闭冲突枚举已定；属边缘阶段验收（§3）。
   多主体 / 多节点下的 Approval 决定权与 Policy 分发转为 Q12。
10. **Q10 agentgateway 复用形态——关闭**（[ADR-0012](../adr/0012-mcp-guard-self-implementation.md)）：`fysun-mcp-guard` 自实现，
    仅借用 `Pass` / `Reject` / `Mutated` 三值语义与 Cedar 策略格式；不作为库嵌入、不进程前置。

### 7.3 新增（v0.7.0）

11. **Q11 Trust 对象**：按 Runtime / 工具 / 外部 Agent 的证据化信任何时引入，以及它与来源标签、Runtime 资格化纪元的关系；MVP 单主体不做，
    多 Runtime 资格化时需要。对外坐标中 Control & Governance Plane 因此表述为"供给（Trust 除外）"（[01 §1.4](01-definition-and-principles.md)、
    [02 §4](02-object-model.md)）。
12. **Q12 多主体 / 多节点分布式一致性**：哪些决定必须在线由中心 `fysund` 裁定、哪些可在 `fysund-edge` 本地先行；edge 上谁是 Principal 及其
    Approval 决定权范围；Policy 分发与版本一致性。[ADR-0010](../adr/0010-edge-offline-ledger-consistency.md) 只解决单主体星型拓扑并把
    "edge 本地 Principal 决定 Approval"列为否决项；与 Q5 相关。
13. **Q13 完整信息流控制（CaMeL 级）**：是否需要 Level-3 控制点，以 M1–M2a 的负例结果与自适应攻击评估为依据再决定；当前只做协议边界上的
    标签 + 引用监视器（ADR-0004 §D6）。
14. **Q14 Provider 路由 / fallback**：`fysund` Provider 代理是否承担多 Provider 路由、失败切换与成本策略，还是留给 Runtime；与 F5 范围
    （只覆盖经代理的请求）联动。
15. **Q15 `gen_ai.*` 与 `fysun.*` 的长期关系**：OTLP 导出中 OpenTelemetry GenAI 语义约定与 FysunOS 自有属性命名空间的对齐、替代或并存；
    守卫系统工具前缀 `fysun.*` 与 OTLP 命名空间 `fysun.*` 为不同域（03 §8）。
16. **Q16 DSH headless 原生工具查证**：需查证 DSH headless 模式是否有独立于 bundle 的原生工具；结果决定 M2a 基线 Runtime 的
    `native_effect_tools[]` 实际内容，不影响规则（[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D8）。
17. **Q17 `modify` 改写表达式形式**：本轮只允许枚举形式 `redact:<param>`、`clamp:<param>:<max>`（写在 `@fysun_obligation` 字符串中）；
    通用改写表达式待有需求再定（承 Q7）。
18. **Q18 `obligations[]` 枚举全集与核对规则**（承 Q7 与 ADR-0003 待后续）。
19. **Q19 `on_expiry = escalate` 的升级目标**：另一 Principal、另一载体如何声明（承 ADR-0005 / ADR-0011 待后续）。
20. **Q20 `downstream_outcome` verifier 的时间窗与归因规则**（承 ADR-0005 待后续）。
21. **Q21 `approval_result_ttl` 超期后已 `authorized` 但未分发的 Effect 的作废语义（待确认）**：MCP-only Runtime 下决定到达而无开放通道时，Effect 停在
    `authorized` 等待 harness 重提或轮询，受 Policy `approval_result_ttl` 约束（[07 §3.4](07-runtime-onboarding.md)、[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D5）。
    07 §3.4 采用的落地为：超期且 Approval `on_expiry = deny`（缺省）→ `fysund` 以 `effect/reconciled`（对账结论"未分发、无外部效果"）关闭该 Effect，
    之后同幂等键候选按新候选重新准入；`keep_waiting` / `escalate` → 放行保持有效直至 Execution 终止或 Approval `cancelled`；作废不产生第二条 `approval/decided`。
    待确认项：用 `effect/reconciled` 表达"从未分发即作废"是否会与 outcome-unknown 对账语义混淆，是否需要独立的 Effect 终态或 `reconciled` 子原因；
    确认前 07 §3.4 的写法为实施基准。
