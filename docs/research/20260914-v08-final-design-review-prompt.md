# FysunOS v0.8 最终设计 · 再评审任务提示词

- 性质：**评审输入提示词**。本文件是给一个全新独立评审会话的完整任务说明，不是评审报告。
- 使用方式：在新窗口中把本文件全文（或其路径 + "按此执行"）作为任务输入。
- 撰写日期：2026-09-14。评审执行日以实际运行日为准，外部信息以**当日**可核验的一手材料为准。

---

## 1. 角色与任务

你是一名独立资深架构评审人，对 FysunOS 重新设计后的**最终设计**做一轮全新的全面复检。这不是对既有评审的复读：
你应当带着"这份设计如果直接交给实现团队，会在哪里失败"的问题去审。

- **评审对象**：`docs/design-v0.8/fysunos-design-spec.md`（综合定稿稿，2026-09-14）与同文件夹 `README.md`。
  它是 `docs/redefinition`（v0.8.0-draft）、`docs/proposal-v0.8`、两份 2026-09-12 独立评审与 v0.7.0 现行设计的综合定稿。
- **允许的结论**（三选一，必须明确给出）：
  1. **按现状采纳**——给出采纳前置条件（若有）；
  2. **部分重构**——列出必改项（每项：证据路径 + 影响 + 改法），并明确"不改则失败"的条目；
  3. **全量重构**——给出理由与替代方案骨架。
  允许对局部（某一章、某一不变量、某一对象）建议推倒重做，同时对其余部分建议冻结。
- **产出**：一份自成体系的评审报告，写入 `docs/research/<执行日>-v08-final-design-review.md`
  （`<执行日>` 用实际日期，格式 `YYYYMMDD`）。报告不得修改任何既有文件。

## 2. 背景速览（先读这段，再读材料）

FysunOS 的文档演进链：v0.5.0 首稿 → v0.6.0（前沿审查后修订：协议优先、Approval/Policy 一级对象、F9）
→ v0.7.0 全量重构（14 篇 + ADR-0001~0012，形态是"`fysund` 内联守卫"：守卫是必经之路、Provider 代理、上下文渲染、三条投递管道）
→ `docs/redefinition`（v0.8.0-draft，2026-09-11：定位重审，`fysund` 退出数据面热路径，执行点 EP，资产化，AaaS，ADR-0013 草案）
→ 两份 2026-09-12 独立评审（一致结论：**部分重构**——四层实体、分层 Lock、04/EP 契约必须继承 ADR-0008 D5、
`published` 以 qualified EP 覆盖为门、MCP 钉 2026-07-28、自进化带 Immutable Zone 的离线插件、不变量 I1–I12）
→ `docs/proposal-v0.8`（2026-09-12：吸收评审的拍板提案，D1–D8、八种资产、三层 Lock、F3 分级、R0–R3）
→ `docs/design-v0.8`（2026-09-14：上述全部的综合定稿稿，即本次评审对象）。

仓库**只有设计文档，没有可运行代码**。所有条目的证据状态为 `specified`（规范存在 ≠ 实现存在 ≠ 测试已执行 ≠ Profile 已符合）。

## 3. 必读材料清单（全部通读，不得只读摘要）

**A. 评审对象**
- `docs/design-v0.8/fysunos-design-spec.md`
- `docs/design-v0.8/README.md`

**B. v0.7.0 现行基线（被继承的合同）**
- `docs/design/00-terminology.md` ~ `13-threat-model.md`（14 篇全读；重点：00 §4 事件族、02 对象模型、05 Runtime ABI、06 MCP 守卫、13 威胁模型）
- `docs/adr/0001` ~ `0012` 与 `docs/adr/README.md`（重点：0002 §D2、0007 §D3/§D5、0008 §D5、0009 §D1、0011、0012）

**C. 演进链**
- `docs/redefinition/`（README、adr-0013-draft、00–11 全套）
- `docs/proposal-v0.8/`（README、CHANGELOG-vs-redefinition、00–05 全套）
- 归档可只读 `docs/archive/v0.6.0/ARCHIVE.md` 与 `docs/archive/v0.5.0/ARCHIVE.md` 的摘要（用于追溯术语变迁，不必全读）

