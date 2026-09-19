# 01 · 内核最小集

- 状态：**提案草案**
- 日期：2026-09-12
- 本文件不替代现行 [`docs/redefinition`](../redefinition/README.md) 直至被采纳。
- 决策：[00](00-decisions.md) D1/D3/D4/D5。继承 v0.7.0 [03](../design/03-architecture.md) 账本与恢复、[05](../design/05-runtime-abi.md) ABI 骨架。

## 1. 结论：内核画在哪

内核是控制面。它不跑模型、不执行工具、不持有 Loop 状态。它保证：谁被允许组合什么、哪个 Runtime 可以跑这个 Lock、每次外部效果是否可审计、服务调用是否落在租户与配额内。

```text
消费者  A2A / MCP / HTTP / CLI
        │
        ▼
┌─ 服务入口 EP（fysun-gateway）──── 数据面 ──────────────────────────┐
│  鉴权 · 配额 · 计量事件 · 调用 → activity.create_or_wake            │
└──────────────┬─────────────────────────────────────────────────────┘
               │ 仅创建/唤醒（控制面 RPC，非每步 tool call）
               ▼
┌─ fysund 控制面 ────────────────────────────────────────────────────┐
│  Catalog：登记 / resolve / 分层 Lock                                │
│  Runtime 控制：qualify · 调度 · Activity/Execution · 恢复/再武装     │
│  Governance：Grant · Policy PDP · Approval · 账本写入                │
│  Service 目录：publish / rollout / Plan→根 Grant                    │
└───────┬───────────────────────────────┬────────────────────────────┘
        │ start/resume + 结构化绑定      │ policy_pull / 核对回报 / secret
        ▼                               ▼
┌─ Runtime（Harness 进程）─┐    ┌─ 工具侧 EP（mcp_guard / adapter / hooks）─┐
│  装配上下文 · 跑 loop     │    │  工具面过滤 · 分级 F3 · sampling 防御     │
│  按需加载 skill           │◄──►│  本地评估 Policy 快照 · 回报事件          │
└──────────────────────────┘    └────────────────────────────────────────┘
```

**热路径**（每步推理 / 工具调用）：Runtime ↔ 工具侧 EP ↔ MCP server / 原生工具。不经过 `fysund`。

**冷路径**：发布、解析、资格化、换 Policy、审批、恢复、发布金丝雀。

## 2. 平面（三个 + 横切）

| 平面 | 一级对象 | `fysund` 做什么 | 热路径？ |
|---|---|---|---|
| 资产/组合 | Asset、Manifest、Capability（契约）、Agent Definition、Lock、Loop Profile（非 kind） | 登记、解析、锁定 | 否 |
| 运行 | Node、Runtime 实例、EP、Activity、Execution、Effect 协议投影 | 资格化、调度、状态机、核对回报 | 否（调度除外） |
| 服务 | Service | 目录、Plan 附着、rollout | 否 |
| 治理（横切） | Principal、Scope、Tenant、Grant、Policy、Approval、Verification、Event | PDP、审批面、唯一写入 | 否 |

Resource（设备、账号、工作区路径、SecretRef）仍是授权与计量对象，不是资产。

## 3. 执行点

执行点站在数据面，按已下发快照强制边界，把事实回报 `fysund`。`fysund` 自己不是执行点。

| kind | 强制 | 回报 | MVP |
|---|---|---|---|
| `gateway` | 租户、配额、Plan 允许的 `effect_category`、协议 | `invocation/*`、`meter/recorded` | R2 |
| `mcp_guard` | `tools/list` 不可区分过滤；`tools/call` 分级 F3；MCP `sampling/createMessage` 预算与上下文隔离 | `action-intent/*`、`effect/*`、`audit/observed` | R2（或由 adapter 覆盖） |
| `hooks` | 原生工具 deny/abstain | `action-intent/rejected{path:hook}` | 可选 |
| `harness_adapter` | 原生工具 persist+receipt、控制点义务 | 同 mcp_guard，`path:adapter` | R1 可用 DSH 适配器 |
| `provider_proxy` | 模型凭据与 token 计量 | `resource/metered` | 可选；自报 token 标 `untrusted_external` |

契约最小面：`describe` / `policy_pull` / `report` / `resolve_secret` / `health`。资格化与 `policy_stale_ttl` 继承 redefinition 04 §4.2：过期只允许 `local_read`。`approve` 与 Grant 扩张必须回中心。同一工具调用只能由一个 EP 产生 Effect。

