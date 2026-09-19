# FysunOS v0.8 综合定稿稿 · 最终设计评审报告

- **性质**：独立架构评审（终审）。本报告只读对照被审对象与全部演进链、基线材料与外部规范，**不修改任何既有文件**；产出仅本文件。
- **评审对象**：[docs/design-v0.8/fysunos-design-spec.md](../design-v0.8/fysunos-design-spec.md)（综合定稿稿，2026-09-14）与 [docs/design-v0.8/README.md](../design-v0.8/README.md)。
- **对照基线**：v0.7.0 现行设计（`docs/design/00–13`，2026-09-08）与已采纳 ADR-0001–0012；演进链 `docs/redefinition`（v0.8.0-draft，2026-09-11）、`docs/proposal-v0.8`（2026-09-12）、两份 2026-09-12 独立评审（`20260912-redefinition-review-and-refactor-report.md`、`20260912-redefinition-audit-and-improvement-plan.md`）；历史评审 `20260907-frontier-review-and-recommendations.md` 与 `20260908-v0.7.0-refactor-blueprint.md`（后者为 v0.7.0 的唯一实施基准，本报告以其 §I 决策记录为已采纳事实的最终形态；frontier 报告的结论以其被 ADR-0002–0006 与蓝图吸收后的定稿形态核对）。
- **外部截止日**：2026-09-14（当日以 WebSearch / WebFetch 核验一手材料；未能当日核验的一律标 [待验证]，不臆造版本与日期）。
- **方法**：(1) 全文精读评审对象并独立形成设计地图；(2) 交叉核对评审对象与 v0.7.0 术语表 / 事件族 / 错误码 / 状态集 / 负例的字面一致性，登记未收录新词；(3) 逐条对照 ADR-0001–0012（重点 0002 §D2、0007 §D2/§D3/§D5/§D8、0008 §D5、0009 §D1/§D5、0011、0012）检验 §13.1 R0 修订清单的完备性；(4) 对每个新机制（F3 分级、EP 微批、I6 发布门、park/thaw、优化器插件、sampling 防火墙、BYO）构造攻击路径并检验负例覆盖；(5) 外部对照（十域，见 §9）；(6) 以实现者视角走 R1/R2 闭环剧本，登记实现期将被迫自行裁定的歧义。
- **证据状态声明**：仓库只有设计文档，没有可运行代码。评审对象全部条目为 `specified`（规范存在 ≠ 实现存在 ≠ 测试已执行 ≠ Profile 已符合，四类状态用语继承 v0.7.0 12 §5）。本报告不引用任何未测量数字作为事实。
- **与此前四份评审报告的关系**：**继承结论、独立重审、收窄改动面**。本报告不是对 20260912 两份评审的复读：那两轮的结论对象是 `docs/redefinition`（11 kind、扁平 Lock、R3 先于 EP 等结构性缺口），本轮对象是已把这些缺口修复后的综合定稿稿。结论是那两轮的"部分重构"清单中的**结构性条目已被本设计吸收**（逐条核对见 §4.2 与 §11），本轮新增的问题集中在**规范闭合度**（实现者会被迫自行裁定的空洞）与**跨文档一致性**（与已采纳 ADR 的残留冲突），规模显著小于前两轮。

---

## 1. 证据分级与陈述约定

| 标记 | 含义 |
|---|---|
| **[事实]** | 可在指定仓库路径 + 章节，或可访问外部 URL（钉版本 / 日期）直接核对 |
| **[推断]** | 由多处原文合取推出，原文未直接写该结论；存在其他合理解释的可能性已评估 |
| **[建议]** | 本报告的改进意见，尚未被仓库采纳 |
| **[待验证]** | 外部规范 / 上游状态在 2026-09-14 未能用一手页面钉死，或依赖未实现的测量 |

硬约束：未在本仓库测量的延迟、命中率、成本一律不写成事实；外部基准数字只说明方向；未核验的规范 revision / 日期不臆造。引用仓库内证据给 `路径 + 章节`；引用外部证据给 `URL + 钉版/检索日`。

---

## 2. 执行摘要

### 2.1 总判断

**[建议] 部分重构（收口级修订）——架构冻结、五项必改。**

v0.8 综合定稿稿在结构层面是四轮演进中第一份可以直接交给实现团队的稿子：定位句稳定、四层对象模型自洽、分层 Lock 与证据继承落实了两份评审的全部结构性 P0、F3 三档显式继承了已采纳的 ADR-0008 §D5、I6 发布门堵住了"先服务平面后执行点"的顺序漏洞、外部协议锚点（MCP 2026-07-28、A2A 1.0、Agent Skills、OWASP 2026、OTel GenAI 状态）经当日核验全部成立。**不应再对骨架做任何重构**。

但"直接交给实现团队"仍会在五个点上失败——全部是规范闭合问题而非架构问题：R0 的 ADR 修订清单不完备（按 `adr/README` 规则"accepted 的 ADR 为准"，未列出的 ADR 冲突会在实现期爆发）；`waiting{reason: audit_backlog}` 是未登记的新状态值且触发机制不可实现；EP 三档 F3 的持久化语义互相矛盾使 I3 / H-2 在崩溃窗口内不可验收；I6 的"EP 覆盖面"没有机械定义且存在两条绕过路径；三层 digest 的切分函数与 Capability 引用解析未写死使 `compose.resolve` 的可重放性（I10）无法实现。这五项各需半天到两天的**写作级**修订，不需要改动任何已冻结的对象、不变量或路线图骨架。**不改则失败**的条目即这五项（§7 P0-1–P0-5）；其余为 P1 / P2 改进项与文档工程。

### 2.2 八条最关键结论