**D. 历史评审与规格**
- `docs/research/20260907-frontier-review-and-recommendations.md`
- `docs/research/20260908-v0.7.0-refactor-blueprint.md`
- `docs/research/20260912-redefinition-review-and-refactor-report.md`
- `docs/research/20260912-redefinition-audit-and-improvement-plan.md`
- `docs/research/sources.md`
- 根 `README.md`、`docs/CHANGELOG.md`

## 4. 外部研究要求（以执行日为准）

用 WebSearch / WebFetch 核验当日仍可访问的一手材料，并与设计说明附录 A 的"2026 锚点"逐一对照：
规范是否有比设计所钉版本更新的修订？工业实践是否已改变前提？至少覆盖：

1. **MCP**：是否有晚于 2026-07-28 的现行规范；Sampling / Roots / Logging 弃用状态；Authorization（OAuth 2.1、RFC 9728、RFC 8707）；Tasks 扩展状态。
2. **A2A**：最新版本与 Agent Card schema；AAIF 治理状态；多租户 / 签名 Card 实践。
3. **Agent Skills**：规范现状与采纳面。
4. **Agent 安全**：OWASP Agentic Top 10 现行版本；CaMeL / Progent 之后的注入防御进展；业界 MCP 网关与 guardrail 实践（AgentCore Policy/Gateway、agentgateway 等）。
5. **自进化**：ACE / GEPA / DGM 及最新综述；评估可靠性（RuVerBench 一类）与 judge 校准实践；SWE-bench 类公开榜污染问题现状。
6. **运行时**：Temporal / durable execution、Agent runtime 环境的最新共识；长任务 HITL 与进程挂起的工业做法。
7. **性能**：prompt / KV prefix caching 的厂商文档与学术进展（前缀稳定性对长程 Agent 的代价）。
8. **可观测性**：OTel GenAI 语义约定稳定性状态（仍是 Development 吗）。
9. **身份与供应链**：SPIFFE / Entra Agent ID 等委派身份实践；SLSA / Sigstore 在 Agent 制品上的应用。
10. **参考架构**：AOS（arXiv:2608.03214）、AIOS 的后续与引用情况。

证据规则（强制）：
- 一手来源优先（规范原文、官方文档、同行评审 / 预印本）；二手只作趋势佐证。
- 每个外部事实钉版本 / 日期 / URL；未核验到的一律标 **[待验证]**，不臆造 revision 与日期。
- 外部基准数字只说明方向，不构成对 FysunOS 的任何性能或安全声明；未在本仓库测量的延迟 / 命中率 / 成本不得写成事实。
- 对照结论逐项给出决策栏：**采纳 / 谨慎采纳 / 延后 / 拒绝**（含义：是否写入 FysunOS 内核规范）。

## 5. 评审方法

1. 全文精读评审对象，先独立形成设计地图（职责、平面、对象、热路径、关键机制）。
2. **交叉核对**：评审对象与 v0.7.0 的术语、事件名、错误码、状态集、不变量、负例编号是否字面一致或有明确改写说明；
   是否出现"未登记进术语表的新词"（例如 `waiting{reason: audit_backlog}`、`change_class`、`park/thaw`、`runtime.cache_alignment_violated` 等）。
3. **与已采纳 ADR 冲突核对**：逐条对照 ADR-0001~0012，特别是 0002 §D2（守卫必经）、0007 §D5（投递路径）、
   0008 §D5（只读快路径）、0009 §D1（Cedar 封闭表）、0011（Approval）、0012（守卫自实现）——设计说明 §13.1 R0 的 ADR 修订清单是否完备、是否遗漏连带修订。
4. **红队 / 威胁评估**：对每个新机制（F3 分级、EP 微批、发布门 I6、park/thaw、优化器插件、sampling 防火墙、BYO）构造至少一个攻击路径，检验负例是否覆盖。
5. **外部对照**（§4）。
6. **可执行性演练**：以"实现者视角"走一遍 R1/R2 闭环剧本，找出会在实现期被迫自行裁定的歧义。
7. 全文使用证据分级标签：**[事实]**（可在指定路径/URL 核对）、**[推断]**、**[建议]**、**[待验证]**。

