# 架构决策记录（ADR）

设计文档之间或设计文档与实现冲突时，以 accepted 的 ADR 为准。

规则（v0.7.0 起）：

- accepted 的 ADR 可在版本重构中直接修订正文，但必须在文内 `## 修订历史` 段注明版本、改动与理由；头部状态标为 `accepted（<版本> 修订）`。
- 推翻既有决策仍需新 ADR 并标注 supersedes；被推翻方的正文在下一次版本重构中按 supersedes 内容修订，并在修订历史中引用推翻方。
- 新增 ADR 从当前最大编号顺延；编号一经分配不复用。
- 归档目录（`docs/archive/`）中的 ADR 副本不再修改。

| 编号 | 标题 | 状态 | 日期 |
|---|---|---|---|
| [0001](0001-founding-decisions.md) | FysunOS 创立决策：定位、DSH/Cordis 角色、独立仓库、Rust daemon、event-sourced 账本 | accepted（v0.7.0 修订） | 2026-09-06 |
| [0002](0002-protocol-first-runtime-boundary.md) | 协议优先的 Runtime 边界：Runtime ABI（MCP 守卫 + 最小生命周期 RPC + 可选控制点）、A2A Handoff、Agent Skills 实现包、DSH 降为声明全部六个控制点的参考适配器、Composition ABI 降为内部接口 | accepted（v0.7.0 修订） | 2026-09-07 |
| [0003](0003-policy-object-and-language.md) | Policy 对象与策略语言：版本化账本对象、`policy_decision`、Cedar 子集 | accepted（v0.7.0 修订） | 2026-09-07 |
| [0004](0004-provenance-labels-and-monotonic-privilege.md) | 来源标签与权限单调收窄（F9） | accepted（v0.7.0 修订） | 2026-09-07 |
| [0005](0005-approval-and-evidence-gated-evolution.md) | Approval 一级对象与证据门控演进：Approval、审批容量、Adjustment 预测与评估、Verification 扩展 | accepted（v0.7.0 修订） | 2026-09-07 |
| [0006](0006-terminology-action-intent.md) | 术语：`Intent` → `ActionIntent` | accepted（v0.7.0 修订） | 2026-09-07 |
| [0007](0007-runtime-native-tools-and-conformance-scope.md) | Runtime 原生工具、资格化与 conformance 范围：`native_effect_tools[]`、资格化纪元、hooks.json 阻断路径（Level-1b）、F5 范围限定、MCP-only Runtime 的投递路径 | accepted | 2026-09-08 |
| [0008](0008-effect-category-and-read-only-determination.md) | Effect 类别分类与只读判定：六值 `effect_category`、三源合一、审计快路径、幂等冲突与补偿规则 | accepted | 2026-09-08 |
| [0009](0009-cedar-subset-and-entity-model.md) | Cedar 子集范围与实体模型：principal / action / resource / context 实体、语法子集、`outcome` 四值表达约定、schema 校验 | accepted | 2026-09-08 |
| [0010](0010-edge-offline-ledger-consistency.md) | 边缘离线账本一致性：节点归属、`origin_seq`、预留预算 escrow、离线行为、合并与冲突封闭枚举 | accepted | 2026-09-08 |
| [0011](0011-approval-result-set-and-approval-surface.md) | Approval 结果集、事件配对与审批面来源：`approved_once / rejected / cancelled / unavailable`、`approval/asked` / `approval/decided` 配对、审批面只由 `fysund` 提供、审批轰炸、重新武装 | accepted | 2026-09-08 |
| [0012](0012-mcp-guard-self-implementation.md) | `fysun-mcp-guard` 自实现，关闭 agentgateway 复用评估 | accepted | 2026-09-08 |

说明：

- ADR-0002 部分 supersede ADR-0001 §D2；ADR-0001 §D2 已于 v0.7.0 按 ADR-0002 修订，不再依赖本索引说明。
- ADR-0009 关闭旧 08 开放问题 7（Cedar 子集范围）；ADR-0010 关闭开放问题 9（边缘账本一致性）；ADR-0012 关闭开放问题 10 与 ADR-0002 §D7（agentgateway 采用形态）。
- ADR-0011 承接 ADR-0005 的 Approval 细则修订（结果集、事件配对、审批面）；ADR-0007 承接 ADR-0002 §D1/§D5 的 Runtime 接入修订。
- ADR-0007、ADR-0008、ADR-0010 为 v0.7.0 新增，其文末 `## 修订历史` 记录的是同一版本内（WP6 收尾）为跨篇字面一致所做的对齐修改，不构成版本修订，状态仍为 `accepted`。
- 开放问题统一登记于 [12 §7](../design/12-mvp-plan.md)；各 ADR"待后续 ADR 的问题"段只登记该 ADR 范围内的细项。