1. **[事实]** 定位句、两项职责、"基座八件事"在 §1 内部及与 §6–§8、§10 的对照中无冲突、可判定；相对 ADR-0001 §D1 原点，本设计实现了"管理"与"运行"的再平衡而非偏航（Q-01/Q-02 通过，论证见 §6）。
2. **[事实]** 两份 2026-09-12 评审的全部结构性 P0 已被吸收并可逐条核对：四层实体（spec §3.1）、分层 Lock + 差分证据（§5.4）、04/EP 契约继承 ADR-0008 D5（§6.6 明确拒绝把 `external_read` 放进无持久化快路径）、`published` 以 qualified EP 覆盖为门（§8.2 I6）、MCP 钉 2026-07-28（§8.6）、自进化 Immutable Zone（§9.3）、不变量 I1–I14 与负例 N-01–N-12（§2.3/§12.4）。
3. **[事实]** **R0 的 ADR 工程清单不完备**（spec §0.3、§13.1）：设计实际触碰了 ADR-0005 §D1 / ADR-0011 的封闭 `Approval.subject` 集（§7.4 增三值）、ADR-0007 §D2（conformance 表形态）、ADR-0009 §D1/§D5 的 action 命名（`Runtime::"qualify"` vs `Action::"runtime/qualify"`），但 R0 清单未列出 0005 / 0007 §D2 / 0009 §D5 / 0011 的修订——按 `adr/README` "设计文档冲突时以 accepted 的 ADR 为准"，这些冲突在实现期裁决权在旧 ADR 手里。
4. **[事实]** `waiting{reason: audit_backlog}`（spec §6.6）未登记进 §3.3 的 Execution 状态集（仍为 `trigger | approval | input` 三值，继承 v0.7.0 00 §4 / ADR-0011 §D9）；"补送失败 → 谁写这个状态"不可实现（EP 不可写权威状态 I1；`fysund` 不可达时更不可达）；`audit/observed` 微批的 EP 本地序号与账本去重规则缺失。
5. **[推断]** F3 三档的 EP 本地持久化语义不一致：`local_read` 走内存环形缓冲（EP 崩溃即丢 `audit/observed`，违反 I3"不允许丢失"）、`external_read` 组批写且"不阻塞同步 fsync"（分发可先于持久化完成，崩溃窗口内的外部读取无法被账本溯源，掏空 H-2 承诺）、仅 mutation 档完整。设计未定义组批提交边界、窗口上限与崩溃对账语义。
6. **[事实]** I6 的"qualified EP 覆盖面"无机械定义（EP `capabilities[]` 是能力位，不声明覆盖哪些工具）；存在两条未堵死的绕过路径：先以纯 `local_read` Lock `published` 再经 Adjustment 扩工具面；EP 失格（`ep/disqualified`）不级联 ServiceVersion `suspended`。
7. **[事实]** 外部锚点当日核验通过（详见 §9）：MCP 2026-07-28 为现行最新修订（SEP-2577 Final，Sampling/Roots/Logging 弃用 + 约 12 个月窗口；OAuth 2.1 / RFC 9728 MUST / RFC 8707 / DCR 弃用→CIMD），A2A 1.0.0 Agent Card 形状与 §8.6 逐字段吻合，OWASP Agentic Top 10 2026（ASI01–ASI10，2025-12-09 发布），OTel GenAI 截至 2026-09 仍全部 Development（独立仓库），ACE / GEPA / DGM / 自进化综述的 arXiv 编号全部真实，SWE-bench Verified 污染有一手与学术陈述。**设计所钉版本无需变更**；唯一日期差异：A2A 加入 AAIF 检索到 2026-08-17/20 两个口径，设计写 2026-08-27，建议复核标注。
8. **[推断]** 实现期演练（R1/R2 剧本）暴露两类歧义：三层 digest 的切分函数与 `capability:` 引用（Capability 非 kind 后的版本 / digest / 解析归属）未定义，`compose.resolve` 可重放性在 R1 验收时无法落笔；R2 单阶段交付面过重（EP 契约 + F3 三档 + sampling 防御 + gateway + A2A Card + `os.observe` + EP 恢复协议），建议拆分。

### 2.3 是否重构：三分表

| 范围 | 判定 | 说明 |
|---|---|---|
| 定位与定义（§1–§2） | **定稿** | 冻结。含 P1–P11、F1–F9、I1–I14 全集。 |
| 对象模型（§3–§4：四层 / 8 kind / Loop Profile / Capability 契约） | **定稿** | 冻结。附两个收口项：`kind=agent` 的 `declares.effects` 填充口径（P2）、Capability 引用解析（并入 P0-5）。 |
| 分层 Lock 与证据继承（§5） | **定稿骨架 + 必改收口** | 切分函数写死（P0-5）、`inherited` 标记载体（P1）、canary 证据类型（P2）。 |
| F3 分级（§6.6） | **部分重构** | external_read 组批与 local_read 审计的持久化语义必须闭合（P0-3），不改则 I3 / N-04 / H-2 不可机械验收。 |
| EP（§6.4–§6.5） | **定稿骨架 + 必改收口** | audit_backlog 状态登记与微批协议（P0-2）、hooks fail-closed 语义澄清（P1）、EP 覆盖面声明（与 P0-4 联动）。 |
| AaaS（§8） | **定稿骨架 + 必改收口** | I6 覆盖判定 + 两条绕过路径（P0-4）。 |
| 演进与自进化（§9） | **定稿** | Q30 姿态可执行；补 `inherited` 载体（P1）与 `subagents[]` 显式入不可变表（P2）。 |
| 文档树与元层面（§0/§14/附录） | **部分重构** | §14 Q8 编号复用自相矛盾（P1）、术语缺口清单（P1）、R0 清单扩充（P0-1）。 |

---

## 3. 设计地图（当前形态）

### 3.1 结构速写（独立复核后的形态）

```text
消费者 ──A2A/MCP/HTTP──▶ fysun-gateway（入口 EP：鉴权·配额·计量·invocation）
                             │ 仅创建/唤醒（控制面 RPC）
                             ▼
                        fysund（控制面：Catalog·Runtime 控制·PDP·Approval·Service 目录·唯一权威账本）
                             │ start/resume + 结构化绑定（Lock + Grant 摘要 + Memory 引用，无渲染文本）
                             │ policy_pull / report 核对 / resolve_secret
                             ▼
        Runtime（Harness 进程，资格化纪元 runtime/qualified）◀──os.observe 查询（审批结果枚举）
                             │ tools/call / 原生工具
                             ▼
        工具侧 EP（mcp_guard / harness_adapter / hooks）：本地 Policy 快照 → F3 三档 → 微批回报
                             ▼
                     MCP server / Provider / 物理执行器
```

- **四层实体**：L1 Capability 契约（非 kind）→ L2 八种资产 kind（`prompt·skill·mcp_server·model·memory_provider·knowledge·harness·agent`）→ L3 Agent Definition + 分层 Lock + Loop Profile（非 kind）→ L4 运行与服务（Runtime 实例 / EP / Activity·Execution / Service / Tenant / Plan）。`agent` 是唯一双栖（L2 制品 + L3 入口）。
- **热路径禁令**：`fysund` 不在 `tools/call` 或模型请求转发链（I2 / §6.5）；例外回中心仅四类（`outcome=approve`、Grant 扩张、secret 解析、Memory 准入需人部分）。
- **治理**：F1 唯一权威写入者；六值 `effect_category` 三源合一（ADR-0008）；Cedar 子集 + 版本化快照下发（ADR-0003/0009 + 新实体待修 ADR-0009）；Approval 封闭结果集与审批面只由 `fysund` 提供（ADR-0011）；来源标签 + `flags`（ADR-0004）。
- **关键新机制**：分层 Lock（contract / substrate / profile 三 digest + `inherited` 差分证据）、F3 三档热路径（local_read 快路径 / external_read 组批 / mutation 同步 WAL）、EP 资格化与 Policy 下发（`policy_stale_ttl` fail-closed）、`session_park`/`thaw`（可选控制点，ExecutionSnapshot 入 Artifact）、优化器插件（只许 profile 层，Immutable Zone，`evolution.immutable_boundary`）、MCP sampling 防火墙（缺省拒绝 + 兼容窗）、I6 发布门、计量可信（Q26：对外 token 配额要求可信计量 EP）。
- **账本**：event-sourced 唯一真源在 `fysund`；EP 本地 WAL 为待核对副本（I1）；准入类事件集扩至 11 类（v0.7.0 七类 + `asset/published`、`service/published`、`ep/qualified`、`plan/attached`）。

### 3.2 职责迁移表（v0.7.0 → 本设计）

