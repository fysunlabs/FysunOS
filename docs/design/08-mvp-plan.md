# 08 · MVP 计划

- 状态：设计稿 v0.6.0（2026-09-07）
- 变更依据：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) 与 ADR-0002 ~ ADR-0006
- 前置：全部设计文档
- 决策依据：[ADR-0001](../adr/0001-founding-decisions.md)、[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md)、[ADR-0003](../adr/0003-policy-object-and-language.md)、[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)、[ADR-0005](../adr/0005-approval-and-evidence-gated-evolution.md)
- 目的：避免再次做成白皮书。第一个可运行闭环必须小、真实、可验证。

## 1. MVP 边界（MVP-first）

首个可运行路径只实现当前闭环与底线 F1–F9 要求的最小面：

- owner-local、单主体、单机；Scope 模型预留但不预建 RBAC、审批链、通用 capability 市场；
- MCP 守卫 + 一个 Level-1 Runtime（任选现成 MCP 兼容 harness）；随后 DSH 作为 Level-2 参考适配器（`fysun-dsh`）；
- 一个不依赖任何业务对象的 Capability，实现包为 Agent Skills 目录；
- Policy 从本地 Cedar 子集文件载入为版本化对象；
- 不可省略：唯一权威写入者、Secret Store、persist-before-dispatch、预算/fencing、独立 Verifier、
  审计边界、Policy 版本引用、来源标签、Approval 对象、F9 负例。

## 2. 首个闭环

以"趋势调研 → 内容包"能力为例（可换成任何不需要外部账号的能力）：

```text
1. capability.register        声明契约（输入源、输出内容包 schema、质量 checklist（07 §1.5）、预算/时间边界、Verifier=schema+来源校验）；implementation 为 Skills 目录引用
2. activity.create + start    建立长期 Activity（周期触发）；"先看过再发布"由 Policy 对发布类 Effect 产出 approve → Approval 对象实现
3. fysund 启动 Runtime         harness 把 fysund 配为 MCP server；start_execution 携带受控上下文视图（带来源标签）与 Grant 摘要
4. Runtime 执行               工具调用经 MCP 守卫 tools/call pre → submit_candidate → ActionIntent/Effect → 执行 → Receipt；tools/list 已按 Grant 与 Policy 过滤
5. 产物独立验证                Verifier 校验内容包 schema 与来源引用；Verification（含 verifier_kind、independence、verdict）写入账本与 Capability.evidence
6. 一次 Approval               发布候选命中 policy_decision.outcome=approve → approval/requested；用户 approve_once 后 Effect 才 authorized；崩溃恢复后未决 Approval 不视为已批
7. 进程重启                    kill fysund 与 Runtime；重启后 Activity 恢复到 waiting（若有未决 Approval 则 input_required），未知结果 Effect 对账
8. 一次 Adjustment             用户校准"语气更克制"，scope=this_activity，带 predicted_effect；下一 Execution 生效，其 Verification 自动填 evaluated_against；adjustment.revert 可回退
9. 一次 Handoff                内容包中的配图交给另一 Capability（子 Execution）；Handoff 与子 Execution 在账本中可见；Handoff 状态可无损映射 A2A TaskState
```

## 3. 阶段划分与验收

| 阶段 | 交付 | 验收（全部为可判定项） |
|---|---|---|
| **M0 账本与对象** | 治理账本 schema（event-sourced）、投影、replay；对象模型的 Rust 类型 | replay 两次得到相同投影 digest；负例：陈旧 epoch 写入被拒 |
| **M1 权威协议** | ActionIntent/Effect/Receipt、Grant/预算、Verifier 框架、Secret Store、Provider 代理、Policy 载入（Cedar 子集）、来源标签赋予 | 重复请求幂等；预算耗尽拒绝；outcome-unknown 进入 reconcile/quarantine；明文 secret 不出现在任何日志/账本/argv；`policy_decision`（含 `policy_version`）附在每个准入事件上；来源标签在 context_view 与 Effect 参数上可见，且 Policy 可基于标签产生 `approve`；负例：`untrusted_external` 来源的收件人参数直接放行 → 失败；F9 负例：扩权候选不产生无 Approval 的 `grant/issued` |
| **M2a MCP 守卫 + Level-1 接入** | `fysun-mcp-guard`：fysund 作为 MCP 中间层，接入任意一个现成 MCP 兼容 harness（选择标准：开源、本地可跑、有 Skills 加载；候选 DSH headless、Goose、OpenCode 等，按当时可用性挑一个）；最小生命周期 RPC | 每个 `tools/call` 配对一个 Effect；`tools/list` 按 Grant 过滤（负例：被过滤的工具不可调用）；F9 负例：扩权 candidate 不产生无 Approval 的 `grant/issued`；Runtime 无法推进 Execution 状态（负例） |
| **M2b DSH Level-2 适配** | `fysun-dsh`：上下文注入、检查点、session_link、subagent → Handoff | 跨账本 replay 双向定位；`runtime/qualified{control_points[]}` 事件正确 |
| **M3 Activity 闭环** | Activity/Execution 状态机（含 `input_required`）、周期触发、检查点、恢复顺序、Handoff、Approval 闭环、A2A Handoff 对齐 | 上述闭环第 1–7、9 步全部通过；崩溃后不重放已 receipt 的 Effect；闭环第 2 步"先看过再发布"通过 Approval 对象实现（不再是策略注释）；Handoff 状态可无损映射到 A2A TaskState；崩溃恢复后未决 Approval 不被视为已批 |
| **M4 演进与入口** | Adjustment（apply/revert/promote、predicted_effect/evaluated_against、evidence_gate）、`fysun` CLI、最小 Web 状态页（Approval 呈现面优先） | 闭环第 8 步；Adjustment 触及扩权时被 Policy 拦截（负例）；一次 Adjustment 带 `predicted_effect`，下一 Execution 的 Verification 自动填充 `evaluated_against`；负例：`evidence_gate` 开启时未跑回归集的 promote 被拒 |
| **M5 第一个应用** | Personal 作为应用切回 FysunOS（角色/团队/业务对象在应用层） | Personal 不再维护第二套执行路径 |