**MCP sampling（R2 义务）。** Server 发起的采样视为 `untrusted_external` 候选；token 计入当前 Execution 预算；不得把 `trusted_system` / Secret / Slot-0 身份提示回传给 server。超出 → `quota.exceeded{dimension:sampling}`。对应 OWASP ASI01/ASI02/ASI06。

无 EP 的 Runtime：只有生命周期与账本；F3/F7/F9 标 `not_covered`；Policy 可拒绝把含非 `local_read` 的 Lock 绑到它。

## 4. 资格化与工具面

Runtime 资格化：`describe()` 五字段与 **Harness 资产** manifest 互证（`control_points[]`、`native_effect_tools[]`、`describe_digest`、`bundle_digest`）。不一致 → 拒绝。失格理由继承 v0.7.0，并加 `harness_asset_revoked`。

有效权限（F9）在 **`start_execution` 之前** 算完：

```text
tool_surface = Lock 声明的工具（MCP 快照 ∩ skill.allowed-tools ∩ 未禁用项）
effective    = Grant ∩ Policy 允许集 ∩ tool_surface
```

结果写入 EP 快照。之后 `tools/list` 只是本地过滤；看不见 ≡ 不存在。扩张只能经 `approved_once` 且仍 ⊆ 父 Grant。

绑定载荷是**结构化视图**（Lock、Grant 摘要、Activity.continuity、Memory 检索引用，各带 label），**不是**渲染好的 prompt。Harness 自己装配。OS 可附带 `prefix_order[]` 建议（身份 → 工具 schema 索引 → 静态 skill 元数据 → 记忆 → 对话）；Harness 若声明 `cache_alignment` 应遵守。未声明者不因此失格。

## 5. Activity / Execution 最终形态

| | Activity | Execution |
|---|---|---|
| 是什么 | 长期工作：目标、租户、service_ref、连续性 | 一次运行 |
| 谁写 | `fysund`（调用、触发、审批唤醒） | `fysund`（start/resume/cancel） |
| 谁读 | 网关、审批面、计量投影、演进归因 | Runtime ABI、EP、恢复器 |
| 热路径 | 否 | 否（状态在控制面；进程在数据面） |
| MVP | 需要 | 需要 |

状态机继承 v0.7.0：`waiting` ≠ `input_required`；检查点双水位（Effect + Approval）；恢复顺序：投影 → Policy 版本 → Runtime 资格化 → fencing → outcome-unknown → Approval 水位 → 重新武装 → `resume_execution`。含非 `local_read` 的 Activity 恢复后缺省 `paused{reason:rearm_required}`。

`session_park`（可选）：长时间等人时 Runtime 返回 `ExecutionSnapshot`（对话摘要、未完成 step、待输入标记；不含权重/KV）。快照入 Artifact；进程释放。唤醒：任意合格 Runtime + `thaw` 或 `resume_execution`。Park 不是透明迁移 Loop 状态——不承诺 KV cache 或模型内部状态（延续 ADR-0001）。

调度：`compose.match(lock, node)` × Policy × 负载。缺省每租户 `dedicated` Runtime；`shared` 要求 Harness 声明 `session_isolation`。

## 6. 治理对象（继承，提案内自洽所需的最小语义）

读者不必翻 v0.7.0 也能实现内核。以下为仍有效的压缩陈述；字段细节实现期出 schema。

**Effect 类别（六值，[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md)）**  
`local_read` · `external_read` · `local_mutation` · `external_mutation` · `credential_use` · `physical_action`。合成三源：资产/`tool_effects[]` ∪ 目录缺省 ∪ Policy；取最高严重度；未分类 = `external_mutation`。

**Grant**  
绑定 `(principal, scope)`。衰减三条不等式：scope ⊆ 父、expiry ≤ 父、budget ≤ 父剩余。可带 `root_of: plan_ref`。Execution 是可授权 Principal，故委托有独立预算。补偿必须新 Grant + 新 ActionIntent，禁止复用被补偿 Effect 的 Grant。

**Policy**  
Cedar 子集；版本化；每个准入决定引用 `policy_version`。唯一对外输出 `outcome ∈ allow | deny | modify | approve`。`approve` 只在 `fysund` 变成 Approval。EP 与中心对同一快照、同一输入必须同 outcome。

**Approval**  
结果集封闭：`approved_once | rejected | cancelled | unavailable`。只有 `approved_once` 可派生衰减 Grant 或扩权 Adjustment。到期缺省 fail-closed。`asked` 与 `decided` 必须配对，落账失败 → `unavailable.approval`，整个请求拒绝。

