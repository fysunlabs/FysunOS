# 来源登记

本设计依据的精确来源。迁入代码或语义时在此追加来源 revision 与路径。

## 代码基线

| 项目 | Revision | 位置（调研时） | 用途 |
|---|---|---|---|
| CognitiveOS / Personal（`agent-kernel`） | `6f562bf1bdb86582955de36eb93e51126573f89c`（分支 `personal/P14-T04-member-join`） | `D:\agent-kernel` | 迁移来源：Intent/Effect、replay、recovery、context、Verifier、Provider 代理、DSH broker 经验 |
| DeepSeek Harness | `cd5ef8148158c3a752a658978873241fdf8e2bbc` | `D:\agent-kernel\artifacts\personal-2.0.0-dev-prep\deepseek-harness`（只读副本） | 默认 Runtime；架构、安全、术语、seam 目录、Agent Teams、workflow、schedule 文档 |
| Cordis（DSH vendored） | 4.0.1（`vendor/cordis/package.json`） | 同上 `vendor/cordis/` | Composition ABI 语义来源 |
| CognitiveOS V2 重构研究 | 2026-08-21，基线 CognitiveOS `742a9346…`、DSH `528c682e…`、Cordis `8cc9e33f…` | `D:\agent-kernel\docs\research\20260821-cognitiveos-v2-restructuring-architecture.md` | 8 primitive 最小内核、双账本、Harness ABI、Composition ABI、DSH/Cordis 策略表 |

## 论文与一手资料

| 资料 | 用途 |
|---|---|
| Mei et al., *AIOS: LLM Agent Operating System*, arXiv:2403.16971（COLM 2025） | Agent OS 内核词汇（scheduler / context / memory / storage / tool / access manager）；FysunOS 在其之上增加 Intent/Effect、fencing、独立验证 |
| Shi, Zhang, Cui, *A Programming Paradigm for Spatiotemporal Composability*, arXiv:2608.25512（2026-08-13） | Cordis 的理论基础：可撤销效果（时间可组合性）与反应式依赖（空间可组合性）；FysunOS Composition ABI 的语义来源 |
| Anthropic, *How we built our multi-agent research system*（engineering blog） | 多 Agent 收益取决于任务结构而非 Agent 数量；支撑"协作是普通能力行为、不默认多 Agent" |
| DeepSeek Harness 文档：`docs/architecture.md`、`docs/cordis-primer.md`、`docs/capability-seams.md`、`docs/glossary.md`、`docs/subsystems/agent-team.md`、`docs/subsystems/workflow.md`、`docs/user/guide/schedule.md`、`SAFETY.md` | DSH 设计哲学与边界的原文依据 |
| Skalse et al., 2022（reward hacking 可形式化）及 2026 loop engineering / reward-hacking benchmark 综述 | 独立验证（F4）的依据 |

## 探讨记录

| 记录 | 内容 |
|---|---|
| `D:\agent-kernel\docs\architecture\rebuild-0906` | 2026-09-06 CognitiveOS 重定义探讨全文（OPC 场景、IoT 场景、可移植性、Capability 单一核心对象、协作不特殊化、Agent Shell 不入定义、四个重叠概念的拆分） |
| 同日续谈（本仓库创立） | DSH / Cordis 调研结论、方案 A/B/C 取舍、FysunOS 命名与四项创立决策 |

## 行业参考（信息性，非规范）

- Agent runtime / durable execution 领域的公开实践（AIOS 后续工作、Pydantic AI durable execution、
  Blaxel 等对 "agent runtime environment" 的表述）用于确认：持久会话、checkpoint/resume、
  幂等工具调用、capability-controlled 副作用是业界共识方向；FysunOS 不以它们的具体实现为依据。
