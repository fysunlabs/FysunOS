# 11 · 边缘与离线账本一致性

- 状态：设计稿 v0.7.0（2026-09-08）
- 变更依据：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md)；ADR-0001 ~ ADR-0006（v0.7.0 修订）与 ADR-0007 ~ ADR-0012
- 前置：[02 对象模型](02-object-model.md)（Node、Grant、ActionIntent / Effect、Policy、Approval）、[03 架构](03-architecture.md)（双账本、恢复模型、Policy 模型）
- 决策依据：[ADR-0001](../adr/0001-founding-decisions.md) §D5、[ADR-0003](../adr/0003-policy-object-and-language.md) §D5、[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md) §D4、[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md)、**[ADR-0010](../adr/0010-edge-offline-ledger-consistency.md)**（§3 全部决策）、[ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md) §D5

## 1. 定位与范围

边缘与演进（[10](10-evolution.md)）是对象模型的两条**设计检验线**：如果对象模型只适合桌面数字员工，它们会最先暴露问题。本篇讨论边缘：
§2 说明同一对象模型如何在边缘节点上运行（场景、部署形态、物理 Effect 扩展、明确边界、可移植性三层），§3 定义边缘节点离线期间与联网后
治理账本如何保持一致（节点归属、预留预算、离线行为、合并、冲突、负例）。

范围与承诺：

- **不是 MVP 必须完整实现的功能。** §3 的全部机制属边缘阶段（M3 之后，[12 §3](12-mvp-plan.md)）；唯一进入 MVP 的是事件通用字段
  `node_id` / `origin_seq` 必须自 M0 起存在（中心节点恒为自身）。
- **只解决单主体、中心—边缘星型拓扑下的离线一致性。** 一个 Principal、一个中心 `fysund`、若干 `fysund-edge`；不引入 CRDT、不引入多主决策。
  多主体 / 多节点分布式一致性（Approval 决定权、Policy 分发）继续开放，登记于 [12 §7](12-mvp-plan.md)。
- 边缘 Runtime 的接入与资格化见 [07 §6](07-runtime-onboarding.md)；`Node` 对象定义见 [02 §2.16](02-object-model.md)；`fysund-edge` 组件定义见
  [00 §9](00-terminology.md)。

## 2. 边缘：同一对象模型，不同 Runtime 与策略

### 2.1 边缘场景对模型的要求

自动灌溉、自动采摘、设备异常诊断这类场景不是"把云端 Agent 部署到边缘"，而是**能力在不同
节点上的部署与协同**。它们要求：

- 长期活动（持续感知、事件响应、维持状态）而非一次性任务 → **Activity**；
- 能力声明延迟、吞吐、能耗、存储、离线行为、可接受不确定性 → **Capability.requirements**；
- 目标环境是否满足、能否用本地模型 / 规则模块 / 远端调用替代、不满足时明确拒绝 → **环境匹配**；
- 执行器动作是不可逆外部效果 → **ActionIntent / Effect + 物理扩展字段**；
- 动作边界随设备而不同 → **绑定到设备 Resource 的包络 Policy**；
- 云端与边缘之间的分工 → **Handoff**（而不是进程迁移）；
- 网络不可用时账本仍可推进且联网后可对账 → **Node 归属 + escrow 预算 + 按 `origin_seq` 合并**（§3）。

### 2.2 边缘部署形态

```text
中心节点  fysund  ──── Handoff / 产物引用 / 包络 Policy 版本下发 / escrow Grant ────  边缘节点  fysund-edge
              ◀─────── 联网后按 origin_seq 推送本地事件（ledger/merged）──────────      ├── 确定性包络校验器
                                                                                      ├── 本地账本（node_id = 该 edge）
                                                                                      └── 边缘 Runtime（Rust / Python）
                                                                                            ├── 决策核心：本地小模型 / 规则 / 远端调用
                                                                                            └── 适配器：传感器、执行器、网关（MCP server）
```

- `fysund-edge` 是同一账本模型的精简实现：本地 Activity / Execution（home 为该边缘节点，§3.1）、本地 escrow Grant 与预算（§3.2）、本地
  Effect 协议、本地 Policy 快照（含包络 Policy）；联网后与中心节点交换 Handoff 与证据，并把离线期间的本地事件推送到中心合并（§3.4）。
