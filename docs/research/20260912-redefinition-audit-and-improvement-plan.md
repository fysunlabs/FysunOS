# FysunOS 重定义提案：全面复检与可执行改进方案

- **本文件路径**：`docs/research/20260912-redefinition-audit-and-improvement-plan.md`
- **性质**：独立审查与改进方案。只读对照 `docs/redefinition/` 与现行 `docs/design/`、`docs/adr/`、`docs/research/`，**不修改任何被审查原文**。
- **审查日期**：2026-09-12
- **被审对象**：`docs/redefinition/` 提案草案 v0.8.0-draft（2026-09-11），含 README、00–10、ADR-0013 草案
- **对照基线**：现行设计 v0.7.0（2026-09-08）；尤其 [ADR-0008](../adr/0008-effect-category-and-read-only-determination.md)（accepted）、[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md)、[ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md)、[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md)、[ADR-0012](../adr/0012-mcp-guard-self-implementation.md)
- **外部核验截止**：2026-09-12；一手来源见 §13。未核验到的事实标 **[待验证]**，不臆造 revision / 日期。
- **与同日另一份审查报告的关系**：工作区可能另有 `docs/research/20260912-redefinition-review-and-refactor-report.md`。本文件是**新路径上的独立完整报告**，不读取后覆盖、不编辑、不依赖该文件。

## 0. 元信息与证据分级

### 0.1 不改原文声明

本次工作**禁止且未执行**下列动作：

- 不修改 `docs/redefinition/` 下任何原文。
- 不读取后写入、不覆盖、不编辑 `docs/research/20260912-redefinition-review-and-refactor-report.md`。
- 不 git commit。
- 不改其他无关文件。

所有「应改成什么」只写在本方案中，作为后续若采纳 ADR-0013 时的修订输入。

### 0.2 证据分级

正文用标签标等级，避免把推断写成规范事实：

| 标签 | 含义 |
|---|---|
| **[事实]** | 可在指定路径/章节或可访问 URL 原文核对 |
| **[推断]** | 由多处原文合取推出，原文未直接写该结论 |
| **[建议]** | 本报告的改进动作；尚未被仓库采纳 |
| **[待验证]** | 外部规范或上游行为在 2026-09-12 未能用一手页面钉死，或依赖尚未实现的测量 |

未在本仓库或外部系统上测量过的延迟、命中率、成本节省**不得写成事实**。下文凡涉及性能，只给可判定机制与度量项，不给未测百分比。

### 0.3 审查方法

1. 通读 `docs/redefinition/README.md`、`00-terminology.md`–`10-roadmap.md`、`adr-0013-draft-repositioning.md`。
2. 对照 `README.md`、`docs/design/**`、`docs/adr/**`（重点 ADR-0008 D5、ADR-0002 D2、ADR-0007、ADR-0009、ADR-0012）、`docs/research/20260908-v0.7.0-refactor-blueprint.md`、`docs/research/sources.md`。
3. 用 WebSearch / WebFetch 核验 2026-09-12 仍可访问的一手规范（MCP 2026-07-28、A2A 1.0、Cedar/AgentCore Policy、OTel GenAI、SPIFFE、prompt caching、Temporal durable execution、DSPy/GEPA、SWE-bench 污染、MCP OAuth 2.1）。

---

## 1. 执行摘要

**是否建议重构：建议部分重构（partial refactor），不建议全量推倒重写，也不建议按现草案原样冻结进 v0.8.0 实现。**

重定义的方向正确：**[事实]** ADR-0013 §D1 把定位收回到「能力管理 + Runtime 控制」运行基座，`fysund` 退出数据面热路径，Harness / Loop 升为可登记资产，多主体进入核心，AaaS 作为第三平面。这些与 ADR-0001 §D1 原点、ADR-0007 已承认的内联守卫结构性极限一致。应保留的内核见 §4。

但现草案若直接按 11 种资产 `kind`、扁平 Lock、「新 Lock 从零累积」、R3 先于 R4 对外 `published` 去实现，会在四个地方把 v0.7.0 已经可判定的底线冲掉，或把热路径重新做厚。建议在采纳 ADR-0013 的**同一轮**做有限重构：实体分层、Lock 分层、04 显式继承 ADR-0008 D5、对外发布以 qualified EP 覆盖为门。不另起第三套对象模型。

### 最关键结论（8 条）

1. **[事实]** ADR-0008 D5（accepted）：只有 `local_read` 走审计快路径（不持久化 ActionIntent / Effect，必须 `audit/observed`）；`external_read` 走完整协议。重定义 04 §4.1–§4.2 **未显式继承**这一分档；01 §7 F3 又把 persist-before-dispatch 收窄成「外部 / 不可逆 mutation」。实现者会二选一走偏。**[建议]** 04 按 D5 分档；**禁止**把 `external_read` 放进无持久化快路径。
2. **[事实]** `runtime` 被放进 00 §3 / ADR-0013 §D2 的封闭 `kind` 枚举，但 02 §1 定义资产为「可复用、可分发、可被 Agent Definition 引用」，三条同时满足；02 §3.10 又写 Runtime「通常不经资产仓库分发」；03 的 Agent Definition **并不引用** `runtime` 资产。这是定义冲突，不是用词问题。**[建议]** `runtime` 降回运行对象（资格化实例），退出可分发 `kind`。
3. **[事实]** 09 §2：「换资产后旧证据留在旧 Lock；新 Lock 从零累积」。这与 01 P3 / P6、09 自身的 Adjustment / 证据门、以及自进化目标冲突。**[建议]** 分层 Lock + 按 digest 子集差分继承证据；换 Harness / Model 仍不迁移资格化证据。
4. **[事实]** 10 §3 把 R3（服务平面、`service.publish`）排在 R4（EP 契约与治理闭合）之前；10 §7 第 3 项把「先对外」列为待所有者决策且草案已按此排列。05 §12 的发布负例**没有**「mutation 工具面必须被 qualified EP 覆盖」。**[建议]** 含非 `local_read` 工具面的 Service 不得进入 `published`，除非至少有一个 qualified EP 覆盖该工具面。Personal 单租户可绕过 gateway，但不能绕过这条门。
5. **[事实]** MCP 规范 2026-07-28：Sampling（及 Roots、Logging）按 SEP-2577 弃用，新实现 SHOULD NOT 采用，至少保留 12 个月兼容窗。现行 HTTP 传输的核心是 Authorization（OAuth 2.1、RFC 9728 PRM、RFC 8707 resource indicator、RFC 9207 `iss`）。07 §1 仍按工具/资源/提示/Tasks/elicitation 列举 MCP，**未列** Authorization，也未钉规范日期。sources.md 仍钉 2025-11-25。**[建议]** 新实现默认拒绝 Sampling；兼容窗才做防火墙；AaaS MCP facade 与下游 MCP client 按 OAuth 2.1 做 audience 绑定。
6. **[事实]** 07 §1 写「`gen_ai.*` 钉 commit」。OpenTelemetry GenAI 约定在 2026-06 迁入独立仓库后，截至 2026-08 公开统计仍全部为 Development，schema URL 在官方 README 仍为 TODO。**[建议]** 导出可带 `gen_ai.*`，但账本 schema 与 SLO 不得把未稳定语义当强制契约；核心键走 `fysun.*`。
7. **[建议]** DSPy / GEPA 只作为离线编译插件：输入回归集与 traces，输出新的 `prompt` / `loop` / `skill` 版本 + Verification；**不得**进入 `tools/call`、模型请求或 `compose.resolve` 热路径。SWE-bench Verified 不得作为对外证据门的回归集（污染与测试缺陷已有一手陈述）。
8. **[推断]** 草案对象面过宽（11 `kind` + Service / Tenant / Plan / Meter / EP / Capability / Lock 全在 R1–R3 铺开），会把「精简高效」做成登记操作系统。**[建议]** 四层实体收口；R1 只冻结可分发制品 + Runtime 对象 + 最小 EP 契约；Meter 作为事件而非一级对象实现。

