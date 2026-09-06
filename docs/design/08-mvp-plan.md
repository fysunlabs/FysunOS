# 08 · MVP 计划

- 状态：设计稿 v0.1（2026-09-06）
- 前置：全部设计文档
- 目的：避免再次做成白皮书。第一个可运行闭环必须小、真实、可验证。

## 1. MVP 边界（MVP-first）

首个可运行路径只实现当前闭环与底线 F1–F8 要求的最小面：

- owner-local、单主体、单机；Scope 模型预留但不预建 RBAC、审批链、通用 capability 市场；
- 一个 Runtime（DSH，经 `fysun` bundle）；
- 一个不依赖任何业务对象的 Capability；
- 部署策略用本地文件；
- 不可省略：唯一权威写入者、Secret Store、persist-before-dispatch、预算/fencing、独立 Verifier、
  审计边界。

## 2. 首个闭环

以"趋势调研 → 内容包"能力为例（可换成任何不需要外部账号的能力）：

```text
1. capability.register        声明契约（输入源、输出内容包 schema、质量要求、预算/时间边界、Verifier=schema+来源校验）
2. activity.create + start    建立长期 Activity（周期触发，"先看过再发布"作为策略而非流程）
3. fysund 启动 DSH 子进程     dsh --profile fysun，start_execution 携带受控上下文视图与 Grant 摘要
4. DSH 执行                    工具调用经 tools/pre-execute → submit_candidate → Intent/Effect → execute → Receipt
5. 产物独立验证                Verifier 校验内容包 schema 与来源引用；Verification 报告写入账本与 Capability.evidence
6. 进程重启                    kill fysund 与 DSH；重启后 Activity 恢复到 waiting，未知结果 Effect 对账
7. 一次 Adjustment             用户校准"语气更克制"，scope=this_activity；下一 Execution 生效；adjustment.revert 可回退
8. 一次 Handoff                内容包中的配图交给另一 Capability（仍在 DSH 内以 subagent 执行）；Handoff 与子 Execution 在账本中可见
```

## 3. 阶段划分与验收

| 阶段 | 交付 | 验收（全部为可判定项） |
|---|---|---|
| **M0 账本与对象** | 治理账本 schema（event-sourced）、投影、replay；对象模型的 Rust 类型 | replay 两次得到相同投影 digest；负例：陈旧 epoch 写入被拒 |
| **M1 权威协议** | Intent/Effect/Receipt、Grant/预算、Verifier 框架、Secret Store、Provider 代理 | 重复请求幂等；预算耗尽拒绝；outcome-unknown 进入 reconcile/quarantine；明文 secret 不出现在任何日志/账本/argv |
| **M2 Runtime 接入** | `fysun` bundle（credentials / approval / session-link / tool-gate / context / rpc）、DSH 子进程管理 | 一次 DSH 会话的每个 tool/result 都能配对到一个 Effect；DSH 无法推进 Execution 状态（负例）；跨账本 replay 双向定位 |
| **M3 Activity 闭环** | Activity/Execution 状态机、周期触发、检查点、恢复顺序、Handoff | 上述闭环第 1–6、8 步全部通过；崩溃后不重放已 receipt 的 Effect |
| **M4 演进与入口** | Adjustment（apply/revert/promote）、`fysun` CLI、最小 Web 状态页 | 闭环第 7 步；Adjustment 触及扩权时被策略拦截（负例） |
| **M5 第一个应用** | Personal 作为应用切回 FysunOS（角色/团队/业务对象在应用层） | Personal 不再维护第二套执行路径 |

阶段可并行局部推进，但 M2 之前不引入第二个 Runtime，M3 之前不做边缘。

## 4. 效率与性能的度量

"精简高效"必须可测。基线先测、阈值后定，不能仅凭架构宣称性能提升：

- daemon 启动耗时、空闲内存；
- 调度延迟（trigger → start_execution）；
- 每次工具调用的账本事务数与 broker 往返延迟；
- 崩溃恢复时间；
- 接入一个新 Capability 所需步骤数。

## 5. 证据纪律（迁移自 CognitiveOS A7，作为流程规则）

- 本地、fixture、普通 CI 证据只推进实现进度；任何"支持"、"发布"、"Gate"类声明需要预登记的
  环境与流程。
- `not-run` 永不等于 pass；被解析器拒绝的命令记 `not-run`，不是测试失败。
- 每个测试/验证单元完成后立即追加到该活动唯一的 running report。

## 6. 非声明

本设计不声明：

- FysunOS 已实现任何代码；
- DSH 已被接入或已通过资格化；
- Cordis 已成为依赖；
- 任何性能、任务完成率、协作收益、边缘可用性数字；
- Personal 已迁移。

## 7. 开放问题（待后续 ADR）

1. `fysund` 与 Runtime 之间的传输：仅 stdio JSON-RPC，还是同时支持本地 socket / gRPC；
2. Secret Store 后端的首选实现（OS keyring vs 文件加密）；
3. Web UI 的技术栈与是否复用 Personal Control Plane 的组件；
4. Capability 契约中"质量目标"的机器可判定表示法（schema + 断言 vs 自然语言 + model-as-judge 的边界）；
5. 多主体（多人 / 多设备）Scope 模型何时激活；
6. 许可证选择。
