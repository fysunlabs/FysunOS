# FysunOS `docs/redefinition` 全面复检与改进方案报告

- **性质**：独立架构审查与演进建议。本报告不修改 `docs/redefinition/` 原文，也不触发 ADR-0013 采纳。
- **审查对象**：`docs/redefinition/` 全套提案草案（v0.8.0-draft，2026-09-11），含 `README.md`、`00`–`10`、`adr-0013-draft-repositioning.md`。
- **对照基准**：现行 `docs/design/**`、`docs/adr/**`（尤其 ADR-0001–0012、ADR-0008）、`docs/research/20260908-v0.7.0-refactor-blueprint.md`、`docs/research/sources.md`。
- **外部截止日**：2026-09-12。优先一手材料（规范原文、官方文档、同行评审/预印本、头部工业实践）。
- **审查日期**：2026-09-12。
- **方法**：全文精读 + 跨文档冲突核对 + 红队评估 + 与 2025–2026 协议/工业实践对照。
- **先前草稿**：工作区曾出现同路径草稿。本稿在核验原文与外部规范后**整篇重写**，纠正了若干过时或过强断言（尤其 MCP Sampling 生命周期、OTel `gen_ai.*` 稳定性、以及把 `external_read` 放进无持久化快路径会与已采纳 ADR-0008 冲突）。

## 0. 证据分级与陈述约定

| 标记 | 含义 |
|---|---|
| **[事实]** | 可在指定文件/URL 直接核对 |
| **[推断]** | 由事实推出，可能有其他合理解释 |
| **[建议]** | 本报告的改进意见，非原文承诺 |
| **[待验证]** | 缺少实现、基准或规范钉版，不能当既定事实 |

定量收益（延迟、成本、打包开销）一律标 **[待验证]**，不把营销口径或未测倍数写成验收指标。

---

## 1. 执行摘要

### 1.1 总判断

**[建议]** 采纳 ADR-0013 的方向，但对对象模型与若干契约做**部分重构**，不要全盘推倒，也不要原样冻结 11 种资产 + 单一 Lock 证据清零。

v0.8.0-draft 把 v0.7.0 的核心错误纠正了：`fysund` 退出数据面热路径、Harness/Loop 进入对象模型、Tenant 进入核心、Service 平面补齐 AaaS。这些应保留。真正卡住「精简、性能、自进化、工业级 AaaS」的，不是定位句，而是：

1. 资产本体论把**实例**和**制品**混在同一封闭枚举里；
2. 单一 `lock_digest` 把「换一句 prompt」做成「证据从零」；
3. 运行时对现代 Agent 热路径（前缀缓存、并行工具、只读快路径）几乎无约束，且 04 与已采纳 ADR-0008 的快路径定义未对齐；
4. 09 的「演进」仍是配置管理，不是带安全包络的可验证自进化；
5. 服务平面对象齐全，但身份/MCP OAuth、计量可信度、长任务零成本挂起、公开 AaaS 与 F3 闭合的先后顺序仍有缺口。

### 1.2 最关键的八条结论

1. **保留** D1–D8 的骨架：能力管理 + Runtime 控制 + AaaS；PDP 在中心、PEP 在执行点；F1–F9；标准优先（MCP / A2A / Agent Skills / OCI / Cedar / OTel）。**[事实]** ADR-0013 D1–D8。
2. **部分重构资产模型**：11 种 `kind` 不必砍到「只剩 6 个名字」，但必须分成 **制品 / 连接件 / 配置档 / 运行实例** 四层；`runtime` 退出资产枚举。**[建议]**
3. **分层 Lock + 差分证据继承**：契约层/基底层变更不可继承；仅配置档变更可折损继承。否则自进化在生产不可用。**[建议]** 与 09 §2「新 Lock 从零累积」对冲。
4. **性能最小集**：把 ADR-0008 D5 的 `local_read` 快路径写进 04；强制 Harness 的确定性前缀槽；并行只读分发。不要把 `external_read` 放进无持久化路径。**[建议]** 对齐已采纳 ADR。
5. **自进化必须有 Immutable Zone**：允许离线编译 prompt/few-shot/loop 常数；禁止自主扩权、降级 effect、突破配额。DSPy/GEPA 是**编译器插件**，不是内核。**[建议]**
6. **AaaS 先闭合 PEP 再对外**：10 §7 第 3 项「先服务平面」对内测可接受，对多租户公开调用顺序危险。**[推断]**
7. **补协议深水区，但钉对版本**：MCP Authorization（OAuth 2.1）是 2026-07-28 现行必做；Sampling 在该修订中**已弃用**，新实现默认拒绝，仅对旧 server 做兼容防火墙。**[事实]** MCP 规范。
8. **不要把未成熟层写进内核**：Arrow Flight/零拷贝 IPC、SPIFFE 全面替换 Tenant token、DSPy 内生循环、强制 `gen_ai.*` 稳定语义——分别属于过早优化、可选身份后端、离线优化器、Development 约定。**[建议]**

### 1.3 是否建议重构

| 范围 | 建议 |
|---|---|
| 定位句、三平面、F1–F9、EP 模型 | **不重构**，定稿 |
| 资产种类、Lock、证据继承、Runtime 归属 | **部分重构**（对象分层，不改安全底线） |
| 04 EP 热路径、前缀槽、checkpoint 冻结 | **契约增补** |
| 09 演进 | **扩展为双轨**：人工 Adjustment + 受限离线进化 |
| 05/08 AaaS 身份与计量 | **增补**，不推翻 Tenant/Plan/Meter |
| 文档树 | **轻重组**（见 §11），采纳 ADR 后再改 `docs/redefinition` |