**来源标签**  
`label{source_principal, integrity, origin_ref, flags?}`。完整性四级与工具结果一律 `untrusted_external` 继承 [ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)。升级只经 Verification 或 Memory 准入。`flags` 含 `authority_claim` / `secret_shaped`。

**ActionIntent → Effect → Receipt → Verification**  
写路径：EP 本地持久化（带幂等键）再分发，回报后 `fysund` 投影。幂等冲突 → `conflict.idempotency`。无 receipt 不得 `verified`。Verifier 独立；`human` 与 `model_judge` 记 `independence` 与 `calibration_ref`。

| 对象 | 谁写 | 谁读 | 热路径 | MVP |
|---|---|---|---|---|
| Grant | 仅 `fysund` | EP 快照、Runtime 摘要 | 快照在启动时 | R1 |
| Policy | 运营者经 `fysund` | PDP 与 EP | 评估在 EP 本地 | R1 |
| Approval | `fysund` 签发/关闭 | 人、`os.observe`（仅枚举） | 否 | R1（可先无 UI） |
| ActionIntent/Effect | EP 持久化 + `fysund` 投影 | 恢复器、Verifier | 写路径是 | R2（R1 可用 adapter 最小集） |
| label | EP / 控制点赋予；`fysund` 记 digest | 装配与准入 | 片段随上下文 | R1 结构，R2 工具结果 |
| Memory 条目 | 准入事件 | 绑定进上下文 | 检索可在热路径 | R3 |
| Tenant/Scope | 运营者 | 全平面过滤 | 鉴权在入口 | R1 单租户；R3 硬隔离 |

## 7. 账本

唯一权威写入者：`fysund`（F1）。事件是真源；状态是投影。模型 chunk、KV cache、Loop step 计数、传感器高频采样不入账本。会话账本在 Harness，用 `execution_id` / `session_id` 关联，不合并。

本提案相对 v0.7.0 新增的事件族（名称稳定，字段可在实现期补 schema）：

- `asset/*`（register/publish/deprecate/withdraw/quarantine/revoke）
- `agent/defined` · `agent/resolved{lock_digest, contract_digest, substrate_digest, profile_digest}`
- `ep/*` · `runtime/qualified`（带 `harness_asset_id`，无 `kind=runtime` 资产 id）
- `service/published` · `service/rollout` · `plan/attached` · `meter/recorded` · `invocation/*`
- `adjustment/*` 作用于 profile 或 Definition 补丁

EP 回报必须带 `ep_id` + `policy_version`。核对失败 → `ep.report_rejected` + `ledger/conflict`。

边缘：`fysund-edge` 是带本地账本的 EP 宿主；escrow Grant、`approve` 离线阻塞、`origin_seq` 合并——继承 [ADR-0010](../adr/0010-edge-offline-ledger-consistency.md)。R3 才做，R1 不做。

## 8. 底线落点（F1–F9）

| ID | 由谁执行 | 热路径？ |
|---|---|---|
| F1 唯一写入 | `fysund` | 否 |
| F2 模型只产候选 | `fysund` + EP | 候选在数据面，准入决策在快照/中心 |
| F3 先记录再执行 | EP 分级；`fysund` 拒无回报的 `verified` | 写路径是 |
| F4 独立验证 | `fysund` 调 Verifier | 否 |
| F5 secret | 只向资格化 EP 按 Grant 解析 | `resolve_secret` 是冷/温路径 |
| F6 契约不为实现让路 | 流程 | — |
| F7 不自扩权 | EP 过滤；`fysund` 签发 Grant | 过滤在热路径本地 |
| F8 未知外部改动 | Harness + EP | 是 |
| F9 权限单调收窄 | 启动时计算；EP 执行 | 计算在冷路径 |

## 9. 负例

1. `fysund` 出现在 `tools/call` 或模型 HTTP 转发链。
2. 每次 `tools/list` 都 RPC 到 `fysund`。
3. `local_read` 强制同步 WAL 才返回结果。
4. 未资格化 EP 的 `report` 被接受。
5. EP 本地写 `approval/decided` 或 `grant/issued`。
6. `start_execution` 携带渲染文本而非 Lock + 结构化视图。
7. 把 Runtime 实例 `asset.pack` 成 OCI。
8. 无 EP 覆盖的原生 mutation 被标 `verified`。