| v0.7.0 | 本设计 | 迁移性质 | 核对 |
|---|---|---|---|
| `fysun-mcp-guard` = `fysund` 内组件、必经之路（ADR-0002 §D2、ADR-0012 §D1） | 独立可选 EP（`mcp_guard`）；"不嵌入"保留、"不前置"按 EP 框架重开 | 架构迁移 | [事实] spec §6.4、§10.2；与 ADR-0012 §D1 冲突由 R0 修订处理（P0-1） |
| `start_execution` 携带渲染 context_view；`fysund` 渲染 | 只携带 Lock + 结构化绑定；Harness 自行装配 | 废止重写 | [事实] spec §6.2 负例 2 |
| `deliver_input` 三条投递管道 + 守卫系统工具 `fysun.await_approval`/`fysun.pending_inputs` | 废止为 OS 规范；`os.observe` 查询；管理 API 保留 `execution.deliver_input` 记账 | 废止 | [事实] spec §3.4、§6.2；对应 ADR-0007 §D5 废止（R0 已列） |
| F5 以 Provider 代理为前提 | `provider_proxy` 为可选 EP；F5 = 只向 qualified EP 解析 | 收窄 | [事实] spec §6.4/§12.3 |
| 扁平 Lock、新 Lock 证据从零 | 三层 digest + `inherited` 差分继承 | 新增 | [事实] spec §5.4 |
| conformance 按 Runtime 类型四列（ADR-0007 §D2） | 按 EP 组合分列六 Profile（§12.5） | 改写 | [事实] spec §12.5；ADR-0007 §D2 修订未列入 R0（P0-1） |
| M0–M5 里程碑 | R0–R3 + Spike S1–S9 | 重写 | [事实] spec §13；M2a/M2b 表述残留在 ADR-0007 §D8（P0-1） |
| 多主体开放（v0.7.0 Q5） | Tenant 核心，R1 字段即存在 | 关闭 | [事实] spec §7.1 |
| 无服务平面 | Service 一级 + Tenant/Plan 附着 + gateway EP | 新增 | [事实] spec §8 |
| 无执行点概念 | EP 六类 + 契约 + 资格化 + 回报核对 | 新增 | [事实] spec §6.4–6.5 |

### 3.3 综合裁定表的核对（spec §0.2 对四轮输入的取舍）

逐条核对 spec §0.2 的 15 项裁定与来源原文：**14 项与来源字面一致**（四层实体、Loop 非 kind、tool 非 kind、分层 Lock、D5 继承、I6、MCP 钉版、A2A 1.0、ADR-0009 先修、park 采纳、Immutable Zone、拒绝数值折损、拒绝内生 Evolution Engine、性能机制采纳）。**1 项有口径偏移**：对"前缀槽硬资格化"的拒绝，评审原建议是"Slot 顺序定为 Harness 资格化输入 + 可选能力位"（refactor-report P1-3），本设计改为纯能力位 + `prefix_order[]` 建议序——这是**更弱**的方案，但与"不渲染文本、不收窄任意 Runtime"的承诺自洽，且以 S9 spike + 度量兜底，[建议] 接受该偏移，但应在 R0 把"若 S9 显示不声明位的 Harness 前缀命中率系统性偏低，是否升级为资格化输入"登记为条件性开放问题。

---

## 4. 应保留的核心（重构不得拆掉的清单）

以下条目经本轮独立核对后判定为**正确且不可拆**；任何后续修订不得为局部便利削弱它们：

1. **定义句与两项职责 + 八件事**（spec §1.1–1.2）：短定义句 + 长形式备查的双轨解决了"六种实现物并列"的歧义，且引用规则（字面一致）可机械检查。
2. **F1–F9 全集与其执行者标注**（spec §2.2）：唯一权威写入者、候选≠事实、按 `effect_category` 分级的先记录再执行、独立验证、secret 隔离、契约不为实现让路、不自扩权、未知外部改动保护、单调收窄。
3. **I1–I14 不变量集**（spec §2.3）：特别是 I1（EP WAL 是 pending replica）、I2（fysund 热路径禁令）、I5（未分类 fail-closed）、I6（发布门）、I9（同 policy_version 同输入同 outcome）、I12（Runtime 非资产）、I13（证据继承门）、I14（thaw 锁校验）。
4. **六值 `effect_category` 三源合一与 ADR-0008 的继承**（spec §6.6）：`max_severity`、Policy 唯一降级来源、annotations 只作输入。
5. **分层 Lock 与证据差分继承**（spec §5.4）：contract/substrate 不继承、profile 可 `inherited`、资格化证据不跨 id、拒绝数值折损内核化。
6. **Approval 封闭结果集、配对事件、审批面只由 `fysund` 提供、审批事实不进模型可见面**（spec §7.4；ADR-0011）。
7. **EP 框架**：六类 EP、`ep/qualified` 资格化纪元、Policy 快照下发 + `policy_stale_ttl` fail-closed、回报核对（`ep_id` + `policy_version` + 幂等键）、同一工具调用只产一个 Effect。
8. **Activity/Execution 状态机、双水位恢复顺序、重新武装、审批容量与限流**（spec §6.7；ADR-0011 §D8）。
9. **供应链**：digest 重算、签名 Policy、namespace 允许列表、`tools_snapshot` 漂移检测、quarantine/revoke 级联（spec §7.7、§12.2 T12）。
10. **协议锚点**（spec §8.6）：MCP 2026-07-28（含 Sampling 缺省拒绝）、A2A 1.0 Card 形状、Agent Skills 唯一 skill 格式、OCI artifact、Cedar 子集、OTel `gen_ai.*` 钉 commit + `fysun.*` 权威。
11. **演进最小闭环**（spec §9.3）：优化器可插拔、Immutable Zone、轨迹脱敏导出、影子评估、canary 可审计、SWE-bench Verified 不得作对外回归集。
12. **路线图纪律**：R0–R3 可判定退出、Spike S1–S9（S1/S2/S4/S5 为 R2 阻断项）、度量先基线后阈值、四类状态用语、全部条目 `specified` 的非声明。

---

## 5. 必答评审问题 Q-01–Q-25 逐题结论

每题给出：结论（通过 / 有保留通过 / 不通过）+ 证据路径 + 一句话论证。详细展开指向 §7 问题清单的对应编号。