---

## 2. 设计地图（当前草案）

### 2.1 职责与平面

**[事实]** README「一句话定义」与 01 §2.1、00 §8、ADR-0013 D1 字面一致：FysunOS 是能力管理与 Runtime 控制的运行基座，目标是 AaaS。

```text
消费者 ──A2A/MCP/HTTP──▶ fysun-gateway (入口 EP)
                              │ invocation / 配额 / 计量
                              ▼
                         fysund (PDP · 账本 · Approval · 目录)
                              │ start_execution(Lock, 结构化视图)
                              ▼
                    Runtime = 资格化 Harness 实例
                              │ tools/call
                              ▼
              mcp_guard / hooks / harness_adapter / provider_proxy
                              │ 本地策略评估 + 回报
                              ▼
                         fysund 核对投影
```

**[事实]** 01 §3、04 §4、05 §5。`fysund` 明确不转发 `tools/call`、不代理模型、不渲染上下文（01 §8、04 §4.4）。

### 2.2 一级对象

**[事实]** 00 §2：Asset / Manifest / Capability / Agent Definition / Lock / Runtime / Activity·Execution / EP / Node / Service / ServiceVersion / Endpoint / Tenant / Plan / Meter，外加继承的 Grant / Policy / Approval / Verification / Adjustment 等。

资产封闭枚举 **11 种**（00 §3、02）：`prompt skill tool mcp_server harness loop model memory_provider knowledge runtime agent`。

### 2.3 热路径与控制路径

| 路径 | 现状 | 评价 |
|---|---|---|
| 模型生成 | Harness 直连或可选 `provider_proxy` | 控制面正确；无前缀槽约束 **[事实]** 02 §3.7、01 §8 |
| 工具调用 | EP 本地评估 + persist-before-dispatch | 04 §4.2 未复述 ADR-0008 只读例外 **[事实/冲突]** |
| 治理决策 | Policy 快照下发，`approve` 回中心 | 与工业 PEP/PDP 一致 **[事实]** ADR-0013 D5 |
| 服务调用 | gateway → activity.create_or_wake → Runtime | AaaS 主路径清楚 **[事实]** 05 §5 |
| 演进 | Adjustment / 新 Lock / 证据门 | 无自主变异与影子门 **[事实]** 09 |

---

## 3. 应保留的核心

1. **原点读回本义**：管理绑定 ≠ 坐在每次 RPC 上。**[事实]** 01 §1、10 §1。
2. **P8–P11**：不规定决策核心但登记它；可复用之物可管理；对外是服务；基座不在热路径。**[事实]** 01 §6。
3. **F1–F9 标注执行者**：F3/F7/F9 由资格化 EP 执行，conformance 按 EP 组合分列。**[事实]** 01 §7、04 §4.3–4.5。这是对 ADR-0007「脚注收窄」的架构化。
4. **Capability 仍是契约，实现改为 Agent Definition**。**[事实]** 03 §1。
5. **Cedar 子集 + 版本化下发 + 本地评估一致性负例**。**[事实]** 06 §3。与 2026 年 Bedrock AgentCore Policy / AWS Cedar 多 Agent 授权实践同向。
6. **供应链**：digest 互证、隔离/撤销级联、MCP `tools_snapshot` 漂移检测。**[事实]** 02 §4、06 §7。
7. **AaaS 对象**：ServiceVersion 钉 Lock、canary 可审计、Plan 派生根 Grant、发现 ≠ 授权。**[事实]** 05。
8. **标准优先、不自造包格式/策略语言/传输协议**。**[事实]** ADR-0013 D7、01 §8。
9. **大声失败、不设兼容等级、不静默降质**。**[事实]** 03 §4、01 §2.2。
10. **里程碑纪律**：本提案期全部 `specified`，不假装已实现。**[事实]** 10 §5。

---

## 4. 问题清单

严重度：P0 不改则实现/安全/演进失败或与已采纳 ADR 冲突；P1 会显著拖慢性能、AaaS 或维护；P2 优化与文档债。

### P0-1 资产本体论：`runtime` 同时是「实例」和「资产」

- **证据 [事实]** 02 §1 判定资产须「可复用、可分发、可脱离某次 Execution」；同节把 `runtime` 列为资产。02 §3.10 又写「通常不经资产仓库分发」。00 §1 明确「仍是资产种类 `runtime`」。ADR-0013 D2 把 11 种封闭枚举写死。
- **影响**：OCI 分发、生命周期（draft/published）、发现接口、撤销级联都要为「某 Node 上的安装描述」发明例外；Registry API 与 Runtime API 双入口（08 §1 把 `runtime.register` 放在 Registry 下）。
- **建议**：`RuntimeInstance` 留在运行平面；资产仓库只存 `harness`。登记事件可以保留，但 `kind` 枚举删除 `runtime`。负例：将 RuntimeInstance 当 OCI artifact `pack` → `asset.kind_mismatch`。

### P0-2 单一 Lock 导致证据饥饿，与自进化目标冲突

