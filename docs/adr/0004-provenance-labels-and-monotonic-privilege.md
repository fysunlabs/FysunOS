# ADR-0004 · 来源标签与权限单调收窄（F9）

- 状态：accepted
- 日期：2026-09-07
- 决策者：owner
- 背景来源：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) §2.2、§3.7、§5.1、§5.2、§7.1、§8.2
- supersedes：无（新增）。ADR-0001 §D7"能力不得自行扩大 Grant"保留有效；本 ADR 给出其可判定表述（F9）并补充数据流维度。

## 背景

v0.5.0 的安全模型只有一个维度：权限与授权。Grant 只能衰减、策略在准入点评估、Runtime 不能推进状态。
审查报告 §3.7 指出缺第二个维度——**数据流来源**：context_view 有 digest 与 losses[]，Effect 有幂等键与
Grant 检查，但没有任何字段表达"这个参数值来自哪里"。一段被注入的网页内容可以经模型改写后作为"收件人"
参数进入一次外部发送 Effect，而策略引擎只看到"principal 有权发送"。

2025–2026 的间接 prompt injection 防御研究已经收敛：

- CaMeL（arXiv:2503.18813）：值级 capability 与控制流 / 数据流分离；
- Progent（arXiv:2504.11703）：符号化工具 / 参数策略，收窄自动生效、扩张需外部批准（monotonic confinement）；
- Narisetty 等（arXiv:2606.26479）：把 CaMeL / FIDES / Progent / RTBAS / FORGE 统一为 Biba 完整性模型 +
  引用监视器 + 最小权限，并指出"没有数据流污点追踪"是 Codex CLI 类 harness 的共同缺口。

Progent 报告在 AgentDojo 上把间接注入成功率从 39.9% 降到 1.0%，在自适应攻击下保持 2.6%–4.2%。这些数字**仅说明
方向，不构成 FysunOS 的任何性能或安全声明**。

报告 §5.1 建议把"收窄自动、扩张需批准"写成一条可形式化的底线 F9，其价值在于可以写成负例向量并在 M1 验收。
AOS 16 节同样把"非扩张委派、无授权不执行、撤销安全"列为可用 TLA+ 表达的候选性质。

## 决策

### D1 · 来源 / 完整性标签

- 每个进入上下文的片段、每个工具结果、每个 Artifact 携带
  `label{source_principal, integrity, origin_ref}`，
  `integrity ∈ {trusted_user, trusted_system, verified_data, untrusted_external}`。
- 标签由 **Resource 子系统在装配时赋予**；Runtime 不能改写。Runtime 传回的任何标签字段被视为声明，
  `fysund` 以自己的记录为准。
- 取值规则：用户直接输入 → `trusted_user`；`fysund` 生成的系统内容（Policy 摘要、Grant 视图、Approval
  呈现）→ `trusted_system`；经 Verification `verdict = pass` 的 Artifact 或经准入的 Memory → `verified_data`；
  其余一切外部内容（网页、文件、邮件、工具结果、外部 A2A Agent 的 Artifact、模型输出）→ `untrusted_external`。
- 标签只降不升：合并、摘要、改写多个片段得到的新片段，其 `integrity` 取输入中最低者；升级到
  `verified_data` 的唯一路径是 Verification + Memory 准入。
- 事件 `context/assembled{labels_digest}` 记录每次上下文装配的标签集合摘要，替代或补充 `request_context`
  的审计事件。
- 负例：Runtime 在 `submit_candidate` 中把某参数标为 `trusted_user` 而 `fysund` 的装配记录中该值来自
  `untrusted_external` 片段 → `fysund` 以 `untrusted_external` 评估，且记录标签冲突；一次上下文装配后无
  `context/assembled` 事件 → 失败。

### D2 · Effect 候选参数的来源标注

- Effect 候选提交时，每个参数值携带来源标签：
  - Level-2 Runtime（声明 `context_injection` 控制点）须在 `os.submit_candidate` 中标注参数来源；
  - Level-1 Runtime 经 MCP 守卫接入，由 OS 在 `tools/call` pre 阶段按参数值与已装配片段匹配**尽力**标注；
    无法匹配的参数值记为 `untrusted_external`（fail-closed）。
- MCP 守卫在 `tools/call` post 阶段为工具结果打标签（缺省 `untrusted_external`），再交回 harness。
- Level-1 的标注为尽力而为，其局限（模型改写后的值可能无法匹配原片段）必须在 conformance 声明与 09 威胁模型
  中如实标注。
- 负例：一个无法匹配任何片段的参数值被标为 `trusted_user` 或 `trusted_system` → 失败。

### D3 · Policy 可引用标签

- 标签进入策略评估输入的 context 组（[ADR-0003](0003-policy-object-and-language.md) §D4）。
- 示例规则：外部发送类 Effect 的收件人参数含 `untrusted_external` 来源 → `policy_decision.outcome = approve`
  （创建 Approval，Runtime 收到 `needs_human(approval_id)`）。
- Memory 准入规则可要求 `integrity ≥ verified_data` 或经 `human` verifier（T7 记忆投毒）。
- 负例：`untrusted_external` 来源的收件人参数被直接放行（`outcome = allow`）→ 测试失败。此负例进入 M1 验收。

### D4 · F9：Execution 内有效权限单调收窄