- 边缘节点作为 `Node{kind: edge}` 在中心登记（`node/registered`），连通状态以 `node/online` / `node/offline` 表达；MVP 单机部署下中心即唯一 Node。
- 边缘 Runtime 实现 Runtime ABI 最小面（MCP 客户端 + 最小生命周期 RPC），并如实声明 `native_effect_tools[]`（[07 §6](07-runtime-onboarding.md)）；
  决策核心可以完全不是 LLM。
- "策略下发" = 下发包络 Policy 的版本（`policy/loaded` 在边缘账本落地，事件带该 edge 的 `node_id`）；不下发模型或进程。
- 每个边缘能力独立资格化；软件模拟中的成功不等于硬件动作已通过验证。

### 2.3 物理 Effect 扩展

**安全包络是绑定到设备 Resource 的 Policy 对象**，不是 Effect 字段：

| 层 | 字段 | 说明 |
|---|---|---|
| 包络 Policy（绑定设备 Resource） | `velocity_bounds`、`geofence`、`interface_allowlist`、`rate_limit`、`deadman`、`emergency_stop_ref` | 版本化、带 `source` 与校准记录；同一能力在不同设备上自然适用不同包络 |
| Effect（物理扩展字段） | `envelope_ref`、`envelope_check` | 引用所用包络 Policy 版本与校验结果（通过 / 拒绝项） |
| Effect（物理扩展字段） | `deadline`、`sensor_timestamp`、`uncertainty`、`reversibility_class`、`actuator_receipt`、`physical_post_state_verifier` | 保留在 Effect 上 |

- 校验器在 `fysund-edge` 内以**确定性代码**运行，不经模型；候选动作不满足包络即 `denied.policy`，
  `envelope_check` 记录拒绝项。
- 物理动作的 `effect_category` 为 `physical_action`（[00 §5](00-terminology.md)），一律走完整协议，不进入审计快路径。
- Policy 可要求高 `reversibility_class` 动作必须人工确认（Approval）或双源传感器确认；离线期间需要 Approval 的候选一律阻塞（§3.3）。
- `emergency_stop_ref` 指向宿主控制器的急停接口；FysunOS 只记录引用与触发事件，不实现急停。

### 2.4 明确边界

- **硬实时不在 FysunOS 内。** 运动控制、急停、时序安全由宿主控制系统（PLC、RTOS、ROS 2
  实时控制器）承担；FysunOS 管理感知解释、决策与动作请求，并记录 receipt。
- **包络声明的校准是部署者责任。** 校验器只能保证动作符合**声明的**包络，不能保证声明与世界
  一致（"通过校验 ≠ 世界安全"）；Policy 模型要求包络声明带来源与校准记录，缺失时 Policy 载入
  仍成功但 `capability.match` 对该设备返回 `not_runnable.environment{missing[]}`。
- **离线保守规则。** `fysund-edge` 离线期间只允许本地 Policy 已预授权的 `effect_category`；需要
  Approval 的一律阻塞（Execution 进入 `waiting{reason: approval}`，Activity 进入 `input_required`）；联网后本地
  事件按 `origin_seq` 推送合并到中心账本。预算、合并与冲突处置的完整规则见 §3（[ADR-0010](../adr/0010-edge-offline-ledger-consistency.md)）。
- **不承诺透明迁移。** 模型内部状态、任意进程、凭据与设备权限不随能力包迁移；跨环境复用
  的单位是 Capability 契约 + 目标环境可用的实现 + 重新签发的 Grant。对象归属（home）也不迁移：需要"换节点"时终止原 Activity、
  在新节点创建新 Activity（§3.1）。
- **算力下沉不自动提供实时保证。** 边缘只是减少网络依赖。

### 2.5 可移植性的三个层次（分别承诺）

| 层次 | 含义 | FysunOS 承诺 |
|---|---|---|
| 能力可移植 | 同一契约与实现包在兼容环境部署 | 是：任何实现 Runtime ABI 最小面的 Runtime + 满足 `requirements` 的工具面（经环境匹配与资格化，[07 §1](07-runtime-onboarding.md)） |
| 执行可恢复 | 活动进行到一半，携带必要状态在另一环境重新绑定资源并继续 | 部分：以 `Activity.continuity` 与产物引用为单位，不搬会话与进程，不搬对象归属 |
| 行为适配 | 换模型、工具、设备后仍满足契约 | 是（换实现，重新验证） |

三者难度不同，不合并为"透明迁移"。

## 3. 离线账本一致性

