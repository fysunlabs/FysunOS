# 来源登记

本设计依据的精确来源。迁入代码或语义时在此追加来源 revision 与路径。v0.7.0（2026-09-08）按
[v0.7.0 全量重构蓝图](20260908-v0.7.0-refactor-blueprint.md) §B 修订：更正 Cordis 版本表述、更新 DSH 用途、补登 v0.7.0 外部调研的一手资料。
未能确定精确 revision 的条目写"待查证"，不臆造。

## 代码基线

| 项目 | Revision | 位置（调研时） | 用途 |
|---|---|---|---|
| CognitiveOS / Personal（`agent-kernel`） | `6f562bf1bdb86582955de36eb93e51126573f89c`（分支 `personal/P14-T04-member-join`） | `D:\agent-kernel` | 迁移来源：Intent / Effect（迁入后为 ActionIntent / Effect）、replay、recovery、context、Verifier、Provider 代理、DSH broker 经验 |
| CognitiveOS 公开仓库（`agentkernel/cognitive-os`，即 `agent-kernel` 的公开形态） | 待查证（2026-09-08 调研时的默认分支 HEAD） | github.com/agentkernel/cognitive-os | v0.7.0 一手资料核对：`core/crates/cognitive-kernel/src/context.rs`（`LoadedContextItemTrustLevel {Control, Authoritative, Verified, Untrusted}`、`UntrustedInput` 分区角色）、`effects.rs`（七属性模型：幂等冲突硬拒绝、补偿需新 Grant + 新请求记录）、README 与 `core/conformance/README.md`（四类状态用语 `specified / implementation available / test executed / implemented`）、`core/crates/cognitive-akp/src/deepseek_harness.rs`（入站 `authority_claim` / `secret_shaped` 标志）、`cognitive-akp/src/harness.rs::LoopDriver`（loop 边界在内核内，FysunOS 不照搬）；见 [09 迁移清单](../design/09-migration-inventory.md) §1 |
| DeepSeek Harness | `cd5ef8148158c3a752a658978873241fdf8e2bbc` | `D:\agent-kernel\artifacts\personal-2.0.0-dev-prep\deepseek-harness`（只读副本） | 参考适配器 `fysun-dsh` 的上游（声明全部六个控制点）；其 headless 模式为 M2a 的 MCP-only 基线；架构、安全、术语、seam 目录、Agent Teams、workflow、schedule 文档 |
| Cordis | 上游 cordis **4.0.0-rc.7**（DSH vendored，见 DSH `docs/rescope.md`；DSH 清单 `vendor/cordis/package.json` 版本号 **4.0.1**）；上游最新 tag **v4.0.0-rc.9**（对应 commit 待查证） | DSH 副本 `vendor/cordis/`；github.com/cordiverse/cordis | `fysun-dsh` 内部适配器生命周期（[08 附录 A](../design/08-dsh-adapter.md)）的语义来源；rc.9 为 CI 每日漂移预警的比对目标（08 §6） |
| CognitiveOS V2 重构研究 | 2026-08-21，基线 CognitiveOS `742a9346…`、DSH `528c682e…`、Cordis `8cc9e33f…` | `D:\agent-kernel\docs\research\20260821-cognitiveos-v2-restructuring-architecture.md` | 8 primitive 最小内核、双账本、Harness ABI、Composition ABI、DSH / Cordis 策略表（登记时原文，其中 Harness ABI 与 Composition ABI 在 FysunOS 中分别对应 Runtime ABI 与内部适配器生命周期） |

v0.7.0 起（[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md) v0.7.0 修订、[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md)）：
DSH 的角色为声明全部六个控制点的参考适配器 `fysun-dsh` 的上游，其 headless 模式为 M2a 的 MCP-only 基线（查证项"DSH headless 是否有独立于 bundle 的
原生工具"登记于 [12 §7](../design/12-mvp-plan.md)）；Cordis 语义仅作 `fysun-dsh` 适配器内部参考。v0.6.0 表中 Cordis 行写的"4.0.1"是 DSH 清单版本号，
不是上游版本；上游 vendored 版本为 4.0.0-rc.7，本表已更正。DSH 与 Cordis 为同一批作者，Cordis rc 变更直接传导到 DSH，这是 `fysun-dsh` 精确钉版的依据。

## 论文与一手资料