| # | 题 | 结论 | 证据与论证 |
|---|---|---|---|
| Q-01 | 定义句稳定、可判定、无章节冲突；八件事有无遗漏/越界 | **通过** | [事实] spec §1.1 短定义句 + 引用规则（字面一致）；§1.2 两项职责与八件事清单与 §6（执行点契约、资格化）、§7（Grant∩Policy∩工具面、Approval、账本）、§8（服务目录）、§10.2（fysund 职责表）逐项对应；§1.5 不做清单与 P8/P11 一致。结构化绑定（§6.2）与 `prefix_order[]`（§6.3）是解析/资格化职责的交付物，不构成越界。 |
| Q-02 | 相对 ADR-0001 §D1 原点是否再平衡 | **通过** | [事实] ADR-0001 §D1"管理绑定并可靠运行绑定"在 v0.7.0 被落实为内联数据面（redefinition 10 §1 诊断，本报告核对 design/05–08 属实）；spec §1.2 把"管理"抬为能力管理职责（L1–L3 平面）且 §1.4 AOS 供给声明改为"供给控制面，不供给数据面"。再平衡成立；定义句的正式替换由 ADR-0013 承接（R0 已列）。 |
| Q-03 | 8 kind + Loop Profile 自洽；agent 双栖歧义 | **通过（附 P2 收口）** | [事实] spec §3.1 四层表、§4.1 kind 表、§4.2 非 kind 表相互封闭；agent 双栖唯一且无递归打包问题（Definition 只含引用，digest 覆盖 manifest）。[推断] 一处口径需一句话写死：`kind=agent` 的 `declares.effects[]` 按 §4.3"必填（可空）"登记时必须为空、由 `compose.resolve` 汇总填充（redefinition 03 §2 原文如此，spec 未复述）→ P2-11。 |
| Q-04 | Capability 非 kind 后的契约生命周期；跨来源同名工具消解 | **有保留通过** | [事实] Capability 保留 `capability.register/describe/search/health` API（spec §11.2）、`contract_version` 与谱系继承（§5.1）、证据归属四层（§9.1）；同名工具 `max_severity` + `tools_allow` 收窄可判定（§5.3），工具名 `<server>/<tool>` 限定继承 v0.7.0 06 §2.1。[推断] 缺口：Capability 非 kind 后**没有 manifest/digest/存储载体定义**——`capability:acme/trend-research@2` 引用走哪张登记表、`contract_digest` 规范序列化包含其哪些字段，未写死 → 并入 P0-5。 |
| Q-05 | I12（Runtime 非资产）是否贯彻 | **通过** | [事实] §3.1 L2 排除 runtime；§4.2 专表；§11.2 "没有 `asset.register(kind=runtime|loop|tool)`"；§3.4 `runtime/qualified` 带 `harness_asset_id`、无 kind=runtime 资产 id；§5.2 "Definition 不引用 Runtime id"；R1 验收含 §4.6 负例 2。无残留 kind=runtime 语义（组件名 `fysund` 与 Runtime 对象不同名，无冲突）。 |
| Q-06 | 三层 digest 切分可判定性（prompt_slots / fallback / tools_allow / memory_provider） | **不通过 → P0-5** | [事实] 明确的：harness（含 `prompt_slots[]`）与 memory/knowledge digest → substrate（§5.4）；Loop params / prompt 变量缺省 / `tools_allow` 允许的覆盖 → profile（§5.4 表把"允许的覆盖、变量缺省"入 profile，但 `mcp_servers[].tools_allow` 同时被 §5.2 列为"工具面收窄的声明来源"且 §5.4 把"工具面收窄"归 contract_digest——**同一声明字段被切分进两层，规范序列化时必须写死归属**）；`model.fallback`：§9.3 把"模型绑定顺序（fallback）"列为优化器 Mutable Zone（暗示 profile 层），§5.4 substrate 覆盖"model 身份"——fallback 中备用 model 的身份归哪层未写死。[推断] `compose.diff` 只要求返回 `change_class` 三值，未给切分算法；R1 闭环验收（I10、可重放）实现者必须自行裁定。 |
| Q-07 | inherited 会不会被滥用夹带 substrate 变更；change_class 完备性；Q30 可执行性 | **有保留通过** | [事实] 夹带防御成立：substrate 任一 digest 变更即失去继承资格（§5.4），profile 微调改不了 `declares.effects[]` 类别（§5.2 overrides 限制 + §5.6 负例 3）；`compose.diff` 返回 `change_class` 为系统判定输入（§5.4）。[推断] 两个缺口：(a) `inherited` 标记的载体（事件 / Lock 字段 / evidence 复制引用的格式）未定义，N-02 无法机械验收 → P1；(b) `change_class` 判定算法（同名工具 severity 合并后的 diff 归 contract 还是 substrate）未给 → 并入 P0-5。Q30 姿态（`deterministic` 自动继承、`model_judge`/`human` 缺省重跑）可执行 ✓（§14 Q30）。 |
| Q-08 | I13 与 canary 累积证据的时间窗矛盾 | **通过（附 P2）** | [事实] 无硬矛盾：canary 阶段对真实流量按 F4 产生本 Lock 的 Verification（`contract_digest` 相同 → 非 inherited 证据），stable 门检查的正是这批证据（§9.2 + §5.4）；I13 只禁止把**不同** `contract_digest` 的旧证据计为 pass。[推断] 残留口径：canary 样本为零 / 极少时"通过率 ≥ 阈值"未定义最小样本数；canary Verification 是否需满足 `independence` 最低要求未写 → P2-12。 |
| Q-09 | F3 三档与 ADR-0008 §D5 字面一致性；external_read 崩溃窗口下 H-2 承诺 | **不通过 → P0-3** | [事实] 三档结构对 D5 字面一致：仅 `local_read` 快路径、`external_read` 完整协议、明确拒绝无持久化快路径（§6.6 引用 H-2）。[推断] 但"可组批写，不阻塞同步 fsync"未定义组批提交边界与崩溃语义：分发若先于组批落盘，EP 崩溃后该 external_read 在账本无 ActionIntent，N-04 在该窗口必然违反、H-2 溯源承诺落空；恢复协议（§6.5"先回报未回报事件"）无法回报从未落盘的事件。必须拍板持久化语义（见 P0-3 三个方案）。 |
| Q-10 | audit_backlog 是否新状态/新 reason；术语/恢复/负例是否同步 | **不通过 → P0-2** | [事实] §6.6 引入 `waiting{reason: audit_backlog}`；§3.3 Execution 状态集仍为 `waiting{reason: trigger | approval | input}`（继承 v0.7.0 00 §4/§5、ADR-0011 §D9）；事件族 §3.4 无对应；恢复顺序（§6.7 固定八步）不含该状态的进入/退出；负例集（§6.9、N-01–N-12）无对应条目；触发归属不可实现（EP 不能写权威状态 I1）。 |
| Q-11 | EP 独立进程后 F1 原始理由（同事务边界）如何成立；I1/I9 机制是否足够 | **有保留通过** | [事实] ADR-0012 §D1 的原理由（判定与账本写入同事务边界）被显式重写为"权威面 = fysund；数据面强制点 = qualified EP（快照与回报受核对）"（§12.1）；I1（EP 不得写终态）+ I9（同 policy_version 同 outcome）+ 回报核对（§6.5）+ "无回报 Effect 不得 verified" + outcome-unknown 对账构成替代机制。[推断] 不足处即 P0-3：EP 本地持久化的**分级语义**未闭合，"pending replica" 在 local_read / external_read 两档的物理含义不同；补齐后 F1 在独立进程下成立。 |
| Q-12 | policy_stale_ttl 取值与续约协议；EP 崩溃恢复（Q25）是否升级为规范 | **通过** | [事实] TTL 为 Policy 回答的部署设定（§7.3 清单含 `policy_stale_ttl`），续约协议完整：`ep.policy_pull` + `policy/revised` 失效通知 + TTL 内未刷新自行 fail-closed（只允许 local_read）（§6.5）+ 负例 §6.9.5；Q25 已升级为 I1–I3 + EP 契约恢复协议并列入 R2 交付（§14 Q25、§13.1 R2）。[建议] 给缺省推荐值（如 5–15 分钟量级）进 §15 决策表，避免每个部署自定 → P2-13。 |
| Q-13 | I6 覆盖的机械判定；两条绕过路径 | **不通过 → P0-4** | [事实] "覆盖面"无定义：EP `capabilities[]` 是 filter_tools/persist_intent/receipt 等能力位（§6.5），不描述覆盖哪些工具；`ep/qualified` 事件也无覆盖集字段。绕过路径 A：纯 local_read Lock 先 `published`，再 Adjustment 扩工具面（§5.5 允许经 Approval 加工具，发布门只在 publish/rollout 检查）未被禁止；绕过路径 B：`ep/disqualified` 不级联 `ServiceVersion.suspended`（§8.2 只写资产 revoked/quarantined 级联）。 |
| Q-14 | 计量可信（Q26）、Trust 顺序（Q8/Q27）、shared 池化（Q24）、BYO（Q29）缺口可接受性；最小合格服务清单 | **通过** | [事实] Q26 处置（对外 token 配额要求 `provider_proxy` 或等价可信 EP；自报只作对照，N-12）可判定；Q24（`dedicated` 缺省、`shared` 需 `session_isolation` + 注入测试负例后开）保守合理；Q29（BYO 槽位 + 类别不可降级在 fysund 校验、EP 再强制）defense in depth 成立；Q8/Q27（Trust R3 后单独立 ADR、跨基座 `public` 前置）顺序正确。最小合格服务清单（§8.2 发布门 + §7.4 审批路由 + §8.5 计量 + Card 签名 + I6）完整。 |
| Q-15 | A2A 1.0 Card 字段、MCP Authorization/Sampling 写法与当日规范一致性 | **通过** | [事实] A2A 1.0.0 规范当日核验（https://a2a-protocol.org/v1.0.0/specification/ ，检索 2026-09-14）：`supportedInterfaces[]`、`protocolVersion` 在 interface 层、`capabilities.extendedAgentCard`、JWS `signatures[]`、interface 可选 `tenant`——与 §8.6/负例 10 完全吻合。MCP 2026-07-28 核验（见 §9 域 1）：Sampling/Roots/Logging 弃用（SEP-2577 Final）、OAuth 2.1、RFC 9728 MUST、RFC 8707 resource indicator、DCR 弃用→CIMD——§8.6/§8.4 逐条吻合。[推断] 唯一差异：A2A 加入 AAIF 的日期，设计写 2026-08-27，当日检索到 2026-08-17（aaif.io）与 2026-08-20（二手）两个口径 → P2-2 复核。 |
| Q-16 | 优化器插件边界可执行性；Immutable Zone 清单完备性 | **通过（附 P2）** | [事实] `optimizer.describe()/propose()` 边界清晰（§9.3）：非 PDP、不能 `grant.issue`、不能标 Verification pass、不能读 Secret 明文、不进 tools/call / 模型请求 / `compose.resolve` 热路径、禁止 Execution 内反射改 prompt、越权 → `evolution.immutable_boundary`（N-07）。[推断] 清单建议补一行：`subagents[]` 白名单扩张（内部 Handoff 目标新增）应显式列入不可变区或"新外部动作"类——运行时已有负例兜底（§5.6.4），但优化器层的禁止清单未点名 → P2-14。`prompt_slots` 结构属 harness manifest（substrate）✓；`prefix_order[]` 属 profile ✓。 |
| Q-17 | 影子评估数据源是否足够；Q32 权限边界 | **通过** | [事实] 影子集 = 已 `verified` 的脱敏输入（§9.2）+ 回归集来源继承 v0.7.0 10 §1.4 / ADR-0005 §D5（Capability 历史 verified Execution 输入 + 租户可选贡献脱敏）；"同一契约/基座、仅换 profile"的对照可由分层 Lock 直接构造（换 profile_digest、复用 contract/substrate）。Q32（优化器缺省只许 profile 层）已答（§14）。 |
| Q-18 | 并行只读与 fencing/幂等/补偿的交互 | **有保留通过** | [事实] 并行规则明确：多 `local_read`（及已 authorized 且 Policy allow 的 `external_read`）并发；mutation / `credential_use` / `physical_action` 与未决 Approval 串行；同批含 mutation 先完成 read 再 persist mutation（§6.6）；fencing 防陈旧 epoch 覆盖（继承）；补偿必须新 Grant + 新 ActionIntent（继承 ADR-0008 §D7）。[推断] 缺口：EP 微批回报的**去重与排序键**未定义——`audit/observed` 无幂等键，EP 崩溃重放微批或乱序补送时账本如何去重（建议 EP 回报带本地单调序号，沿 `origin_seq` 机制）→ 并入 P0-2。 |
| Q-19 | session_park 与重新武装、Approval 水位、thaw 锁校验次序；快照信任面 | **有保留通过** | [事实] 次序自洽：唤醒走"重新武装 → thaw / `resume_execution`"（§6.7 恢复顺序固定八步），未决 Approval 不入快照、水位由 `fysund` 账本承载、`thaw` 到错误 `lock_digest` 拒绝（I14 / N-09）。[推断] 新信任面未定义：ExecutionSnapshot（对话摘要等）由 Harness 生成、入 Artifact 后以何种 `label.integrity` 登记、thaw 后作为初始投递注入是否走 F2 候选/来源标签检查——未写 → P1。 |
| Q-20 | T12–T15 与 NV-T* 重复性；缺口 | **通过（附 P2）** | [事实] T12–T15 与 T1–T11 无重复（供应链扩展到全部 kind、EP 冒充/失格、跨租户、服务面为新增面）；"v0.7.0 的 NV-T* 向量全部保留并按 EP 组合重登记"（§12.4）。[推断] 三个缺口：EP 微批丢失（P0-2 修复后需负例）；Card 签名密钥泄漏无向量（建议 T16 或并入 T15 负例 + 密钥轮换开放问题 → P2-4）；优化器投毒诱导错误归因（包络 + 证据门兜底，低危，trajectory.export 脱敏义务建议补一条 → P2-15）。 |
| Q-21 | conformance 表与 v0.7.0 按 Runtime 类型四列的关系 | **通过** | [事实] 附录 B 明确"13 conformance 按 EP 组合分列"为 R0 回填操作；§12.5 六 Profile 表 + 状态用语四类继承；§12.5 保留"一致性负例（继承 ADR-0007 §D2）"。[建议] R0 grep 清单加一条"v0.7.0 13 §6 四列表已替换"以防双表并存 → 并入 P0-1。 |
| Q-22 | R0–R3 依赖与退出条件可判定性；R0 清单遗漏 | **有保留通过** | [事实] 各阶段退出条件均落到负例编号 / grep / 闭环剧本（§13.1），依赖串行清晰，S1/S2/S4/S5 为 R2 阻断项。[推断] R0 清单遗漏四项 ADR 修订（0005 §D1 subject 集、0007 §D2 conformance 形态、0009 §D5 action 命名、0011 审批路由）+ adr/README 索引 + 归档动作细节 → P0-1。 |
| Q-23 | R2 是否过重；拆分建议 | **不通过（路线图级）→ P1** | [事实] R2 交付清单：EP 契约完整五方法、`mcp_guard` 或 adapter 覆盖 MCP 工具面、F3 三档、sampling 防御、`fysun-gateway` A2A Endpoint + 签名 Card（1.0 形状）、单 Tenant 调用、`os.observe`、EP 恢复协议（§13.1）。对比 R1（登记 + 单 Harness 最小 ABI + 恢复）与 R3（多租户 + 演进 + 第二 Harness），R2 是三个阶段中交付面最大的一个，且 sampling 防御 + OAuth audience 属于与闭环正交的协议深水区。[建议] 拆 R2a（EP 契约 + F3 + adapter-min 覆盖闭环工具面 + `os.observe`）/ R2b（gateway + A2A Card + 单 Tenant 调用），sampling 防御与 OAuth 可随 R2b 但不阻断 R2a 退出 → P1。 |
| Q-24 | 术语表缺口；双源收敛路径 | **有保留通过** | [事实] 双源收敛明确：redefinition 采纳后 superseded/归档、`docs/design` R0 回填（§0.3）；采纳前以本说明为准（spec 头部）。[推断] 术语缺口（§3/§4 使用但 v0.7.0 00 未登记、附录 B 只登记了部分）：`change_class`（附录 B 已列）、`audit_backlog`（未列，P0-2）、`ExecutionSnapshot`、`runtime.cache_alignment_violated`（事件还是错误码未定）、`prefix_order[]`、`cache_hint`、`session_park`/`thaw`（附录 B 已列 park）、`policy_stale_ttl`、`profile.put`、`lock_overrides_digest`、`ep_coverage`（若采纳 P0-4）→ P1 清单。 |
| Q-25 | 开放问题合并清单的编号冲突 / 丢失 / 与正文矛盾 | **不通过（文档级）→ P1** | [事实] §14 头部声明"编号只增不复用"，但 Q8 被复用给 Trust（"Q8（原 Q11/Q27）"）——v0.7.0 12 §7 明言"Q8 `Intent` 改名——删除……编号保留以备追溯，不复用"；且 §14 同时存在"Q7–Q10（v0.7.0 已关闭）"行与 Q8 行，**同一清单内 Q8 出现两次、语义矛盾**；Q11 整体缺号、Q27 被折叠进 Q8 行。其余核对：Q21 并入 Q25 ✓、Q5/Q14/Q33 关闭与正文一致 ✓、Q23"建议关闭"与 §4.2 最小 params 口径需统一（P2-8）。 |