本节是 [ADR-0010](../adr/0010-edge-offline-ledger-consistency.md) 的设计正文，逐小节对应其 §D1–§D7。它回答 v0.6.0 离线保守规则留下的三个问题：
同一 Activity 由谁推进（归属）、离线消耗的预算如何与中心对账（预算）、边缘 `seq` 与中心 `seq` 如何合并（顺序）。
术语、事件名与字段名以 [00 §4 / §5](00-terminology.md) 为准；本节不新增 00 之外的字段名。

### 3.1 节点与归属

**Node 对象**（[02 §2.16](02-object-model.md)）：`Node{node_id, kind: central | edge, clock}`，归 Authority 子系统；状态 `registered` / `online` / `offline`，
事件 `node/registered | node/online | node/offline`。中心节点恒存在且唯一，恒 `online`；MVP 单机部署下中心即唯一 Node。

**归属规则（home）**：

- 每个 Activity / Execution / Effect 在创建时确定**唯一归属节点**（`home_node`，记入对象），**只有 home 节点可以推进其状态**。
- 中心节点可以创建 home 为某 edge 的 Activity（下发）；edge **不能**创建 home 为中心的对象。
- Approval 的 home 恒为中心（§3.3）；Policy 的 home 恒为中心，edge 只持有版本快照。
- 跨节点协作只经 Handoff 与 Artifact 引用（§2.2），**不迁移对象归属**；需要"换节点"时终止原 Activity、在新 home 创建新 Activity。
- `fencing_epoch` 由 home 节点分配，含 `node_id` 前缀以避免跨节点碰撞（02 §2.3）。

**事件的节点字段**：全部事件带通用字段 `node_id`（产生该事件的节点）与 `origin_seq`（该节点本地的单调序号）。中心节点自身产生的事件
`node_id` 为中心、`origin_seq = seq`；MVP 单机下二者恒相等，但字段必须存在（M0 验收）。合并到中心账本的 edge 事件**保留**原 `node_id` 与
`origin_seq`，中心为其分配的 `seq` 只表示"进入中心账本的顺序"，不表示因果顺序：跨节点因果由 `causation_id` / `correlation_id` 表达，
同节点内顺序由 `origin_seq` 表达。

负例：非 home 节点对某 Execution 写出状态推进事件 → 合并时判为 `ledger/conflict{kind: home_violation}`，事件 quarantine（§3.5）；
合并后的 edge 事件 `origin_seq` 被改写为中心 `seq` → 失败；两条同 `node_id` 同 `origin_seq` 的事件同时存在于中心账本 → 失败。

### 3.2 预留预算（escrow）

预算是有上界的稀缺资源，离线期间必须在**不与中心通信**的前提下仍满足 F9 三条不等式（`scope ⊆ parent.scope`、`expiry ≤ parent.expiry`、
`budget ≤ parent.remaining`）。做法是预扣，不是事后合并：

- edge 离线前（通常在 `node/registered` 之后、预期断网之前，或按 Policy 周期续签），中心为其签发**子 Grant**，事件
  `grant/issued{escrow: {node_id}}`（字段 `Grant.escrow = {node_id}`，[00 §5](00-terminology.md)、[02 §2.5](02-object-model.md)），
  `budget ≤ parent.remaining`，并在中心账本中把该预算从父 Grant 的可用余量中**预扣**。
- escrow Grant 的 `scope ⊆ parent.scope`、`expiry ≤ parent.expiry`；到期后 edge 本地新候选 `denied.grant`，**不因离线延长**。
- 离线期间 edge **只消耗本地 escrow Grant**；每次消耗照常产生 `grant/consumed`（`node_id` 为该 edge）；escrow 用尽即 `denied.budget`，
  不得透支到父 Grant，也不得在本地"再签发"。
- 联网合并后，中心按 edge 的 `grant/consumed` 事件把实际消耗记入父 Grant；未消耗的 escrow 余量在 `grant/revoked{escrow}` 或到期时归还父 Grant。
- 中心在 edge 离线期间撤销父 Grant（`grant/revoked`）时，edge 本地无法得知；此后 edge 继续在 escrow 内的消耗在合并时判为
  `revoked_offline` 冲突（§3.5），不自动补偿。
- 长期离线设备的 escrow 自动续期策略待后续 ADR（ADR-0010 待后续段）。

负例：edge 消耗超出 escrow `budget` 的候选被放行 → 失败；escrow `budget > parent.remaining` 的 `grant/issued` → 失败；
edge 本地出现 `grant/issued`（非中心签发）→ 合并时 `home_violation`。

### 3.3 离线行为

`node/offline` 后，`fysund-edge` 按以下规则运行，直到 `node/online`：

