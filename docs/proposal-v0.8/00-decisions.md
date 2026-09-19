# 00 · 拍板决策

- 状态：**提案草案**
- 日期：2026-09-12
- 本文件不替代现行 [`docs/redefinition`](../redefinition/README.md) 直至被采纳。
- 相对草案：[ADR-0013](../redefinition/adr-0013-draft-repositioning.md)。继承已采纳：[ADR-0001](../adr/0001-founding-decisions.md)、[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md)（守卫必经之路一条由本提案 supersede）。

每条格式：决策 → 理由 → 否决的替代 → 后果。

## D1 · 定义与职责边界

**决策。** FysunOS 是 Agent 能力管理与 Runtime 控制的运行基座。两项职责、一个目标：

1. **能力管理**：可复用之物登记为带 digest 的资产或命名 Profile，组合成 Agent Definition，解析为分层 Lock。
2. **Runtime 控制**：Harness 安装实例的资格化、绑定、Activity/Execution 生命周期、预算、取消、恢复、重新武装、执行边界。
3. **目标**：Agent as a Service——发布、发现、调用、计量、版本、回滚。

基座做八件事，其余在内核之外：登记、解析锁定、资格化 Runtime/EP、执行点契约、Grant∩Policy∩工具面、Approval 封闭结果集、权威账本、服务目录。

**理由。** ADR-0001 §D1「管理绑定并可靠运行绑定」仍正确；v0.7.0 把它落实成 `fysund` 坐在每次 `tools/call` 上，redefinition 把职责拆对了，但定义句把六种实现物并列，暗示它们同级入核。2026 工业默认是控制面/数据面分离（AWS AgentCore），协议在 MCP / A2A / Agent Skills。

**否决。** (a) 维持 v0.7.0 内联守卫为必经路径；(b) 只做纯注册表、不做生命周期；(c) 基座自带参考 loop / prompt 装配；(d) 把服务平面拆成独立产品。

**后果。** 定义句不再枚举「十一种资产」。文档不得把基座写成「实现 loop / 沙箱 / 计费」。`fysund` 出现在工具或模型转发链上 → 失败。

## D2 · 八种资产；Loop 与 Tool 不是一级 `kind`；Runtime 不是资产

**决策。** 可登记资产 `kind` 封闭为：

`prompt` · `skill` · `mcp_server` · `model` · `memory_provider` · `knowledge` · `harness` · `agent`

| 被建议抬成一级的东西 | 本提案裁决 |
|---|---|
| Loop | **拒绝独立 `kind`。** 它是 Harness 的 Loop Profile：内联在 Agent Definition `spec.loop`，或作为命名 Profile 存在登记表（无强制 OCI）。校验 `params ⊨ harness.loop_schema`。 |
| Tool | **拒绝独立 `kind`。** 工具声明属于 `mcp_server.tools_snapshot`、`skill.allowed-tools`、`harness.native_effect_tools[]`。非 MCP 的 HTTP/本地动作打成最小 `mcp_server`（或 harness 原生工具），不维护第二套工具身份。 |
| Runtime | **拒绝作为资产。** Runtime = Harness 资产 × Node × `runtime/qualified` 纪元。由 `runtime.register` 产生，不进 OCI 仓库、不被 Agent Definition 当资产引用。 |

每种一级资产的「若不存在则无法完成 X」见 [02](02-assets.md) §2。

**理由。** Redefinition 自己写：Loop「是配置不是代码」、Runtime「通常不经资产仓库分发」——却仍享受与 Skill/Harness 相同的 OCI 生命周期。Tool 与 `mcp_server.tools_snapshot` 双源会在漂移时撕裂（OWASP ASI04）。Agent Skills 与 MCP 已是工具与技能的标准容器。

**否决。** (a) 十一类平级；(b) 复检报告的「3 层 9 类」重命名（仍是九个一级，只是换了分层标签）；(c) 把 prompt/model 也降到「不可单独引用」（组合将无法复用人格包与模型目标）。

**后果。** `asset.register(kind=loop|tool|runtime)` → `asset.kind_mismatch`。Loop Profile 变更只影响 Lock 的 `profile_digest`。

## D3 · 内核 vs 可插拔

**决策。**

