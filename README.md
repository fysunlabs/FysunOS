# FysunOS

> **FysunOS 是 Agent 能力的操作系统：它定义描述能力、访问资源、运行与交接工作的
> 公共契约，并提供让同一能力在任意 Agent 实现、Runtime 与环境中持续、受控、可恢复地
> 运行并从反馈中改进的运行服务。**

一句话职责：**管理"决策核心 × 资源 × 环境"的绑定，并可靠地运行这种绑定。**
决策核心（loop、prompt、工具选择、继续/停止策略）由 Runtime 提供；FysunOS 自己不写 loop。
"任意 Agent 实现"指任何实现 Runtime ABI 最小面（MCP 客户端 + 最小生命周期 RPC）并在 `describe()` 中
**如实声明其原生（非 MCP）工具**（`native_effect_tools[]`）的 Runtime；声明不实的 Runtime 不在承诺范围。
[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 是声明全部六个控制点的参考适配器（`fysun-dsh`），
其 headless 模式是 MCP-only 接入的验收基线；它不是唯一路径。

**对外坐标**（相对 [AOS 参考架构](https://arxiv.org/abs/2608.03214) 两平面的供给声明摘要）：Control & Governance
Plane（策略、授权、审批、审计、可观测性）**供给（Trust 除外）**；Runtime & Coordination Plane 中的生命周期、调度、交接、
上下文/记忆协调、验证**供给**，决策核心 loop、沙箱、流量管理**不供给**；平台服务（宿主 OS、容器、物理设施）不供给。
完整表见 [01 §1.4](docs/design/01-definition-and-principles.md)，按 Runtime 类型分列的 conformance 声明见
[13 威胁模型与 conformance 声明](docs/design/13-threat-model.md)。

- 状态：**设计阶段**，设计文档 **v0.7.0**（2026-09-08）。本仓库当前只有设计文档，没有可运行代码。
  v0.7.0 依据 [v0.7.0 全量重构蓝图](docs/research/20260908-v0.7.0-refactor-blueprint.md)（前置分析 + 外部调研 + 所有者两轮决策）
  对 v0.6.0（[归档](docs/archive/v0.6.0/ARCHIVE.md)）做全量重构：设计文档从 10 篇重组为 14 篇，ADR-0001 ~ 0006 正文直接修订，
  新增 ADR-0007 ~ 0012；v0.6.0 此前依据[前沿对照审查报告](docs/research/20260907-frontier-review-and-recommendations.md)修订自
  v0.5.0（[归档](docs/archive/v0.5.0/ARCHIVE.md)）。变更见 [CHANGELOG](docs/CHANGELOG.md)。
- 前身：CognitiveOS / CognitiveOS Personal（`agent-kernel` 仓库）。FysunOS 是基于对该项目、
  DeepSeek Harness 与 Cordis 的调研重新设计的系统，**不是**其原地重构；有价值的部分按
  [迁移清单](docs/design/09-migration-inventory.md) 选择性迁入。
- 决策记录：[docs/adr/](docs/adr/README.md)。设计文档之间冲突时，以 ADR 为准；术语冲突时以
  [00 术语表](docs/design/00-terminology.md) 为准；开放问题统一登记于 [12 §7](docs/design/12-mvp-plan.md)。

## 文档索引

编号顺序为"基础（00–03）→ 接口（04–06）→ Runtime 接入（07–08）→ 迁移 / 演进 / 边缘（09–11）→ 计划与保证（12–13）"；
文件名以 [00 §0](docs/design/00-terminology.md) 的文档地图为定稿。

| 编号 | 文档 | 内容 |
|---|---|---|
| 00 | [术语与对象命名对照表](docs/design/00-terminology.md) | 全仓库对象名、状态名、事件名、字段名、错误码、组件名的唯一基准；v0.5.0 → v0.6.0 → v0.7.0 改名对照；Level 速记禁令；事件族全集 |
| 01 | [定义与设计原则](docs/design/01-definition-and-principles.md) | FysunOS 定义（含可判定限定）、对外坐标、Capability 定义、八条设计原则、底线 F1–F9（按 Runtime 类型限定范围）、不做什么、五个正交概念 |
| 02 | [对象模型](docs/design/02-object-model.md) | 一级对象（Capability、Activity/Execution、Grant、ActionIntent/Effect、Verification、Handoff、Resource、Adjustment、Approval、Policy、Memory、Node）、状态机、字段、横切来源标签、AOS 映射 |
| 03 | [系统架构、账本与恢复](docs/design/03-architecture.md) | 四个子系统、进程拓扑、双账本、event-sourced 账本（事件族只引用 00 §4）、恢复模型（含重新武装）、Policy 模型、可观测性 |
| 04 | [Capability API 与错误语义](docs/design/04-capability-api.md) | 应用 ↔ OS 的全部方法（含 `approval.*`、`policy.*`、`runtime.*`）；Agent Skills → Capability 映射；错误码全表 |
| 05 | [Runtime ABI：最小生命周期 RPC、控制点与 A2A](docs/design/05-runtime-abi.md) | `describe` / `start` / `resume` / `cancel` / `deliver_input` / `checkpoint` / `health`；六控制点义务；Runtime → OS `os.*`；不得调用清单；`session_link`；A2A 外部 Handoff |
| 06 | [MCP 守卫规范（`fysun-mcp-guard`）](docs/design/06-mcp-guard.md) | `tools/list` 过滤与不可区分性；`tools/call` pre / 执行 / post；Effect 类别三源合一；审计快路径与 `audit/observed`；结果标签与 `flags`；tasks / elicitation；守卫系统工具；下游 server 凭据与会话；hooks 阻断路径 |
| 07 | [Runtime 接入与资格化](docs/design/07-runtime-onboarding.md) | Runtime 作为通用 Resource 类型；`native_effect_tools[]`；资格化纪元与 `runtime/disqualified`；MCP-only 直接接入（context_view 渲染与 `deliver_input` 投递路径）；hooks.json 阻断路径；边缘 Runtime；通用钉版与 CI 漂移预警 |
| 08 | [DSH 参考适配器 `fysun-dsh`](docs/design/08-dsh-adapter.md) | DSH 定位与不吸收项、七组件归类、`approval/request` waterfall 拦截、DSH 原生工具治理路径、headless 模式、精确钉版、`guard/` 包消歧、Cordis 映射（附录 A） |
| 09 | [迁移清单](docs/design/09-migration-inventory.md) | 从 CognitiveOS / Personal / DSH / Cordis 各迁什么、不迁什么；来源标签先例更正；Agent Teams 收窄；新建项 |
| 10 | [演进](docs/design/10-evolution.md) | 三种变化、规则、机制（Adjustment 预测与评估、证据门控、健康查询）、质量目标表示法、技能库生命周期对照 |
| 11 | [边缘与离线账本一致性](docs/design/11-edge-and-offline-ledger.md) | 边缘场景、部署形态、物理 Effect 扩展、明确边界、可移植性三层；§3 离线账本一致性（节点归属、escrow、合并与冲突枚举） |
| 12 | [MVP 计划](docs/design/12-mvp-plan.md) | 边界、首个闭环、M0–M5（M2a / M2b）验收、边缘阶段、度量、证据纪律（四类状态用语）、统一开放问题清单 |
| 13 | [威胁模型与 conformance 声明](docs/design/13-threat-model.md) | 资产、信任边界、T1–T11 攻击向量与机制、明确不防、负例向量索引、按 Runtime 类型分列的 conformance 表 |
| — | [来源登记](docs/research/sources.md) | 本设计依据的精确 revision、论文、协议规范与文档 |
| — | [前沿对照审查报告](docs/research/20260907-frontier-review-and-recommendations.md) | 2026-09-07 对 v0.5.0 的逐条审查；v0.6.0 的变更依据 |
| — | [v0.7.0 全量重构蓝图](docs/research/20260908-v0.7.0-refactor-blueprint.md) | 2026-09-08 所有者两轮决策、逐篇改动规格、ADR 计划；v0.7.0 的实施基准（完成后转为历史记录） |
| — | [CHANGELOG](docs/CHANGELOG.md) | 版本变更日志 |
| — | [归档 v0.6.0](docs/archive/v0.6.0/ARCHIVE.md) | v0.7.0 重构前的完整快照 |
| — | [归档 v0.5.0](docs/archive/v0.5.0/ARCHIVE.md) | v0.6.0 修订前的完整快照 |

## 命名（暂定）

与 [00 §9](docs/design/00-terminology.md) 一致；冲突时以 00 为准。

| 名称 | 含义 |
|---|---|
| `fysund` | 权威 daemon（Rust），唯一权威写入者；唯一审批面提供者（Web UI / CLI / `fysund` 发起的 elicitation） |
| `fysun` | CLI；子命令含 `fysun hook pre-tool-use`（hooks.json PreToolUse 阻断路径的命令端，调 `fysund` 本地 RPC，Policy `deny` 时退出码 2） |
| `fysun-mcp-guard` | MCP 守卫组件（`fysund` 内），自实现，借用 agentgateway `Pass` / `Reject` / `Mutated` 语义与 Cedar 格式（[ADR-0012](docs/adr/0012-mcp-guard-self-implementation.md)）；与 DSH 的 `guard/` 包（loop 卫生插件）无关 |
| `fysun-dsh` | DSH 参考适配器，声明全部六个控制点；技术形态为叠在 `dsh-base` 之上的 bundle + `fysun` profile，bundle 校验 `dsh_version` 与 `schema_digest`；DSH 原生工具经 `approval/request` 拦截 + `tools/post-execute` receipt 路径治理 |
| `fysund-edge` | 边缘节点上的精简 daemon（同一账本模型，内含确定性包络校验器）；持有 escrow Grant，离线按 [11 §3](docs/design/11-edge-and-offline-ledger.md) 运行，联网后按 `origin_seq` 推送合并 |

## 许可证

待定。依赖与参照物的许可证：DeepSeek Harness 与 Cordis 为 MIT；Cedar 与 Agent Skills 规范为 Apache-2.0。
