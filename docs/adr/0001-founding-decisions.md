# ADR-0001 · FysunOS 创立决策

- 状态：accepted（v0.7.0 修订）
- 日期：2026-09-06
- 决策者：owner
- 背景来源：`agent-kernel` 仓库中关于 CognitiveOS 重定义的探讨记录
  （`docs/architecture/rebuild-0906`）及同日对 DeepSeek Harness 与 Cordis 的调研讨论。
- 修订：v0.7.0（2026-09-08）按 [ADR-0002](0002-protocol-first-runtime-boundary.md) 落实对 §D2 的 supersede 并消除过时表述；见文末修订历史。

## 背景

CognitiveOS / CognitiveOS Personal 已实现一个 owner-local 的持久 Agent 控制平面（Rust
daemon 唯一权威写入者、SQLite WAL、Intent/Effect、独立 verifier、replay），但其架构叙事
（双内核 / 七层 / 三平面 / 六族资源）远大于运行需求，且可复用执行能力与 Personal 业务语义
混在一起。与此同时，DeepSeek Harness 已提供完整、可替换、以 Cordis 组合的 Agent 决策核心
与 Runtime。owner 决定：不再原地重构 CognitiveOS，而是基于本轮调研新设计 FysunOS，选择性
迁移有价值的部分。

## 决策

### D1 · 定位与定义

FysunOS 是 **Agent 能力的操作系统**：定义描述能力、访问资源、运行与交接工作的公共契约，并
提供让同一能力在任意 Agent 实现、Runtime 与环境中持续、受控、可恢复地运行并从反馈中改进的
运行服务。核心职责是"管理决策核心 × 资源 × 环境的绑定，并可靠地运行这种绑定"。
可靠是系统承担的底线，提高 Agent 完成实际工作的有效性是目的。

采用五个正交概念：**Capability / 决策核心（Agent Core）/ Resource / Activity·Execution /
Runtime**。"认知资源"归入 Resource；"Loop"归入决策核心的控制策略；"Harness"降为工程术语；
"Runtime"收窄为承载与适配。一句话定义中"自进化"表述为"从反馈中改进"。

### D2 · DSH 为声明全部六个控制点的参考适配器；Cordis 语义仅在适配器内部

（v0.7.0 按 ADR-0002 §D1/§D5/§D6 修订；原文见 `docs/archive/v0.6.0/adr/0001-founding-decisions.md`。）

- DeepSeek Harness 是**声明全部六个控制点的参考适配器** `fysun-dsh`，不是默认 Runtime，也不是唯一路径；任何实现 Runtime ABI 最小面
  （MCP 客户端 + 最小生命周期 RPC）的 Runtime 都可接入。技术形态为叠在 `dsh-base` 之上的 bundle + `fysun` profile，以**隔离子进程**
  经 `sdk` / `acp` profile 进程外接入。**不 fork、不 vendor、不进程内链接。**
- Cordis 语义（可撤销效果、反应式依赖）仅作 `fysun-dsh` 适配器内部对 Cordis 生命周期的映射参考（[08 附录 A](../design/08-dsh-adapter.md)）；
  **不作为内核基底**，不进入公共接口。
- FysunOS 不再自建 loop、prompt 装配、工具流水线、会话日志、compaction、subagent 传输、
  sandbox、聊天 UI、skills、终端、LSP。

否决的替代方案：
- 以 DSH 为 OS、FysunOS 写成 DSH 插件——权威层将与模型驱动的工具执行和可自我修改的动态
  Cordis 包同进程，违背底线 F1–F5；且绑定 Node 单语言与 developer preview 的破坏性变更。
- 以 Cordis 重写内核——Cordis 解决进程内动态组合，不解决持久、权威、外部效果与跨进程。

### D3 · 独立仓库

FysunOS 位于独立仓库（本机 `D:\fysun-os`，稍后同步到新的 GitHub 仓库），从 `agent-kernel`
按 [迁移清单](../design/09-migration-inventory.md) 选择性迁移。不在 `agent-kernel` 内新增
子项目，不原地改名。

