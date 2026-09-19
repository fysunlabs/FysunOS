# ADR-0006 · 术语：`Intent` → `ActionIntent`

- 状态：accepted（v0.7.0 修订）
- 日期：2026-09-07
- 决策者：owner
- 背景来源：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) §2.2、§2.3、§3.8、§7.3、§8.2
- supersedes：无（新增；纯术语变更，不改语义）。
- 修订：v0.7.0（2026-09-08）§D3 随 ADR 修订规则变化而更新；见文末修订历史。

## 背景

v0.5.0 02 §2.6 的 `Intent` 是"持久化的效果级请求记录（幂等键 = canonical 参数 digest）"，是
Intent → Effect → Receipt → Verification 协议的第一环，继承自 CognitiveOS 的 Intent/Effect 机制
（ADR-0001 §D4）。

审查报告 §2.3 指出：AOS（arXiv:2608.03214）、MCP 社区、A2A 文档以及 Anthropic Managed Agents 文档里的
"intent" 一律指用户或系统的**顶层目标**（目标 + 约束 + 权限 + 成功标准 + 时限）。报告 §2.2 的对象映射表把
AOS 的 Intent 对应到 FysunOS 的 `Activity.target + constraints + policy_binding`。两种用法在同一份 Runtime ABI
文档中并存，会让读过 AOS 或 A2A 的 Runtime 实现者把 `submit_candidate` 产生的记录误解为目标声明。

协议优先（[ADR-0002](0002-protocol-first-runtime-boundary.md)）之后，Runtime ABI 的读者主要是外部 harness
的实现者，术语与业界一致的价值高于与 CognitiveOS 历史命名的连续性。

## 决策

### D1 · 对象改名 `ActionIntent`，事件族改名 `action-intent/*`

- 对象 `Intent` 改名为 **`ActionIntent`**。
- 事件族 `intent/recorded | intent/rejected | intent/replayed` 改名为
  `action-intent/recorded | action-intent/rejected | action-intent/replayed`。
- 语义不变：persist-before-dispatch（F3）；幂等键 = canonical 参数 digest；恢复后与崩溃前
  `dispatch-started` 但无 receipt 的 Effect 幂等键相同的 candidate 命中 `action-intent/replayed` 而不是新建 Effect。
- 协议链写法：**ActionIntent → Effect → Receipt → Verification**。
- 负例：v0.6.0 及之后的设计文档（归档与审查报告除外）中出现独立的对象名 `Intent` 或事件名 `intent/*` → 失败。

### D2 · 否决 `EffectRequest`

报告 §2.3 给出两个候选：`ActionIntent` 与 `EffectRequest`。选择前者的理由：

- `EffectRequest` 与 `Effect` 对象在名称上过近，读者容易把"请求记录"与"效果事实"混为一个对象的两个状态，
  而二者是协议链上不同的对象（ActionIntent 可能被 `rejected`，永不成为 Effect）。
- "intent" 保留了"尚未成为事实的意图"含义，与 F2（概率组件只产候选）的语义一致；前缀 `Action` 把它限定在
  单个动作层面，与 AOS 的顶层目标义区分。
- 与 CognitiveOS 迁移来源的 `Intent/Effect` 机制（[09 迁移清单](../design/09-migration-inventory.md)）保持可辨识的对应关系，便于迁移比对。

### D3 · 全仓库自 v0.6.0 起统一

- `docs/design/`、`docs/adr/`、`README`、`CHANGELOG` 自 v0.6.0 起统一使用新名。
- `docs/archive/v0.5.0/`、`docs/archive/v0.6.0/` 与审查报告 `docs/research/20260907-frontier-review-and-recommendations.md` 保留旧名
  不改（历史记录）。v0.7.0 起 ADR 正文允许在版本重构中直接修订（[adr/README](README.md) 规则），ADR-0001 §D4 的括注已改为
  `ActionIntent/Effect` 并注明 CognitiveOS 原名，以便迁移比对。
- [00-terminology](../design/00-terminology.md) §1 的改名对照表是唯一权威对照；其他文档与之冲突时先修正冲突方。
- 负例：为统一术语而修改归档目录中的任何文件 → 失败。

### D4 · AOS 意义上的"Intent（顶层目标）"不另设对象

- AOS 的 Intent（顶层目标 + 约束 + 权限 + 成功标准 + 时限）在 FysunOS 中对应
  `Activity.target`（不可变，修订走 Adjustment）+ `Activity.constraints` + `Activity.policy_binding`。
- 不新增名为 `Intent`、`Goal` 或类似的一级对象；对外坐标（01）与 conformance 声明中以"对应 AOS Intent"的
  注释说明，不改对象模型。
- 负例：对象模型中出现承载顶层目标的独立对象 → 失败。

## 否决的替代方案

- **保留 `Intent`。**与 AOS / MCP / A2A / Managed Agents 用法冲突；协议优先后 Runtime ABI 读者以外部实现者
  为主，冲突成本上升。
- **`EffectRequest`。**理由见 D2。
- **`Candidate`。**"候选"在 FysunOS 中是更宽的概念（`submit_candidate` 的输入、Adjustment 候选、Memory
  提案都是候选）；用作单一对象名会与 F2 的一般表述混淆。
- **同时新增 AOS 义的 `Intent` 对象。**Activity 已承载该语义；新增对象只会增加对齐表的一行而不增加能力。

## 后果

- 所有 v0.6.0 设计文档执行改名：02 §2.6 与协议图、03 §5.1 事件族、Runtime ABI 中 `submit_candidate`
  的返回与事件引用、迁移清单的对照项、MVP 验收项（v0.7.0 起对应新 02、00 §4、05、09、12）。
- 未来实现的 schema、事件名、错误码、CLI 输出沿用 `ActionIntent` / `action-intent/*`；从 CognitiveOS 迁移的
  代码在迁入时改名，不保留兼容别名。
- 迁移清单（[09](../design/09-migration-inventory.md)）需要在 CognitiveOS `Intent` 与 FysunOS `ActionIntent` 之间保留一行对照，便于代码比对。
- 无运行时语义变化；不产生新的验收项，只产生文档一致性检查（D1、D3 的负例）。

## 修订历史

- v0.7.0（2026-09-08）：§D3 删除"ADR-0001 正文中的 `Intent/Effect` 保留不改（ADR 只追加不改写）"与对应负例"为统一术语而修改 ADR-0001 正文
  → 失败"，改为引用 adr/README 的 v0.7.0 修订规则并注明 ADR-0001 §D4 已改括注；归档范围增 `docs/archive/v0.6.0/`；§D1 负例与后果段的文档
  编号更新为 v0.7.0 地图。为什么：蓝图 §I.1 决策 2（允许直接修订 ADR 正文）、决策 3（v0.6.0 归档）。对应蓝图：§C.1 ADR-0006 行。
