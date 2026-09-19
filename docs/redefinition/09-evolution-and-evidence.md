# 09 · 演进与证据

- 状态：提案草案 v0.8.0-draft（2026-09-11）
- 继承：v0.7.0 [10 演进](../design/10-evolution.md)、[ADR-0005](../adr/0005-approval-and-evidence-gated-evolution.md)（证据门控）
- 前置：[02 能力资产模型](02-asset-model.md)、[03 组合](03-composition.md)、[05 Agent as a Service](05-agent-as-a-service.md)

## 1. 三种变化（继承并映射到资产）

| 变化 | 载体 | 检查 |
|---|---|---|
| 调整（Adjustment） | Lock 级覆盖 / Definition 补丁版本 | 触及扩权、新外部动作、预算上调、设备安全时走 Policy；其余直接生效、留痕、可回退 |
| 资产升级 | 新资产版本 → 新 Lock → 新 ServiceVersion | 解析约束 + 可选证据门 + rollout |
| 契约变化 | Capability 新版本或新 Capability（谱系） | 契约不兼容默认派生新 Capability；Service 需新发布 |

## 2. 证据的层次

```text
Verification ──归属──▶ Capability（契约层：这个契约能被满足吗）
                  ├──▶ Agent Definition 版本（组合层：这套组合行不行）
                  ├──▶ Lock（精确组合：这组 digest 在这类环境的结果）
                  └──▶ 各资产版本（资产层：按 verifier_kind 与贡献标注）
downstream_outcome ──▶ Service（服务层：调用后的后续结果）
conformance 结果  ──▶ harness / runtime / EP 资产（资格化证据）
```

- 配置声明不算证据（继承）；`compatibility` 是声明。
- 发现接口按层展示证据摘要与 `health`。
- 换资产后旧证据留在旧 Lock；新 Lock 从零累积（可由 `compose.diff` 提示"只换了 prompt 版本"作为人工判断参考，不作为系统推断）。

## 3. 质量目标表示法

继承 v0.7.0 10 §1.5：Skill Contract 骨架 `(P, O, A, V, F)`，V 为 checklist，每项标 `verifier_kind`；`human` 项以 Approval 的 `approved_once` 为该项 Verification。
新增：checklist 项可标 `attributable_to: [asset kinds]`，用于证据归属到资产层。

## 4. 预测与评估

`predicted_effect?` / `evaluated_against` / `evidence_gate` 继承（可选、缺 `statement` 拒绝、为空恒 `inconclusive`）。
`evidence_gate` 的回归集扩展来源：该 Capability 历史已 `verified` 的 Execution 输入 + 消费者标记为"基准"的调用（租户可选贡献，脱敏后入回归集，Policy 控制）。

## 5. 发布证据门

- `service.publish` 与 `service.rollout(stable)` 可由 Policy 要求 `evidence_gate`：对回归集跑一次，Verification 全部 `pass` 且无 `not-run`。
- `canary` 阶段自动累积 Lock 级证据；Policy 可设 "canary 期间 Verification 通过率 ≥ 阈值 且 人工介入率 ≤ 阈值" 作为进入 `stable` 的前置。
- 回滚率、验证独立性比例、人工介入率、审批不可用率——继承的度量在服务层按 Service 折叠。

## 6. 消费反馈

- 反馈只以 Verification 形式进入系统：`downstream_outcome`（后续 Activity 结果反推）、租户人工判定（`human` verifier，独立性 `different_principal`）。
- 不设评分 / 点赞对象；主观反馈若需要保留，作为 Artifact 引用挂在 Verification 的 `items[]` 上，`integrity = trusted_user`（租户）。
- 反馈驱动的调整仍是 Adjustment，归属发布者或租户（BYO 槽位），不自动传播到其他租户。

## 7. 弃用与迁移

- 资产 `deprecated{successor}`：`compose.resolve` 告警并建议 successor；`asset.health` 标"依赖已弃用"。
- Service 版本 `retire`：钉住它的 Plan 收到通知；宽限期由 Policy 决定；宽限结束后调用返回 `service.version_retired`。
- Capability 谱系（`derived_from` / `forked_from` / `evolved_from`）继承；Service 可声明 `successor_service`。

## 8. 健康查询

`asset.health` / `agent.health` / `service.health` 是投影：冗余（契约近似）、过期（证据全部来自已失格 Runtime / 已弃用资产）、反复失败的 checklist 项、
未分类工具、被隔离依赖、`policy_stale` 频发的 EP。健康不是等级；不设自动降级。

## 9. 负例（R5）

1. 换 Harness 后旧 Lock 的证据被计入新 Lock → 失败。
2. `evidence_gate` 开启时未跑回归集的 `stable` rollout 被接受 → 失败。
3. 出现独立于 Verification 的"评分"对象进入账本 → 失败。
4. 租户 A 的反馈驱动的 Adjustment 自动生效于租户 B → 失败。