---

## 6. 问题清单

优先级约定：**P0** 不改则实现期被迫自行裁定、与已采纳 ADR 冲突、或安全/一致性承诺（I1/I3/H-2/I6/I10）不可机械验收——"不改则失败"；**P1** 显著拖慢实现、验收或产生文档级自相矛盾；**P2** 优化与文档债。每条含：路径 + 章节、证据、影响、建议。

### 6.1 P0（不改则失败）

#### P0-1 · R0 的 ADR 工程清单不完备，且 Cedar action 命名存在跨文档冲突

- **证据 [事实]**
  - spec §0.3：ADR 修订范围列为"0002 §D2、0007 §D3/§D5、0009 §D1、0012 §D1 按 §13.1 R0 清单修订或增补"；§13.1 R0 行同。
  - 但设计正文实际改动了更多已采纳 ADR 的封闭集：
    - §7.4 增 `Approval.subject` 三值 `asset_publish / service_publish / plan_attach`——该集合在 ADR-0005 §D1 与 ADR-0011 中是**封闭枚举**（`effect_candidate / grant_expansion / adjustment / handoff_accept / memory_admit / verification_override`）；
    - §12.5 conformance 按 EP 组合分列——ADR-0007 §D2 明文"conformance 表按 Runtime 类型分列"；
    - §7.3 新增 action `Runtime::"qualify"` / `EP::"qualify"` 等——ADR-0009 §D1/§D5 的 action 实体均为 `Action::"<…>"` 形态，资格化 action 写作 `Action::"runtime/qualify"`；两种命名方案不能同时进 Cedar schema；
    - §7.4 增审批路由（发布者+租户双签 `quorum: 2`）——ADR-0011 未含路由概念；
    - R0–R3 替换 M0–M5——ADR-0007 §D8 仍写"M2a 基线 = DSH headless"。
  - `docs/adr/README.md` 规则："设计文档之间或设计文档与实现冲突时，以 accepted 的 ADR 为准。"