## 6. 必答评审问题（逐条给出结论与证据）

**定位与定义**
- Q-01 定义句是否稳定、可判定、且不与其余章节冲突？"基座八件事"是否有遗漏或越界？
- Q-02 相对 ADR-0001 §D1 原点（管理绑定并可靠运行绑定），本设计是否实现了"管理"与"运行"的再平衡，还是又一次偏航？

**对象模型**
- Q-03 8 种 kind + Loop Profile 是否自洽？`agent` 的"制品与入口双栖"是否会引发分发 / 资格化歧义？
- Q-04 Capability 非 kind 后，契约的版本、发现、证据归属是否仍有完整生命周期？`tool` 身份并入三处来源（mcp_server / skill / harness）后，跨来源同名工具的冲突消解是否可判定？
- Q-05 I12（Runtime 非资产）在 API、事件、Lock 引用各处是否真正贯彻（无残留 `kind=runtime` 语义）？

**分层 Lock 与证据**
- Q-06 三层 digest 的切分是否可判定？边界案例：harness 的 `prompt_slots` 变更、`model.fallback` 顺序变更、`tools_allow` 收窄、`memory_provider` 换后端——各属哪层？设计是否写死？
- Q-07 `inherited` 证据机制会不会被滥用（用 profile 微调夹带 substrate 变更）？`compose.diff` 的 `change_class` 判定规则是否完备？Q30 的校准姿态是否可执行？
- Q-08 I13 与"canary 累积 Lock 级证据"之间是否存在时间窗矛盾（canary 尚无非 inherited 证据时如何进入 stable）？

**F3 分级与 EP**
- Q-09 F3 三档与 ADR-0008 §D5 是否字面一致？`external_read` "可组批、不阻塞同步 fsync" 的崩溃窗口下，"账本能回答 untrusted_external 来自哪次外部读取"的承诺是否仍然成立？
- Q-10 `audit/observed` 异步微批 + `waiting{reason: audit_backlog}`：这是新状态/新 reason 吗？术语表、恢复顺序、负例是否同步？
- Q-11 EP 独立进程后，F1"判定与账本写入同一事务边界"的原始理由（ADR-0012 背景）如何成立？I1/I9 的可验证机制（核对、重放、金样）是否足够？
- Q-12 `policy_stale_ttl` 的取值与续约协议是否定义？EP 崩溃恢复协议（Q25）是否已从"开放问题"升级为可实现的规范？

**AaaS 与服务平面**
- Q-13 发布门 I6 的"覆盖"如何机械判定：按 EP 能力位声明，还是按 Lock 工具面与 EP `capabilities` 交集？绕过路径（先 publish 纯 local_read、再 Adjustment 加 mutation）是否被堵死？
- Q-14 计量可信（Q26）、Trust 顺序（Q8）、`shared` 池化（Q24）、BYO（Q29）的缺口是否可接受？对外 AaaS 的最小合格服务清单是否完整？
- Q-15 A2A 1.0 Agent Card 字段、MCP Authorization / Sampling 防御的写法与当日最新规范是否一致？

**演进与自进化**
- Q-16 优化器插件边界（§9.3）是否可执行？Immutable Zone 清单是否完备（prompt slot 结构、`prefix_order[]`、`subagents[]` 白名单是否也该入列）？
- Q-17 影子评估的数据来源（已 verified 脱敏输入）是否足够构造"同一契约/基座、仅换 profile"的对照？Q32 的权限边界是否明确？

**性能**
- Q-18 并行只读与 fencing epoch、幂等键、补偿的交互是否已定义（并发只读的 `audit/observed` 顺序、mutation 屏障的语义）？
- Q-19 `session_park` 与重新武装、Approval 水位、`thaw` 锁校验的次序是否自洽？park 快照入 Artifact 是否引入新的信任面？