| 在内核 | 不在内核 |
|---|---|
| 资产登记与 digest 核对 | Prompt 写作、Skill 正文 |
| `compose.resolve` / 分层 Lock | Loop 实现、compaction 算法、工具选择 |
| Runtime / EP 资格化 | 具体 Harness（DSH / Claude Code / Codex / 边缘） |
| 执行点契约与 Policy 快照下发 | 记忆引擎实现、模型 Provider |
| Grant∩Policy∩工具面；Approval | 优化器（ACE/GEPA/DGM 类） |
| event-sourced 账本 | 计费结算、聊天 UI、沙箱 |
| 服务目录与 Plan→Grant | Trust 联邦对象（R3 后，单独 ADR） |

优化器若要入目录，按普通进程或 `harness` 资产登记，经 Adjustment API 写回 Profile，**不能**写 Grant、不能改 `effect_category`、不能跳过证据门。

**理由。** ADR-0001 §D2 已否决自建 loop。2026 自进化文献（ACE / DGM）证明演化发生在上下文、prompt、脚手架，不是 OS 内核。内核一旦内置某优化器，就会把证据门与算法绑死。

**否决。** 内生「Evolution Engine」作为第五平面；DSPy/TextGrad 作为必选依赖。

**后果。** 路线图不得把「实现 ACE」列为内核里程碑；R3 只验收闭环接口。

## D4 · 控制面 / 数据面；热路径不经 `fysund`

**决策。** `fysund` = PDP + 账本 + 生命周期。数据面 = 执行点（`gateway` / `mcp_guard` / `hooks` / `harness_adapter` / 可选 `provider_proxy`）。

资格化、策略评估投影、工具面裁剪在 **绑定/启动时** 完成，打进 EP 的 Policy 快照。每次 `tools/call`：**不**回 `fysund` 问「可不可以」。例外（回中心）：`outcome=approve`、Grant 扩张、secret 解析、Memory 准入中需人的部分。

F3 分级（对齐 [ADR-0008](../adr/0008-effect-category-and-read-only-determination.md)，修正 redefinition 的一刀切）：

- `local_read`：EP 内存策略 + `audit/observed`（可异步刷盘），不持久化 ActionIntent。
- `external_read`：本地记录意图（可组批），不阻塞同步 fsync；回报可微批。
- 其余四类：persist-before-dispatch，同步 WAL + receipt。

**理由。** Redefinition 正确把 `fysund` 拉出热路径，但 EP 契约仍把所有调用写成同步落盘。v0.7.0 已有 `local_read` 快路径；高频 Agent 的只读占比高，一刀切会惩罚 TTFT/TPOT。Anthropic 工具搜索与 `defer_loading` 的前提是：模型看到的工具面在请求构建时已裁剪。

**否决。** (a) 每次 tool call 问 `fysund`；(b) 所有类别同步 WAL；(c) 规定 Arrow/UDS 零拷贝为内核义务；(d) OS 做推测执行调度。

**后果。** 负例：`fysund` 在转发链上；未资格化 EP 的回报被接受；`policy_stale` 的 EP 放行非 `local_read`。

## D5 · Activity 与 Execution 保持两级

**决策。** Activity = 可跨多次运行的工作（目标、租户、连续性、HITL 挂起）。Execution = 一次 run（Runtime 绑定、fencing、检查点、本次计量）。一次服务调用创建或唤醒 Activity，并启动一条 Execution。不引入第三级 Invocation 对象；入口事件是 `invocation/*`。

可选控制点 `session_park`：Activity 进入长时间 `input_required` / `waiting{approval}` 时，Runtime 可 `park` 交出快照并释放进程；恢复走既有重新武装 + `thaw`/`resume_execution`。

**理由。** 计费、审批、崩溃恢复、金丝雀（旧 Execution 跑完）都需要「工作 ≠ 这一次进程」。合成一级会把 fencing/水位/租户用量缠进同一状态机。Park 解决 redefinition 承认但不设计的「HITL 数天占着 Runtime」。

**否决。** 取消 Execution；把 Invocation 抬成一级。

**后果。** MVP 必须实现两级状态机（继承 v0.7.0）。`park` 可返回 `unsupported`。

## D6 · 服务对象不五类平级

**决策。** 服务平面只有 **Service** 一个一级对象。