---

## 2. 设计地图

### 2.1 提案自己声称的结构 **[事实]**

按 README 文档地图与 01 §3：

```text
消费者（A2A / MCP / HTTP）
        │
   服务平面    Service · ServiceVersion · Endpoint · Tenant · Plan · Meter
               入口 EP = fysun-gateway
        │
 资产/组合平面  11 种 kind 的 Registry · Agent Definition · Capability · Lock
        │
   运行平面    Runtime 资格化 · Activity/Execution · 执行点 · 账本 · Node
        │
   治理横切    Principal/Scope · Grant · Policy(Cedar 子集) · Approval · 标签 · Verification
        │
   生态供给    Harness / 工具 / MCP server / 模型 / 记忆 / 外部 Agent
```

`fysund`：资产登记、组合解析、Runtime 管理、服务目录、PDP、Approval、唯一权威账本写入者；**不在数据面热路径**（01 P11、04 §4.4）。

数据面 EP `kind`（04 §4.1）：`gateway` / `mcp_guard` / `hooks` / `harness_adapter` / `provider_proxy` / `external_gateway`。

### 2.2 实际耦合（原文未画、由交叉引用推出） **[推断]**

| 调用路径 | 实际穿过的对象 | 热路径风险 |
|---|---|---|
| AaaS 一次调用（05 §5 时序图） | gateway → `fysund`（`activity.create` / Grant 衰减 / `compose.match` / `start_execution`）→ Runtime → 工具侧 EP → `ep.report` 回 `fysund` | 控制面在请求上；工具转发不经 `fysund`（符合 P11）。但 04 把每次 `tools/call` 写成同步持久化，会把 EP 做成新的热路径瓶颈 |
| 组合解析（03 §4） | 全部资产引用 → 最高 `published` → digest → 单块 Lock | Lock 不可分，任何一处 digest 变都整包作废证据 |
| 发布（05 §3） | Lock → Policy → 可选 evidence_gate → Agent Card → `service/published` | 不检查 EP 覆盖 |
| 自进化（09） | Adjustment → 新 Definition 补丁或 Lock 覆盖 → 新 Lock 从零累积证据 | 差分升级的证据被丢弃 |

### 2.3 与 v0.7.0 的职责迁移 **[事实]**

| v0.7.0 | 重定义 | 落点 |
|---|---|---|
| `fysun-mcp-guard` 为 `fysund` 内组件、harness 唯一 MCP server（ADR-0002 D2、ADR-0012 D1） | 独立可选 EP | 04 §4、ADR-0013 §D4；ADR-0012「不前置」被明确重开 |
| `Capability.implementation` = Skills 目录 | 删除；由 Agent Definition 实现 | 03 §1 |
| Runtime = Resource 类型 | Runtime = Harness 资格化实例，且仍是资产 `kind` | 00 §1、02 §3.10 |
| F5 以 Provider 代理为前提 | 改为只向 qualified EP 解析 SecretRef | 01 §7 |
| 投递管道 / `fysun.await_approval` | 废止为 OS 规范；改 `os.observe` | 04 §2 |
| Q5 多主体、Q11 Trust | Tenant 核心；Trust 改 R4 必做 | 10 §6 |

---

## 3. 应保留的核心（不要在重构中拆掉）

下列条目在重定义与 v0.7.0 之间连续，且与 2026 工业实践同向。重构应把它们写得更可判定，而不是换成另一套词。

1. **定义句与两项职责**：能力管理 + Runtime 控制；基座不写 loop / prompt 装配 / 沙箱（01 §2、§8、P8）。
2. **F1–F9**，尤其：唯一权威账本、候选≠准入、独立验证、secret 隔离、能力不自扩权、F9 单调收窄。F3 的**类别分档**必须回到 ADR-0008，而不是回到「一切 mutation」或「一切调用」。
3. **event-sourced 治理账本**；EP 本地 WAL 只是待核对副本，权威投影只在 `fysund`（F1）。
4. **Cedar 子集 + 版本化 Policy 下发**；`approve` 与 Grant 扩张回中心（ADR-0013 §D5）。这与 AgentCore Gateway 用 Cedar 在工具调用点做 default-deny 的工业形态同向，但 FysunOS 的四值 `outcome` 与实体模型仍以 ADR-0009 为根。
5. **六值 `effect_category`、三源合一、`max_severity`、未分类 → `external_mutation`、annotations 不作安全依据**（ADR-0008 D1–D4）。
6. **Activity / Execution 生命周期、双水位恢复、重新武装、幂等键、`conflict.idempotency`、补偿用新 Grant**。
7. **标准优先**：MCP（工具面）、A2A（服务端点）、Agent Skills（`skill` 格式）、OCI artifact（分发）、不自造包格式 / 策略语言 / 传输协议。
8. **conformance 按覆盖面分列、声明不实不在承诺范围、大声失败、不设能力等级制度**。
9. **Approval 封闭结果集、审批面只由 `fysund` 提供**（ADR-0011）；Harness 文本不构成审批。
10. **P11：`fysund` 不在数据面转发链**（ADR-0013 负例 2）。这是性能与安全的同一条禁令。

---

## 4. 问题清单

优先级约定：

- **P0**：与已采纳 ADR 或底线冲突，或对外发布会把 mutation 放到无 PEP 的面上；采纳 ADR-0013 的同一轮必须改方案（仍只改后续文档，不改本审查所读原文）。
- **P1**：不改也能做出 R1 登记，但会卡住性能、自进化或 AaaS 治理。
- **P2**：文档债、钉版、可延后到 R4/R5。

每条含：路径 + 章节、影响、建议。建议是方案，不是对原文的直接补丁。

### 4.1 P0

#### P0-1 ADR-0008 D5 未写入 Runtime / EP 契约

- **证据 [事实]**
  - ADR-0008 §D5：审计快路径**仅**对 `effective effect_category = local_read` 开放：不持久化 ActionIntent / Effect，必须 `audit/observed{..., effect_category: local_read, path: guard|hook|adapter}`；`external_read` 走完整协议（ActionIntent → Effect → Receipt）。否决项写明 owner 否决「`external_read` 白名单快路径」（H-2）。
  - 重定义 01 §7 F3：「每个**外部 / 不可逆 mutation** 先持久化…」。按字面，`external_read` 不是 mutation，会被读成可跳过持久化。
  - 重定义 04 §4.1 `mcp_guard`：`tools/call` pre 写「ActionIntent 持久化」，不分档。
  - 重定义 04 §4.2：「persist-before-dispatch 在 EP 本地成立：EP 先本地持久化 ActionIntent / Effect…再分发」——对**所有**调用一刀切。
  - 04 §4.2 / §8 负例 5 只在 `policy_stale` 时提到「只允许 `local_read`」，承认 `local_read` 特殊，但未定义快路径事件与禁止项。
- **影响**：实现者要么对每次 `tools/call` 同步 WAL（把 EP 做成新的 `fysund` 热路径），要么把 `external_read` 也做成无记录快路径。后者破坏「这条 `untrusted_external` 来自哪次外部读取」的溯源，与 ADR-0008 已采纳决策直接冲突。
- **建议**：后续 04 / 01 / EP 契约写三档，而不是两档：
  1. `local_read`：内存 Cedar + Grant + 标签 + `tools/list` 过滤 → 分发 → **必须** `audit/observed`（可异步微批刷盘）；不建 Effect。
  2. `external_read`：完整协议；EP 本地记录意图可组批，**不得**免持久化；Policy 通常 `allow`。
  3. `local_mutation` 及以上：完整 F3 + 同步持久化后再分发。
  - **明确拒绝**：把 `external_read` 放进无持久化快路径。
  - 负例沿用 ADR-0008：`external_read` 只有 `audit/observed` 无 `action-intent/recorded` → 失败；`local_read` 无 `audit/observed` → 失败。

