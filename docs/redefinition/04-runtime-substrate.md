# 04 · Runtime 基座：接入、资格化、生命周期、执行点、账本

- 状态：提案草案 v0.8.0-draft（2026-09-11）
- 决策依据：[ADR-0013 草案](adr-0013-draft-repositioning.md) §D3、§D4、§D5、§D8
- 前置：[00 术语表](00-terminology.md)、[02 能力资产模型](02-asset-model.md)、[03 组合](03-composition.md)
- 继承：v0.7.0 [03 架构 §5–§6](../design/03-architecture.md)（账本与恢复）、[05 Runtime ABI](../design/05-runtime-abi.md)、[07 §1、§7](../design/07-runtime-onboarding.md)（资格化与钉版）、[11 §3](../design/11-edge-and-offline-ledger.md)（边缘）

本文回答"Runtime 控制"这一半职责：Harness 怎样成为可被控制的 Runtime、基座控制什么、不控制什么、数据面拦截由谁承担、事实如何进账本。

## 1. Harness → Runtime

```text
harness 资产（published）──install 到 Node──▶ runtime.register(descriptor) ──▶ asset/registered{kind: runtime}
   ──▶ runtime.qualify(runtime_id) ──describe()──▶ 形状校验 · describe_digest 重算 · bundle_digest · 与 harness manifest 比对 · Policy
   ──▶ runtime/qualified{runtime_id, harness_asset_id, control_points[], native_effect_tools[], describe_digest, bundle_digest, policy_decision}
```

- `describe() -> {type, version, control_points[], native_effect_tools[], describe_digest}`（五字段字面继承）。新增校验：`describe()` 的 `control_points[]`
  与 `native_effect_tools[]` 必须与该 Harness 资产 manifest 的 `provides.control_points[]` / `native_effect_tools[]` 一致，不一致 → 拒绝资格化
  （资产声明与实例声明互证）。
- 资格化纪元与四种 `runtime/disqualified.reason`（`describe_changed` / `bundle_changed` / `policy_revoked` / `adapter_precondition_failed`）继承；
  新增第五种 `harness_asset_revoked`（该 Harness 资产版本被 `asset/revoked`）。
- 多 Runtime 并存、资格化证据不跨 id 转移、精确钉版与 CI 漂移预警继承 v0.7.0 07 §1.4、§7。
- Runtime 归属 Node；`fysund` 可管理多个 Node 上的 Runtime（`cloud` / `central` / `edge`），资格化恒由中心完成。

## 2. Runtime ABI 最小面

| 方向 | 方法 | 说明 |
|---|---|---|
| OS → Runtime | `describe()` | 五字段 |
| | `start_execution(execution, lock, bindings, grants_summary, inputs)` | 携带 Lock（资产 digest 集）与绑定；**不携带渲染好的上下文**——Harness 按 Lock 自行装配 prompt / skills；大声失败清单继承（缺控制点、未资格化、Policy 拒绝原生工具、Lock 含被隔离资产） |
| | `resume_execution(execution, checkpoint_ref, watermarks)` | Effect 水位 + Approval 水位继承 |
| | `cancel(execution, reason)` / `health()` | |
| | `deliver_input(execution, input)` | **可选**：Harness 若支持进程外输入则实现；否则 Runtime 以 `execution.observe` 轮询或由适配器投递。基座不再规定投递管道 |
| | `checkpoint(execution)` | 可返回 `unsupported` |
| Runtime → OS | `os.report_progress` / `os.report_failure`（脱敏 ≤ 4096 B）/ `os.request_resource` / `os.propose_adjustment` / `os.submit_handoff` | 继承 |
| | `os.observe(execution)` | 查询 Approval 结果、待投递输入、Grant 摘要、预算余量——替代 `fysun.await_approval` / `fysun.pending_inputs` 系统工具 |
| | `os.report_observed` / `os.submit_candidate` / `os.report_receipt` | 仅当 Harness **自身作为执行点**（`harness_adapter` 类 EP）时使用；见 §4 |

不得调用清单继承：Runtime 不能签发 Grant、推进 Execution 状态、决定 Approval、改写标签、写权威账本。

六个控制点与义务继承 v0.7.0 05 §3；`context_injection` 的语义改为"OS 提供**结构化视图**（Lock、Grant 摘要、Activity.continuity、Memory 检索结果，各带 label），
Harness 负责渲染"，OS 不再产出渲染文本。

## 3. Activity / Execution 生命周期

继承 v0.7.0 02 §2.2–2.3、03 §2.2、§6：状态机、`waiting{reason}`、`input_required`、`paused{reason: rearm_required}`、检查点双水位、恢复顺序（账本投影 → Policy 版本 →
Runtime 资格化 → fencing → outcome-unknown 对账 → Approval 水位 → 重新武装 → `resume_execution`）、审批限流与连续拒绝告警、Handoff 与 A2A TaskState 对照。

变化：

