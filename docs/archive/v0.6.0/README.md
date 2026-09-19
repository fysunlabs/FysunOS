# FysunOS

> **FysunOS 是 Agent 能力的操作系统：它定义描述能力、访问资源、运行与交接工作的
> 公共契约，并提供让同一能力在任意 Agent 实现、Runtime 与环境中持续、受控、可恢复地
> 运行并从反馈中改进的运行服务。**

一句话职责：**管理"决策核心 × 资源 × 环境"的绑定，并可靠地运行这种绑定。**
决策核心（loop、prompt、工具选择、继续/停止策略）由 Runtime 提供；FysunOS 自己不写 loop。
"任意 Agent 实现"指任何实现 Runtime ABI 最小面（MCP 客户端 + 最小生命周期 RPC）的 Runtime；
[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 是第一个 Level-2 参考适配器，不是唯一路径。

**对外坐标**（相对 [AOS 参考架构](https://arxiv.org/abs/2608.03214) 两平面的供给声明摘要）：Control & Governance
Plane（策略、授权、审批、审计、可观测性）**全量供给**；Runtime & Coordination Plane 中的生命周期、调度、交接、
上下文/记忆协调、验证**供给**，决策核心 loop、沙箱、流量管理**不供给**；平台服务（宿主 OS、容器、物理设施）不供给。
完整表见 [01 §1.4](docs/design/01-definition-and-principles.md)，conformance 声明见 [09 威胁模型](docs/design/09-threat-model.md)。

- 状态：**设计阶段**，设计文档 **v0.6.0**（2026-09-07）。本仓库当前只有设计文档，没有可运行代码。
  v0.6.0 依据[前沿对照审查报告](docs/research/20260907-frontier-review-and-recommendations.md)修订自 v0.5.0
  （[归档](docs/archive/v0.5.0/ARCHIVE.md)）；变更见 [CHANGELOG](docs/CHANGELOG.md)。
- 前身：CognitiveOS / CognitiveOS Personal（`agent-kernel` 仓库）。FysunOS 是基于对该项目、
  DeepSeek Harness 与 Cordis 的调研重新设计的系统，**不是**其原地重构；有价值的部分按
  [迁移清单](docs/design/06-migration-inventory.md) 选择性迁入。
- 决策记录：[docs/adr/](docs/adr/README.md)。设计文档之间冲突时，以 ADR 为准；术语冲突时以
  [00 术语表](docs/design/00-terminology.md) 为准。

## 文档索引

| 编号 | 文档 | 内容 |
|---|---|---|
| 00 | [术语与对象命名对照表](docs/design/00-terminology.md) | 全仓库对象名、状态名、事件名、字段名、错误码、组件名的唯一基准；v0.5.0 → v0.6.0 改名对照 |
| 01 | [定义与设计原则](docs/design/01-definition-and-principles.md) | FysunOS 定义、对外坐标、Capability 定义、八条设计原则、底线 F1–F9、不做什么 |
| 02 | [对象模型](docs/design/02-object-model.md) | 五个正交概念与一级对象（Capability、Activity/Execution、Grant、ActionIntent/Effect、Verification、Handoff、Adjustment、Approval、Policy、Memory）与来源标签 |
| 03 | [系统架构](docs/design/03-architecture.md) | 四个子系统、进程拓扑（MCP 守卫）、双账本、event-sourced 账本、事件族、恢复模型、Policy 模型、可观测性 |
| 04 | [接口与 ABI](docs/design/04-abi.md) | Capability API、Runtime ABI（MCP 守卫 + 最小生命周期 RPC + 可选控制点）、A2A 外部 Handoff、Agent Skills 实现包、错误语义 |
| 05 | [Runtime 接入与 DSH 适配器](docs/design/05-runtime-and-dsh-integration.md) | Level-1 直接接入；DSH 作为第一个 Level-2 参考适配器（`fysun-dsh`）；双账本关联 |
| 06 | [迁移清单](docs/design/06-migration-inventory.md) | 从 CognitiveOS / Personal / DSH / Cordis 各迁什么、不迁什么；v0.6.0 新建项 |
| 07 | [演进与边缘](docs/design/07-evolution-and-edge.md) | 轻量演进模型（Adjustment 预测与评估）、质量目标表示法、能力谱系、IoT/边缘运行与安全包络 Policy |
| 08 | [MVP 计划](docs/design/08-mvp-plan.md) | 首个可运行闭环、阶段划分（M2a/M2b）、验收标准、度量、开放问题 |
| 09 | [威胁模型与 conformance 声明](docs/design/09-threat-model.md) | 资产、信任边界、T1–T10 攻击向量与机制、明确不防、负例向量索引、conformance 声明 |
| — | [来源登记](docs/research/sources.md) | 本设计依据的精确 revision、论文、协议规范与文档 |
| — | [前沿对照审查报告](docs/research/20260907-frontier-review-and-recommendations.md) | 2026-09-07 对 v0.5.0 的逐条审查；v0.6.0 的变更依据 |
| — | [CHANGELOG](docs/CHANGELOG.md) | 版本变更日志 |
| — | [归档 v0.5.0](docs/archive/v0.5.0/ARCHIVE.md) | 修订前的完整快照 |

## 命名（暂定）

| 名称 | 含义 |
|---|---|
| `fysund` | 权威 daemon（Rust），唯一权威写入者 |
| `fysun` | CLI |
| `fysun-mcp-guard` | `fysund` 内的 MCP 守卫组件，位于 harness 与所有工具/MCP server 之间；是否复用 agentgateway 待评估（[08 开放问题 10](docs/design/08-mvp-plan.md#7-开放问题待后续-adr)） |
| `fysun-dsh` | DSH 的 Level-2 参考适配器（技术形态为叠在 `dsh-base` 之上的 bundle + `fysun` profile） |
| `fysund-edge` | 边缘节点上的精简 daemon（同一账本模型，内含确定性包络校验器） |

## 许可证

待定。依赖与参照物的许可证：DeepSeek Harness 与 Cordis 为 MIT；agentgateway 与 Agent Skills 规范为 Apache-2.0。