- **证据 [事实]** 03 §4 Lock 钉全部资产 digest；09 §2「换资产后旧证据留在旧 Lock；新 Lock 从零累积」；`compose.diff` 只作「人工判断参考，不作为系统推断」。09 §9 负例 1：换 Harness 后旧证据计入新 Lock → 失败。
- **影响**：prompt 微调、few-shot 替换、`max_steps` 调整都会清空 Lock 级证据。09 标题是演进，机制是 SCM。与「显著提升自进化」目标直接冲突。
- **建议**：Lock 改为分层 digest（契约 / 基底 / 配置档）。仅配置档变化时标记 `inherited` 并折损；契约或工具效果集变化必须全量回归。`compose.diff` 升级为系统判定输入，而不是旁注。

### P0-3 04 的 persist-before-dispatch 未继承 ADR-0008 D5

- **证据 [事实]** ADR-0008 D5（accepted）：**只有** `local_read` 走审计快路径，不持久化 ActionIntent/Effect，必须 `audit/observed`；`external_read` 走完整协议。01 §7 F3 写的是「外部 / 不可逆 mutation」。04 §4.1 `mcp_guard` 对 `tools/call` pre 写「ActionIntent 持久化」未分档；04 §4.2 写「EP 先本地持久化 ActionIntent / Effect 再分发」未引用 D5。
- **影响**：实现者会按 04 对每次 `tools/call` 同步 WAL；或反过来把 `external_read` 也做成无记录快路径，违反已采纳 ADR（注入溯源、外部配额、访问痕迹）。
- **建议**：04 §4.2 显式继承 D5。热路径分级：
  - `local_read`：内存 Cedar + Grant + 标签 → 并发分发 → `audit/observed` 异步微批；
  - `external_read` 及以上：完整 F3；
  - 单轮多工具：只读并发，遇到 mutation 设屏障。
- **明确拒绝**：把 `external_read` 放进无持久化快路径。**[建议]** 与 ADR-0008 H-2 一致。

### P0-4 公开 AaaS 与 F3 闭合顺序

- **证据 [事实]** 10 §3：R3 服务平面在 R4 执行点之前。10 §7 第 3 项：本文按「先服务平面」，理由是 R2 已有最小 EP。04 §4.5：无 EP 的 Runtime 对 F3/F7/F9 为 `not_covered`。
- **影响 [推断]** 若 R3 对外多租户而 PEP 仍是「某个适配器尽力」，配额、跨租户隔离、Plan.effect 强制会停在 gateway 入口，工具面仍可能 not_covered。
- **建议**：R3 对内可先做目录/Plan；**对外 `service/published` 的资格条件**改为「该 Lock 的非 `local_read` 工具面被至少一个 qualified EP 覆盖」。Personal 单租户允许绕过 gateway（10 §7 第 5 项可保留）。

### P0-5 MCP 反向能力面未入 EP 契约（含已弃用但窗口内仍存在的 Sampling）

- **证据 [事实]** 07 §1 MCP 用于 `tools/list` / `tools/call` / `resources` / `prompts` / `tasks` / `elicitation`，**未列** `sampling/*`、roots、OAuth。MCP 2025-11-25 定义 `sampling/createMessage`；**2026-07-28 起 Sampling 弃用**，新实现 SHOULD NOT 采用，至少保留 12 个月。MCP Authorization（OAuth 2.1、RFC 8707 resource、RFC 9728 PRM）是现行 HTTP 传输的核心。
- **影响**：旧 MCP server 仍可能向 Host 要模型；`includeContext` 可把宿主上下文交给 server。EP 若只过滤出站 `tools/call`，会漏入站采样与错误的 token audience。
- **建议**：
  - 新 Runtime/EP：**默认不声明 sampling capability**；
  - 兼容窗口：若必须开，采样请求视为 `untrusted_external` 候选，计入 Execution token 预算，剥离 `trusted_system` 与 Secret，Policy 缺省 deny；
  - HTTP MCP：gateway / mcp_guard 必须按 2026-07-28 做 audience 校验，不自造会话层 token。

### P1-1 `tool` 与 `mcp_server.tools_snapshot` 双源

- **证据 [事实]** 02 §3.3 与 §3.4；03 §2 同时有 `tools[]` 与 `mcp_servers[]`。
- **影响 [推断]** 同一逻辑工具两个 digest，漂移检测只覆盖 snapshot 一侧。
- **建议**：外部工具身份以 `mcp_server` + 工具名为准；独立 `tool` 仅用于非 MCP 的本地/HTTP 实现。禁止同一 `(server, tool)` 再登记孤立 `tool` 资产。

### P1-2 `loop` / 短 `prompt` / `model` 强制走与 Harness 同级的 OCI 生命周期

- **证据 [事实]** ADR-0013 负例 1：prompt/skill/tool/mcp_server/harness/loop 被引用必须有登记与 digest。02 §3.6：Loop 是配置不是代码。P9：无 digest 不能进 Definition。
- **影响**：几行 YAML 也要 pack/push/publish，组合噪声大；但完全内联又破坏追溯。
- **建议**：保留 `kind` 与 digest，允许 **Agent Definition 内联 Profile**（内容纳入 Definition digest）。独立资产变为可选复用，而不是强制。Q23（跨 Harness loop 最小子集）应在 R2 关闭。

### P1-3 无缓存友好装配约束