| 资料 | 用途 |
|---|---|
| Mei et al., *AIOS: LLM Agent Operating System*, arXiv:2403.16971（COLM 2025） | Agent OS 内核词汇（scheduler / context / memory / storage / tool / access manager）；FysunOS 在其之上增加 ActionIntent / Effect、fencing、独立验证 |
| Shi, Zhang, Cui, *A Programming Paradigm for Spatiotemporal Composability*, arXiv:2608.25512（2026-08-13） | Cordis 的理论基础：可撤销效果（时间可组合性）与反应式依赖（空间可组合性）；`fysun-dsh` 内部适配器生命周期（08 附录 A）的语义来源 |
| Cordis 源码 `packages/core/src/context.ts`（`isolate(name)`）与上游 tag v4.0.0-rc.9（github.com/cordiverse/cordis；commit 待查证） | 一手资料确认：`isolate(name)` 只是服务名的命名空间符号，不是安全隔离，不构成信任边界（[08 §1](../design/08-dsh-adapter.md)、[09 §4](../design/09-migration-inventory.md)）；rc.9 为漂移预警比对目标 |
| Anthropic, *How we built our multi-agent research system*（engineering blog） | 多 Agent 收益取决于任务结构而非 Agent 数量；支撑"协作是普通能力行为、不默认多 Agent" |
| DeepSeek Harness 文档（v0.6.0 登记）：`docs/architecture.md`、`docs/cordis-primer.md`、`docs/capability-seams.md`、`docs/glossary.md`、`docs/subsystems/agent-team.md`、`docs/subsystems/workflow.md`、`docs/user/guide/schedule.md`、`SAFETY.md` | DSH 设计哲学与边界的原文依据；`docs/subsystems/agent-team.md` 标注 Agent Teams（roster / mailbox / task DAG）为实验特性 → FysunOS 只有 Handoff 状态迁移与 deliverables 入治理账本（02 §2.8） |
| DeepSeek Harness 文档（v0.7.0 补登，revision 同上表 DSH 基线）：`docs/tool-execution-pipeline.md`、`docs/agent-lifecycle.md`、`docs/rescope.md`、`docs/subsystems/approval.md`、`docs/subsystems/permission-presets.md`、`docs/subsystems/scope.md`、`docs/subsystems/subagent.md`、`docs/subsystems/credentials.md`、`packages/guard/README.md` | 一手资料确认：`tools/post-execute` 为工具流水线 accept / block / replace 阶段 → DSH 原生工具 receipt 采集点（08 §2）；goal `armed / disarmed` 故意不持久化 → 重新武装入账本（03 §6）；`docs/rescope.md` 记载 vendored Cordis 4.0.0-rc.7；approval policy / sandbox mode / permission preset 以会话日志最后一条事件为真源 → FysunOS 不照搬（02 §2.12）；scope 过滤掉的工具不在 prompt 里且拒绝执行、与不存在的工具不可区分 → `tools/list` 不可区分性（06 §2）；subagent provider 能力位在 `start()` 前校验、缺失则 `UNSUPPORTED_CAPABILITY` 大声失败 → `start_execution` 大声失败（05 §2）；`SubagentResult.diagnostic` 剥离工具输入 / 文件内容 / 环境值 / 凭据、上限 4096 字节 → `deliverables[].summary` / `error_report`（02 §2.3、§2.8）；凭据每操作重解析、描述面无值槽、空值即未配置 → SecretRef 三规则（02 §2.9）；`guard/` 是 loop 卫生插件 → 与 `fysun-mcp-guard` 消歧（00 §9） |
| CognitiveOS `core/crates/cognitive-kernel/src/context.rs`（`LoadedContextItemTrustLevel {Control, Authoritative, Verified, Untrusted}`；revision 见上表公开仓库行，待查证） | 一手资料确认：上下文条目信任等级只覆盖上下文装配阶段，未沿参数流传播到 Effect → 来源标签 `label` 的上下文层部分先例，FysunOS 扩展新建（[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)、02 §2.15、09 §1） |
| Skalse et al., 2022（reward hacking 可形式化）及 2026 loop engineering / reward-hacking benchmark 综述 | 独立验证（F4）的依据 |
| Sharma, Ye, Ge, Zhang 等，*The Agent Operating System (AOS): A Reference Operating Architecture for Distributed Agentic Systems*（AOS-0001），arXiv:2608.03214（2026-08） | 对外坐标（01 §1.4）、对象映射（02 §4）、不变量、威胁模型与 conformance 声明（13）的参照；Confidence 控制动作六元组 → `assurance`（映射到 Policy `outcome` 四值，控制动作为 Activity 内部词汇） |
| Debenedetti 等，*Defeating Prompt Injections by Design*（CaMeL），arXiv:2503.18813 | 值级 capability 与控制 / 数据流分离；来源标签的范围界定（不做完整 CaMeL） |
| Shi 等，*Progent: Securing AI Agents with Privilege Control*，arXiv:2504.11703 | 符号化策略、SMT 收窄 / 扩张判定、单调限制 → F9 |
| Narisetty 等，*Adaptive Evaluation of Out-of-Band Defenses Against Prompt Injection in LLM Agents*，arXiv:2606.26479（2026-06） | 引用监视器 + 最小权限 + 完整性标签的统一框架；静态基准局限 |
| *Dynamic Agent Skills: A Lifecycle Survey and Taxonomy of Evolving Skill Libraries*，arXiv:2607.10113（2026-07） | 八阶段生命周期 → 演进模型缺口（[10 §1.6](../design/10-evolution.md)） |
| *SkillOps: Managing LLM Agent Skill Libraries as Self-Maintaining Software Ecosystems*，arXiv:2605.13716 | Skill Contract (P,O,A,V,F) → 质量目标表示法（10 §1.5） |
| Lin 等，*Agentic Harness Engineering: Observability-Driven Automatic Evolution of Coding-Agent Harnesses*（AHE），arXiv:2604.25850 | 决策可观测性 → `Adjustment.predicted_effect`（可选字段） |
| *Self-Improvements in Modern Agentic Systems: A Survey*，arXiv:2607.13104 | 自改进的参数 / 脚手架双回路 |
| Weng，*Harness Engineering for Self-Improvement*，lilianweng.github.io（2026-07-04） | harness 定义；harness-updating vs harness-benefit |
| *Can LLM-as-a-Judge Reliably Verify Rubrics in Agentic Scenarios?*（RuVerBench），arXiv:2606.29920 | judge 噪声 → Verification 校准字段 |
| *Measurement Without Validity: The Compounding Reliability Problem in Agentic AI Evaluation*，arXiv:2608.00794 | 可靠性阈值 α≥0.67 / 0.70 / 0.80 → judge 项替代人工的条件 |
| *AdaRubric*，arXiv:2603.21362 | 任务自适应 rubric |
| Li 等，*MemOS: A Memory OS for AI System*，arXiv:2507.03724 | 记忆作为可调度资源；不自建记忆引擎的依据（Memory 对象） |
| *ROSClaw: An OpenClaw ROS 2 Framework for Agentic Robot Control and Interaction*，arXiv:2603.26997 | 执行层契约、前置校验器、安全包络 Policy（[11 §2.3](../design/11-edge-and-offline-ledger.md)） |
| URML（robot intent language + five-pass validator），github.com/URML-MARS/URML | "通过校验 ≠ 世界安全" → 包络校准为部署者责任（11 §2.4） |
| Pan 等，*Measuring Agents in Production* / *Characterizing Agents in Production*（CAP），arXiv:2512.04123（ICML 2026） | 生产 Agent 的 HITL、评估与可靠性数据 → Approval 一级对象 |