- 新增底线 **F9**：在一次 Execution 的生命周期内，有效权限集
  **有效权限集 = Grant ∩ Policy 允许集 ∩ 工具面**
  只能因以下两类事件变化：
  - (a) Runtime 或用户提出的**收窄**自动生效（`grant/attenuated`、`tools/list` 过滤、Policy 收紧）；
  - (b) 任何**扩张**必须经 Approval（[ADR-0005](0005-approval-and-evidence-gated-evolution.md) §D2），
    且扩张后的集合仍 ⊆ 父 Grant。
- "父 Grant"指 Execution 启动时绑定的 Grant 或 Handoff 委派链上游的 Grant；扩张不得越过它。跨越父 Grant
  的需求只能通过新 Activity 由 Principal 重新授权。
- F9 是 ADR-0001 §D7"能力不得自行扩大 Grant"的可判定表述，同时把 Agent Skills `allowed-tools` 作为初始上界
  （[ADR-0002](0002-protocol-first-runtime-boundary.md) §D4）纳入同一不等式。
- Grant 委派的三条不等式写成负例：`scope ⊆ parent.scope`、`expiry ≤ parent.expiry`、`budget ≤ parent.remaining`；
  任一不成立的 `grant/issued` → 失败。

### D5 · 负例向量进入 M1 验收

以下负例登记为 M1 验收项，编号按 [00-terminology](../design/00-terminology.md) §11 的
`NV-T<n>-<序号>` 格式在实现阶段分配：

| 向量 | 断言 |
|---|---|
| T3 扩权候选 | 构造一条试图扩权的 candidate（请求 `allowed-tools` 之外的工具，或高于当前 Grant 的预算），账本中不出现无 `approval/granted` 前置的 `grant/issued` |
| T1 注入收件人 | `untrusted_external` 来源的收件人参数直接放行 → 失败；期望 `outcome = approve` 与 `approval/requested` |
| T3 委派越界 | Handoff 派生的 Grant 违反三条不等式之一 → `grant/issued` 不出现 |
| T1 标签伪造 | Runtime 声明的标签高于 `fysund` 记录 → 以低者评估并留痕 |
| T9 标签改写 | Runtime 试图改写已装配片段的标签 → ABI 中无此调用，请求被拒 |

### D6 · 范围界定

- 本 ADR 实现的是 FIDES / Progent 一级的"标签 + 引用监视器"：在协议边界（MCP 守卫、`submit_candidate`）
  即可实现，不要求控制 Runtime 的执行语义。
- 本 ADR **不是**完整的 CaMeL 解释器。CaMeL 需要 OS 控制决策核心的执行语义（代码生成与解释），与 P8
  （不自建 loop）和 ADR-0001 §D2"不自建 loop、prompt 装配、工具流水线"冲突。
- 完整信息流控制列为后续研究，可能的形态是一个 Level-3 控制点（报告附录 B），由 Runtime 自愿声明。
- 负例：设计文档把 FysunOS 描述为"防止所有 prompt injection" → 失败；正确表述为"T1 向量在标签 + Policy + F9
  下的可判定防护，局限见 09 威胁模型"。

## 否决的替代方案

- **只依赖 Grant 与 Policy（无数据流维度）。**策略引擎只看到"principal 有权发送"，看不到"收件人来自不可信
  内容"；这是 Narisetty 等指出的共同缺口。
- **完整 CaMeL。**要求控制 Runtime 执行语义，与 P8、ADR-0001 §D2 冲突；且会使 FysunOS 重新绑定到单一
  Runtime 实现。
- **把标签交给 Runtime 维护。**Runtime 是非权威方且运行模型输出；标签一旦可由 Runtime 改写就失去引用监视器
  的意义（T9）。
- **把 F9 只写成叙述性原则而不写成负例。**v0.5.0 的 F7 已经是叙述性原则，审查报告指出其价值在于可在 M1 机械
  验收；不写成负例则与 F6 冲突。

## 后果

- context_view 与 Effect 候选参数的 schema 增加 `label` 字段；Artifact 元数据增加 `label`。
- MCP 守卫需在 `tools/call` pre 阶段做参数值匹配、post 阶段为结果打标签；这是 M2a 的一部分。
- Level-1 标注为尽力而为，conformance 声明必须如实标注"Level-1 Runtime 下参数来源标注存在漏标可能，漏标
  按 `untrusted_external` 处理"；这会使 Level-1 下的 Approval 触发率高于 Level-2，是可接受的代价。
- Memory 准入获得一条机械规则（`integrity` 门槛），T7 记忆投毒有了第一道确定性防线。
- 09 威胁模型的 T1、T3、T7、T9 条目直接引用本 ADR 的负例向量。
- 每个上下文装配增加 `context/assembled` 事件；账本体量上升，但仍在 ADR-0001 §D5 的"有边界"范围内
  （不记录片段正文，只记录 `labels_digest`）。

## 待后续 ADR 的问题

- 完整信息流控制是否作为 Level-3 控制点引入（报告附录 B）。
- `verified_data` 之外是否需要更细的完整性等级（例如区分"经确定性 verifier"与"经 `human` verifier"）。
- Trust 对象（按 Runtime / 工具 / 外部 Agent 的证据化信任）何时引入，以及它与标签的关系。

其余见 [08 MVP 计划 §7](../design/08-mvp-plan.md#7-开放问题待后续-adr)。