#### P0-2 对外 `published` 与 qualified EP 脱钩

- **证据 [事实]** 10 §3 R3 交付 `service.publish / rollout` 与 gateway；R4 才交付 EP 契约、`fysun-mcp-guard` 独立进程、`policy_stale_ttl`。10 §7 第 3 项已按「先服务平面」排列，理由是「F3 在 R2 已由 `harness_adapter` 或 `mcp_guard` 的最小实现覆盖**首个闭环**」。05 §3 发布流程的门是 Policy + 可选 evidence_gate + Agent Card，**没有** EP 覆盖检查。05 §12 负例 1 只禁止未 `service/published` 的调用，不禁止「已 published 但 mutation 面无 PEP」。04 §4.3 只说 Policy **可**拒绝把无 EP Runtime 用于含非 `local_read` 的 Lock，不是发布前置。
- **影响**：按草案字面，R3 可以把带 `external_mutation` 工具的 Agent Card 公开出去，而 PEP 仍是「指定」状态。这不是「先做目录」，而是把无守卫的 mutation 面标成服务。
- **建议**：
  - 对内目录 / Plan 草稿可以在 R3。
  - **`service/published` 的资格条件**：该 Lock 的非 `local_read` 工具面被至少一个 `ep/qualified` 的 EP 覆盖；否则只允许 `draft` / `staged`，或只对纯 `local_read` Lock 发布。
  - Personal 单租户允许绕过 gateway（10 §7 第 5 项可保留），**不允许**绕过 EP 覆盖门。
  - 负例：Lock 含 `external_mutation` 工具、无 qualified EP，`service.status = published` 且 Endpoint 接受调用 → 失败。

#### P0-3 F1 与「EP 本地持久化再回报」未写成不变量

- **证据 [事实]** 01 §7 F1：只有 `fysund` 可以推进 Effect 状态、写权威账本。04 §4.2：EP 先本地持久化 ActionIntent / Effect 再分发，随后回报；`fysund` 以回报重建投影。ADR-0012 §D1 当初把守卫放进 `fysund` 进程的理由正是「判定与账本写入同一事务边界」；ADR-0013 后果段重开「不前置」，但未重写 F1 的可判定含义。
- **影响**：崩溃窗口里出现「EP 已 dispatch、账本尚无 Effect」或「EP 本地有、回报被拒」。若把 EP WAL 当成权威，F1 被静默废除；若完全不认 EP WAL，则 F3 在独立进程上不成立。
- **建议**：写三条不变量（后续 04 / ADR-0013 修订用）：
  - I1 权威账本 ⊆ `fysund`。EP WAL 是 pending replica，不是第二账本。
  - I2 无回报的 Effect 不得 `verified`（已有）；有 `dispatch-started` 无 receipt 走 outcome-unknown（已有）。
  - I3 `local_read` 不建 Effect，但必须最终在权威账本出现 `audit/observed`（允许异步，不允许丢失）。
  - 增加负例：EP 本地写出 `effect/verified` 或 `approval/decided` → 失败（04 已有 Approval 项，缺 Effect 终态项）。

#### P0-4 `runtime` 作为资产 `kind` 与「可分发」定义冲突

- **证据 [事实]**
  - 00 §3 封闭枚举含 `runtime`；ADR-0013 §D2 同样列出。
  - 02 §1：「**资产** = 可复用、**可分发**、**可被 Agent Definition 引用**之物。判定标准三条同时满足：有稳定身份与版本；内容可 digest；可脱离某次 Execution 存在。」
  - 02 §3.10：Runtime 由 `runtime.register` 产生，**通常不经资产仓库分发**。
  - 03 Agent Definition YAML：引用 harness / loop / model / prompts / skills / tools / mcp_servers / memory / knowledge / subagents，**没有** `runtime:` 引用。绑定发生在 `compose.match`（03 §4）。
  - 09 §2 把 conformance 记到「harness / runtime / **EP 资产**」——EP 甚至不在 `kind` 枚举里。
- **影响**：R1 要为 11 种 `kind` 写统一 pack/push/register。把 Node 本地安装实例塞进 OCI 分发模型，会迫使假 digest、假仓库，或在实现里给 `runtime` 开例外——例外会污染全部资产 API。Agent Definition「无 manifest 不能引用」（P9）与 Runtime 对象被误当成可组合制品，也会让「换 Runtime ≠ 换 Agent」说不清。
- **建议**：资产 `kind` 去掉 `runtime`。Runtime 回到「Harness 资产在某 Node 的资格化实例」（ADR-0013 §D3 的定义保留，分类改正）。EP 是运行对象，不是资产。09 的资格化证据挂在 `harness` 版本 + `runtime/qualified` 事件，不发明 `ep` kind。

### 4.2 P1

#### P1-1 扁平 Lock +「新 Lock 从零累积」阻断自进化

- **证据 [事实]** 03 §4 Lock 是一张精确 digest 表，不可变。09 §2：「新 Lock 从零累积（可由 `compose.diff` 提示…作为**人工判断**参考，**不作为系统推断**）。」09 §9 负例 1：换 Harness 后旧 Lock 证据计入新 Lock → 失败。03 §7 Adjustment 的 `this_activity` 只记 `lock_overrides_digest`，不改 Lock。
- **影响**：只换 prompt 版本也丢弃 skill / tool 上已有的 deterministic 证据，自进化每次都从零交学费。负例 1 要保护的是「资格化证据不跨 id」，不是「资产层证据不可差分继承」。
- **建议**：分层 Lock，见 §9.2。`compose.diff` 对**未改 digest** 的资产层证据自动继承；对改了的 digest 从零；Harness / Runtime / EP 资格化证据仍禁止跨 id。

#### P1-2 MCP 反向能力面与 Authorization 缺口

- **证据 [事实]** 07 §1 MCP 用于 `tools/list` / `tools/call` / `resources` / `prompts` / `tasks` / `elicitation`，未列 `sampling/*`、roots、OAuth。MCP 2026-07-28 Sampling 页原文：feature deprecated as of `2026-07-28` (SEP-2577)；New implementations SHOULD NOT adopt it；至少十二个月后才有资格移除。Authorization 页原文：HTTP 传输 SHOULD 遵循该规范；基于 OAuth 2.1；MCP server MUST 实现 RFC 9728；client MUST 用 PRM 发现；DCR deprecated，CIMD SHOULD。05 Endpoint.auth 只写「tenant token / mTLS / OAuth，由 gateway 强制」，无 audience / resource indicator。
- **影响**：新实现若按 2025-11-25 去做 Sampling，会把「server 向 client 要模型」做成默认能力，扩大 TCB，且与现行规范相反。MCP facade 若只验租户 token、不绑 RFC 8707 audience，会让 token 被另一个 MCP server 转用。
- **建议**：EP / gateway 默认拒绝 `sampling/createMessage`（及已弃用的 roots/logging 作为安全边界宣称）。兼容窗：对声明旧协议版本的 server 做防火墙（记录、拒绝或经 Policy `approve`），不在新协议连接上实现 Sampling。HTTP MCP 按 OAuth 2.1 + PRM + resource indicator 做；stdio 仍用环境凭据（规范允许）。钉规范日期 2026-07-28，在 07 做与 2025-11-25 的差异表。

#### P1-3 Cedar 实体扩展未走 ADR-0009

- **证据 [事实]** 06 §3 增 principal `Tenant` / `EnforcementPoint` / `Publisher`，resource `Asset` / `Service` / `Endpoint` / `Plan`，一批新 action 与 context 键。ADR-0009 §D1 明文：「未列出的键不得出现在策略中；新增 context 键需修订本 ADR」。
- **影响**：不先修订 ADR-0009 就写 EP 本地评估，`policy.load` 按现行 schema 应拒绝这些键；或实现偷偷扩展，破坏「子集范围已关闭」（v0.7.0 Q7）。
- **建议**：R0 产出 ADR-0009 增补草案，与 ADR-0013 一起采纳。新增键保持封闭表，禁止 EP 自造 context。