- **影响**：实现者按 ADR 为准会拒绝 `asset_publish` 等 subject（schema 校验失败）或按旧四列表写 conformance；Cedar schema 按哪套 action 命名生成将直接分叉；R0 的 grep 验收无法覆盖这些点。
- **建议**：扩充 §13.1 R0 清单为：(a) ADR-0002 §D2 supersede（已列）；(b) **ADR-0005 §D1 subject 集增补**（新 ADR 或修订历史）；(c) **ADR-0007 §D2（conformance 形态）+ §D8（M2a→R1 措辞）修订**；(d) **ADR-0009 §D1 + §D5 增补：统一 action 命名为 `Action::"<域>/<动作>"`（`Action::"runtime/qualify"`、`Action::"ep/qualify"`、`Action::"asset/publish"`…）并增实体/context 封闭表**；(e) **ADR-0011 增补审批路由**；(f) ADR-0012 增补（已列）；(g) adr/README 索引 + ADR-0013 定稿 + `docs/redefinition` 归档动作；(h) R0 grep 增一条："v0.7.0 13 §6 四列表已替换、`Approval.subject` 含七值"。

#### P0-2 · `waiting{reason: audit_backlog}` 未登记、触发归属不可实现、`audit/observed` 微批缺去重/排序键

- **证据 [事实]**
  - spec §6.6："补送失败 → Execution `waiting{reason: audit_backlog}`，不得静默丢"。
  - spec §3.3：Execution 状态集为 `waiting{reason: trigger | approval | input}`——三值，继承 v0.7.0 00 §4/§5 与 ADR-0011 §D9（D9 负例："因 pending Approval 等待却 waiting{reason: trigger} → 失败"，枚举封闭）。
  - spec §3.4 事件族与 §6.7 恢复顺序均无 `audit_backlog`；负例集（§6.9、§12.4 N-01–N-12）无对应条目。
  - 机制空洞：EP 不能写权威状态（I1）；"补送失败"多发生在 EP 与 `fysund` 之间链路异常时，此时 `fysund` 恰恰收不到通知。
  - `audit/observed` 无幂等键（v0.7.0 00 §4），EP 微批乱序 / 崩溃重放补送时账本去重与排序规则缺失；EP 回报事件是否携带本地单调序号未写。
- **影响**：I3（"允许异步微批，不允许丢失"）无法机械验收；实现者必须自行发明状态值、触发协议与去重键。
- **建议**：(1) 状态登记：把该等待登记为 **Activity `blocked{reason: audit_backlog}`**（沿 v0.7.0 `blocked{reason}` 携带原因与恢复动作的既有形态；它需要干预才能解除，比 Execution `waiting` 更贴切）——或在 §3.3/00 术语中正式把 Execution `waiting.reason` 扩为四值，二者择一，全仓一致；(2) 触发协议：EP 在本地 WAL 记录审计缺口（`audit_gap{from_seq, to_seq}`），恢复链路后随 `ep.report` 上报，`fysund` 落账并置状态，补送完成后自动解除；(3) EP 回报事件带 EP 本地单调序号（复用 `node_id`/`origin_seq` 机制，EP 视作事件产生节点），账本按 `(ep_id, seq)` 去重；(4) 新增负例 N-13（见 §13.2）。

#### P0-3 · F3 三档的 EP 本地持久化语义不一致：崩溃窗口掏空 I3 与 H-2

- **证据 [事实]**
  - spec §6.6 三档：`local_read`——"（可异步微批刷盘，经 EP 环形缓冲补送……）**不建 Effect、不持久化 ActionIntent**"；`external_read`——"EP 本地记录 ActionIntent（**可组批写，不阻塞同步 fsync**）→ Policy → 分发 → receipt"；mutation——"persist-before-dispatch（**同步 WAL**）"。
  - ADR-0008 §D5 与否决段（H-2）：`external_read` 必须走完整协议，理由是"账本必须能回答这条 `untrusted_external` 内容来自哪次外部读取"；N-04："`external_read` 只有 `audit/observed` 无 `action-intent/recorded` → 失败"。
  - spec §6.5："EP 崩溃恢复时先回报未回报事件再接受新调用"。
- **推断（三处不一致的后果）**
  - `local_read`：审计事件只进内存环形缓冲——EP 崩溃即丢 `audit/observed`，违反 I3"不允许丢失"；若要求审计事件也进本地 WAL，则"经 EP 环形缓冲"的表述错误、成本模型改变，设计未择一。
  - `external_read`：组批写 + 不阻塞 fsync 意味着分发可以发生在该 ActionIntent 尚未持久化（或未 fsync）时；该窗口内 EP 崩溃 → 已发生的外部读取在账本无记录，N-04 在窗口内必然违反，且恢复协议无法回报从未落盘的事件。
  - mutation：唯一完整档（同步 WAL）。
  - 设计未定义：组批的提交边界（分发是否等待组批提交）、窗口上限（条数 / 时长）、崩溃后对已分发未记录 Effect 的对账语义。
