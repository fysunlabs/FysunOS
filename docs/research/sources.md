# 来源登记

本设计依据的精确来源。迁入代码或语义时在此追加来源 revision 与路径。

## 代码基线

| 项目 | Revision | 位置（调研时） | 用途 |
|---|---|---|---|
| CognitiveOS / Personal（`agent-kernel`） | `6f562bf1bdb86582955de36eb93e51126573f89c`（分支 `personal/P14-T04-member-join`） | `D:\agent-kernel` | 迁移来源：Intent/Effect、replay、recovery、context、Verifier、Provider 代理、DSH broker 经验 |
| DeepSeek Harness | `cd5ef8148158c3a752a658978873241fdf8e2bbc` | `D:\agent-kernel\artifacts\personal-2.0.0-dev-prep\deepseek-harness`（只读副本） | 默认 Runtime；架构、安全、术语、seam 目录、Agent Teams、workflow、schedule 文档 |
| Cordis（DSH vendored） | 4.0.1（`vendor/cordis/package.json`） | 同上 `vendor/cordis/` | Composition ABI 语义来源 |
| CognitiveOS V2 重构研究 | 2026-08-21，基线 CognitiveOS `742a9346…`、DSH `528c682e…`、Cordis `8cc9e33f…` | `D:\agent-kernel\docs\research\20260821-cognitiveos-v2-restructuring-architecture.md` | 8 primitive 最小内核、双账本、Harness ABI、Composition ABI、DSH/Cordis 策略表 |

v0.6.0 起（[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md)）：DSH 的角色为第一个 Level-2 参考适配器；
Cordis 语义仅作 `fysun-dsh` 适配器内部参考；上表"用途"列为登记时原文，不改写。

## 论文与一手资料

| 资料 | 用途 |
|---|---|
| Mei et al., *AIOS: LLM Agent Operating System*, arXiv:2403.16971（COLM 2025） | Agent OS 内核词汇（scheduler / context / memory / storage / tool / access manager）；FysunOS 在其之上增加 Intent/Effect、fencing、独立验证 |
| Shi, Zhang, Cui, *A Programming Paradigm for Spatiotemporal Composability*, arXiv:2608.25512（2026-08-13） | Cordis 的理论基础：可撤销效果（时间可组合性）与反应式依赖（空间可组合性）；FysunOS Composition ABI 的语义来源 |
| Anthropic, *How we built our multi-agent research system*（engineering blog） | 多 Agent 收益取决于任务结构而非 Agent 数量；支撑"协作是普通能力行为、不默认多 Agent" |
| DeepSeek Harness 文档：`docs/architecture.md`、`docs/cordis-primer.md`、`docs/capability-seams.md`、`docs/glossary.md`、`docs/subsystems/agent-team.md`、`docs/subsystems/workflow.md`、`docs/user/guide/schedule.md`、`SAFETY.md` | DSH 设计哲学与边界的原文依据 |
| Skalse et al., 2022（reward hacking 可形式化）及 2026 loop engineering / reward-hacking benchmark 综述 | 独立验证（F4）的依据 |
| Sharma, Ye, Ge, Zhang 等，*The Agent Operating System (AOS): A Reference Operating Architecture for Distributed Agentic Systems*（AOS-0001），arXiv:2608.03214（2026-08） | 对外坐标（01 §1.4）、对象映射（02 §4）、不变量、威胁模型与 conformance 声明（09）的参照；Confidence 控制动作六元组 → `assurance` |
| Debenedetti 等，*Defeating Prompt Injections by Design*（CaMeL），arXiv:2503.18813 | 值级 capability 与控制/数据流分离；来源标签的范围界定（不做完整 CaMeL） |
| Shi 等，*Progent: Securing AI Agents with Privilege Control*，arXiv:2504.11703 | 符号化策略、SMT 收窄/扩张判定、单调限制 → F9 |
| Narisetty 等，*Adaptive Evaluation of Out-of-Band Defenses Against Prompt Injection in LLM Agents*，arXiv:2606.26479（2026-06） | 引用监视器 + 最小权限 + 完整性标签的统一框架；静态基准局限 |
| *Dynamic Agent Skills: A Lifecycle Survey and Taxonomy of Evolving Skill Libraries*，arXiv:2607.10113（2026-07） | 八阶段生命周期 → 演进模型缺口（07 §1.6） |
| *SkillOps: Managing LLM Agent Skill Libraries as Self-Maintaining Software Ecosystems*，arXiv:2605.13716 | Skill Contract (P,O,A,V,F) → 质量目标表示法（07 §1.5） |
| Lin 等，*Agentic Harness Engineering: Observability-Driven Automatic Evolution of Coding-Agent Harnesses*（AHE），arXiv:2604.25850 | 决策可观测性 → `Adjustment.predicted_effect` |
| *Self-Improvements in Modern Agentic Systems: A Survey*，arXiv:2607.13104 | 自改进的参数/脚手架双回路 |
| Weng，*Harness Engineering for Self-Improvement*，lilianweng.github.io（2026-07-04） | harness 定义；harness-updating vs harness-benefit |
| *Can LLM-as-a-Judge Reliably Verify Rubrics in Agentic Scenarios?*（RuVerBench），arXiv:2606.29920 | judge 噪声 → Verification 校准字段 |
| *Measurement Without Validity: The Compounding Reliability Problem in Agentic AI Evaluation*，arXiv:2608.00794 | 可靠性阈值 α≥0.67 / 0.70 / 0.80 → judge 项替代人工的条件 |
| *AdaRubric*，arXiv:2603.21362 | 任务自适应 rubric |
| Li 等，*MemOS: A Memory OS for AI System*，arXiv:2507.03724 | 记忆作为可调度资源；不自建记忆引擎的依据（Memory 对象） |
| *ROSClaw: An OpenClaw ROS 2 Framework for Agentic Robot Control and Interaction*，arXiv:2603.26997 | 执行层契约、前置校验器、安全包络 Policy |
| URML（robot intent language + five-pass validator），github.com/URML-MARS/URML | "通过校验 ≠ 世界安全" → 包络校准为部署者责任 |
| Pan 等，*Measuring Agents in Production* / *Characterizing Agents in Production*（CAP），arXiv:2512.04123（ICML 2026） | 生产 Agent 的 HITL、评估与可靠性数据 → Approval 一级对象 |

