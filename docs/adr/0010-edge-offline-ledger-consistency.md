# ADR-0010 · 边缘离线账本一致性

- 状态：accepted
- 日期：2026-09-08
- 决策者：owner
- 背景来源：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md) §I.1 决策 4、§B-11 §3、§C.2 ADR-0010（关闭旧 08 开放问题 9）；[ADR-0001](0001-founding-decisions.md) §D5（有边界的 event-sourced 账本）；[ADR-0003](0003-policy-object-and-language.md) §D5 离线保守规则；[ADR-0004](0004-provenance-labels-and-monotonic-privilege.md) §D4 F9 三条不等式
- supersedes：无（新增）。旧 07 §2.4 留待后续 ADR 的合并冲突规则由本 ADR 落定。

## 背景

v0.6.0 对边缘的账本处理只有一条保守规则（旧 07 §2.4（现 11 §2.4）、03 §7、ADR-0003 §D5）："`fysund-edge` 离线期间只允许本地 Policy 已预授权的 Effect
类别；需要 Approval 的一律阻塞；联网后本地事件按 `seq` 合并到中心账本"，并把合并冲突规则留待后续 ADR。它有三个未回答的问题：

- **归属**：同一 Activity 能否同时在中心与边缘推进？如果能，两份投影如何合并；如果不能，谁有权推进。
- **预算**：边缘离线期间消耗的 Grant 预算如何与中心账本对账；中心在边缘离线期间撤销了 Grant 怎么办。
- **顺序**：边缘事件的 `seq` 与中心 `seq` 是两个独立序列，"按 `seq` 合并"没有定义。

owner 决策（蓝图 §I.1 决策 4）：远期开放问题中只有"边缘离线账本一致性"纳入本轮正式设计；多主体 / 多节点分布式一致性（Approval 决定权、
Policy 分发）继续开放。本 ADR 只解决**单主体、中心—边缘星型拓扑**下的离线一致性；不引入 CRDT、不引入多主决策。设计文档正文在
[11](../design/11-edge-and-offline-ledger.md) §3；本 ADR 记录决策与否决项。

## 决策

### D1 · 节点归属唯一（home）

- 新增对象 **`Node{node_id, kind: central | edge, clock}`**（[02](../design/02-object-model.md) §2 / [00 §2](../design/00-terminology.md)），
  归 Authority 子系统；事件 `node/registered | node/online | node/offline`。中心节点恒存在且唯一；MVP 单机部署下中心即唯一 Node。
- 每个 Activity / Execution / Effect 有**唯一归属节点（home）**，在创建时确定并记入对象；**只有 home 节点可以推进其状态**。中心节点可以
  创建 home 为某 edge 的 Activity（下发），edge 不能创建 home 为中心的对象。
- 跨节点协作只经 Handoff 与 Artifact 引用（旧 07 §2.2 已定），不迁移对象归属。需要"换节点"时终止原 Activity、在新 home 创建新 Activity。
- 负例：非 home 节点对某 Execution 写出状态推进事件 → 合并时判为 `home_violation`（D6），事件 quarantine。

### D2 · 事件携带 `node_id` 与 `origin_seq`；中心 `seq` 只是合并序

- 事件通用字段新增 **`node_id`**（产生该事件的节点）与 **`origin_seq`**（该节点本地的单调序号）。中心节点自身产生的事件 `node_id` 为中心、
  `origin_seq = seq`；MVP 单机下二者恒相等，但字段必须存在（M0 验收）。
- 合并到中心账本的 edge 事件**保留**原 `node_id` 与 `origin_seq`；中心为其分配的 `seq` 只表示"进入中心账本的顺序"，不表示因果顺序。
  跨节点因果由 `causation_id` / `correlation_id` 表达；同节点内顺序由 `origin_seq` 表达。
- `fencing_epoch` 由 home 节点分配，含 `node_id` 前缀以避免跨节点碰撞。
- 负例：合并后的 edge 事件 `origin_seq` 被改写为中心 `seq` → 失败；两条同 `node_id` 同 `origin_seq` 的事件同时存在于中心账本 → 失败。

### D3 · 预留预算：escrow Grant

- edge 离线前，中心为其签发**子 Grant** `grant/issued{escrow: {node_id}}`（字段 `Grant.escrow = {node_id}`，形状以 [00 §4 / §5](../design/00-terminology.md) 为准），
  `budget ≤ parent.remaining`（沿 F9 三条不等式），并在中心账本中把该预算从父 Grant 的可用余量中**预扣**。