#### P1-4 确定性前缀槽未进入 Harness / prompt 资产契约

- **证据 [事实]** 02 §3.1 / §3.5：基座不装配 prompt；Harness 有 `prompt_slots[]`。全文无 prefix 稳定性、工具面排序、cache breakpoint 约束。Anthropic / OpenAI 2026 文档均把 agent 多步工具循环的缓存建立在「tools → system → messages 前缀不变」上。
- **影响**：Lock 每次微调工具顺序或在 system 槽插入动态时间戳，会打掉 provider 前缀缓存。这不是基座「实现缓存」，而是**禁止自己制造不可缓存的装配约定**。未测量命中率，故不在此宣称节省幅度。
- **建议**：`harness` manifest 增 `prompt_layout`：slots 顺序固定；工具描述符按 Lock 内 canonical 排序；动态内容只进最后一个 user 槽。`loop` 资产不得在每步改工具面。度量：`cache_read` / `cache_write`（provider 回传），基线先测、阈值后定。

#### P1-5 调用热路径上的同步点过多

- **证据 [事实]** 05 §5：每次调用 gateway 都打 `fysund` 做 `activity.create_or_wake` 与 `start_execution`。04 §4.2 `ep.report` 批量回报已有，但 pre 路径仍暗示同步持久化。08 未规定 `ep.report` 的延迟 SLO。
- **影响**：P11 只把 `fysund` 移出**转发**链，没有移出**每次调用的控制 RPC**。对短任务 AaaS，控制 RPC + 全量 WAL 会主导 TPOT。无测量，不断言倍数。
- **建议**：
  - 控制面：`session_mode: per_context` 时 gateway 可持有 execution lease，wake 走本地快路径，lease 失效再回 `fysund`。
  - 数据面：只读并行分发（多 `local_read` / 已授权 `external_read` 在 Grant 允许下并发）；mutation 仍串行 + 持久化。
  - `ep.report` 微批；`audit/observed` 允许丢失窗口用本地环形缓冲补送，补送失败 → Execution `waiting{reason: audit_backlog}`，不得静默丢。
  - 度量先建：EP p50/p95、`audit/observed` 占比、回报拒绝率。阈值后定。

#### P1-6 自进化没有「离线编译插件」槽，却也没有禁止热路径优化器

- **证据 [事实]** 09 把改进策略留给 Adjustment + 证据门；01 §8 不做 loop。未提到 DSPy/GEPA，也未禁止在 Execution 内改 prompt 资产。
- **影响**：后续实现容易把反射优化器挂进每轮 loop（热路径、不可复现、且会改 Immutable Zone）。GEPA 论文与 DSPy 文档描述的是离线/预算内的程序演化，不是请求内编译。
- **建议**：R5 增加 `optimizer` **插件约定**（不必先成为资产 `kind`）：只读 traces + 回归集 → 产出新资产版本草案 → 必须经 `evidence_gate` 与 F9；默认关闭。禁止 Execution 内调用优化器。SWE-bench Verified 不得进入对外回归集（见 §7）。

#### P1-7 A2A 1.0 Agent Card 形状未对齐

- **证据 [事实]** 05 §2 / §4：`/.well-known/agent-card.json`；`skills[]` ↔ `capabilities[]`。A2A 1.0（2026-03-12）打破 0.3：`url` / `preferredTransport` 收入 `supportedInterfaces[]`；`protocolVersion` 下放到每个 interface；`supportsAuthenticatedExtendedCard` 迁到 `capabilities.extendedAgentCard`。
- **影响**：按 0.3 形状签名的 Card 会被 1.0 client 拒。Q28 只问 streaming / pushNotifications，没问 Card schema 版本。
- **建议**：gateway 只发 A2A 1.0 Card；`A2A-Version: 1.0`；负例：Card 顶层仍用 0.3 的 `url` + `preferredTransport` 且无 `supportedInterfaces` → 失败。

#### P1-8 `shared` Runtime 池化证据不足即进入 R3

- **证据 [事实]** 04 §6 缺省 `dedicated`；`shared` 要求 Harness 声明 `session_isolation`。Q24 仍开放「隔离证明」。10 把 Runtime 池化放进 R3 验收。
- **建议**：R3 只验收 `dedicated`。`shared` 延后到 Q24 有负例（会话串扰、tenant_id 泄漏）再开。缺省保持 `dedicated`。

### 4.3 P2

| ID | 证据 | 建议 |
|---|---|---|
| P2-1 | 06 §9 / 07 §1：`gen_ai.*` 钉 commit；sources.md 记 Development | 钉仓库 + schema URL（若仍 TODO 则钉 commit SHA）；核心 SLO 用 `fysun.*`；准备 collector 改名层 |
| P2-2 | 02 十一种 `kind` 的 spec 深浅不一；`tool` 与 `mcp_server` 工具目录重复 | R1 先做 `prompt/skill/mcp_server/harness/loop/model/agent`；`tool` 可作 mcp 的单工具视图而非独立分发格式；`knowledge` / `memory_provider` R2 |
| P2-3 | Meter 是一级对象（00 §2）但 05 §7 说由事件折叠 | 实现上 Meter 是投影，API 保留 `meter.query`；不要第二套记账表 |
| P2-4 | 05 `sla_declared` vs `health` | 保持「声明≠证据」；发现接口并列，禁止把 sla 当 Verification |
| P2-5 | 08 六组 API 方法表已多于首个闭环 | R1 只实现 Registry + Runtime.qualify + 最小 Governance；Composition 在 R2；Service 在 R3 |
| P2-6 | 07 Tasks 仍当 MCP 核心能力；2026-07-28 Tasks 在扩展框架下 | 跟随规范：Tasks 当扩展能力位，不写进最小 EP 必选 `capabilities[]` |
| P2-7 | Q26 计量可信度未决 | 多租户对外 Service：token 计量以 `provider_proxy` 或账单 API 为准；Harness 自报保持 `untrusted_external`，缺省不计入配额 |
| P2-8 | SPIFFE 未出现在重定义 | 见 §7：EP / Runtime 跨机 mTLS 的可选身份后端，不替换 Tenant 对象 |

---

## 5. 跨文档冲突（redefinition vs design / adr）

冲突表只列**现在就会让实现选边**的条目。redefinition README 已声明「ADR-0013 采纳前现行文档不因本文件夹改动」——这是状态事实，不是冲突本身。冲突是：采纳后若按 10 §2「逐篇处置」而不先改下列决策，v0.8.0 会自相矛盾。