- **证据 [事实]** 01 §8 / P8：基座不规定 prompt 装配。02 §3.5 已有 `prompt_slots[]`。04 §2：`start_execution` 不带渲染文本。无前缀稳定性、无 cache 命中度量。
- **工业对照**：Anthropic Prompt Caching、OpenAI/Google prefix cache 均要求稳定前缀；长程 Agent 任务上前缀抖动会打穿缓存（arxiv:2601.06007）。
- **建议**：不写装配算法，但把 **Slot 顺序**定为 Harness 资格化输入：Slot0 系统身份与硬约束 → Slot1 工具 schema → Slot2 静态 skills → Slot3 租户/会话准静态 → Slot4 轨迹 → Slot5 易变值。`runtime.qualify` 增加 cache-alignment 可选能力位；度量 `cache_read_tokens / input_tokens`。不要规定 GPU 实现。

### P1-4 长周期 `input_required` 仍可能占住 Runtime 进程

- **证据 [事实]** 01 §8：不持有 Loop 状态，不承诺进程透明迁移。04 §2 `checkpoint` 可 `unsupported`。04 §3 继承 waiting / rearm。05 §5：`session_mode: per_call | per_context`。
- **工业对照**：Temporal 对 HITL 使用 durable wait，阻塞不占 worker 算力。
- **建议**：在 Runtime ABI 增加可选 `freeze/thaw`（或把 `checkpoint` 语义升级为「可杀死进程的快照」）。不支持则 `shared` 池化 + 长审批任务由 Policy 拒绝。不要承诺 KV cache/模型权重迁移。

### P1-5 自进化安全包络缺失

- **证据 [事实]** 09 §1 三种变化都是人/CI 驱动；§6「反馈驱动的调整仍是 Adjustment……不自动传播」。无 mutation operator、无影子集、无 Immutable Zone。仓库 `sources.md` 已登记自改进综述、AHE、SkillOps，但 09 未吸收。
- **建议**：进化引擎作为 **R5 可选子系统**：轨迹 → 归因 → 仅配置档候选 → 离线回归/影子 → canary。Cedar 禁止变异 `bounds.tool_effects`、`allowed_tools` 扩张、Plan 配额。优化器（DSPy/GEPA 等）产出的是新 prompt/loop Profile 资产，经现有 `service.rollout` 晋升。

### P1-6 多租户身份停留在「tenant token / mTLS / OAuth」枚举

- **证据 [事实]** 05 §2.1 Endpoint.auth；08 无 workload identity；Q2 Secret Store 后端仍开放。
- **建议**：身份分层——Tenant 人类/组织用 OIDC/OAuth；Execution/EP/Runtime 用短寿命工作负载身份（SPIFFE 为**可选后端**，不是内核依赖）。禁止长寿命静态 tenant token 作为多租户生产缺省。

### P1-7 计量可信度未决却已进入 Plan 预算

- **证据 [事实]** 05 §7：Harness 自报 tokens 标 `untrusted_external`；Q26 开放「`provider_proxy` 是否成为多租户资格前提」。
- **建议**：凡 Plan 含 token 配额或对外账单导出的 Service，`provider_proxy`（或等价可信计量 EP）为资格化前提。自报不得单独触发 `quota.exceeded` 放行，只能作对照。

### P1-8 MCP HTTP 授权模型未设计

- **证据 [事实]** 07 未提 OAuth 2.1 / PRM / resource indicator。05 Endpoint.auth 是网关入站，不是 MCP server 作为 resource server。
- **建议**：`mcp_server` 资产增 `auth: none | oauth2`；EP 负责发现 PRM、带 `resource` 参数、校验 audience。SecretRef 只持 refresh/client 凭证，不把下游 token 写进账本。

### P2 项（摘要）

| ID | 问题 | 建议 |
|---|---|---|
| P2-1 | 11 种 kind × 六组 API × 大量事件族，心智负担高 | 按四层分类展示，API 分组不变 |
| P2-2 | 06 §9：`gen_ai.*` 关系仍开放；sources.md 已记 Development | 钉 schema URL + 自有 `fysun.*`，准备改名层 |
| P2-3 | Q22 签名方案未选 | R1 选 Sigstore keyless 或策略可配 Cosign |
| P2-4 | 08 Runtime ABI 传输「R3 定」 | 本机 UDS/stdio JSON-RPC，跨机 gRPC+mTLS；不要前置 Arrow |
| P2-5 | Trust 对象 R4 必做但无 ADR | 与 Q27 合并，赶在跨基座 A2A 之前 |
| P2-6 | 07 列举 Goose/OpenClaw 等，conformance 全 `specified` | 接入表改为「候选」，避免文档承诺 |
| P2-7 | 定义句括号罗列 6 个能力名词 | 对外可用短定义「能力资产与 Runtime 控制的运行基座」；正式句保留不改写规则可开例外 |

---

## 5. 跨文档冲突