**威胁、conformance 与路线图**
- Q-20 T12–T15 与 N-01–N-12 是否与 v0.7.0 的 `NV-T*` 向量重复或冲突？缺口（例如 EP 微批丢失、优化器供应链、Card 签名密钥泄漏）是否应新增向量？
- Q-21 conformance profile 表与 v0.7.0 按 Runtime 类型四列的关系是否交代（避免两套表并存）？
- Q-22 R0–R3 的依赖与退出条件是否可判定？R0 ADR 工程清单是否遗漏（ADR-0003/0004/0011 的连带修订、adr/README 规则、归档动作）？
- Q-23 R2 同时交付 EP 契约 + F3 分级 + sampling 防御 + gateway + Agent Card 是否过重？是否需要拆分或并行轨道？

**文档工程与元层面**
- Q-24 术语表缺口清单（上文 Q-10 提到的词，以及 `Loop Profile`、`change_class`、`ExecutionSnapshot` 等）；双源风险（`docs/design-v0.8` vs `docs/design` vs `docs/redefinition`）的收敛路径是否明确？
- Q-25 开放问题合并清单与决策项是否有编号冲突、丢失或与正文矛盾的"已关闭"声明？

## 7. 报告结构（产出文件的章节骨架）

1. 元信息：性质、评审对象与版本、对照基线、外部截止日（执行日）、方法、与此前四份评审报告的关系（继承/重写声明）。
2. 证据分级与陈述约定。
3. 执行摘要：总判断；**八条最关键结论**（带 [事实]/[推断] 标注）；"是否重构"三分表（定位 / 对象模型 / Lock / F3 / EP / AaaS / 演进 / 文档树各一行：定稿 / 部分重构 / 重构）。
4. 设计地图（当前形态）与职责迁移表（v0.7.0 → 本设计）。
5. 应保留的核心（重构不得拆掉的清单）。
6. 问题清单：**P0**（不改则实现 / 安全 / 演进失败或与已采纳 ADR 冲突）、**P1**、**P2**；每条含：路径 + 章节、证据、影响、建议。
7. 跨文档冲突表（评审对象 vs `docs/design` / `docs/adr` / `docs/redefinition` / `docs/proposal-v0.8`），逐条给处置建议。
8. 前沿学术与工业实践对照表（§4 的十个领域，逐项：当日可核对状态、与本设计关系、决策栏）。
9. 专项分析：性能、治理、自进化、AaaS、复杂度与对象面宽度（至少五节；可用量化框架但不得虚构测量数字）。
10. 若建议重构：目标架构（实体分层、Lock、不变量增删 I-编号、可删 / 合并 / 延后清单）；若部分重构：逐篇 / 逐节修订方案。
11. 建议的文档树与 ADR 工程（修订哪些 ADR、是否需要新 ADR、归档与双源收敛）。
12. 分阶段实施与可判定验收；**新增负例表**（ID、情境、期望，格式对齐 N-01–N-12 与 `NV-T<n>-<序号>`）。
13. 度量（先基线后阈值的清单增删）。
14. 开放问题（继承 + 新增，给出建议姿态）与给 owner 的决策顺序。
15. 来源清单：≥ 15 条，可访问 URL + 一句话用途；含本仓库内部关键路径。
16. 结论：三选一明确回答（§1"允许的结论"），并给出"若采纳，第一步做什么"。

## 8. 硬约束

- **只读评审**：不修改 `docs/design`、`docs/adr`、`docs/design-v0.8`、`docs/redefinition`、`docs/proposal-v0.8` 及任何既有文件的任何字节；
  产出只写入第 1 节指定的新报告文件。不执行 `git commit`。
- 报告必须**自成一体**：读者不需要本次或任何历史对话的上下文即可理解与执行。
- 不把未测量数字写成事实；不臆造规范版本、日期、上游 revision。
- 引用仓库内证据一律给 `路径 + 章节`；引用外部证据一律给 URL + 钉版。
- 评审结论独立作出：不因"前两轮评审都建议部分重构"而预设结论；若你认为应当全量重构或按现状采纳，给出同等强度的论证。