- `Execution.runtime_binding` 增 `lock_digest`、`harness_asset_id`、`ep_ids[]`（本次 Execution 生效的执行点）。
- `Execution.tenant_id`、`Activity.tenant_id`、`Activity.service_ref?`（若由 Service 调用创建）。
- 调度：`fysund` 拥有触发（人工、周期、事件、webhook、**Service 调用**）与 Runtime 选择（`compose.match` 结果 × Policy × 负载），
  Runtime 池化策略（每租户独占 / 共享）由 Policy 决定（§6）。
- 预算：Grant 预算由 Plan（服务调用）或 Activity `constraints`（直接使用）派生；`fysund` 记账，执行点回报消耗。

## 4. 执行点（Enforcement Point，EP）

### 4.1 定义

执行点是**站在数据面上**、按 `fysund` 下发的 Policy 与 Grant 强制边界、并把事实回报 `fysund` 的组件。`fysund` 自身不是执行点。

| `kind` | 位置 | 强制什么 | 回报什么 | 参考实现 |
|---|---|---|---|---|
| `gateway` | 服务入口 | 租户鉴权、配额、速率、Plan 允许的 `effect_category`、Endpoint 协议 | `invocation/*`、`meter/recorded` | `fysun-gateway` |
| `mcp_guard` | Harness 与工具 / MCP server 之间 | `tools/list` 过滤（Grant ∩ Policy ∩ 工具面，不可区分性）、`tools/call` pre（ActionIntent 持久化、类别合成、Policy、Grant、标签）/ post（receipt、结果标签与 `flags`、`modify` / `deny`）、下游凭据注入、下游会话持有 | `action-intent/*`、`effect/*`、`audit/observed{path: guard}`、`resource/metered` | `fysun-mcp-guard`（独立进程，可选） |
| `hooks` | Harness 进程内 hook | 对原生工具 deny / abstain | `action-intent/rejected{path: hook}`、`audit/observed{path: hook}` | `fysun hook pre-tool-use` |
| `harness_adapter` | Harness 内部拦截点 | 原生工具的持久化 + receipt（如 DSH `approval/request` + `tools/post-execute`）、控制点义务 | 同 `mcp_guard`，`path: adapter` | `fysun-dsh` |
| `provider_proxy` | Harness 与模型 Provider 之间 | 凭据注入、Token / 费用计量、模型允许列表 | `resource/metered` | 可选组件 |
| `external_gateway` | 第三方网关（如 agentgateway） | 接受 Policy 快照（Cedar）并按 EP 契约回报 | 同 `mcp_guard` 子集 | 待评估（重开 ADR-0012） |

### 4.2 EP 契约

```text
ep.describe() -> {kind, version, capabilities[], describe_digest}          # capabilities: filter_tools / persist_intent / receipt / labels / flags / secret_inject / meter
ep.policy_pull(scope) -> {policy_version, digest, bundle}                  # 版本化 Cedar 快照 + Grant 有效集投影 + 工具面（Lock 派生）
ep.report(events[])                                                        # 批量回报：action-intent/* · effect/* · audit/observed · meter/recorded · invocation/*
ep.resolve_secret(SecretRef, execution_id) -> value                        # 只对 qualified EP、按 Grant；每次重解析
ep.health()
```

规则：

- **资格化**：`ep.register` → `ep.qualify`（`describe_digest` 重算、Policy 评估、必要能力位检查）→ `ep/qualified`；能力位变化 / Policy 换版 → `ep/disqualified`。
  未资格化 EP 的回报被拒（`ep.report_rejected`），其 `resolve_secret` 被拒。
- **Policy 下发为主**：EP 拉取快照本地评估，回报事件带 `policy_version`；`fysund` 在 `policy/revised` 后推送失效通知，EP 在 `policy_stale_ttl` 内未刷新则自行 fail-closed
  （只允许 `local_read`）。`outcome = approve`、任何 Grant 扩张、Memory 准入中需人的部分**回中心**：EP 提交候选，`fysund` 签发 Approval，EP 以 `os.observe` 语义等待或返回 `needs_human(approval_id)`。
- **persist-before-dispatch 在 EP 本地成立**：EP 先本地持久化 ActionIntent / Effect（带幂等键、`policy_version`、`ep_id`）再分发，随后回报；`fysund` 以回报重建 Effect 投影。
  EP 崩溃恢复时先回报未回报事件再接受新调用；`fysund` 对"有 `dispatch-started` 无 receipt"的 Effect 走 outcome-unknown 对账（继承）。
- **回报核对**：`fysund` 对每条回报校验 `ep_id` 资格化、`policy_version` 在允许窗口、幂等键唯一（`conflict.idempotency`）、Grant 未过期；核对失败 → `ep.report_rejected` 并写 `ledger/conflict{kind: ep_report_rejected}`。
- 一个 Execution 可同时有多个 EP（gateway + mcp_guard + hooks）；同一工具调用只能由一个 EP 产生 Effect（EP `capabilities` 与 Lock 决定归属），重复 → `conflict.idempotency`。