### D4 · 权威 daemon 用 Rust

`fysund` 用 Rust 实现，迁移 CognitiveOS 现有 kernel / store 中经验证的机制（ActionIntent/Effect
——CognitiveOS 中原名 Intent/Effect，ADR-0006——、replay、recovery、context、Verifier、Provider 代理）。否决 TypeScript 方案：权威层与 Runtime
同技术栈的便利不足以抵消 TCB 扩大与边缘可移植性的损失。

### D5 · 治理账本采用有边界的 event-sourced 模型

事件是唯一真源；状态是可重建的投影；缓存与索引是派生数据。模型流式 chunk、KV cache、高频
传感器采样、Runtime 内部步骤不进入治理账本。会话账本（Runtime 拥有）与治理账本关联不合并。
否决"可变状态表 + 事件表并列"的 CognitiveOS 现状：两个并列事实来源会在恢复与审计时产生
分歧。

### D6 · 不进入 OS 的概念

角色、团队、数字员工、Project、Routine、业务流程、Agent Shell、聊天消息属于应用层或
Runtime 会话账本，不进入 FysunOS 对象模型。协作不设专门子系统，作为 Capability 可用的普通
行为通过 Handoff 记录。

### D7 · 部署策略声明、能力不自扩权

不内置统一风险等级模型；部署环境声明什么动作可自动执行、何时需人工；OS 执行边界。能力不得
自行扩大 Grant，不得隐瞒不满足的运行条件。

### D8 · 演进轻量

能力调整以 Adjustment 记录（作用范围、生效时点、回退目标、风险类），默认直接生效并自动
留痕；只有触及扩权、新外部动作、预算上调、设备安全时执行部署策略。不设候选→评估→灰度→晋升
的强制流水线；契约不兼容变化默认派生新 Capability 并保留谱系。

## 后果

- 需要维护 `fysun-dsh` 对一个**钉住 revision** 的 DSH 的适配与适配测试，并以 CI 每日对上游最新 tag 跑 conformance 作非阻塞漂移预警
  （[08 §6](../design/08-dsh-adapter.md)）；DSH 破坏性变更是持续成本，但不在关键路径上（M2a 以 DSH headless 作 MCP-only 基线，不依赖适配器）。
- 双账本带来跨账本 replay 测试的硬成本，换来 OS 真源不依附任何 harness。
- Personal 1.x 自研 Runtime 片段（Pi adapter、conversation projection 等）进入废弃路径；
  Personal 作为第一个应用切回 FysunOS。
- 边缘场景需要另一个实现 Runtime ABI 最小面的 Runtime，这是 OS ≠ DSH 的直接后果与理由。

## 待后续 ADR 的问题

见 [12 MVP 计划 §7](../design/12-mvp-plan.md)。

## 修订历史

- v0.7.0（2026-09-08）：§D2 由"DSH 为默认参考 Runtime；Cordis 为 TS 侧组合后端"重写为"DSH 为声明全部六个控制点的参考适配器 `fysun-dsh`；
  Cordis 语义仅在适配器内部"，保留"不 fork / 不 vendor / 不进程内链接"与"不自建 loop 等"清单；§D3 迁移清单链接改指新 09；§D4 括注
  `Intent/Effect` → `ActionIntent/Effect`；后果段"与 DSH 版本逐版对应的 `fysun` bundle" → "`fysun-dsh` 对一个钉住 revision 的适配与 CI 漂移预警"，
  "Harness ABI" → "Runtime ABI 最小面"；待后续段链接改指新 12。为什么：ADR-0002 §D1/§D5/§D6 已 supersede §D2 但正文未动，导致三处过时表述
  （蓝图 §0 C1-7）；v0.7.0 起 ADR 正文允许直接修订（蓝图 §I.1 决策 2）。对应蓝图决策：§I.1 决策 1、2；§C.1 ADR-0001 行。