| # | 现行（v0.7.0，accepted） | 重定义 | 处置建议 **[建议]** |
|---|---|---|---|
| C1 | ADR-0008 D5：仅 `local_read` 快路径；`external_read` 完整协议 | 01 F3 收窄为 mutation；04 又对所有调用 persist | **现行 ADR 赢**。重定义 01/04 必须改方案以继承 D5，而不是用 F3 新措辞覆盖 |
| C2 | ADR-0002 D2：MCP 守卫必经；harness 视 `fysund` 为唯一 MCP server | README / ADR-0013：废止该强制形态，EP 可选 | 采纳 ADR-0013 时 **supersede D2** 为「EP 契约」；conformance 按 EP 组合分列（已有方向）。不得在无 EP 时仍宣称 F3/F7/F9 |
| C3 | ADR-0012 D1：`fysun-mcp-guard` **进程内**、**不独立进程前置**；Q10 已关闭 | ADR-0013 重开「不前置」；04 独立可选进程 | 允许独立进程，但必须带 P0-3 不变量。保留「不嵌入 agentgateway」。10 §2 写「重开」不够，需要 ADR-0012 修订历史或新 ADR 条款 |
| C4 | ADR-0007 D5：`deliver_input` 三条路径 + 守卫系统工具 | 废止为 OS 规范 | 可废止；04 `os.observe` 替代成立。MCP-only 薄包装仍需**某一**投递实现，下放适配器文档，不要从负例里消失 |
| C5 | ADR-0007 / 13：conformance 按 Runtime 类型分列 | 04 §4.5 按 EP 组合分列 | 采纳 EP 分列；Runtime 类型改为「缺省 EP 组合」列，避免两张表互斥 |
| C6 | ADR-0009 实体/context 封闭表 | 06 §3 直接增实体与键 | 先改 ADR-0009，再实现 06 |
| C7 | design 00：Runtime 是 Resource；`fysun-mcp-guard` 在 `fysund` 内 | 00：Runtime 是资产 `kind`；guard 独立进程 | Runtime 分类按 P0-4；组件表按 C3 |
| C8 | design 07 §3：OS 渲染 context_view 进 harness | 04：只给结构化视图，Harness 渲染 | 与 P11 一致，采纳重定义；负例已有（04 §8.2） |
| C9 | design 12：MVP 单主体；Q5/Q11 开放 | Tenant 核心；Trust 改 R4 必做 | 采纳多主体，但 R1 仍允许单 Tenant。Trust ADR 必须在任何跨基座 `published` 之前，不能只写「R4 必做」而无字段 |
| C10 | sources.md：MCP 2025-11-25 | 07 未钉版本；现行公开规范 2026-07-28 | 更新来源登记（属 `docs/research` / 后续 07，不在本次修改范围则列入 R0 工作项） |
| C11 | ADR-0003 / 0009：Cedar `approve`/`modify` 靠注解约定 | 06 要求 `fysund` 与 EP 同输入同 outcome | 需要同一 `policy_version` 字节相同的 bundle；EP 禁止本地补丁。负例 06 §10.1 保留 |
| C12 | F5：经 `fysund` 代理的 Provider | F5：只向 qualified EP 解析；`provider_proxy` 可选 | 采纳新 F5。多租户计量若要用可信 token，把 `provider_proxy` 当 **Plan 级资格条件**，不是全局 F5 前提（Q26） |

10 §2 把 ADR-0002 / 0007 / 0012 标「修订 / 重开」是正确的流程提示，但**未列出 ADR-0008**。这是遗漏：**[事实]** 04 继承了 05 ABI 与 07 资格化，没有继承 06/ADR-0008 的快路径分档。R0 必须把 ADR-0008 列入「保留且 04 必须引用」。

---

## 6. 前沿对照（截至 2026-09-12）

列「采纳 / 谨慎 / 延后 / 拒绝」。不把博客业绩数字当本仓库事实。

| 主题 | 2026-09-12 可核对状态 | 与草案关系 | 决策 |
|---|---|---|---|
| **MCP 2026-07-28** | 无协议会话的 stateless 核心；Authorization 收紧；Sampling / Roots / Logging 弃用，≥12 个月窗口；Tasks / Apps 走扩展 | 07 仍按 2025-11 工具面列举 | **采纳** Authorization、stateless HTTP、扩展位；**拒绝**新实现默认 Sampling；**谨慎** Tasks 必选化 |
| **MCP OAuth 2.1** | HTTP：server = resource server；MUST RFC 9728；client MUST resource indicator（RFC 8707）；`iss`（RFC 9207）；DCR 弃用、CIMD SHOULD | 05 只写「OAuth」 | **采纳**为 gateway MCP facade 与下游 HTTP MCP 的必做；stdio 不套该规范 |
| **A2A 1.0** | 2026-03-12；Card 结构破坏性变更；`A2A-Version` 头 | 05 用 Card 但未钉 1.0 字段 | **采纳** 1.0 Card 与 TaskState 映射（已有方向） |
| **Cedar / AgentCore Policy** | AgentCore Policy GA（2026-03 公开材料）；Gateway 拦截 MCP 工具调用；Cedar default-deny、forbid-wins | 已选 Cedar 子集；EP 下发快照 | **采纳**「工具调用点 PEP + Cedar」；**拒绝**照搬 AgentCore 产品边界（FysunOS 有四值 outcome、Approval、Effect 协议） |
| **OTel GenAI** | 2026-06 迁出主仓；独立仓 README schema URL = TODO；2026-08 公开统计 `gen_ai.*` 均 Development | 07「钉 commit」 | **谨慎采纳**导出；**拒绝**当稳定语义强制进账本 |
| **SPIFFE** | SVID + Workload API 仍是跨机工作负载身份的规范底座 | 重定义未提；08 跨机 gRPC mTLS | **延后**：作为 EP/Runtime 跨机身份**后端**，不替换 Tenant / Principal 对象。**拒绝**写成资格化前提 |
| **Prompt caching** | Anthropic `cache_control` / 自动缓存；OpenAI 前缀缓存 + 新模型 breakpoint；工具定义变化使整段前缀失效 | 草案无前缀稳定性 | **采纳**为 Harness 装配约束（P1-4）；基座不实现缓存层 |
| **Temporal durable execution** | Workflow = 耐久编排；LLM/工具放 Activity；Agent Harness 早期项目给审批与可恢复 loop | Activity/Execution + 恢复模型已有 | **谨慎**：借鉴「工具副作用入可重放历史、审批可耐久等待」。**拒绝**把 Temporal 换成账本，或把 loop 搬进 `fysund` |
| **DSPy / GEPA** | GEPA = 离线/预算内的 reflective 程序演化；输入 traces + metric feedback | 09 无优化器槽 | **采纳为离线插件**；**拒绝**进请求热路径 |
| **SWE-bench 污染** | OpenAI：Verified 不再衡量前沿编码，存在金补丁复现与测试缺陷；独立研究显示与更新基准的 localization 差距 | 若用 Verified 当 evidence_gate | **拒绝**用 SWE-bench Verified 作为对外质量证据。回归集用自有 Execution + 租户基准；编码类可考虑 SWE-bench Pro / SWE-rebench，须单独 ADR |
| **Agent Skills** | 仍是 `skill` 目录事实标准 | 02 §3.2 已用 | **采纳**，不自造 skill 包格式 |
| **OCI artifact** | 分发身份 = digest | 02 §4 / 07 §3 | **采纳** |
| **Arrow Flight / 零拷贝 IPC** | 高性能数据面技术 | 无 | **拒绝**现阶段写入内核（过早优化） |
| **CaMeL 级信息流** | v0.7.0 Q13 仍开放 | 06 仍继承标签而非值级 taint | **延后**，与 Q13 一致 |

---

## 7. 专项：性能、治理、自进化、AaaS

### 7.1 性能

目标：在**不削弱 ADR-0008** 的前提下，让未来 agent 的 TTFT/TPOT 不被基座写成同步日志系统。

可执行机制（全部 **[建议]**）：

1. **快路径分档** = P0-1。这是最大的结构性能点，也是安全点。不要用「把 external_read 免盘」换延迟。
2. **确定性前缀** = P1-4。让 provider 缓存有可能命中；命中率先测。
3. **并行只读分发**：同一 turn 内多个 `local_read`（及已 `authorized` 且 Policy `allow` 的 `external_read`）允许并发；任何 mutation / `credential_use` / `physical_action` 与未决 Approval 保持串行。
4. **工具面裁剪**：执行点 `tools/list` 过滤已经是 F7/F9 机制；同时它是上下文体积与缓存的前提。不要为「让模型看见被拒工具」而放弃不可区分性。
5. **`fysund` 继续禁止出现在 `tools/call` / 模型转发链**（已有负例）。
6. **控制面减 RPC**：长 Activity 用 lease，避免每条用户消息都 `activity.create`。

度量（10 §4 已要求「基线先测、阈值后定」——保持）：

- `audit/observed` / (`audit/observed` + `action-intent/recorded`)
- EP `tools/call` pre 的 p50/p95（按 `effect_category` 分位）
- provider `cache_read` 与 `cache_write`（若 Harness 回报）
- `ep.report` 延迟与 `ep.report_rejected` 率
- `compose.resolve` 时延

