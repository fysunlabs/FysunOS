# 10 · 路线图与开放问题

- 状态：提案草案 v0.8.0-draft（2026-09-11）
- 前置：全部本文件夹文档；[v0.7.0 MVP 计划](../design/12-mvp-plan.md)

## 1. 为什么重定义

2026-09-11 对 v0.7.0 的定位审查结论（摘录）：

- 原点 ADR-0001 §D1 "管理绑定并可靠运行绑定"在 v0.7.0 被落实为 `fysund` 内联数据面：唯一 MCP server、下游会话与凭据注入、Provider 代理、上下文渲染、三条审批投递管道。
- 篇幅上，06 / 07 / 08 / 13 约 160 KB 围绕内联拦截；Capability 的注册、组合、谱系、演进（10 演进 9.6 KB 为全仓库最短）不到守卫规范一半；Harness 与 Loop 被排除在对象模型外；
  多主体 Scope 停在开放问题。
- 内联模型的结构性极限已被自身承认：ADR-0007 把 F3 / F7 / F9 收窄到"经守卫的 MCP 工具面"，hooks "只能拦不能记"，F5 收窄到"经代理的 Provider"。
- owner 的目标表述："Agent 能力（提示词、Skills、工具、MCP、Harness、Loop）管理与 Runtime 控制的运行基座，服务 Agent as a Service 生态。"

结论：把"管理"与"可靠运行"分别落为**能力管理**与 **Runtime 控制**两项职责；`fysund` 退出数据面；Harness / Loop 升为资产；新增服务平面；多主体进入核心。

## 2. 与 v0.7.0 文档的逐篇关系

| v0.7.0 | 关系 | 去向 |
|---|---|---|
| 00 术语 | 改写 | 本文件夹 00 合并回去；增资产 / 服务 / EP 术语 |
| 01 定义与原则 | **重写** | 本文件夹 01 |
| 02 对象模型 | 扩展 | 增 Asset / Manifest / Agent Definition / Lock / Service / ServiceVersion / Endpoint / Tenant / Plan / Meter / EP；Capability 去 `implementation`；Resource 收窄 |
| 03 架构、账本与恢复 | **重写分层与拓扑**；账本与恢复继承 | 本文件夹 01 §3 + 04 §3、§7 |
| 04 Capability API | 扩展 | 本文件夹 08（六组 API） |
| 05 Runtime ABI | 修订 | `start_execution` 不带渲染文本；`os.observe` 替代系统工具；`deliver_input` 可选 |
| 06 MCP 守卫规范 | **降级** | 变为"`mcp_guard` 类 EP 参考实现规范"，从属 04 §4；`fysund` 内组件 → 独立进程 |
| 07 Runtime 接入 | §1、§2、§6、§7 继承；**§3–§5 废止** | 投递路径、系统工具、F5 部署指引删除或移入适配器文档 |
| 08 DSH 适配器 | 降级 | `fysun-dsh` = Harness 资产打包 + Runtime ABI + `harness_adapter` 类 EP；从属 04 §4 与 07 §2 |
| 09 迁移清单 | 缩小 | Effect 协议进入参考 EP；Personal 三合一应用 |
| 10 演进 | 扩展 | 本文件夹 09 |
| 11 边缘与离线账本 | 继承 | `fysund-edge` = 带本地账本的 EP 宿主 |
| 12 MVP 计划 | **重写** | 本文 §3 |
| 13 威胁模型与 conformance | 修订 | conformance 按 EP 组合分列；增 T12–T15 |
| ADR-0001 | 保留；§D1 加注 | ADR-0013 承接定义 |
| ADR-0002 | 修订 | §D2 "守卫必经之路" → "EP 契约"；§D5 DSH 归类 |
| ADR-0007 | 修订 | §D3 hooks = `hooks` 类 EP；§D5 投递路径废止 |
| ADR-0011 | 保留 | 审批面规则不变；增审批路由 |
| ADR-0012 | 重开 | "不嵌入"保留；"不前置"改为"可作为 `external_gateway` EP" |
| 其余 ADR | 保留 | |

## 3. 里程碑（R0–R5）

原则不变：第一个可运行闭环必须小、真实、可验证；不再全量重写 14 篇文档，先以本文件夹 + ADR-0013 为准推进实现，实现过程中回填 `docs/design`。