阶段可并行局部推进，但 M2a 完成前不做 DSH Level-2 适配；M3 之前不做边缘。
负例向量的编号与归属见 [09 威胁模型 §5](09-threat-model.md#5-负例向量索引)。

## 4. 效率与性能的度量

"精简高效"必须可测。基线先测、阈值后定，不能仅凭架构宣称性能提升：

- daemon 启动耗时、空闲内存；
- 调度延迟（trigger → start_execution）；
- 每次工具调用的账本事务数与 MCP 守卫往返延迟；
- 崩溃恢复时间；
- 接入一个新 Capability 所需步骤数；
- 人工介入率（`approval/requested` 数 / Execution 数）；
- 回滚率（`adjustment/reverted` 数 / `adjustment/applied` 数）；
- 验证独立性比例（`independence != not_independent` 的 Verification 占比）。

后三项直接从治理账本事件折叠，并经 OTLP 导出（见 [03 §8](03-architecture.md#8-可观测性)）；
不另建统计表。

## 5. 证据纪律（迁移自 CognitiveOS A7，作为流程规则）

- 本地、fixture、普通 CI 证据只推进实现进度；任何"支持"、"发布"、"Gate"类声明需要预登记的
  环境与流程。
- `not-run` 永不等于 pass；被解析器拒绝的命令记 `not-run`，不是测试失败。
- 每个测试/验证单元完成后立即追加到该活动唯一的 running report。
- 审查报告引用的所有外部基准数字（AgentDojo、Terminal-Bench、LoCoMo 等）只说明方向，不构成对
  FysunOS 的任何性能或安全声明。

## 6. 非声明

本设计不声明：

- FysunOS 已实现任何代码；
- DSH 已被接入或已通过资格化；
- 任何 Runtime 已通过 Level-1 / Level-2 资格化；
- agentgateway 已被复用或评估完成；
- Cordis 已成为依赖；
- 任何性能、任务完成率、协作收益、边缘可用性数字；
- Personal 已迁移。

## 7. 开放问题（待后续 ADR）

1. `fysund` 与 Runtime 之间的传输，拆为两问：(a) MCP 传输——stdio + Streamable HTTP，跟随 MCP 规范；
   (b) 最小生命周期 RPC——stdio JSON-RPC（MVP），是否同时支持本地 socket / gRPC 待定；
2. Secret Store 后端的首选实现（OS keyring vs 文件加密）；
3. Web UI 的技术栈与是否复用 Personal Control Plane 的组件；Approval 呈现面为 UI 第一优先级；
4. Capability 契约中"质量目标"的机器可判定表示法——已由 [07 §1.5](07-evolution-and-edge.md#15-质量目标的表示法)
   采纳 Skill Contract 骨架 + checklist 化 V + `verifier_kind` 标注；剩余问题为 `model_judge` 校准集的来源
   （谁构造、多大、如何更新）；
5. 多主体（多人 / 多设备）Scope 模型何时激活；对外暴露 A2A server（签名 Agent Card）是多主体的第一个触发点；
6. 许可证选择；兼容性说明：agentgateway 与 Agent Skills 规范为 Apache-2.0，DSH / Cordis 为 MIT；
7. 策略语言——已由 [ADR-0003](../adr/0003-policy-object-and-language.md) 选 Cedar 子集；剩余：子集范围
   的精确边界，以及与 CEL（agentgateway `mcpAuthorization`）的互操作；
8. `Intent` 改名——已由 [ADR-0006](../adr/0006-terminology-action-intent.md) 决定 `ActionIntent`；剩余：无，
   记录以备追溯；
9. 边缘账本离线一致性——保守规则先行（[07 §2.4](07-evolution-and-edge.md#24-明确边界)）；离线期间本地
   Grant 消耗、Effect 与 Approval 联网后与中心账本的合并与冲突规则待 ADR；
10. agentgateway 复用形态——作为库嵌入 `fysund` / 作为独立进程前置 / 仅借用 Pass / Reject / Mutated 语义与
    Cedar 策略格式自实现；无论哪种，语义与策略格式保持一致以便切换。