- `versions[]`：钉 `lock_digest` + rollout 阶段（staged / canary / stable / rolled_back / retired）。
- `endpoints[]`：协议绑定，由 gateway 暴露，不是可独立生命周期的对象。
- **Tenant**：治理对象（Principal + Scope），资产/Memory/Activity 的归属。
- **Plan**：Tenant×Service 附着，派生根 Grant；`plan/attached` 为准入事件。
- **Meter**：事件 `meter/recorded` + 查询投影。

**理由。** AaaS 需要发布/发现/调用/计量/版本/回滚，不需要五套 CRUD。AWS AgentCore 用 Runtime 版本 + Endpoint 指向版本，Endpoint 不是与 Runtime 平级的产品概念。

**否决。** redefinition 的五对象平级；把 Tenant 做成可打包资产。

**后果。** 无 `endpoint.create` 独立资源；改 Endpoint 配置不是新 Service 版本（版本单位是 Lock）。

## D7 · 分层 Lock 与证据继承

**决策。** Lock 不可变，含三个 digest：

| 层 | 覆盖 | 变更时证据 |
|---|---|---|
| `contract_digest` | Capability 契约 + 汇总 `tool_effects[]` | **不继承**，全量证据门 |
| `substrate_digest` | Harness + MCP 快照 + model 身份 + memory/knowledge digest | **不继承** |
| `profile_digest` | Prompt、Loop Profile、允许的覆盖、工具面收窄 | 旧 Lock 证据标记 `inherited`，健康查询降权展示；**不**自动当作本 Lock 的 Verification |

`service.rollout(stable)` 的证据门看本 Lock 的非 inherited 证据，或 Policy 明确允许「仅 profile 变更 + 回归集通过」。禁止把 inherited 证据计为新契约/基座的 pass。

**理由。** Redefinition「换资产则证据清零」会杀死 prompt/loop 微调（ACE/GEPA 的主战场）。但数值型 `confidence_discount: 0.85` 无法校准（RuVerBench / 评估可靠性文献），不进内核。

**否决。** 单一扁平 digest 且任何变更清零；自动数值折损作为门控输入。

**后果。** `compose.diff` 必须返回变更落在哪一层。优化器只能提交 profile 层 Adjustment。

## D8 · 协议优先；身份在 R0–R3 保持最小

**决策。** 不自造包格式、策略语言、传输协议。MCP（含 2026-07-28 扩展与 sampling）、A2A、Agent Skills、OCI artifact、Cedar 子集、OTel 导出。

身份：Tenant 凭据（token / mTLS）→ gateway；Execution 是可授权 Principal；Handoff 衰减 Grant 即委托链。SPIFFE/SVID **不**进入 R0–R3 内核。跨基座 Trust 对象延后，单独立 ADR。

可观测性：OTLP 导出；`gen_ai.*` 按 commit 钉住（约定仍为 Development）；治理键放 `fysun.*`（`tenant_id`、`lock_digest`、`ep_id`、`activity_id`）。账本 schema 不依赖未稳定的 `gen_ai.*`。

**理由。** OWASP ASI03/ASI07 需要委托链与 A2A 完整性，v0.7.0 的 Execution Principal + Grant 衰减已经覆盖 MVP。SPIFFE 是部署选项，不是对象模型。

**否决。** 把 SPIFFE 写成资格化前提；把 `gen_ai.*` 当作稳定契约。

## 相对 ADR-0013 草案的差异

| ADR-0013 | 本提案 |
|---|---|
| D1 定义句列举 Harness、Loop 为并列管理对象 | 定义句不把 Loop 与 Harness 并列；Loop 随 Harness 管理 |
| D2 十一类资产含 loop / tool / runtime | 八类；三者降级或删除 |
| D3 Runtime = 资格化实例，但仍是资产 `kind` | 实例属于运行平面，删除 `kind=runtime` |
| D4 三平面对象把 Tenant/Plan/Meter/Endpoint 画进服务平面一级 | 服务平面一级只有 Service；其余附着或治理 |
| D5–D8 Policy 下发、多主体、标准、F1–F9 | **保持**；F3 恢复并明确读路径分级 |
| 负例 1「无资产登记不能引用」含 loop/tool | 改为：无登记不能引用八类资产；Loop 必须能追溯到 Profile id 或内联 digest |

ADR-0013 若采纳，应以本文件 D1–D8 替换其 D2 资产枚举与服务对象清单，而不是原文落地。