- **Policy 评估 context `offline = true`**（[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md) §D1 的固定键），使部署者可以为离线状态写更保守的规则。
- **只允许本地 Policy 已预授权的 `effect_category`**（六值之一或其子集，通常为 `local_read`、`external_read`、受包络 Policy 约束的
  `physical_action`）；其余类别的候选 `denied.policy` 并留痕（`action-intent/rejected`）。`local_read` 照常走审计快路径产生 `audit/observed`。
- **`outcome = approve` 的候选一律阻塞**：Execution `waiting{reason: approval}`、Activity `input_required`；`approval/asked` 在 edge 本地落账
  （它记录的是"候选在等人"这一事实），但 **edge 不能决定 Approval**——`approval/decided` 只能由中心节点在联网合并后写出
  （沿 [ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md) §D5，审批面只由中心 `fysund` 提供）。到期处置同样由中心在合并后按
  `on_expiry` 执行；`approval/asked` 与 `approval/decided` 的配对约束不因跨节点而改变。
- **Policy 快照按版本使用**：edge 持有的 `active` Policy 版本在离线期间不变；每个准入类事件照常携带 `policy_decision.policy_version`，
  合并时中心据此判断 `policy_retired` 冲突（§3.5）。edge 不能加载、修订或退役 Policy。
- **本地对象照常推进**：home 为该 edge 的 Activity / Execution / Effect 在 escrow 与预授权类别内正常走 ActionIntent → Effect → Receipt →
  Verification；`outcome_unknown` 只能本地 reconcile 或 quarantine，不能假定成功。
- **edge 重启**：与中心同样的恢复顺序（[03 §6](03-architecture.md)），含重新武装——含非 `local_read` 类别的 Activity 缺省 `paused{reason: rearm_required}`
  （[02 §2.2](02-object-model.md)）。离线状态下 Principal 无法经中心的审批面与 Capability API 到达 edge，因此只有本地 Policy 快照预授权的
  `auto_resume` 能解除该暂停；否则保持 `paused` 直到联网后由 Principal `activity.resume`。

负例：离线期间 `approve` 类候选被放行 → 失败；edge 本地写出 `approval/decided` → 合并时判为 `home_violation`；离线期间未预授权类别的候选出现
`effect/authorized` → 失败；edge 事件的 `policy_decision.policy_version` 不属于其离线前持有的快照版本 → 失败。

### 3.4 合并

`node/online` 后，edge 与中心执行一次合并：

1. edge 按 `origin_seq` 顺序推送离线期间（自上一次 `ledger/merged.to_seq + 1` 起）的全部本地事件。
2. 中心逐条校验 **digest 链**：每条事件的 digest 覆盖其前一条事件的 digest，链不得断裂；区间首条事件的链锚定到上一次 `ledger/merged` 所记的
   `digest`。链断裂或校验失败 → 拒绝整个区间，不部分追加。
3. 中心按 `(node_id, origin_seq)` **去重**：已存在的事件跳过，不产生第二份。
4. 通过校验的事件**逐条追加**到中心账本（分配中心 `seq`，保留 `node_id` / `origin_seq`），并逐条判定冲突（§3.5）；发现冲突时追加
   `ledger/conflict{node_id, kind, refs[]}`，**原事件保留**。
5. 区间末写 **`ledger/merged{node_id, from_seq, to_seq, digest}`**：`from_seq` / `to_seq` 为该 edge 的 `origin_seq` 区间，`digest` 为该区间
   digest 链的末值，供下次合并锚定与重放校验。

规则：

- 中心**不改写、不删除、不重排** edge 事件（ADR-0001 §D5 append-only）。
- 合并是**幂等的**：同一区间重复推送命中去重，不产生第二条 `ledger/merged`。
- 合并期间中心对该 edge 的 home 对象只读；合并完成后中心投影重建，这些对象的状态以 edge 事件为准（它是 home）。
- 每次离线 / 上线周期产生一条 `ledger/merged`；相邻两条的区间衔接（`from_seq = 上一条 to_seq + 1`），`origin_seq` 不连续而无对应
  `ledger/conflict` 即为失败。
- 合并完成后中心对该 edge 上仍 `pending` 的 Approval 接管决定（写 `approval/decided`），并把 `grant/consumed` 记入父 Grant（§3.2）。

负例：合并后 `origin_seq` 不连续而无 `ledger/conflict` → 失败；`ledger/merged` 的 `digest` 与重放该区间得到的 digest 不一致 → 失败；
中心对 edge 事件做任何改写 → 失败；同一区间重复推送产生第二条 `ledger/merged` → 失败。