| 冲突 | 草案 | 现行/已采纳 | 处置建议 |
|---|---|---|---|
| 守卫必经之路 | ADR-0013 后果：重开 ADR-0002 §D2 | ADR-0002 仍有效 | 采纳 0013 时同步改 0002：必经的是 **EP 契约**，不是 `fysund` 内组件 |
| 投递路径 | 04 `deliver_input` 可选；08 仍有 `execution.deliver_input` | ADR-0007 §D5、design/05 | 管理 API 保留投递记账；OS 规范不再规定三条管道（草案已如此，design 待 v0.8 修订） |
| F5 Provider 代理 | 可选 `provider_proxy` | design/13 威胁模型仍画 `fysund` 代理 | 威胁模型改为「凭据只在 qualified EP 注入」 |
| 只读快路径 | 04 未写 D5 | ADR-0008 D5 已采纳 | **04 必须继承**，见 P0-3 |
| Runtime 类型 | 资产 `kind=runtime` | v0.7.0 Runtime 是 Resource | 建议回到「运行对象」，不要两种都是 |
| Capability.implementation | 删除，改 Agent Definition | design/02 Skills 目录 | 这是有意改变，采纳后改 design |
| OTel | 07「`gen_ai.*` 钉 commit」 | sources.md：截至 2026-08 全部 Development；主仓曾 deprecate 迁出 | 钉 **仓库+schema URL**，不要暗示稳定 |
| MCP 版本 | sources.md 钉 2025-11-25 | 现行公开规范 2026-07-28（Sampling 弃用、无协议会话、Authorization 收紧） | 07 增加规范钉版与差异表 |
| R3/R4 顺序 | 先 AaaS | F3 依赖 EP | 见 P0-4 |
| 双文档世界 | redefinition 自称独立；00 称与 design/00 冲突以本文件夹为准 | design 仍是现行 | 在 ADR 采纳前避免实现对照两套术语 |

---

## 6. 与前沿研究 / 工业实践对照

决策栏：采纳 / 谨慎采纳 / 延后 / 拒绝（指是否写入 FysunOS **内核规范**）。

1. **MCP 2026-07-28**  
   工具/资源/提示/Tasks 仍是工具面标准。Sampling 弃用。Authorization 为 OAuth 2.1。  
   **含义**：07 的 MCP 表过时一个大版本。  
   **决策**：采纳现行 Authorization 与 Tasks；Sampling 默认拒绝；钉规范日期。

2. **A2A 1.0（Linux Foundation，2026-04 宣布企业生产与签名 Agent Card、多租户）**  
   草案已用 A2A 作主 Endpoint、签名 Agent Card、TaskState 映射。  
   **决策**：采纳。补 Card 签名密钥与 `supportedInterfaces`（JSON-RPC/gRPC/HTTP+JSON）钉版。Q28 在 R3 关闭。

3. **Cedar + 网关 PEP（Bedrock AgentCore Policy GA 2026-03；AWS 多 Agent Cedar 链）**  
   与「EP 本地 Cedar、approve 回中心」同构。  
   **决策**：采纳并保持子集；EP 编译缓存为实现细节。不要引入 NL→Cedar 自动生效。

4. **Prompt / KV prefix caching（Anthropic 官方文档；arxiv:2601.06007）**  
   长程工具循环的成本结构已被工业验证。  
   **决策**：采纳为 Harness 装配**约束**（槽序），拒绝规定厂商 cache API。

5. **Durable execution（Temporal AI agent 参考架构）**  
   协调与 I/O 分离；HITL 零成本等待；并行工具。  
   **决策**：谨慎采纳模式（Activity 已存在）；不把 Temporal 做成依赖。freeze/thaw 对齐 wait_condition，不重写账本。

6. **DSPy / GEPA（arXiv:2310.03714；2507.19457）**  
   离线编译 prompt/演示；生产应 compile 一次、当制品发布。  
   **决策**：延后到 R5 **插件**。拒绝运行时在热路径上做优化循环。

7. **SWE-bench 系评测**  
   OpenAI 已停止用 SWE-bench Verified 报告前沿能力（污染）。确定性 verifier + 私有回归集比追公开榜更重要。  
   **决策**：采纳「容器化、可复现、独立 verifier」；拒绝把公开榜当 evidence_gate 唯一门。

8. **SPIFFE/SPIRE**  
   短寿命工作负载身份工业标准。  
   **决策**：谨慎采纳为 EP/Runtime 身份后端之一；Tenant 计划仍走 OIDC。

9. **OTel GenAI semantic-conventions-genai（schema `gen-ai-dev/1.42.0-dev`，stability: development）**  
   **决策**：导出时带 schema URL；核心 SLO 用 `fysun.*` 与稳定 OTel HTTP/RPC 约定。拒绝「强制稳定 gen_ai.system」。

10. **CaMeL / Progent（已在 sources.md）**  
    值级信息流与符号化权限。Q13 已保留。  
    **决策**：延后完整 CaMeL；保持 label 与 F9。

11. **Agent 生产测量 CAP（arXiv:2512.04123）**  
    HITL、评估、可靠性是生产常态。  
    **决策**：采纳为 SLA 投影维度（05 §8 已有人工介入率）。

12. **MemOS 等记忆 OS**  
    不自建引擎、要租户命名空间——草案已如此。  
    **决策**：采纳现状。

13. **SLSA / Sigstore**  
    对应 Q22。  
    **决策**：R1 选一种签名；Policy 可要求。

14. **Microsoft Entra Agent ID**  
    sources.md 已作委派身份对照。  
    **决策**：联邦 Trust（Q27）时参考，不绑厂商。

---

## 7. 专项分析

### 7.1 精简与复杂度

草案用「一切可复用之物是资产」换来了可追溯，也换来了种类爆炸。**[推断]** 真正高频的组合是：`agent + harness + model + 若干 skill/mcp + 可选 prompt/loop`。`runtime` 不应在此列；`tool` 应降为 mcp 的附属或原生特例。

目标不是「种类越少越好」，而是：**制品走 OCI；配置可内联；实例走运行平面**。这样 P9（必须有 digest）仍成立，打包流水线不必为 20 行 loop YAML 空转。

### 7.2 性能

现代 Agent 延迟/成本主因通常是：**前缀未命中、工具串行、同步磁盘、上下文膨胀**，不是 Cedar 微秒级评估。**[推断]**