| 阶段 | 交付 | 验收（可判定） |
|---|---|---|
| **R0 定位与术语** | ADR-0013 采纳；本文件夹定稿；`docs/design` 01 / 03 / 12 重写，其余标注继承 / 降级 | 一致性 grep：`fysund` 不再被描述为 MCP server / 代理 / 渲染方；ADR-0013 负例 6 |
| **R1 资产与 Runtime 登记** | manifest schema（11 种 `kind`）；`fysun asset pack / push / register / publish`；`asset.*` API；治理账本（继承 M0）；Runtime 登记与资格化（`describe()` 与 manifest 互证）；Policy 载入与 `policy.simulate` | 02 §9 负例 1–7；04 §8 负例 1；replay digest 稳定；`asset/published` 带 `policy_decision`；`runtime/qualified` 带 `harness_asset_id` |
| **R2 组合与首次运行** | Capability 契约；`agent.define` / `compose.resolve` / `compose.match`；Lock；一个 Harness（DSH headless 薄包装或 `fysun-dsh`）实现 Runtime ABI；Activity / Execution 状态机、恢复与重新武装（继承 M3 核心）；`os.observe` | 03 §8 负例 1–5；04 §8 负例 2、8；首个闭环：登记 prompt + skill + mcp_server + harness + loop + model → 定义 Agent → 解析 → 匹配 → 运行 → 独立验证 → 崩溃恢复 |
| **R3 服务平面** | `fysun-gateway`（A2A Endpoint + Agent Card + MCP facade）；Tenant / Plan / Meter；`service.publish / rollout`；Runtime 池化；租户隔离 | 05 §12 负例 1–8；04 §8 负例 7；两个租户各自调用同一 Service，用量分离、Memory 分离、审批路由正确 |
| **R4 执行点与治理闭合** | EP 契约；`fysun-mcp-guard` 独立进程（继承 06 规则）；`hooks` EP；Policy 下发与 `policy_stale_ttl`；Approval 与审批路由；Trust 对象（跨基座）；conformance 按 EP 组合分列 | 04 §8 负例 3–6；06 §10 负例 1–5；`fysund` 不在任何转发链（ADR-0013 负例 2）；v0.7.0 M1 / M2a 的 Effect 协议负例在 EP 上通过 |
| **R5 演进与生态** | Adjustment 作用于组合；证据分层；发布证据门；弃用与迁移；`fysun-dsh` 全控制点；边缘 `fysund-edge`；Personal 三合一接入；第二个 Harness 接入 | 09 §9 负例 1–4；Personal 不再维护第二套执行路径；两个不同 Harness 运行同一 Lock（变体）并分别累积证据 |

## 4. 度量

继承 v0.7.0 12 §4 全部项，新增：资产登记到可解析的步骤数；一个新 Harness 从打包到 `runtime/qualified` 的步骤数；`compose.resolve` 时延；Service 发布到首个调用的步骤数；
EP 回报延迟与拒绝率；每租户用量分离正确率（负例通过率）。基线先测、阈值后定。

## 5. 证据纪律与非声明

继承 v0.7.0 12 §5（四类状态用语）与 §6。本提案期：本文件夹全部条目为 `specified`；不声明任何资产已登记、任何 Harness 已资格化、任何 Service 已发布。

## 6. 开放问题

继承 v0.7.0 12 §7 的 Q1–Q21，处置如下；Q22 起为本提案新增。

- 关闭：**Q5 多主体 Scope**（Tenant 进入核心）；**Q11 Trust 对象**（R4 必做，设计待 ADR）；**Q14 Provider 路由**（不做路由，`model.fallback` 是绑定顺序）；
  **Q16 DSH headless 原生工具查证**（R2 前查证，不影响规则）；**Q21 `approval_result_ttl`**（投递管道废止后不再需要；Effect 停在 `authorized` 的 TTL 由 EP 契约定义，转 Q25）。
- 保留：Q1 传输（增 EP 契约跨机传输）、Q2 Secret Store 后端、Q3 Web UI、Q4 judge 校准集、Q6 许可证、Q12 多节点一致性、Q13 CaMeL 级信息流、Q15 `gen_ai.*`、Q17 `modify` 表达式、
  Q18 `obligations[]`、Q19 `on_expiry = escalate`、Q20 `downstream_outcome` 时间窗。
- 新增：
  - **Q22 manifest schema 的规范化与签名格式**：OCI artifact config 的 media type、canonical digest 算法、签名方案（Sigstore / 自签）。
  - **Q23 Harness `loop_schema` 的通用子集**：是否定义跨 Harness 的最小 Loop 参数集（`max_steps`、`stop_conditions`），还是完全由各 Harness 声明。
  - **Q24 `shared` Runtime 池化的隔离证明**：`session_isolation` 能力位的验收方法。
  - **Q25 EP 本地持久化与回报的一致性**：EP 崩溃时未回报事件的恢复协议；`authorized` 未分发 Effect 的 TTL 与作废语义（承 Q21）。
  - **Q26 计量的可信度**：Harness 自报 token 计入与否；`provider_proxy` 是否成为多租户 Service 的资格化前提。
  - **Q27 Trust 对象设计**：字段、来源（签名 / conformance / 历史 Verification）、与 Policy 的引用方式、跨基座传递。
  - **Q28 gateway 的 A2A 能力位**：streaming / pushNotifications / 扩展的支持范围与 Agent Card 生成规则。
  - **Q29 租户 BYO 资产的边界**：允许覆盖的槽位集合、效果类别不可降级的校验位置（`fysund` 还是 EP）。

## 7. 待所有者决策项

1. 是否采纳 ADR-0013（重定位）；采纳后 `docs/design` 按 §2 逐篇处置，版本号 v0.8.0。
2. R2 首个 Harness 选 DSH headless 薄包装（最小面、快）还是 `fysun-dsh`（全控制点、重）。
3. R3 与 R4 的先后：先服务平面（尽早对外）还是先执行点（尽早闭合 F3）。本文按"先服务平面"排列，理由是 F3 在 R2 已由 `harness_adapter` 或 `mcp_guard` 的最小实现覆盖首个闭环。
4. `fysun-mcp-guard` 是否保留为第一方参考 EP，或以 hooks + `fysun-dsh` 为 MVP 执行点、把工具侧 EP 交给外部网关。
5. 单机个人部署（Personal）是否强制经过服务平面（Tenant = owner、Plan = unlimited），还是允许直接 `activity.create` 绕过 gateway。本文按"允许绕过，但 Tenant 与 Scope 仍存在"处理。