### 4.3 EP 与 F3 / F7 / F9 的关系

F3 / F7 / F9 的可判定范围 = 该 Execution 生效 EP 覆盖的工具面之并集。conformance 表按 **EP 组合**分列（§4.5），取代 v0.7.0 按 Runtime 类型分列。
无 EP 的 Runtime：只有生命周期控制、预算（以时间与调用数计）、账本；F3 / F7 / F9 声明为 `not_covered`，Policy 可拒绝把此类 Runtime 用于含非 `local_read` 效果的 Lock。

### 4.4 `fysund` 的热路径禁令

`fysund` 不转发 `tools/call`、不代理模型请求、不持有下游 MCP 会话、不渲染上下文。它与数据面的全部交互是：下发（`ep.policy_pull`）、接收回报（`ep.report`）、
解析 secret（`ep.resolve_secret`）、签发 Approval / Grant。负例：`fysund` 进程出现在任一工具调用或模型请求的转发链上 → 失败（ADR-0013 负例 2）。

### 4.5 conformance 分列（骨架）

| EP 组合 | F3 | F5 | F7 | F9 | 备注 |
|---|---|---|---|---|---|
| `mcp_guard` + `gateway` | MCP 工具面 | 经 `resolve_secret` 的 secret | MCP 工具面 | MCP 工具面 | 原生工具 `not_covered` |
| `mcp_guard` + `hooks` | 同上 | 同上 | 同上 + 原生工具阻断（尽力） | 同上 | hooks 只拦不记 |
| `harness_adapter`（`fysun-dsh`） | MCP + 原生工具 | 同上 | 同上 | 同上 | 依赖适配器前置条件 |
| `harness_adapter` + `provider_proxy` | 同上 | 同上 + 模型凭据 | 同上 | 同上 | |
| 无 EP | `not_covered` | 仅 OS 管理 secret 不外泄 | `not_covered` | `not_covered` | 只有生命周期与账本 |

状态用语继承四类（`specified / implementation available / test executed / implemented`）；本提案期全部 `specified`。

## 5. Policy 下发模型

- Policy 对象、版本、Cedar 子集、`outcome` 四值、`policy_decision` 随准入事件继承（ADR-0003 / 0009）。
- 实体模型扩展：principal 增 `Tenant`、`EnforcementPoint`；resource 增 `Asset`、`Service`、`Endpoint`；context 增 `tenant_id`、`plan_ref`、`ep_kind`、`lock_digest`（[06 §3](06-governance.md)）。
- 下发单位 = `(scope, policy_version)` 快照 + Grant 有效集投影 + Lock 派生的工具面；EP 记录 `ep/policy-pulled`。
- 边缘 `fysund-edge` 是"带本地账本的 EP 宿主"，离线规则继承 11 §3（escrow Grant、`approve` 阻塞、`origin_seq` 合并）。

## 6. 多租户隔离（运行侧）

- 每个 Activity / Execution 有 `tenant_id`；Grant 绑定 `(principal, scope)`，scope 含 tenant 前缀；跨租户 Handoff 必须经 Service 调用（A2A），不能直接引用对方 Execution。
- Runtime 池化：Policy 决定某 Service 用 `dedicated`（每租户独立 Runtime 实例）或 `shared`（同一 Runtime 顺序服务多租户，要求 Harness 声明 `session_isolation` 能力位）。
  缺省 `dedicated`。
- Memory / Knowledge：`memory.scope = tenant` 时 `memory_provider` 必须支持命名空间隔离；跨租户 Memory 读取一律拒绝。
- Secret：`SecretRef` 归属 tenant scope；EP `resolve_secret` 校验 Execution 的 tenant。

## 7. 账本

- 治理账本（`fysund`，event-sourced，SQLite WAL 单机 / 可替换存储多节点）与会话账本（Harness）关联不合并——继承。
- 新增事件族见 00 §5；EP 回报事件保留 `ep_id` / `policy_version`；跨账本 replay 对 MCP-only Harness 退化为 `tools/call` 配对 + `audit/observed`（继承）。
- 投影新增：资产目录与 health、Lock、Runtime 与 EP 资格化状态、Service 目录、租户用量。

## 8. 负例（R1 / R2 / R4）

1. `describe()` 声明与 Harness 资产 manifest 不一致却 `runtime/qualified` → 失败（R1）。
2. `start_execution` 载荷含渲染文本上下文而非结构化视图 + Lock → 失败（R2）。
3. 未资格化 EP 的 `ep.report` 被接受 → 失败（R4）。
4. EP 本地写出 `approval/decided` 或 `grant/issued` → 失败（R4）。
5. `policy_stale_ttl` 超期的 EP 放行非 `local_read` 候选 → 失败（R4）。
6. 同一工具调用由两个 EP 各产生一个 Effect → 失败（R4）。
7. `shared` 池化下 Harness 无 `session_isolation` 能力位仍被绑定多个租户 → 失败（R3）。
8. `fysund` 进程出现在工具调用或模型请求转发链 → 失败（R2）。
