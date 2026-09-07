# 架构决策记录（ADR）

设计文档之间或设计文档与实现冲突时，以 accepted 的 ADR 为准。ADR 只追加，不改写已 accepted
的内容；推翻既有决策需要新 ADR 并标注 supersedes。

| 编号 | 标题 | 状态 | 日期 |
|---|---|---|---|
| [0001](0001-founding-decisions.md) | FysunOS 创立决策：定位、DSH/Cordis 角色、独立仓库、Rust daemon、event-sourced 账本 | accepted | 2026-09-06 |
| [0002](0002-protocol-first-runtime-boundary.md) | 协议优先的 Runtime 边界：Runtime ABI（MCP 守卫 + 最小生命周期 RPC + 可选控制点）、A2A Handoff、Agent Skills 实现包、DSH 降为 Level-2 参考适配器、Composition ABI 降为内部接口 | accepted | 2026-09-07 |
| [0003](0003-policy-object-and-language.md) | Policy 对象与策略语言：版本化账本对象、`policy_decision`、Cedar 子集 | accepted | 2026-09-07 |
| [0004](0004-provenance-labels-and-monotonic-privilege.md) | 来源标签与权限单调收窄（F9） | accepted | 2026-09-07 |
| [0005](0005-approval-and-evidence-gated-evolution.md) | Approval 一级对象与证据门控演进：Approval、审批容量、Adjustment 预测与评估、Verification 扩展 | accepted | 2026-09-07 |
| [0006](0006-terminology-action-intent.md) | 术语：`Intent` → `ActionIntent` | accepted | 2026-09-07 |

ADR-0002 部分 supersede ADR-0001 §D2；ADR-0001 正文保持原样，以本索引与新 ADR 头部的 supersedes 字段为准。