- **影响**：F3/H-2 的核心承诺在崩溃窗口内不可验证；conformance 对 N-04 的测试在故障注入下结果不确定；两个实现团队会做出两种都"符合文本"的系统。
- **建议**（三选一，必须拍板并写入 EP 契约）：
  1. **[推荐]** `external_read` 的 ActionIntent 必须在**分发前完成 WAL `write`**（fsync 可由组批窗口延后，窗口显式上限如 ≤64 条或 ≤100 ms），恢复时以 WAL 尾部校验重建；`local_read` 的 `audit/observed` 同样先入本地 WAL 再微批回报（EP 本地 WAL 是 I1 认可的 pending replica，不违反"不建 Effect、不持久化 ActionIntent"——被禁的是 ActionIntent/Effect，不是审计事件）；负例 N-14（§13.2）验收。
  2. 明确降级承诺："`external_read` 在 EP 崩溃窗口内的溯源为 best-effort，正常路径 100%"——这需要修订 ADR-0008 §D5 措辞（新 ADR），并接受 N-04 仅在无故障注入下验收；不建议。
  3. `external_read` 放弃组批、回到同步持久化——最简单，放弃的性能收益由度量决定（若 13.2 的 `audit/observed` 占比与 EP p50/p95 基线显示组批无收益，这是合理终点）。

#### P0-4 · I6 的"覆盖"无机械定义，存在两条绕过路径

- **证据 [事实]**
  - spec §8.2 / §12.2 I6："该 Lock 的非 local_read 工具面被至少一个 qualified EP 覆盖"；N-11 只测"无 mutation EP 覆盖的 Lock 被 `service/published` 且可调用 → 失败"。
  - spec §6.5：EP `capabilities[]` = `filter_tools / persist_intent / receipt / labels / flags / secret_inject / meter`——**能力位，不含覆盖的工具集**；`ep/qualified` 事件字段（§3.4）亦无覆盖集。
  - 绕过路径 A：Lock 全 `local_read` → `published` 合法（§8.2 允许"仅对纯 local_read Lock 发布"）；之后 Adjustment（`this_execution` / `capability_default`）经 Approval 启用新工具（§5.5 明确允许"启用新工具 → Approval"）——Service 仍 `stable`、Endpoint 仍可调用，而新工具面无 EP 覆盖。
  - 绕过路径 B：`ep/disqualified{reason: 能力位变化 / Policy 换版}`（§6.5）后，引用该 EP 为覆盖的 ServiceVersion 未定义级联——§8.2 只写了资产 `quarantined/revoked` → `suspended`。
- **影响**：I6 作为不变量无法写全负例；"无守卫的 mutation 面对外"可经两条路径重现——这正是两份评审 P0 要求堵死的原始威胁。
- **建议**：(1) 定义覆盖判定：`ep/qualified` 事件与 `ep.describe()` 增 `covered_tools[]`（`mcp_guard` 按其配置的 server×snapshot 工具清单、`harness_adapter` 按 Harness `native_effect_tools[]` + 其代理的 MCP 面、`gateway` 按入口协议面）；发布时计算 `ep_coverage = Lock.非 local_read 工具面 ∩ ⋃ covered_tools` 并随 `service/published` 落账；(2) Adjustment 扩工具面时，若该 Lock 被 `published` 的 ServiceVersion 引用且扩出面 ⊄ 当前 `ep_coverage` → 拒绝（或要求 republish 走发布门）；(3) `ep/disqualified` → 其覆盖的 ServiceVersion 自动 `suspended`（事件级联），补负例 N-15/N-16（§13.2）。

#### P0-5 · 三层 digest 切分函数与 Capability 引用解析未写死，`compose.resolve` 可重放性（I10）无法落笔

- **证据 [事实]**
  - spec §5.4：`lock_digest = H(contract_digest, substrate_digest, profile_digest)`、"三层规范序列化后再哈希，可重放"；§14 Q22："canonical digest 算法 R1 必须先定"。
  - 但每层的**输入字段集**未定义：`tools_allow` 的双归属（contract 的"工具面收窄" vs profile 的"允许的覆盖"，见 Q-06）；`model.fallback` 归层（§9.3 暗示 profile、§5.4 substrate 覆盖"model 身份"）；`compose.diff` 无切分算法。
  - Capability 非 kind 后（§5.1）：`capability:acme/trend-research@2` 引用不在 §4.1 八值引用体系内；Capability 的存储载体、版本（`contract_version`）与 digest 规则未定义——`contract_digest` 必须覆盖"Capability 契约 + 确认后的 `tool_effects[]` + 工具面收窄"，但 Capability 契约本身没有 digest 语义。
- **影响**：R1 验收首条即"闭环：……`compose.resolve`……账本 replay digest 稳定"（§13.1 R1）；切分函数不定，同一 Definition 两次 resolve 的层归属可能不同，I10 / replay 金样 / `compose.diff` 全部无法实现；Capability 证据归属（§9.1"Verification → Capability（契约层）"）无落点。
- **建议**：(1) 在 §5.4 增"层归属规范表"：逐字段写死（建议：`tools_allow` 的**声明位置**入 profile、其**收窄结果**入 contract——与 §5.4 现有两行兼容：contract 覆盖收窄结果、profile 覆盖声明；`model.fallback` 及其目标身份整体入 profile，substrate 只含主 model 身份；`harness.prompt_slots` 结构入 substrate；`memory_provider` 身份入 substrate、`memory.scope` 选择入 profile）；(2) 给 Capability 定义登记载体（建议：Capability 为 Catalog 内的版本化契约记录，登记时计算 `contract_digest_c = H(behavior, quality, bounds.tool_effects[], verification)`，`capability:` 引用解析到 `(capability_id, contract_version, contract_digest_c)`，证据按 `attributable_to` 挂靠）；(3) `compose.diff` 给出确定性算法（按字段→层映射逐字段比较）；(4) 负例 N-17（§13.2）。

### 6.2 P1