最小性能契约：

| 机制 | 规范位置 | 不做 |
|---|---|---|
| 确定性前缀槽 | harness `prompt_slots` + qualify | 不写渲染器 |
| `local_read` 快路径 | 04 继承 ADR-0008 | 不把 `external_read` 免持久化 |
| 单轮并行只读 | EP `call_batch` 或等价 | 不对 mutation 推测提交 |
| 审计微批 | EP ring buffer → `ep.report` | 不让 fysund 进热路径 |
| 本机 UDS | 08 | 不前置 Arrow/FlatBuffers |

### 7.3 管理与治理

治理是草案最强部分：多租户 Scope、Plan→Grant、审批路由、T12–T15、policy.simulate。缺口在 **动态性**（证据不能折损继承）和 **身份**（工作负载短身份、MCP OAuth）。Cedar 实体扩展（Tenant/EP/Asset/Service）应保持。

### 7.4 自进化

09 与 ADR-0005 的证据门是正确的**发布门**，不是进化引擎。学术上的自改进（参数回路 + 脚手架回路）若直接改 Grant/工具面，会打穿 F7/F9。

推荐闭环：

```text
轨迹（含失败 checklist） → 归因（prompt / tool / loop）
  → 仅 Mutable Zone 生成候选 Profile
  → 离线回归 ∪ 影子（同一 Lock 契约层）
  → Adjustment 或新 ServiceVersion canary
  → Pareto 不劣于基线才升 stable
```

Immutable Zone：`tool_effects` 严重度、allowed tools 扩张、预算上调、设备/凭据、Policy 文本。

### 7.5 Agent as a Service

已有工业级骨架：版本钉 Lock、canary 审计、配额维度、数据驻留、跨租户审批、MCP facade、不做计费。要达到「可对外卖」还需：

1. 非只读工具面被 EP 覆盖才允许 `published`（P0-4）；
2. 可信计量（P1-7）；
3. 入站 OAuth + 工作负载身份（P1-6/P1-8）；
4. 长任务 freeze 或明确「不支持跨天 HITL 的 Plan」（P1-4）；
5. SLO 投影已有，缺错误预算与隔离失败注入测试；
6. Trust 对象（Q27）作为跨基座前置，不要在 R3 假装联邦已成。

---

## 8. 目标架构（部分重构后）

### 8.1 最小核心（建议写入下一版草案，而非现在改文件）

**四层实体**

```text
运行实例（非资产）     Node · RuntimeInstance · EP · Secret 后端
L1 可执行制品          agent · skill · harness
L2 连接件              mcp_server · memory_provider · knowledge
L3 配置档（可内联）    prompt · loop · model · 独立 tool（仅非 MCP）
```

**分层 Lock**

```text
contract_digest   = Capability bounds + 确认后的 tool_effects + 工具面收窄
substrate_digest  = harness + mcp snapshots + memory/knowledge digests + model family
profile_digest    = prompt/loop/overrides/variables
lock_digest       = H(contract, substrate, profile)
```

证据：contract/substrate 变 → 不继承；仅 profile 变 → `inherited` + 折损；canary 指标可加速折损回补。

**不变量（建议增补，不削弱 F1–F9）**

- I1 `fysund` 不在工具/模型转发链（已有）。
- I2 有效权限单调收窄（已有 F9）。
- I3 `local_read` 可不建 Effect，但必须 `audit/observed`（ADR-0008）。
- I4 新 Lock 不得把不同 `contract_digest` 的 Verification 计为 pass。
- I5 自进化输出不得修改 Immutable Zone。
- I6 `service.rollout(stable)` 的 Lock 其 mutation 工具面被 qualified EP 覆盖。
- I7 计量用于配额的维度必须来自 qualified EP，而非仅 Harness 自报。

### 8.2 API 增量（相对 08，只增不拆组）

```text
compose.resolve → Lock 含三层 digest 与 inheritance 声明
compose.diff    → 系统判定 change_class ∈ {contract, substrate, profile}
runtime.freeze / thaw   （可选能力位）
ep.report       → 允许 audit 微批；sampling 事件（兼容窗口）
evolution.propose / evaluate   （R5；产出候选 Profile，不签发 Grant）
identity.bind   （OIDC 租户；SVID 可选）
```

删除：把 `runtime.register` 从「资产 Registry」叙事中移到 Runtime API（08 已有 Runtime 组，Registry 段应去掉 kind=runtime 表述）。

### 8.3 可删 / 合并 / 延后

| 动作 | 内容 |
|---|---|
| 删除（概念） | `runtime` 作为资产 kind |
| 合并 | MCP 工具并入 `mcp_server`；独立 `tool` 仅原生 |
| 降级 | 短 loop/prompt/model 默认可内联 |
| 延后 | DSPy 内核化、SPIFFE 强制、Arrow IPC、完整 CaMeL、跨基座共享账本 |
| 拒绝 | 证据跨 contract 自动迁移；NL 生成 Cedar 不经审查生效；公开榜单一证据门 |

---

## 9. 建议的文档树（仅方案，不改原文）

定稿并采纳 ADR-0013 之后：