- 离线期间 edge **只消耗本地 escrow Grant**；escrow 用尽即 `denied.budget`，不得透支到父 Grant。
- 联网合并后，中心按 edge 的 `grant/consumed` 事件把实际消耗记入父 Grant，未消耗的 escrow 余量在 `grant/revoked{escrow}` 或到期时归还。
- escrow Grant 的 `expiry ≤ parent.expiry`；到期后 edge 本地新候选 `denied.grant`，不因离线延长。
- 负例：edge 消耗超出 escrow `budget` 的候选被放行 → 失败；escrow `budget > parent.remaining` 的 `grant/issued` → 失败。

### D4 · 离线行为

- `node/offline` 后，edge 上的 Policy 评估 context `offline = true`（[ADR-0009](0009-cedar-subset-and-entity-model.md) §D1）。
- 只允许本地 Policy 已预授权的 `effect_category`（[ADR-0008](0008-effect-category-and-read-only-determination.md) 六值之一或其子集，
  通常为 `local_read`、`external_read`、受包络 Policy 约束的 `physical_action`）；其余类别的候选 `denied.policy`。
- `outcome = approve` 的候选**一律阻塞**：Execution `waiting{reason: approval}`、Activity `input_required`；`approval/asked` 在 edge 本地
  落账但 **edge 不能决定 Approval**——`approval/decided` 只能由中心节点在联网后写出（沿 [ADR-0011](0011-approval-result-set-and-approval-surface.md)
  §D5，审批面只由中心 `fysund` 提供）。
- Policy 快照按版本使用：edge 持有的 `active` Policy 版本在离线期间不变；每个准入类事件照常携带 `policy_decision.policy_version`，
  合并时中心据此判断 `policy_retired` 冲突（D6）。
- 负例：离线期间 `approve` 类候选被放行 → 失败；edge 本地写出 `approval/decided` → 合并时判为 `home_violation`（Approval 的 home 恒为中心）。

### D5 · 合并

- `node/online` 后 edge 按 `origin_seq` 顺序推送离线期间的事件；中心校验每条事件的 digest 链（每条事件的 digest 覆盖其前一条事件的 digest，链不得断裂）、按 `(node_id, origin_seq)`
  去重、逐条追加到中心账本，最后写 **`ledger/merged{node_id, from_seq, to_seq, digest}`**。
- 中心**不改写、不删除、不重排** edge 事件；发现冲突时追加 `ledger/conflict{node_id, kind, refs[]}`（D6），原事件保留。
- 合并是幂等的：同一区间重复推送命中去重，不产生第二条 `ledger/merged`。
- 合并期间中心对该 edge 的 home 对象只读；合并完成后中心投影重建，Activity 状态以 edge 事件为准（它是 home）。
- 负例：合并后 `origin_seq` 不连续而无 `ledger/conflict` → 失败；`ledger/merged` 的 `digest` 与重放该区间得到的 digest 不一致 → 失败；
  中心对 edge 事件做任何改写 → 失败。

### D6 · 冲突封闭枚举与处置

`ledger/conflict.kind` 是封闭枚举：

| `kind` | 触发 | 处置 |
|---|---|---|
| `revoked_offline` | 中心在 edge 离线期间 `grant/revoked` 了 escrow 的父 Grant，而 edge 在此后仍有 `grant/consumed` | 关联 Activity 进入 `blocked{reason: revoked_offline}`；**不自动补偿**（补偿是新 Effect，需 Principal 决定） |
| `policy_retired` | edge 事件的 `policy_decision.policy_version` 在中心已 `policy/retired` | Activity 重绑到当前 `active` 版本（`activity/policy-bound`）并继续；若新版本拒绝该 Activity 的 Capability 则 `blocked{reason: policy_retired}` |
| `home_violation` | 事件的 `node_id` 不是其目标对象的 home | 事件 quarantine（不折叠进投影）；关联对象 `blocked` 待人工处置 |
| `clock_skew` | edge 事件 `time` 与中心接收时间偏差超出 Policy 阈值 | 记录；排序**以 `origin_seq` 为准**，`time` 只作展示 |