| # | 问题 | 证据（路径+章节） | 影响 | 建议 |
|---|---|---|---|---|
| P1-1 | §14 开放问题清单：Q8 编号复用且与"Q7–Q10 已关闭"行同表矛盾；Q11/Q27 缺号 | spec §14 vs v0.7.0 12 §7.2（"Q8……编号保留以备追溯，不复用"） | 清单自称"编号只增不复用"却自违；交叉引用（两份评审大量引用 Q8/Q24/Q26/Q29/Q30）失配 | Trust 恢复为 Q27（或新编号 Q34）；删除 Q8 复用行；补 Q11 占位说明；grep 校验编号唯一性 |
| P1-2 | `inherited` 证据标记的载体未定义 | spec §5.4（"复制引用而非改写，继承集合须可审计"）、N-02；事件族 §3.4 无对应 | "保留 `inherited` 并降权展示"无法落 schema；N-02 无法机械验收 | 定义载体：建议 `evidence[]` 条目增 `inherited_of: lock_digest?` 字段 + `adjustment/applied` 或新事件 `evidence/inherited{from_lock, items[]}`；R0 并入 00 术语 |
| P1-3 | park 快照的信任级与 thaw 注入语义未定义 | spec §6.7（"快照入 Artifact……不含权重/KV cache"） | ExecutionSnapshot 由 Harness 生成却入 Artifact 治理面；thaw 后注入是否按 `untrusted_external` 候选处理、是否过来源标签检查未写——构成新的注入面（被注入的 Harness 可在快照里埋指令，绕过逐 turn 检查） | 写死：快照 Artifact `label.integrity = untrusted_external`（生成者非 `fysund`）；thaw 注入视同 `context_injection` 片段走装配与标签检查；负例 N-18 |
| P1-4 | hooks EP 的 fail-closed 语义与 Policy 下发模型矛盾 | spec §6.4 hooks 行（"`fysund` 不可达时退出码 2 fail-closed"，继承 v0.7.0 06 §9）vs §6.5（Policy 下发为主、EP 本地评估、负例 §6.9.9"每次 tools/list 都 RPC 到 fysund → 失败"） | hooks 若每次调用都 RPC `fysund`，则 `fysund` 成为原生工具热路径（违反 P11 精神）；若本地评估，"不可达即退出码 2"过严（快照有效时应可判定） | 改为：hooks 拉取快照本地评估；fail-closed 条件 = 快照缺失或 `policy_stale`；`fysund` 不可达但快照有效 → 按 `offline` 语境评估（沿 ADR-0010 离线规则）；负例 N-19 |
| P1-5 | R2 单阶段交付面过重 | spec §13.1 R2 行 vs R1/R3 | 关键路径拉长；sampling/OAuth 等协议深水区与闭环强耦合，任一外部依赖（A2A SDK 形状、OAuth 测试床）滑期即阻塞整阶段 | 拆 R2a（EP 契约 + F3 三档 + adapter-min 覆盖闭环工具面 + `os.observe` + EP 恢复协议）/ R2b（gateway + A2A 1.0 Card + 单 Tenant + sampling 防御 + OAuth audience）；R2a 退出条件 = 现 R2 验收中负例 §6.9.1–10 + N-03–N-06；R2b = §8.8 相关负例 |
| P1-6 | Loop Profile 双形态的 digest 口径与 `profile.put` 生命周期未定义 | spec §4.2（内联或命名登记）、§11.2（`profile.put/describe`）；无 put 的覆盖/删除/版本规则 | 内联 params 是否计入 `agent` 资产 digest 还是仅 `profile_digest`；命名 Profile 的登记表字段（`{harness_ref, params, digest?}`）与幂等/覆盖语义未写 | 写死：内联 params 只入 `profile_digest`（不入 agent manifest 的规范序列化，避免同一配置双重 digest）；`profile.put` 语义 = 按键幂等写入 + 新 version 即新键值；登记表存 `{harness_ref, params, params_digest, publisher, time}` |
| P1-7 | 术语表缺口清单 | spec 附录 B 只列 change_class/park/EP/Tenant；正文另用：`audit_backlog`、`ExecutionSnapshot`、`runtime.cache_alignment_violated`、`prefix_order[]`、`cache_hint`、`policy_stale_ttl`、`lock_overrides_digest`、`profile.put`、`ep_coverage`（若采纳 P0-4） | R0 回填 00 术语表时漏登记会重演 v0.7.0 的"多处字面不一致"（蓝图 §0 C1 系列教训） | R0 清单增"术语补登记表"（逐词给出：对象/字段/事件/错误码/Policy 设定的归类）；`runtime.cache_alignment_violated` 明确归类（建议：OTel 事件/告警键，非账本事件、非错误码） |
| P1-8 | `canary→stable` 的最小样本与证据口径 | spec §9.2（通过率阈值、影子评估） | 零样本/低流量 canary 的阈值除零；canary Verification 是否计入 `evidence_gate` 的"非 inherited 证据"未明说 | 写死：canary 进入 stable 需 ≥N 次 Verification（N 为 Policy 设定，缺省建议 ≥10）且全 pass；canary 产生的本 Lock Verification 即非 inherited 证据，计入 evidence_gate |

### 6.3 P2

| # | 问题 | 证据 | 建议 |
|---|---|---|---|
| P2-1 | `runtime/qualified` 事件字段 `runtime_id` vs v0.7.0 `resource_id`，未登记改名 | spec §6.1 vs design/00 §4 | R0 术语对照表登记改名，或恢复 `resource_id` |
| P2-2 | A2A 加入 AAIF 日期口径（设计 2026-08-27；检索 2026-08-17/20） | spec §8.6 vs 本报告 §9 域 2 | 复核 aaif.io / a2a-protocol.org 公告原文后钉准确日期 |
| P2-3 | sampling 兼容窗口的协议版本协商记录未入账本 | spec §8.6（"兼容窗口内（旧协议版本 server）"） | `mcp_guard` 把协商的协议版本随回报事件记录，作为"窗口内"判定的审计依据 |
| P2-4 | Card 签名密钥泄漏无威胁向量与轮换机制 | spec §12.2 T15；A2A 1.0 `signatures[]` 支持 JWS 多签 | 增向量 T16（Card 签名密钥泄漏 → 伪造服务）与密钥轮换/透明日志开放问题（Q36） |
| P2-5 | gateway execution lease 的失效/接管协议未定义 | spec §6.7（"lease 失效回 fysund"）、§8.3 | 写 lease TTL、心跳与 `fysund` 强制回收（`execution/cancelled` 通知）语义 |
| P2-6 | `model.access: direct` 与 Plan token 配额 / Q26 的交互未写 | spec §5.2、§8.5 | 多租户 Plan 含 token 配额时 Policy 应拒绝 `access: direct`（不可信计量），或配额降级为 calls 维度；写入 §7.3 Policy 问题清单 |
| P2-7 | `resource/metered` 与 `meter/recorded` 的聚合规则未写 | spec §6.4/§3.4 | 定义折叠规则（`provider_proxy` 的 token 计量事件如何成为 `meter/recorded{dimension: tokens}`） |
| P2-8 | Q23"最小集 = max_steps + stop_conditions[]"与 §4.2 六字段"最小 params"口径不一 | spec §14 Q23 vs §4.2 | 统一表述：§4.2 为"跨 Harness 建议子集（advisory）"，Q23 的最小集为"校验强制下限" |
| P2-9 | 消费者入站授权（AaaS MCP facade 的 authorization server 归属）未登记开放问题 | spec §8.1 endpoints[].auth | 新增 Q37：多租户对外服务的 OAuth authorization server 由运营者还是第三方 IdP 承担（R3 后） |
| P2-10 | canonical digest 算法的决策归属未列入 §15 决策表 | spec §14 Q22、§15 | §15 增决策项："R1 前钉 canonical JSON 序列化（建议 RFC 8785 JCS）与 SHA-256" |
| P2-11 | `kind=agent` 的 `declares.effects[]` 登记口径 | spec §4.3 vs §5.3 | 写一句："kind=agent 登记时 declares.effects 必须为空数组，由 compose.resolve 汇总填充" |
| P2-12 | canary 证据口径 | 见 P1-8 | — |
| P2-13 | `policy_stale_ttl` 缺省推荐值 | spec §7.3 | §15 决策表给推荐值 |
| P2-14 | 优化器不可变区未点名 `subagents[]` | spec §9.3 表 | 不可变表增一行"委派白名单（`subagents[]`）扩张" |
| P2-15 | `trajectory.export` 脱敏义务未成文 | spec §9.3/§11.2 | 补：与 `os.report_failure` 同级的脱敏规范（剥离凭据/宿主路径/租户标识；上限与 `diagnostics_ref` 机制） |
| P2-16 | AAIF Agent Plugins 1.0（skills+MCP 的签名包格式）出现，Q22 候选应纳入观察 | 本报告 §9 域 3 | Q22 候选方案增"对齐 AAIF Agent Plugins 1.0（watch，不抢跑）" |

<!-- 待续：第 7 节 跨文档冲突表 -->