### 3.5 冲突规则（封闭枚举）

`ledger/conflict{node_id, kind, refs[]}` 的 `kind` 是**封闭枚举**，取值与 [00 §4](00-terminology.md) 字面一致：

| `kind` | 触发 | 处置 |
|---|---|---|
| `revoked_offline` | 中心在 edge 离线期间 `grant/revoked` 了 escrow 的父 Grant，而 edge 在此后仍有 `grant/consumed` | 关联 Activity 进入 `blocked{reason: revoked_offline}`；**不自动补偿**——补偿是新 Effect（新 Grant + 新 ActionIntent），需 Principal 决定 |
| `policy_retired` | edge 事件的 `policy_decision.policy_version` 在中心已 `policy/retired` | Activity 重绑到当前 `active` 版本（`activity/policy-bound`）并继续；若新版本拒绝该 Activity 的 Capability 则 `blocked{reason: policy_retired}` |
| `home_violation` | 事件的 `node_id` 不是其目标对象的 home（含 edge 写出 `approval/decided`、edge 本地 `grant/issued`） | 事件 quarantine（不折叠进投影）；关联对象 `blocked` 待人工处置 |
| `clock_skew` | edge 事件 `time` 与中心接收时间偏差超出 Policy 阈值 | 记录；排序**以 `origin_seq` 为准**，`time` 只作展示 |

- `refs[]` 指向冲突涉及的事件与对象（edge 事件的 `(node_id, origin_seq)`、中心侧的 `grant/revoked` / `policy/retired` 事件等）。
- 枚举扩展需修订 ADR-0010。
- Activity `blocked` 的原因值 `revoked_offline` / `policy_retired` 与 [00 §3](00-terminology.md) 一致；恢复动作为 Principal `activity.resume`
  或按 Policy 重绑。

负例：出现枚举外的 `kind` → 失败；`revoked_offline` 触发自动 `effect/compensated` → 失败；`ledger/conflict` 无对应原事件保留（原事件被删除或改写）→ 失败。

### 3.6 负例汇总

以下负例在边缘阶段登记为正式向量（编号按 [00 §11](00-terminology.md) 格式在实现阶段分配，归属 T8 / T5，[13 §5](13-threat-model.md)）：

| 断言 | 期望 |
|---|---|
| edge 消耗超出 escrow `budget` 的候选被放行 | 失败；期望 `denied.budget` |
| escrow `budget > parent.remaining` 的 `grant/issued` | 失败 |
| 离线期间 `outcome = approve` 的候选被放行（出现 `effect/authorized` 而无中心写出的 `approval/decided{result: approved_once}`） | 失败 |
| edge 本地写出 `approval/decided` | 合并时 `ledger/conflict{kind: home_violation}`，事件 quarantine |
| 合并后 `origin_seq` 不连续而未报 `ledger/conflict` | 失败 |
| `ledger/merged.digest` 与重放该区间得到的 digest 不一致 | 失败 |
| 中心改写、删除或重排 edge 事件 | 失败 |
| `revoked_offline` 冲突触发自动 `effect/compensated` | 失败；期望 Activity `blocked{reason: revoked_offline}` |
| `ledger/conflict.kind` 出现四值之外的取值 | 失败 |
| 中心节点事件的 `node_id` 不是中心或 `origin_seq ≠ seq` | 失败（M0 即可验收） |

### 3.7 非 MVP：边缘阶段验收

- 本节全部机制属**边缘阶段**（M3 之后，非 MVP）；唯一进入 MVP 的是事件通用字段 `node_id` / `origin_seq`（M0：存在且中心恒为自身）。
- 边缘阶段验收项（登记于 [12 §3](12-mvp-plan.md)）：escrow 越界拒绝；离线 `approve` 阻塞；`ledger/merged` 正常写出且
  `ledger/conflict{revoked_offline}` → Activity `blocked`；T8 全部负例；§3.6 表全部向量。
- `fysund-edge` 需要实现：escrow 消耗记账、事件 digest 链、离线 Policy 评估（`offline = true`）、合并推送；中心 `fysund` 需要实现合并校验、
  去重与冲突判定。
- 继续开放（[12 §7](12-mvp-plan.md)）：多主体 / 多节点下 Approval 决定权与 Policy 分发一致性；edge 之间（非经中心）的 Handoff 是否允许及其账本归属；
  escrow 的自动续期策略。