## 协议与规范

| 资料 | 版本 / 日期 | 用途 |
|---|---|---|
| Model Context Protocol 规范 | 2025-11-25；SEP-1686（Tasks）、SEP-1036（URL elicitation）、SEP-1577（sampling with tools）、SEP-1649 / SEP-1960（server cards，草案）、SEP-2663（Tasks 移入扩展，草案） | Runtime ABI 的 MCP 守卫层（04 §2.1） |
| Agent2Agent 协议 | v1.0（2026-03） | Handoff 外部形态与状态机对齐（02 §2.8、04 §2.7） |
| Agent Skills 规范 | agentskills.io，2025-12-18 公开；Discussion #415 SemVer 提案（2026-06-05） | `Capability.implementation` 的实现包格式（04 §2.8） |
| agentgateway | AAIF 项目（Rust，Apache-2.0）；1.3 版 `mcpGuardrails` | MCP 守卫层复用候选；Pass / Reject / Mutated 语义；Cedar/CEL 策略 |
| Cedar 策略语言 | AWS 开源（Apache-2.0） | Policy 对象的策略语言（ADR-0003） |
| OpenTelemetry GenAI 语义约定 | `semantic-conventions-genai` 仓库（2026-06 起），截至 2026-08-13 全部 Development | 可观测性导出的钉版策略（03 §8） |
| Microsoft Entra Agent ID | learn.microsoft.com/entra/agent-id（2026） | 委派身份与继承权限的工业对照 |

## 探讨记录

| 记录 | 内容 |
|---|---|
| `D:\agent-kernel\docs\architecture\rebuild-0906` | 2026-09-06 CognitiveOS 重定义探讨全文（OPC 场景、IoT 场景、可移植性、Capability 单一核心对象、协作不特殊化、Agent Shell 不入定义、四个重叠概念的拆分） |
| 同日续谈（本仓库创立） | DSH / Cordis 调研结论、方案 A/B/C 取舍、FysunOS 命名与四项创立决策 |
| [`20260907-frontier-review-and-recommendations.md`](20260907-frontier-review-and-recommendations.md) | 2026-09-07 前沿对照审查报告；v0.6.0 修订与 ADR-0002 ~ ADR-0006 的依据 |

## 行业参考（信息性，非规范）

- Agent runtime / durable execution 领域的公开实践（AIOS 后续工作、Pydantic AI durable execution、
  Blaxel 等对 "agent runtime environment" 的表述）用于确认：持久会话、checkpoint/resume、
  幂等工具调用、capability-controlled 副作用是业界共识方向；FysunOS 不以它们的具体实现为依据。

| 资料 | 用途 |
|---|---|
| Anthropic，*Scaling Managed Agents: Decoupling the brain from the hands*；Claude Managed Agents 文档（2026-04-08 发布） | session / harness / sandbox 三分对双账本设计的印证 |
| Amazon Bedrock AgentCore 开发者指南（Runtime / Memory / Gateway / Identity / Policy / Observability / Evaluations） | 企业级模块划分对照；Policy 采用 Cedar 的工业先例 |
| OpenClaw 架构文档与安全指南；Cisco / Snyk 对 ClawHub skill 的审计报道 | 本地优先需求与供应链威胁（09 T2） |
| Zylos Research，*Durable Execution for AI Agent Runtimes*（2026-04-24 / 04-27）；LangGraph durable execution 文档 | 恢复模型对照（03 §6） |
| LangChain，*State of Agent Engineering*（2026） | 用户需求数据 |
| Forrester / Anaconda 2026 面板、Gartner 预测（经二手汇编，仅作趋势） | 用户需求数据；不作为任何单条决策的唯一依据 |

所有外部基准数字只用于说明方向，不构成对 FysunOS 任何性能或安全声明（见 [08 §5](../design/08-mvp-plan.md)）。