**不要**在无测量时写「快路径覆盖 70%」或「缓存节省 60%」。

### 7.2 治理

应保持：Cedar 下发、`approve` 回中心、F9、供应链 digest/签名/隔离、标签。

必须补的可判定规则：

1. **PEP 覆盖门**（P0-2）。
2. **Policy bundle 字节相同**（C11）：`ep.policy_pull` 返回的 Cedar + Grant 投影 + 工具面 digest 记入 `ep/policy-pulled`；评估输入不得含 EP 自造键。
3. **OAuth audience**：MCP HTTP 的 access token 必须绑定该 Endpoint / 下游 server 的 resource indicator；错 audience → 拒绝（负例）。
4. **Sampling 缺省 deny**（P1-2）。
5. **租户硬边界**已有负例，保留；R3 不要验收 `shared` 池化（P1-8）。
6. **T12–T15**（06 §8）方向对，补一条 T13 变体：EP 把 `external_read` 标成 `local_read` 以走快路径 → 失败（类别合成仍三源合一，EP 不能单方降级）。

### 7.3 自进化

v0.7.0 10 演进已经把「改进策略当普通 Capability、不进内核」写对。重定义 09 继承了 Adjustment / 证据门 / 不设评分对象，然后用「新 Lock 从零累积」把资产层学习切断。

目标机制：

```text
Immutable Zone（不可被优化器或 Adjustment 静默改写）
  Grant 边界 · effect_category 合成结果 · Policy 引用 · F9 工具面收窄
  扩权 / 新外部动作 / 预算上调 / 设备安全 → 必须 Approval

Mutable Zone（可进化）
  prompt 文本 · loop.params（⊆ harness.loop_schema）· skill 资源 · 模型绑定顺序

离线编译
  optimizer 插件（DSPy/GEPA/自研）读 traces + 回归集
  → 新资产版本（draft）
  → evidence_gate（Verification 全 pass、无 not-run）
  → 新 Overlay 或新 Base Lock
  → canary 累积 Lock 级证据 → stable
```

证据继承规则：

- 资产层：相同 `(kind, digest)` → 继承该 digest 的 evidence 摘要。
- 组合层：Agent Definition 版本变了 → 组合层证据从零，但资产层仍可继承。
- Lock 层：分层后，Base 不变、Overlay 变 → Base 证据保留，Overlay 从零。
- 资格化：Harness / Runtime / EP id 变 → 不继承（保持 09 §9 负例 1 的原意）。

**拒绝**：Execution 内反射改 prompt；优化器改 `declares.effects[]` 类别；把 SWE-bench Verified 分数写成 Capability 证据。

### 7.4 Agent-as-a-Service

05 的角色分离（发布者 / 运营者 / 租户 / 审核者）和「不做计费、不做聊天 UI」应保留。

对外发布的最小合格服务：

1. 钉住 Lock（已有）。
2. **PEP 覆盖**非 `local_read` 工具面（P0-2）。
3. A2A 1.0 Card 由 `fysund` 签名，字段与 Lock `capabilities[]` 一致（已有负例，需补 1.0 形状）。
4. Plan → 根 Grant 衰减（已有）。
5. 计量事件可导出；Harness 自报 token 缺省不计配额（P2-7）。
6. Trust 对象：跨基座调用前必须存在（05 §11 已把 Q11 收成 R4 必做）。**[建议]** 在任何 `tenant_visibility = public` 且允许跨基座之前落地 Trust ADR，而不是「R3 先公开、R4 再补信任」。

联邦：多 `fysund` 不共享账本（已有）正确。不要为了 AaaS 再引入中心市场对象。

---

## 8. 目标架构（方案，不修改原文）

### 8.1 四层实体

把「一切都是资产」收成四层。P9 改为：**一切可分发、可被 Definition 引用的制品是资产**；运行实例与服务实例不是资产。

| 层 | 对象 | 是什么 | 不是什么 |
|---|---|---|---|
| **L1 契约** | Capability | 行为 / 质量 / 边界 / 验证 | 不是 OCI 制品；无 `implementation` 字段（03 已删，保持） |
| **L2 制品** | Asset `kind` ∈ `prompt · skill · tool · mcp_server · harness · loop · model · memory_provider · knowledge` | 可 pack/push、有 digest、可被 Definition 引用 | 不含 `runtime`；不含 EP |
| **L3 组合** | Agent Definition（`kind = agent`）+ **分层 Lock** | 引用 + 覆盖 + 变体 → 可重放的 digest 集 | 不绑定具体 Node/进程 |
| **L4 运行与服务** | Runtime、EP、Node、Activity/Execution、Service、Tenant、Plan | 资格化实例、PEP、生命周期、对外入口 | 不进资产仓库；Meter 为事件投影 |

`agent` 同时是 L2 制品（可分发 Definition）与 L3 入口，这是允许的唯一双栖。

### 8.2 分层 Lock

```text
Lock
├── base        解析时刻的资产 digest 全集（不可变）
├── overlay     this_activity / this_execution 的覆盖（lock_overrides_digest）
├── bindings    slots、Prompt 变量缺省、tools_allow
├── effects     合成后的 tool_effects[]（只能收窄）
└── zone
    ├── immutable  Grant 相关边界、effect_category、Policy refs、reject_native_tools
    └── mutable    prompt/loop.params/model.fallback 顺序等
```

规则：

- ServiceVersion 钉 `base.lock_digest`（+ 允许的 overlay schema 版本），不钉浮动范围（03 已有）。
- Overlay 不能降低 `effect_category`、不能启用 Definition 已禁用的工具（03 §8 负例 6 保持）。
- 证据：对 `base.assets[digest]` 差分继承；overlay 与新 digest 从零。
- `compose.diff` 从「人工提示」升级为「系统推断的继承集合」，但继承集合必须可审计（列出哪些 evidence id 被复制引用，而不是改写）。

### 8.3 不变量

实施与 conformance 用。编号 I1 起：

| ID | 不变量 | 主要负例 |
|---|---|---|
| I1 | 权威账本只在 `fysund` | EP 写出 `grant/issued` / `approval/decided` / `effect/verified` |
| I2 | `fysund` 不在工具或模型转发链 | 进程出现在 `tools/call` 路径 |
| I3 | 仅 `local_read` 可无 Effect；必须有 `audit/observed` | `local_read` 无该事件 |
| I4 | `external_read` 及以上必须有 ActionIntent | `external_read` 只有 `audit/observed` |
| I5 | 未分类工具按 `external_mutation` | `readOnlyHint` 导致快路径 |
| I6 | 非 `local_read` 工具面 ⊆ qualified EP 覆盖面，才允许 `service/published` | 无 PEP 的 mutation 服务对外 |
| I7 | 有效权限 = Grant ∩ Policy ∩ 工具面，Execution 内单调收窄 | Adjustment 启用禁用工具且无 Approval |
| I8 | Secret 只向 qualified EP 按 Grant 解析 | 未资格化 EP `resolve_secret` 成功 |
| I9 | 同 `policy_version` + 同输入 → `fysund` 与 EP 的 `outcome` 相同 | 两处结果不同 |
| I10 | 资产内容 digest 与 Lock 不一致不得 `start_execution` | 已有 07 负例 |
| I11 | Immutable Zone 不能被优化器或 overlay 静默改写 | GEPA 输出直接降级 effect_category |
| I12 | `runtime` 不是可分发资产；Definition 不引用 Runtime id | `kind=runtime` 被 `fysun asset push` 成功且进入 Lock.assets |

### 8.4 可删 / 合并 / 延后