| 篇 | 职责 |
|---|---|
| ADR-0013 | 决策；增补：资产分层、分层 Lock、04 继承 ADR-0008、MCP 钉 2026-07-28 |
| 00 | 术语；Runtime 不再是 kind；增 change_class、freeze、workload identity |
| 01 | 定位原则；定义句可增加「对外短形式」例外 |
| 02 | 四层模型 + manifest；kind 专有字段 |
| 03 | 组合、分层 Lock、内联 Profile、证据继承规则 |
| 04 | Runtime 实例、ABI、EP、**快路径分级**、freeze、conformance |
| 05 | AaaS；对外发布资格；计量可信；OAuth/A2A Card |
| 06 | 治理变化 + 进化 Immutable Zone 的 Policy 实体 |
| 07 | 生态；MCP/A2A/OTel **钉版本差异表** |
| 08 | API；Runtime 登记挪出 Registry 叙事 |
| 09 | 证据门 + 演进双轨（人工 / 离线进化插件） |
| 10 | 路线图；调整 R3/R4 门；关闭 Q23/Q26 的推荐答案 |

不建议再拆「性能专篇」「AaaS 计费篇」。性能进 04，计费继续不做。

---

## 10. 分阶段实施与验收

在原文 R0–R5 上**插入门禁**，不另起炉灶。

| 阶段 | 增量 | 可判定验收（建议） |
|---|---|---|
| **R0** | 采纳 ADR-0013 + 本报告分层/快路径/MCP 钉版修订 | grep：`runtime` 不再出现在资产 kind 表；04 引用 ADR-0008 D5；07 写明 MCP 2026-07-28 与 Sampling 缺省拒绝 |
| **R1** | manifest；pack/push/register；RuntimeInstance 资格化 | 02 原负例 + N-01（Runtime 当 artifact 被拒）；签名策略可配 |
| **R2** | compose 分层 Lock；一个 Harness 闭环 | 首个闭环仍成立；仅改 prompt 产生新 `profile_digest`、同一 `contract_digest`；`compose.diff` 返回 `profile` |
| **R2b 性能** | local_read 快路径；前缀槽；并行只读 | N-03/N-04；度量 cache 命中与 `audit/observed` 占比（阈值基线后定） |
| **R3** | gateway + Tenant/Plan；**对外 publish 需 EP 覆盖** | 05 原负例 + 两租户隔离；mutation 无 EP 不得 `published` |
| **R4** | mcp_guard 独立进程；OAuth audience；Sampling 防火墙；Trust ADR | 04/06 负例；sampling 缺省 deny；token 错 audience 拒绝 |
| **R5** | 证据继承折损；evolution.propose；freeze 可选；第二 Harness | 09 原负例收窄为「contract 变更不继承」；N-07/N-08；Personal 不维护第二执行路径 |

### 10.1 建议新增负例（进入修订后的 ADR/04/09，而非现在改文）

| ID | 情境 | 期望 |
|---|---|---|
| N-01 | pack RuntimeInstance | `asset.kind_mismatch` |
| N-02 | 仅 profile 变更却丢弃同 contract 证据 | 必须保留 `inherited` |
| N-03 | 资格化 Harness 把每轮时间戳写入 Slot0 | `runtime.cache_alignment_violated`（若声明该能力位） |
| N-04 | `local_read` 无 `audit/observed` 或 `external_read` 无 ActionIntent | 失败（ADR-0008 原文） |
| N-05 | 兼容窗口内 sampling 带回 `trusted_system` 或 Secret | EP 拒绝 |
| N-06 | sampling/模型调用超过 Execution 预算 | `quota.exceeded` |
| N-07 | 进化提议提升 effect_category 或新增工具 | `evolution.immutable_boundary_violated` |
| N-08 | 影子集未通过即 `stable` | `evidence_gate` 失败 |
| N-09 | freeze 后 thaw 到错误 lock_digest | 拒绝 resume |
| N-10 | 租户 A 凭据调用租户 B Service | 网关拒绝 |
| N-11 | 无 mutation EP 覆盖的 Lock 被 `service/published` | 失败 |
| N-12 | 仅 Harness 自报 token 导致他户 `quota.exceeded` | 失败 |

### 10.2 基线后再定阈值的度量

- `compose.resolve` 时延；资产登记到可解析步骤数（10 §4 已列）
- 前缀缓存命中率、`local_read` 快路径占比
- EP 回报延迟与拒绝率
- 每租户用量分离正确率
- canary→stable 的 Verification 通过率、人工介入率、回滚率
- 进化提议接受率与安全拦截率

---

## 11. 开放问题与待验证假设

| ID | 内容 | 建议默认 |
|---|---|---|
| Q22 | 签名格式 | Sigstore/Cosign，Policy 可要求 |
| Q23 | 跨 Harness loop 子集 | **定义最小集** `max_steps / stop_conditions / retry`，其余 schema 开放 |
| Q24 | shared 池化隔离证明 | R3 缺省 `dedicated`；shared 需注入测试 |
| Q25 | EP 崩溃未回报 | 继承 outcome-unknown；明确 `authorized` TTL |
| Q26 | 计量可信 | 对外 token 配额 **要求** provider_proxy 或等价 |
| Q27 | Trust 对象 | R4 独立 ADR，阻断无 Trust 的跨基座自动执行 |
| Q28 | A2A 能力位 | 与 gateway 实现同步写入 Agent Card，禁止虚报 |
| Q29 | BYO 边界 | 效果类别校验在 `fysund` 解析时，EP 再强制一次 |
| **[待验证]** | 前缀槽对 TTFT 的实际收益 | R2b 用同一 Lock 开关对比 |
| **[待验证]** | 内联 Profile 是否降低发布摩擦 | 用真实 Agent Definition 计数 |
| **[待验证]** | freeze 快照最小字段集 | 先支持「丢弃模型状态、保留 Lock+水位+待输入」 |
| **[待验证]** | GEPA 等优化器在 Fysun 回归集上是否 Pareto 改进 | 未测前不写入内核 |