## 协议与规范

| 资料 | 版本 / 日期 | 用途 |
|---|---|---|
| Model Context Protocol 规范 | 2025-11-25；SEP-1686（Tasks）、SEP-1036（URL elicitation）、SEP-1577（sampling with tools）、SEP-1649 / SEP-1960（server cards，草案）、SEP-2663（Tasks 移入扩展，草案） | Runtime ABI 的 MCP 守卫层（[06](../design/06-mcp-guard.md)）；Tasks 与 elicitation 为 MCP-only Runtime 的审批投递载体（07 §3） |
| Agent2Agent 协议 | v1.0（2026-03） | Handoff 外部形态与状态机对齐（02 §2.8、[05 §7](../design/05-runtime-abi.md)） |
| Agent Skills 规范 | agentskills.io，2025-12-18 公开；Discussion #415 SemVer 提案（2026-06-05） | `Capability.implementation` 的实现包格式（[04 §2](../design/04-capability-api.md)） |
| Claude Code hooks 文档（`hooks.json`，`PreToolUse` command hook；退出码 2 阻断本次工具调用） | 文档版本 / 日期待查证 | hooks 阻断路径（06 §9、07 §4、ADR-0007 §D3）：`fysun hook pre-tool-use` 的宿主接口；仅阻断，不计入 conformance 主路径 |
| Codex hooks 文档（`hooks.json`，`PreToolUse`；与 Claude Code 兼容格式） | 文档版本 / 日期待查证 | 同上；DSH 亦兼容该格式 |
| agentgateway | AAIF 项目（Rust，Apache-2.0）；1.3 版 `mcpGuardrails` | 语义借用来源：`Pass` / `Reject` / `Mutated` 三值与 Cedar 格式；`fysun-mcp-guard` 自实现，不作为依赖、不进程前置（[ADR-0012](../adr/0012-mcp-guard-self-implementation.md)） |
| Cedar 策略语言 | AWS 开源（Apache-2.0） | Policy 对象的策略语言（ADR-0003）；子集范围与实体模型见 [ADR-0009](../adr/0009-cedar-subset-and-entity-model.md) |
| OpenTelemetry GenAI 语义约定 | `semantic-conventions-genai` 仓库（2026-06 起），截至 2026-08-13 全部 Development | 可观测性导出的钉版策略（03 §8）；`gen_ai.*` 与 `fysun.*` 长期关系为开放问题（12 §7） |
| Microsoft Entra Agent ID | learn.microsoft.com/entra/agent-id（2026） | 委派身份与继承权限的工业对照 |