| 动作 | 对象 | 理由 |
|---|---|---|
| **从资产 `kind` 删除** | `runtime` | P0-4 |
| **不要做成资产 `kind`** | EP、Meter、Trust | 运行/投影/治理对象 |
| **合并（实现，不必先改词）** | `tool` 与 `mcp_server.tools_snapshot` | 避免双重目录；对外仍可搜单工具 |
| **延后 R1 实现** | `knowledge`、`memory_provider` 的完整 spec、`shared` 池化、跨基座 Trust、`external_gateway`、边缘 `fysund-edge` | 不阻塞首个闭环 |
| **延后产品化为 kind** | optimizer / GEPA | 先当 CLI 插件 |
| **保留但降级篇幅** | 06 MCP 守卫细节 | 变「`mcp_guard` EP 参考规范」，从属 04，与 10 §2 一致 |
| **不要删** | Loop 资产、Harness 资产、Lock、Capability 契约、Approval、Cedar | 定位的载荷 |

---

## 9. 建议文档树（仅方案）

**不在本次创建这些文件。** 若采纳 ADR-0013，建议 v0.8.0 的 `docs/design` 按此收口，避免再维持「redefinition 与 design 双树」过久。

```text
docs/adr/
  0013-repositioning.md                 # 本草案定稿
  0002/0007/0009/0012 修订历史          # supersede 守卫必经、投递路径、实体扩展、独立 EP
  （0008 保留，04 必须引用 D5）

docs/design/                            # 仍是实现规范；redefinition/ 在定稿后归档或合并
  00-terminology.md                     # 唯一术语基准；kind 无 runtime
  01-definition-and-principles.md       # 定义句、P1–P11、F1–F9（F3 按 ADR-0008 分档）
  02-object-model.md                    # 四层实体 + 分层 Lock + 服务对象
  03-architecture.md                    # 三平面拓扑、账本、恢复、不变量 I1–I12
  04-capability-and-registry.md         # 制品 Registry + Capability API
  05-runtime-abi.md                     # 最小面；结构化视图；os.observe
  06-enforcement-and-policy.md          # EP 契约 + Cedar 下发 + ADR-0008 分档 + OAuth
  07-runtime-onboarding.md              # Runtime 对象（非资产）资格化
  08-dsh-adapter.md                     # 参考 harness_adapter
  09-aaas.md                            # Service/Tenant/Plan；发布门含 PEP
  10-evolution.md                       # 分层证据 + 离线优化器插件 + Immutable Zone
  11-edge-and-offline-ledger.md         # 继承
  12-mvp-plan.md                        # 按下面 R0–R5
  13-threat-model.md                    # T12–T15 + Sampling/OAuth；conformance 按 EP

docs/redefinition/                      # 定稿后标 superseded by v0.8.0，不再双源
```

过渡期（ADR-0013 已采纳、design 尚未改完）：`docs/redefinition` 保持只读快照，实现以 ADR-0013 + 本方案的 P0 修正为准，避免第三份「口头真理」。

---

## 10. 分阶段路线与可判定验收 / 负例

原则：第一个闭环仍然要小、真实、可验证（10 §3 原则保留）。调整的是**门**与**对象面宽度**，不是再写一份空计划。

| 阶段 | 交付 | 验收（可判定） | 关键负例 |
|---|---|---|---|
| **R0 定位** | ADR-0013 采纳；kind 去掉 `runtime`；04 方案继承 ADR-0008 D5；07 钉 MCP 2026-07-28；ADR-0009 增补实体表 | grep：资产 kind 表无 `runtime`；04/EP 契约出现 `local_read`/`external_read` 分档；MCP 页写明 Sampling 缺省拒绝 | 设计把基座写成实现 loop（ADR-0013 负例 6）；把 `external_read` 写成无持久化 |
| **R1 登记** | 可分发制品 subset：`prompt/skill/mcp_server/harness/loop/model/agent`；pack/push/register/publish；Runtime **对象** register/qualify；Policy.simulate | 02 现有负例 1–7 中与 kind 相关者按新枚举改写后通过；`runtime/qualified` 带 `harness_asset_id`；digest 重算 | `fysun asset push` 接受 `kind=runtime`；缺 `native_effect_tools` 字段仍 qualify |
| **R2 组合与首运行** | Capability；`compose.resolve`；分层 Lock（允许先实现 base，overlay API 可薄）；一个 Harness 最小 ABI；Activity/Execution 恢复；**最小 qualified EP**（adapter 或 mcp_guard）覆盖该闭环的非只读工具 | 03 负例 1–5；首个闭环：登记 → 定义 → 解析 → 匹配 → 运行 → 独立验证 → 崩溃恢复；I3/I4 在该 EP 上成立 | Lock 含版本范围；`external_read` 无 ActionIntent；`fysund` 在转发链 |
| **R2b 性能（可与 R2 并行，不挡 R3 目录）** | local_read 快路径；前缀槽声明；并行只读；`ep.report` 微批 | 度量管道存在（无强制阈值）；N-03/N-04 | `local_read` 无 `audit/observed`；`external_read` 无 ActionIntent |
| **R3 服务目录** | gateway；Tenant/Plan；**仅 `dedicated` 池化**；对内 search | 两租户用量/Memory 分离；未 PEP 覆盖的 mutation Lock **不能** `published` | 无 qualified EP 的 mutation Service 为 `published` 且可调用；跨租户读 Memory |
| **R4 治理闭合与对外发布** | 独立 mcp_guard；OAuth audience；Sampling 防火墙；policy_stale；Approval 路由；Trust ADR；conformance 按 EP | 04/06 负例；错 audience 拒绝；sampling 缺省 deny；I6 满足后才允许 public Endpoint | 未资格化 EP 回报被接受；gateway 本地 `approval/decided` |
| **R5 演进与生态** | 证据差分继承；离线 optimizer 插件；`fysun-dsh` 全控制点；第二 Harness；边缘 | 09 负例 1 **改写为**：换 Harness 不得继承资格化证据，**但**未改 digest 的 skill 证据必须仍可查询；优化器改 Immutable Zone → 失败 | 新 Lock 丢掉未改 digest 的资产证据；GEPA 进入 `tools/call` 路径 |

与草案 10 §3 的显式差异：

- R3 **不再**等于对外 `published`。对外发布并入 R4（或 R3 仅限纯 `local_read` + PEP）。
- R2 就必须有**覆盖该闭环工具面**的 qualified EP，不能把 F3 全部推到 R4。
- R1 不做 11 种 kind 的全 schema。

---

## 11. 开放问题

在 v0.7.0 Q1–Q21 与重定义 Q22–Q29 之上，按本方案重排。不关闭 owner 尚未决策的项，只标注建议姿态。

### 11.1 建议关闭或收口（仍须 owner 点头）

| ID | 题 | 建议姿态 |
|---|---|---|
| Q5 | 多主体 | 保持关闭为 Tenant 核心；R1 单 Tenant 合法 |
| Q11 | Trust | 不保持「远期」；R4 / 跨基座发布前置；需独立 ADR |
| Q14 | Provider 路由 | 保持重定义：不做路由；`fallback` 是绑定顺序 |
| Q16 | DSH headless 原生工具 | 保持：R2 前查证，不影响规则 |
| Q21 | `approval_result_ttl` | 保持关闭为 OS 投递管道问题；语义并入 Q25 |
| 10 §7.3 | R3 vs R4 | **建议改默认**：先 PEP 覆盖，再 `published`（见 P0-2） |
| 10 §7.4 | 是否自研 mcp_guard | 建议保留第一方参考 EP；允许 `external_gateway` 后接，不嵌入 |

### 11.2 保留

Q1 传输、Q2 Secret Store、Q3 Web UI、Q4 judge 校准集、Q6 许可证、Q12 多节点、Q13 CaMeL、Q15 `gen_ai.*`（按 P2-1 收窄为「并存 + 改名层」）、Q17 `modify`、Q18 `obligations[]`、Q19 `on_expiry=escalate`、Q20 `downstream_outcome` 时间窗。

### 11.3 重定义已新增、本方案附修正