继承仍开放的 Q1（传输）、Q2（Secret 后端）、Q12（多节点一致性）、Q13（CaMeL）、Q15（OTel 关系）——Q15 建议改为「钉 Development schema + `fysun.*` 为权威 SLO」。

---

## 12. 来源清单（可访问）

1. MCP Specification 2026-07-28 — https://modelcontextprotocol.io/specification/2026-07-28 — 现行协议；Authorization、传输与弃用策略。  
2. MCP Sampling（2026-07-28）— https://modelcontextprotocol.io/specification/2026-07-28/client/sampling — Sampling 弃用（SEP-2577），新实现 SHOULD NOT 采用。  
3. MCP Sampling（2025-11-25）— https://modelcontextprotocol.io/specification/2025-11-25/client/sampling — 弃用窗口内仍可能出现 `sampling/createMessage`。  
4. MCP Authorization — https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization — OAuth 2.1、RFC 9728、RFC 8707 audience。  
5. A2A Protocol — https://a2a-protocol.org/latest/ — Linux Foundation 托管的 Agent 互操作标准。  
6. A2A 1.0 规范 — https://a2a-protocol.org/v1.0.0/specification/ — Agent Card、Task、多绑定。  
7. Linux Foundation A2A 一周年（2026-04-09）— https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year — 1.0、签名 Card、企业多租户。  
8. Anthropic Prompt Caching — https://platform.claude.com/docs/en/build-with-claude/prompt-caching — 稳定前缀与 cache breakpoint 的官方约束。  
9. *Don’t Break the Cache* — https://arxiv.org/html/2601.06007v2 — 长程 Agent 任务中前缀抖动的代价。  
10. OpenTelemetry GenAI conventions — https://github.com/open-telemetry/semantic-conventions-genai — 独立仓库；`stability: development`。  
11. Cedar 语言 — https://www.cedarpolicy.com/ （语言与工具）— FysunOS Policy 已选；与 AgentCore/AVP 同族。  
12. AWS Security Blog：多 Agent Cedar 链 — https://aws.amazon.com/blogs/security/enforce-least-privilege-authorization-in-multi-agent-ai-chains-using-cedar/ — 委派深度与 originating user。  
13. SPIFFE 规范 — https://spiffe.io/docs/latest/spiffe-specs/spiffe/ — 工作负载身份。  
14. Temporal AI Agent Reference Architecture — https://go.temporal.io/platform-hub/ai-engineering/ai-reference-architecture — durable wait、工具并行、I/O 进 Activity。  
15. DSPy — https://arxiv.org/html/2310.03714v1 与 https://dspy.ai/ — 离线编译式优化，非请求路径循环。  
16. GEPA — https://arxiv.org/abs/2507.19457 — 反思式 prompt 进化，适合当 R5 插件。  
17. OpenAI：SWE-bench Verified 不再衡量前沿 — https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/ — 公开榜污染；私有回归更可靠。  
18. SWE-bench 评估指南 — https://github.com/swe-bench/swe-bench/blob/main/docs/guides/evaluation.md — 容器化确定性 verifier 先例。  
19. CaMeL — https://arxiv.org/abs/2503.18813 — 按设计对抗注入；对应 Q13 延后。  
20. Progent — https://arxiv.org/abs/2504.11703 — 特权控制与单调收窄，支撑 F9。  
21. AOS-0001 — https://arxiv.org/abs/2608.03214 — 草案 01 对外坐标来源。  
22. *Self-Improvements in Modern Agentic Systems* — https://arxiv.org/abs/2607.13104 — 参数/脚手架双回路；09 应引用而非内嵌实现。  
23. MemOS — https://arxiv.org/abs/2507.03724 — 不自建记忆引擎的依据。  
24. CAP / Measuring Agents in Production — https://arxiv.org/abs/2512.04123 — HITL 与生产可靠性。  
25. 本仓库 ADR-0008 — `docs/adr/0008-effect-category-and-read-only-determination.md` — `local_read` 快路径与 `external_read` 完整协议。  
26. 本仓库 sources.md — `docs/research/sources.md` — MCP 2025-11-25 钉版与 OTel Development 已登记，需随 07 升级。

---

## 13. 结论

v0.8.0-draft 的战略选择是对的：从内联网关回到「管绑定、控 Runtime、服务化 Agent」。不应当再退回 v0.7.0 的 `fysund` 热路径。

它目前还不是一条精简高效、能显著抬升 Agent 性能与自进化的实现规格。主要阻力是概念层把实例当资产、证据模型惩罚微调、运行时契约与已采纳的只读快路径以及 2026-07 MCP 现状脱节、演进章停留在 SCM。

推荐路径是 **部分重构**：四层实体、分层 Lock、04 继承 ADR-0008、前缀槽与并行只读、对外 AaaS 以 PEP 覆盖为门、MCP OAuth + Sampling 缺省拒绝、自进化作为带 Immutable Zone 的离线插件。保留 F1–F9、Cedar、账本、EP 与标准优先。

本报告不修改 `docs/redefinition/`。若采纳，下一步应是修订 ADR-0013 草案正文与 00/02/03/04/07/09/10，而不是先改实现或现行 `docs/design`。