- 枚举扩展需修订本 ADR。
- 负例：出现枚举外的 `kind` → 失败；`revoked_offline` 触发自动 `effect/compensated` → 失败。

### D7 · 非 MVP；边缘阶段验收

- 本 ADR 的全部机制属**边缘阶段**（M3 之后，非 MVP）；唯一进入 MVP 的是 D2 的通用字段（M0：`node_id` / `origin_seq` 存在且中心恒为自身）。
- 边缘阶段验收项（12 §3）：escrow 越界拒绝；离线 `approve` 阻塞；`ledger/merged` 与 `ledger/conflict{revoked_offline}` → Activity `blocked`；
  T8 全部负例。

## 否决的替代方案

- **CRDT 式自动合并预算。**预算是有上界的稀缺资源，CRDT 合并会在双方都"合法"消耗后得到超支的总和；F9 要求 `budget ≤ parent.remaining`
  在任何时刻成立，只有预扣（escrow）能保证。
- **中心回滚 edge 事件。**违背 ADR-0001 §D5（事件是唯一真源、append-only）；edge 上的物理 Effect 已进入世界，回滚账本不会回滚世界，
  只会让账本与世界不一致。冲突以追加 `ledger/conflict` + Activity `blocked` 表达。
- **edge 本地 Principal 决定 Approval。**这是多主体问题（谁在 edge 上是 Principal、其决定权范围），owner 决定继续开放；本轮 Approval 的 home
  恒为中心。
- **以 `time` 排序合并。**边缘设备时钟不可信；`origin_seq` 是节点本地单调序，不依赖时钟。
- **允许同一 Activity 多 home（中心与 edge 都能推进）。**需要分布式共识或冲突合并语义，超出单主体星型拓扑的范围。

## 后果

- 事件通用字段增 `node_id`、`origin_seq`（全部事件，含 MVP）；`grant/issued` 增 `escrow?: {node_id}`；新增事件 `node/*`、`ledger/merged`、
  `ledger/conflict`；新增对象 `Node`。
- `fysund-edge` 需要实现：escrow 消耗记账、事件 digest 链、离线 Policy 评估（`offline = true`）、合并推送；中心 `fysund` 需要实现合并校验与
  冲突判定。
- Activity `blocked` 新增原因 `revoked_offline`、`policy_retired`。
- 旧 08 开放问题 9 关闭；多主体 / 多节点一致性（Approval 决定权、Policy 分发）作为独立开放问题保留在 12 §7。
- 03 §7 Policy 模型问题清单增"边缘预留预算与冲突处置"。
- **与重新武装（[ADR-0011](0011-approval-result-set-and-approval-surface.md) §D8）的交互**：edge 重启走与中心相同的恢复顺序（03 §6），含非 `local_read`
  类别的 Activity 缺省 `paused{reason: rearm_required}`；离线状态下 Principal 无法经中心的审批面与 Capability API 到达 edge，因此**只有本地 Policy 快照
  预授权的 `auto_resume` 能解除该暂停**，否则保持 `paused` 直到 `node/online` 后由 Principal `activity.resume`（[11 §3.3](../design/11-edge-and-offline-ledger.md)）。
  这是 D4"edge 不能决定 Approval、Policy 快照按版本使用"在恢复场景下的直接推论，不引入新决策。

## 待后续 ADR 的问题

- 多主体 / 多节点下 Approval 决定权与 Policy 分发一致性（与 ADR-0003、ADR-0005 待后续项合并）。
- edge 之间（非经中心）的 Handoff 是否允许，及其账本归属。
- escrow 的自动续期策略（长期离线设备）。

其余见 [12 MVP 计划 §7](../design/12-mvp-plan.md)。

## 修订历史

- v0.7.0（2026-09-08，WP6 收尾）：§D3 `grant/issued` 的 escrow 载荷写法由 `{escrow: true, node_id}` 改为 `{escrow: {node_id}}`，与 00 §4 / §5、02 §2.5、11 §3.2
  字面一致（WP1 裁定以 00 为准）；后果段补"离线时只有 Policy 快照预授权的 `auto_resume` 能解除 `paused{reason: rearm_required}`"的引用（WP4 在 11 §3.3 的推导，
  经 WP6 复核与 02 §2.2、03 §6、本 ADR §D4、ADR-0011 §D8 无冲突）。对应蓝图：§I.1 决策 4、§I.2 H-7。
