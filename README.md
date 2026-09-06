# FysunOS

> **FysunOS 是 Agent 能力的操作系统：它定义描述能力、访问资源、运行与交接工作的
> 公共契约，并提供让同一能力在任意 Agent 实现、Runtime 与环境中持续、受控、可恢复地
> 运行并从反馈中改进的运行服务。**

一句话职责：**管理"决策核心 × 资源 × 环境"的绑定，并可靠地运行这种绑定。**
决策核心（loop、prompt、工具选择、继续/停止策略）由 Runtime 提供，默认 Runtime 是
[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)；FysunOS 自己不写 loop。

- 状态：**设计阶段**（2026-09-06 起）。本仓库当前只有设计文档，没有可运行代码。
- 前身：CognitiveOS / CognitiveOS Personal（`agent-kernel` 仓库）。FysunOS 是基于对该项目、
  DeepSeek Harness 与 Cordis 的调研重新设计的系统，**不是**其原地重构；有价值的部分按
  [迁移清单](docs/design/06-migration-inventory.md) 选择性迁入。
- 决策记录：[docs/adr/](docs/adr/README.md)。设计文档之间冲突时，以 ADR 为准。

## 文档索引

| 编号 | 文档 | 内容 |
|---|---|---|
| 01 | [定义与设计原则](docs/design/01-definition-and-principles.md) | FysunOS 定义、Capability 定义、八条设计原则、不做什么 |
| 02 | [对象模型](docs/design/02-object-model.md) | 五个正交概念与核心对象（Capability、Activity/Execution、Grant、Intent/Effect、Handoff、Adjustment…） |
| 03 | [系统架构](docs/design/03-architecture.md) | 四个子系统、进程拓扑、event-sourced 账本、事件族 |
| 04 | [接口与 ABI](docs/design/04-abi.md) | Capability API、Harness ABI、Composition ABI |
| 05 | [Runtime 与 DSH 接入](docs/design/05-runtime-and-dsh-integration.md) | DSH 作为默认参考 Runtime 的接入方式：`fysun` bundle、seam 替换、双账本 |
| 06 | [迁移清单](docs/design/06-migration-inventory.md) | 从 CognitiveOS / Personal / DSH / Cordis 各迁什么、不迁什么 |
| 07 | [演进与边缘](docs/design/07-evolution-and-edge.md) | 轻量演进模型（Adjustment）、能力谱系、IoT/边缘运行 |
| 08 | [MVP 计划](docs/design/08-mvp-plan.md) | 首个可运行闭环、阶段划分、验收标准、非声明 |
| — | [来源登记](docs/research/sources.md) | 本设计依据的精确 revision、论文与文档 |

## 命名（暂定）

| 名称 | 含义 |
|---|---|
| `fysund` | 权威 daemon（Rust），唯一权威写入者 |
| `fysun` | CLI |
| `fysun-dsh` | 叠在 `dsh-base` 之上的 DSH bundle/profile，把 DSH 接成 FysunOS Runtime |
| `fysund-edge` | 边缘节点上的精简 daemon（同一账本模型） |

## 许可证

待定（依赖的 DeepSeek Harness 与 Cordis 均为 MIT）。