## 探讨记录

| 记录 | 内容 |
|---|---|
| `D:\agent-kernel\docs\architecture\rebuild-0906` | 2026-09-06 CognitiveOS 重定义探讨全文（OPC 场景、IoT 场景、可移植性、Capability 单一核心对象、协作不特殊化、Agent Shell 不入定义、四个重叠概念的拆分） |
| 同日续谈（本仓库创立） | DSH / Cordis 调研结论、方案 A/B/C 取舍、FysunOS 命名与四项创立决策 |
| [`20260907-frontier-review-and-recommendations.md`](20260907-frontier-review-and-recommendations.md) | 2026-09-07 前沿对照审查报告；v0.6.0 修订与 ADR-0002 ~ ADR-0006 的依据 |
| [`20260908-v0.7.0-refactor-blueprint.md`](20260908-v0.7.0-refactor-blueprint.md) | 2026-09-08 v0.7.0 全量重构蓝图；WP1–WP6 的唯一规格来源；所有者两轮决策（§I）、逐篇改动规格（§B）、ADR-0001 ~ 0006 修订与 ADR-0007 ~ 0012 新增计划（§C）、外部调研（CognitiveOS / DeepSeek Harness / Cordis）结论；v0.7.0 完成后转为历史记录 |

## 行业参考（信息性，非规范）

- Agent runtime / durable execution 领域的公开实践（AIOS 后续工作、Pydantic AI durable execution、
  Blaxel 等对 "agent runtime environment" 的表述）用于确认：持久会话、checkpoint / resume、
  幂等工具调用、capability-controlled 副作用是业界共识方向；FysunOS 不以它们的具体实现为依据。

| 资料 | 用途 |
|---|---|
| Anthropic，*Scaling Managed Agents: Decoupling the brain from the hands*；Claude Managed Agents 文档（2026-04-08 发布） | session / harness / sandbox 三分对双账本设计的印证 |
| Amazon Bedrock AgentCore 开发者指南（Runtime / Memory / Gateway / Identity / Policy / Observability / Evaluations） | 企业级模块划分对照；Policy 采用 Cedar 的工业先例 |
| OpenClaw 架构文档与安全指南；Cisco / Snyk 对 ClawHub skill 的审计报道 | 本地优先需求与供应链威胁（[13](../design/13-threat-model.md) T2） |
| Zylos Research，*Durable Execution for AI Agent Runtimes*（2026-04-24 / 04-27）；LangGraph durable execution 文档 | 恢复模型对照（03 §6） |
| LangChain，*State of Agent Engineering*（2026） | 用户需求数据 |
| Forrester / Anaconda 2026 面板、Gartner 预测（经二手汇编，仅作趋势） | 用户需求数据；不作为任何单条决策的唯一依据 |

所有外部基准数字只用于说明方向，不构成对 FysunOS 任何性能或安全声明（见 [12 §5](../design/12-mvp-plan.md)）。