| ID | 题 | 本方案修正 |
|---|---|---|
| Q22 | manifest 规范化与签名 | 保持；canonical digest 算法 R1 必须先定，否则 Lock 不可重放 |
| Q23 | loop_schema 通用子集 | 建议：跨 Harness **最小**字段仅 `max_steps` + `stop_conditions[]`；其余各 Harness 声明。不要在 R1 做统一 loop 语言 |
| Q24 | shared 隔离证明 | R3 不做 `shared` |
| Q25 | EP WAL 与回报一致性 | 升级为 P0-3；R2 就要有恢复协议，不能等 R4 |
| Q26 | 计量可信度 | 对外多租户：自报缺省不计配额 |
| Q27 | Trust 字段 | R4 前出 ADR |
| Q28 | gateway A2A 能力位 | 补 A2A 1.0 Card schema |
| Q29 | BYO 边界 | 效果类别不可降级的校验在 `fysund`（compose / adjustment）；EP 再强制一次（defense in depth） |

### 11.4 本报告新增

| ID | 题 | 说明 |
|---|---|---|
| Q30 | Overlay 证据的校准 | 差分继承是否要求 `verifier_kind=deterministic` 才自动继承，`model_judge` 必须重跑 |
| Q31 | MCP 2026-07-28 无会话下，下游 MCP 会话持有如何迁到 EP | 原守卫「持有下游会话」依赖旧会话模型；stateless 后应改为 per-request 凭据 + 可选应用层任务 id |
| Q32 | optimizer 插件的权限 | 是否允许改 `loop.params` 以外的 Harness 绑定；缺省否 |
| Q33 | `tool` 是否保留独立 kind | 见 §8.4；R1 可只实现 mcp_server |

---

## 12. 建议给 owner 的决策顺序

不在本次代为决策。若进入 v0.8.0，建议按此顺序签字，避免实现期再分叉：

1. 是否采纳 ADR-0013 的定位（能力管理 + Runtime 控制 + AaaS 目标）。
2. 是否接受 **部分重构** 四件套：四层实体（`runtime` 退出 kind）、分层 Lock、04 继承 ADR-0008 D5、`published` 依赖 qualified EP。
3. R2 首个 Harness：DSH headless 薄包装 vs `fysun-dsh`（10 §7.2 仍开放）。
4. `fysun-mcp-guard` 独立进程 + F1 不变量（P0-3）是否接受。
5. Personal 是否绕过 gateway（可绕过）vs 是否绕过 PEP（不可绕过）。

---

## 13. 来源（≥15，可访问 URL + 一句话）

下列为 2026-09-12 核验所用。内部路径不是 URL，但作为已采纳决策的对照列出。

1. https://modelcontextprotocol.io/specification/2026-07-28/client/sampling — MCP 2026-07-28 Sampling 弃用页：SEP-2577；新实现 SHOULD NOT 采用；≥12 个月后才有资格移除。
2. https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization — MCP Authorization：HTTP 传输以 OAuth 2.1 为基；RFC 9728 MUST；DCR 弃用、CIMD SHOULD。
3. https://blog.modelcontextprotocol.io/posts/2026-07-28/ — MCP 2026-07-28 规范发布说明：stateless 核心、授权硬化、Roots/Sampling/Logging 弃用与十二个月窗口。
4. https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/ — 同修订 RC 说明：弃用对照表（Sampling → 直连 LLM API）。
5. https://a2a-protocol.org/v1.0.0/specification/ — A2A Protocol 1.0.0 规范（Latest Released Version `1.0.0`）。
6. https://github.com/a2aproject/A2A/releases/tag/v1.0.0 — A2A v1.0.0 发布（2026-03-12）与破坏性变更列表入口。
7. https://a2a-protocol.org/latest/whats-new-v1/ — A2A 1.0 相对 0.3 的 Card 字段迁移（`supportedInterfaces`、`extendedAgentCard` 等）。
8. https://github.com/open-telemetry/semantic-conventions-genai — OTel GenAI 语义约定独立仓库；README 中 Schema URL 仍为 TODO。
9. https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-understanding-cedar.html — AgentCore Policy 的 Cedar 评估语义（default-deny、forbid-wins）。
10. https://aws.amazon.com/blogs/security/why-policy-in-amazon-bedrock-agentcore-chose-cedar-for-securing-agentic-workflows/ — AWS 说明为何在 AgentCore Gateway 用 Cedar 拦截工具调用。
11. https://spiffe.io/docs/latest/spiffe-specs/spiffe/ — SPIFFE 规范总览：SPIFFE ID、SVID、Workload API。
12. https://platform.claude.com/docs/en/build-with-claude/prompt-caching — Anthropic prompt caching：前缀缓存、breakpoint、TTL。
13. https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching — 工具定义与 `defer_loading` 如何影响缓存前缀。
14. https://developers.openai.com/api/docs/guides/prompt-caching — OpenAI prompt caching：相同前缀复用、breakpoint / `prompt_cache_key`。
15. https://temporal.io/blog/temporal-agent-harness-durable-agent-infrastructure — Temporal Agent Harness：agent 作为耐久 Workflow、工具审批与可恢复执行（早期项目，不当产品依赖）。
16. https://dspy.ai/api/optimizers/GEPA/overview/ — DSPy GEPA：基于 traces 与文字反馈的 reflective 优化器，按 metric 预算运行。
17. https://arxiv.org/abs/2507.19457 — GEPA 论文（Agrawal et al., 2025）：reflective prompt evolution，不是请求内编译器。
18. https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/ — OpenAI：SWE-bench Verified 存在污染与测试缺陷，不再用于衡量前沿编码能力。
19. https://arxiv.org/abs/2512.10218 — 关于 SWE-Bench-Verified 更像测记忆而非能力的 localization 对比研究。
20. https://workos.com/blog/mcp-2026-spec-agent-authentication — 对 MCP 2026-07-28 授权变化的工业解读（OAuth 2.1 resource server、RFC 8707）；作对照，规范条文以第 2 条一手页为准。
21. 本仓库 `docs/adr/0008-effect-category-and-read-only-determination.md` — 已采纳：`local_read` 快路径与 `external_read` 完整协议（H-2）。
22. 本仓库 `docs/adr/0012-mcp-guard-self-implementation.md` — 已采纳：守卫自实现、不嵌入 agentgateway；「不前置」被 ADR-0013 草案重开，须显式修订。
23. 本仓库 `docs/redefinition/adr-0013-draft-repositioning.md` — 被审定位决策草案。
24. 本仓库 `docs/research/sources.md` — v0.7.0 来源登记（MCP 条目仍钉 2025-11-25，与现行公开规范差一个修订）。
25. https://john-hodge.com/blog/opentelemetry-genai-semantic-conventions/ — 2026-07 对 GenAI 约定仍为 Development、主仓 v1.42.0 迁出的状态综述（二手时间戳，稳定性结论与独立仓库 README 一致）。

---

## 14. 收束

重定义把 FysunOS 从「内联 MCP 守卫的操作系统」拉回「能力资产 + Runtime 控制」基座，这个方向应采纳。现草案的主要风险不是文风或篇幅，而是四件已经可在原文核对的结构性缺口：

1. 04 没有继承 ADR-0008 D5，F3 措辞与「一切 persist」互相覆盖；
2. `runtime` 既当可分发资产又当 Node 实例；
3. 新 Lock 从零累积证据，切断自进化；
4. R3 允许在 PEP 闭合前把 mutation 服务标成 `published`。

推荐路径是 **部分重构**：四层实体、分层 Lock、04 按 D5 分档（**不**把 `external_read` 免持久化）、对外 AaaS 以 qualified EP 为门、MCP 钉 2026-07-28（Sampling 缺省拒绝、OAuth 2.1 为现行重点）、自进化作为带 Immutable Zone 的离线插件。保留 F1–F9、Cedar、账本、EP 与标准优先。

原文保持不动。本文件只作为采纳 ADR-0013 之后的修订输入。
