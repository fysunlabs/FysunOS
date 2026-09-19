# FysunOS 重新设计 · 完整设计说明（v0.8 综合定稿稿）

- 状态：**综合定稿稿（待所有者采纳）**，2026-09-14。本文档综合 `docs/redefinition`（v0.8.0-draft）、
  `docs/proposal-v0.8`、两份 2026-09-12 独立评审报告与 v0.7.0 现行设计（`docs/design` + ADR-0001–0012），
  给出重新设计后 FysunOS 的**单份完整设计说明**。采纳后它将成为 v0.8.0 正式设计文档集合（`docs/design` 回填）与 ADR-0013 定稿的基础。
- 证据纪律：本文档全部条目为 **`specified`**（规范存在 ≠ 实现存在 ≠ 测试已执行 ≠ Profile 已符合，四类状态用语继承 v0.7.0 12 §5）。
  所有外部基准数字只说明方向，不构成 FysunOS 的任何性能或安全声明；未测量过的延迟、命中率、成本一律不写为事实。
- 本文档不修改任何既有文件；与现行 v0.7.0 文档冲突处，以本文档 §0.3 的裁定关系为准。

---

## 0. 本说明的由来与综合方法

### 0.1 演进脉络

```text
v0.5.0（2026-09-06） 首轮设计稿
v0.6.0（2026-09-07） 前沿对照审查后修订：协议优先、Approval/Policy/Memory 一级对象、F9、威胁模型
v0.7.0（2026-09-08） 全量重构：14 篇设计 + ADR-0001–0012；"fysund 内联守卫"形态（MCP 守卫在 fysund 内、
                     守卫是必经之路、Provider 代理、上下文渲染、三条投递管道）
redefinition（2026-09-11） v0.8.0-draft：定位重审——"管理绑定"被 v0.7.0 落实为 fysund 坐在每次 tools/call 上，
                     Capability 管理被挤成附属；提出双职责、执行点、资产化、服务平面（ADR-0013 草案）
两份独立评审（2026-09-12） 对 redefinition 的部分重构方案：四层实体、分层 Lock、04 继承 ADR-0008 D5、
                     published 以 EP 覆盖为门、MCP 钉 2026-07-28、自进化 Immutable Zone、不变量 I1–I12
proposal-v0.8（2026-09-12） 吸收评审的拍板提案：D1–D8、八种资产、三层 Lock、F3 分级、R0–R3
本说明（2026-09-14） 上述全部的综合定稿
```

### 0.2 综合裁定表（评审建议的采纳与拒绝）

| 评审主张 | 本说明裁定 |
|---|---|
| Runtime 不是可分发资产 | **采纳**（不变量 I12） |
| Loop 不占 OCI 一级 kind | **采纳**（Loop Profile：可内联或命名登记，非 kind） |
| 取消独立 `tool` kind | **采纳**（工具身份归 `mcp_server.tools_snapshot` / `skill.allowed-tools` / `harness.native_effect_tools[]`） |
| 分层 Lock + 微调可继承证据 | **采纳**（D7；拒绝数值折扣内核化） |
| 04/EP 契约必须继承 ADR-0008 D5（仅 `local_read` 快路径） | **采纳**（F3 分级，§6.6；明确拒绝把 `external_read` 放进无持久化快路径） |
| `published` 以 qualified EP 覆盖为门 | **采纳**（不变量 I6，§8.2） |
| MCP 钉 2026-07-28；Sampling 缺省拒绝；OAuth 2.1 | **采纳**（§8.6） |
| A2A 1.0 Agent Card 形状（`supportedInterfaces[]` 等） | **采纳**（§8.6） |
| Cedar 实体扩展先修 ADR-0009 | **采纳**（R0 工作项，§13.1） |
| freeze/thaw | **采纳为可选控制点 `session_park`**（不承诺 KV/模型状态迁移） |
| 自进化优化器不进内核、Immutable Zone | **采纳**（§9.3） |
| 前缀槽作为资格化硬失败 | **拒绝**——改为 Harness 声明 `cache_alignment` 能力位；OS 只给结构化绑定顺序 `prefix_order[]`，不渲染文本（MCP-only 薄包装往往做不到稳定前缀，硬失败会收窄"任意 Runtime"承诺） |
| SPIFFE 替换 Tenant token | **拒绝**进 R0–R3 内核（Execution Principal + 租户凭据已覆盖 MVP；SPIFFE 为后置可选身份后端） |
| Arrow Flight / 零拷贝 IPC 为规范 | **拒绝**（过早优化；传输是开放问题 Q1） |
| 3 层 9 类新本体论命名 | **拒绝**——压缩为 8 kind + Loop Profile，不另造 L1/L2/L3 产品名（分层仅用于 Lock digest 与心智模型展示） |
| 数值型 `confidence_discount` 进入证据门 | **拒绝**（无法校准；`inherited` 只标记与降权展示，不自动当作新 Lock 的 Verification） |
| 内生 Evolution Engine 作为第五平面/必做阶段 | **拒绝**（优化器是可插拔生产者；内核只提供证据 → Adjustment → 门控 → 回滚） |
| 确定性前缀槽、并行只读、EP 微批等性能机制 | **采纳**（§6.6、§6.3；度量先基线后阈值） |

### 0.3 与既有文档的关系

| 文档集 | 处置 |
|---|---|
| `docs/design`（v0.7.0，14 篇） | 现行实现规格。采纳本说明后按 §13.1 R0 清单回填为 v0.8.0：01/03/12 重写，02 增资产与服务对象，06/07/08 降级为执行点与适配器规范，13 conformance 按 EP 组合分列 |
| `docs/adr`（0001–0012） | 0001/0003/0004/0005/0006/0008/0010/0011 保留；0002 §D2、0007 §D3/§D5、0009 §D1、0012 §D1 按 §13.1 R0 清单修订或增补；0008 D5 原样保留并被 EP 契约继承 |
| `docs/redefinition` | 采纳后标记 superseded by v0.8.0，归档或合并，不再双源 |
| `docs/proposal-v0.8` | 其 D1–D8 与本说明一致处全部继承；本文档是其超集 |

---

## 1. 定位与定义

### 1.1 一句话定义（唯一正式定义句）

> **FysunOS 是 Agent 能力管理与 Runtime 控制的运行基座：登记并组合可版本化的能力资产，资格化 Runtime，
> 在执行点强制边界，使 Agent 作为服务被发布、调用、计量与演进。**

完整描述（不改变定义句，不缩略、不改写；引用时字面一致）：它管理的能力包括提示词、Skills、工具（经 MCP）、
Harness 及其 Loop 策略；Loop 是 Harness 的配置，不是与 Harness 平级的一级资产。基座不实现 loop、不装配 prompt、
不在热路径转发工具或模型请求。

redefinition 版定义句（"Agent 能力管理（提示词、Skills、工具、MCP、Harness、Loop）和 Runtime 控制的运行基座"）
作为对外长形式保留备查；正式定义句采用上方短形式（两份评审 P2-7 建议 + proposal 定稿），避免把六种实现物并列暗示同级入核。

### 1.2 两项职责、一个目标、基座八件事

**两项职责：**

1. **能力管理（Capability Management）**：可复用之物登记为带 digest 的资产或命名 Profile，组合成 Agent Definition，解析为分层 Lock；
   基座负责登记、版本、来源、兼容、效果声明、组合解析与证据——**不写 prompt、不写 skill、不实现工具**。
2. **Runtime 控制（Runtime Control）**：Harness 安装实例的资格化、绑定、Activity/Execution 生命周期、预算、取消、恢复、重新武装、执行边界
   （Grant ∩ Policy ∩ 工具面）；控制与决策在 `fysund`，数据面拦截在**执行点**。

**一个目标：Agent as a Service**——Agent 的对外形态是服务，有租户、配额、计量、版本、Agent Card 与证据；
消费者经 A2A / MCP / HTTP 调用，不关心它由哪个 Harness 在哪个 Runtime 上跑。服务平面建立在前两项职责之上，不替代它们。

**基座八件事（其余全部在内核之外）：** 登记、解析锁定、资格化 Runtime/EP、执行点契约、Grant∩Policy∩工具面、
Approval 封闭结果集、权威账本、服务目录。

### 1.3 与 v0.7.0 的继承 / 改变 / 废止

**继承（不变量与合同原样保留）：** event-sourced 治理账本与唯一权威写入者（F1）；F1–F9 底线；Grant ∩ Policy ∩ 工具面
有效权限模型与 F9 单调收窄；ActionIntent → Effect → Receipt → Verification 协议；六值 `effect_category` 三源合一（ADR-0008）；
Cedar 子集与版本化 Policy（ADR-0003/0009）；Approval 封闭结果集与"审批面只由 `fysund` 提供"（ADR-0011）；来源标签与
`flags`（ADR-0004）；Activity/Execution 状态机、双水位恢复与重新武装；Runtime 资格化纪元与 `native_effect_tools[]`（ADR-0007）；
边缘离线一致性（ADR-0010）；协议优先（MCP / A2A / Agent Skills / OCI / Cedar / OTel）；conformance 四类状态用语；
大声失败、不设兼容等级、不静默降质。

**改变：** Harness 与 Loop 策略升为可登记对象（Harness 为一级资产 kind，Loop 为其 Profile）；Capability 的实现从单一
Skills 目录扩展为 **Agent Definition（资产组合）**；执行点（Enforcement Point，EP）替代"`fysund` 内联守卫是必经之路"；
新增服务平面（Service / Tenant / Plan）；多主体 Scope 从开放问题变为核心；扁平 Lock 改为**分层 Lock**（contract /
substrate / profile）并配套证据差分继承；MCP 规范钉版升级至 2026-07-28。

**废止（降级为执行点或适配器的实现细节）：** `fysund` 作为 harness 唯一 MCP server 的强制形态；`fysund` 代理模型请求
作为 F5 前提；`fysund` 渲染 context_view 进 harness 启动输入（改为只下发结构化绑定视图，Harness 自行装配）；
`deliver_input` 三条投递管道与守卫系统工具 `fysun.await_approval` / `fysun.pending_inputs` 作为 OS 规范
（替代物：Runtime 以 `os.observe` 查询 Approval 结果与待投递输入）。

### 1.4 对外坐标（AOS 两平面）

| AOS 平面 / 职责 | FysunOS 对应 | 供给声明 |
|---|---|---|
| Control & Governance Plane — intent、policy、trust、authority、confidence、auditability、observability、human oversight | 治理横切（Grant / Policy / Approval / 标签 / Verification）+ 治理账本；Trust 由服务平面联邦需要引入（R3 之后单独 ADR） | **供给（Trust 除外）** |
| Runtime & Coordination Plane — lifecycle、workflow coordination、model/tool routing、context & memory coordination、scheduling、traffic management、runtime assurance | 运行平面（生命周期、调度、交接、资格化）；服务平面（流量入口）；组合平面（模型 / 工具 / 记忆的**绑定**而非路由） | **供给控制面，不供给数据面**：决策核心 loop、prompt 装配、沙箱、请求转发留给 Harness 与执行点 |
| 平台服务 — 宿主 OS、容器、物理设施 | 不做 | 不供给 |

### 1.5 明确不做的事

- 不实现 loop、prompt 装配、工具选择、compaction、subagent 传输、会话日志、沙箱、终端、LSP、聊天 UI——由 Harness 资产提供；
- 不做数据面转发：`fysund` 不作为 MCP server 转发工具调用、不代理模型请求、不渲染上下文文本、不持有下游 MCP 会话；
- 不做计费结算：只产生 `meter/recorded` 事件与导出，账单由外部系统生成；不做定价引擎；
- 不自造包格式（OCI artifact + Agent Skills）、策略语言（Cedar 子集）、传输协议（MCP / A2A / HTTP）；
- 不内置角色、团队、数字员工、业务流程、Agent Shell；
- 不替代宿主 OS、容器编排或设备控制系统；硬实时与急停留在宿主控制器；
- 不区分能力等级、不设兼容等级制度；不持有 Runtime 的 Loop 状态；不承诺模型内部状态或进程的透明迁移；
- 不内置优化器（ACE / GEPA / DGM 类为可插拔插件）；不把未稳定的外部约定（`gen_ai.*` Development 状态）写进账本 schema。

---

## 2. 设计原则、底线与不变量

### 2.1 设计原则 P1–P11

| # | 原则 | 含义 |
|---|---|---|
| P1 | 标准化必要边界，让能力自由变化 | 只结构化机器必须检查或跨组件交接的部分（manifest、契约、事件） |
| P2 | 可靠性由系统承担，不变成用户流程 | 权威写入、资格化、效果回报核对、独立验证在系统内自动发生 |
| P3 | 契约可移植，过程不承诺 | 换 Harness / Loop / Model / Runtime 不改契约；证据按资产版本归属 |
| P4 | 长期活动是一级生命周期 | Activity 可无终点；Execution 是一次运行；均不依附会话或进程 |
| P5 | 协作是普通能力行为 | 子 Agent 与外部服务经 Handoff / A2A 引用；无专门多 Agent 子系统 |
| P6 | 调整默认轻量，检查只在风险边界 | Adjustment 直接生效、留痕、可回退；扩权 / 新外部动作 / 预算上调 / 设备安全走 Policy |
| P7 | 部署环境声明策略，基座执行边界 | Policy 版本化下发；能力不自扩权 |
| P8 | 不规定决策核心，但登记并承载它 | 不写 loop / prompt 装配 / 沙箱；Harness 与其 Loop Profile 作为可登记对象被登记、版本化、绑定、度量 |
| P9 | 一切可分发、可被 Definition 引用的制品是资产 | 无 manifest、无 digest 的东西不能进入 Agent Definition；运行实例与服务实例不是资产 |
| P10 | Agent 的对外形态是服务 | 发布即有租户、配额、计量、版本、Agent Card；接口只用 A2A / MCP / HTTP |
| P11 | 基座不在热路径 | `fysund` 只做管理与决策；拦截、转发、渲染在执行点或 Harness 内 |

### 2.2 底线 F1–F9（含执行者与热路径标注）

| ID | 底线 | 可判定表述 | 由谁执行 | 热路径？ |
|---|---|---|---|---|
| F1 | 唯一权威写入者 | 只有 `fysund` 可以登记 / 发布资产与 Service、签发 Grant、推进 Activity / Execution / Effect / Verification 状态、签发与关闭 Approval、记账预算、写权威账本。EP 的本地 WAL 是待核对副本，不是第二账本 | `fysund` | 否 |
| F2 | 概率组件只产候选 | 模型与任何第三方 Agent 的输出都是候选；准入、调度、Effect 状态推进、验收是确定性操作（配套机制：来源标签，§7.5） | `fysund` + EP | 候选在数据面，准入决策在快照/中心 |
| F3 | 外部效果先记录再执行（**按 `effect_category` 分级**，§6.6） | `local_read`：不建 Effect，但必须 `audit/observed`；`external_read`：完整协议（不得免持久化）；`local_mutation` 及以上：EP 本地 persist-before-dispatch（同步 WAL）+ receipt。无回报的 Effect 不得进入 `verified` | EP 分级执行；`fysund` 核对投影 | 写路径是 |
| F4 | 完成由独立验证判定 | Activity 阶段完成需独立 Verifier 对照契约与已回报 Effect 判定；`inconclusive ≠ pass` | `fysund` 调度 Verifier | 否 |
| F5 | secret 隔离 | OS 管理的 secret 只以 `SecretRef` 出现，**只向资格化执行点按 Grant 解析**；Harness 与 Runtime 进程不见明文；Harness 自持凭据不在范围（列入不防） | `fysund`（解析）+ EP（注入） | `resolve_secret` 是冷/温路径 |
| F6 | 契约与负例不为实现让路 | 公共契约、manifest schema、状态迁移、错误、负例不因某个实现不通过而删改；范围含 Policy | 流程 | — |
| F7 | 部署策略声明、能力不自扩权 | Policy 决定什么可自动执行；Agent Definition 与 Harness 不得自行扩大 Grant、不得隐瞒未满足条件 | EP 强制；`fysund` 签发 | 过滤在热路径本地 |
| F8 | 未知外部改动受保护 | 自主代理不覆盖、回退、混入非本任务的并发改动 | Harness + EP | 是 |
| F9 | Execution 内有效权限单调收窄 | 有效权限集 = Grant ∩ Policy 允许集 ∩ 工具面；在 `start_execution` **之前**算完写入 EP 快照；收窄自动生效，扩张只经 Approval（`approved_once`）且仍 ⊆ 父 Grant | `fysund` 签发；EP 过滤 | 计算在冷路径，执行在热路径本地 |

### 2.3 架构不变量 I1–I12（实施与 conformance 用）

| ID | 不变量 | 主要负例 |
|---|---|---|
| I1 | 权威账本只在 `fysund`；EP 本地 WAL 是 pending replica | EP 本地写出 `grant/issued` / `approval/decided` / `effect/verified` → 失败 |
| I2 | `fysund` 不在工具或模型转发链 | `fysund` 进程出现在 `tools/call` 或模型请求路径 → 失败 |
| I3 | 仅 `local_read` 可不建 Effect，但必须最终在权威账本出现 `audit/observed`（允许异步微批，不允许丢失） | `local_read` 无 `audit/observed` → 失败 |
| I4 | `external_read` 及以上必须有 ActionIntent（`external_read` 的 EP 本地意图记录可组批，不得免持久化） | `external_read` 只有 `audit/observed` 无 `action-intent/recorded` → 失败 |
| I5 | 未分类工具按 `external_mutation`（fail-closed） | `readOnlyHint = true` 导致走快路径 → 失败 |
| I6 | **非 `local_read` 工具面 ⊆ qualified EP 覆盖面，才允许 `service/published`** | 无 PEP 覆盖的 mutation Service 对外 `published` 且可调用 → 失败 |
| I7 | 有效权限 = Grant ∩ Policy ∩ 工具面，Execution 内单调收窄 | Adjustment 启用禁用工具且无 Approval → 失败 |
| I8 | Secret 只向 qualified EP 按 Grant 解析 | 未资格化 EP `resolve_secret` 成功 → 失败 |
| I9 | 同 `policy_version` + 同输入 → `fysund` 与 EP 的 `outcome` 相同 | 两处结果不同 → 失败 |
| I10 | 资产内容 digest 与 Lock 不一致不得 `start_execution` | 拉取内容 digest 不匹配仍启动 → 失败 |
| I11 | Immutable Zone 不能被优化器或 overlay 静默改写 | 优化器输出降低 `effect_category` → 失败 |
| I12 | `runtime` 不是可分发资产；Agent Definition 不引用 Runtime id | `kind=runtime` 被 `fysun asset push` 成功且进入 Lock.assets → 失败 |
| I13 | 新 Lock 不得把不同 `contract_digest` 的 Verification 计为 pass（证据继承规则，§5.4） | 契约层变更后旧 Verification 被计为新 Lock 的 pass → 失败 |
| I14 | `session_park` 后 `thaw` 到错误 `lock_digest` 拒绝恢复 | 快照 lock 与当前 Lock 不一致仍 resume → 失败 |

### 2.4 热路径与冷路径

- **热路径**（每步推理 / 工具调用）：Runtime ↔ 工具侧 EP（`mcp_guard` / `harness_adapter` / `hooks`）↔ MCP server / 原生工具。
  **不经过 `fysund`**。EP 按已下发快照本地评估；只读并发；审计微批。
- **冷路径**：发布、解析、资格化、换 Policy、审批、恢复、canary。`fysund` 拥有全部冷路径决策。
- **例外回中心**（热路径允许的一次控制面 RPC）：`outcome = approve`、Grant 扩张、secret 解析、Memory 准入中需人的部分。
  每次调用**不**回 `fysund` 问"可不可以"。

---

## 3. 对象模型总览：四层实体

### 3.1 四层

| 层 | 对象 | 是什么 | 不是什么 |
|---|---|---|---|
| **L1 契约** | Capability | 行为 / 质量 / 边界 / 验证；`behavior` / `quality`（Skill Contract (P,O,A,V,F)）/ `bounds`（含汇总后确认的 `tool_effects[]`）/ `verification` | 不是 OCI 制品；无 `implementation` 字段（实现关系写在 Agent Definition 侧） |
| **L2 制品（资产）** | `kind ∈ prompt · skill · mcp_server · model · memory_provider · knowledge · harness · agent` | 可 pack/push、有 digest、可被 Definition 引用 | 不含 `loop` / `tool` / `runtime`；不含 EP |
| **L3 组合** | Agent Definition（`kind = agent`）+ 分层 Lock + Loop Profile（非 kind） | 引用 + 覆盖 + 变体 → 可重放的三层 digest 集 | 不绑定具体 Node / 进程 |
| **L4 运行与服务** | Node、Runtime 实例、EP、Activity/Execution、Service、Tenant、Plan | 资格化实例、PEP、生命周期、对外入口 | 不进资产仓库；Meter 为事件投影 |

`agent` 同时是 L2 制品（可分发 Definition）与 L3 入口——这是允许的唯一双栖。

### 3.2 一级对象速查

| 一级 | 平面 | 一句话 |
|---|---|---|
| Asset / Manifest | 资产 | 八种可登记可分发之物 |
| Capability | 资产 | 行为契约，不是 kind |
| Agent Definition / Lock / Loop Profile | 资产 | 组合与三层钉死；Profile 是配置不是 kind |
| Runtime 实例 / Node / EP | 运行 | 资格化后的进程与拦截点 |
| Activity / Execution | 运行 | 长期工作 / 一次 run |
| Service | 服务 | 对外发布形态（唯一服务一级对象） |
| Principal / Tenant / Scope / Grant / Policy / Approval | 治理 | 谁、在哪、能做什么、谁批 |
| Event | 治理 | 账本真源 |

**不是一级**：Loop Profile（配置）、Endpoint（Service 绑定）、ServiceVersion（`versions[]` 记录）、Plan（附着约定）、
Meter（事件投影）、Resource（设备 / 账号 / 工作区路径 / SecretRef，授权与计量对象）、Memory 条目 / Artifact（治理对象）、
单次装配出的上下文（Harness 会话账本）。

### 3.3 状态机（继承与新增）

| 对象 | 状态 |
|---|---|
| Asset（版本） | `draft` / `published` / `deprecated` / `withdrawn` / `quarantined`（+撤销 `revoked` 事件；版本不可变） |
| Agent Definition（版本） | 同 Asset；另有 `resolved`（存在 Lock）投影 |
| Capability | `draft` / `active` / `withdrawn`（继承） |
| Runtime 实例 | 资格化纪元：有生效 `runtime/qualified`，或最近 `runtime/disqualified{reason}`；失格原因五值 `describe_changed / bundle_changed / policy_revoked / adapter_precondition_failed / harness_asset_revoked` |
| Enforcement Point | `registered` / `qualified` / `disqualified{reason}` |
| Service | `draft` / `published` / `suspended` / `retired` |
| ServiceVersion（rollout） | `staged` / `canary{percent}` / `stable` / `rolled_back` / `retired` |
| Tenant | `registered` / `active` / `suspended` / `closed` |
| Activity | `created` / `active` / `waiting` / `input_required` / `paused{reason}` / `blocked{reason}` / `completed` / `cancelled`（继承 v0.7.0） |
| Execution | `starting` / `running` / `waiting{reason: trigger \| approval \| input}` / `checkpointed` / `finished` / `failed` / `cancelled`；恢复后含非 `local_read` 类别缺省对应 Activity `paused{reason: rearm_required}` |
| Effect | `authorized` / `dispatch_started` / `receipt` / `outcome_unknown` / `reconciled` / `quarantined` / `compensated`（`local_read` 不进 Effect 状态机） |
| Approval | `pending` / `approved_once` / `rejected` / `cancelled` / `unavailable`（封闭，fail-closed） |
| Policy | `active` / `retired`（版本只增不改） |
| Handoff | A2A v1.0 TaskState 九态 + FysunOS 叠加态 `verified` |
| Verification.verdict | `pass` / `pass_with_warning` / `fail` / `inconclusive` |
| Node | `registered` / `online` / `offline` |

### 3.4 事件族（新增与变更；其余继承 v0.7.0 00 §4）

**新增：**

- `asset/registered{asset_id, kind, version, digest, source, publisher}` · `asset/published`（准入类）· `asset/deprecated{successor?}` ·
  `asset/withdrawn{reason}` · `asset/quarantined{reason}` · `asset/revoked{reason}`（供应链撤销，级联 Lock 与 Service）
- `agent/defined{asset_id, capabilities[]}` · `agent/resolved{asset_id, lock_digest, contract_digest, substrate_digest, profile_digest, bindings_digest}`
- `ep/registered{ep_id, kind}` · `ep/qualified{ep_id, kind, describe_digest, policy_version}`（准入类）· `ep/disqualified{ep_id, reason}` ·
  `ep/policy-pulled{ep_id, policy_version}`
- `service/published{service_id, version, lock_digest, endpoints[]}`（准入类）· `service/rollout{service_id, version, stage}` ·
  `service/suspended` · `service/retired`
- `tenant/registered` · `tenant/suspended` · `plan/attached{tenant_id, service_id, plan}`（准入类）· `plan/detached`
- `meter/recorded{tenant_id, service_id, execution_id, dimension, amount, ep_id?, policy_version?}`
- `invocation/received{tenant_id, service_id, endpoint, correlation_id}` · `invocation/completed{state}`
- `adjustment/*` 作用于 profile 层或 Definition 补丁（继承事件族，target 扩展）

**变更：**

- 准入类事件（携带 `policy_decision{policy_version, rule_id, outcome, obligations[]}`）在 v0.7.0 七类
  （`effect/authorized`、`action-intent/rejected`、`approval/asked`、`grant/issued`、`memory/admitted`、`memory/rejected`、`runtime/qualified`）基础上
  增 `asset/published`、`service/published`、`ep/qualified`、`plan/attached`。
- `effect/*`、`action-intent/*`、`audit/observed` 增 `ep_id`（哪个执行点回报）与 `policy_version`（EP 本地评估所用快照）。
- `runtime/qualified` 增 `harness_asset_id`（该 Runtime 是哪个 Harness 资产版本的实例；**无 `kind=runtime` 资产 id**）。
- 通用字段增 `tenant_id`（可空：系统级事件）；边缘字段 `node_id` / `origin_seq` 继承。
- **废止**：`fysun.await_approval` / `fysun.pending_inputs` 作为 OS 规范（由 `os.observe` 取代）；三条 `deliver_input` 投递管道
  作为 OS 规范（管理 API 保留 `execution.deliver_input` 投递记账，投递实现下放适配器文档）。

### 3.5 错误码

稳定前缀继承 v0.7.0 全表（`denied.*`、`not_runnable.*`、`conflict.*`、`unavailable.*`、`needs_human(approval_id)`、
`input_required(request_ref)`、`auth_required(approval_id)`、`unknown_outcome`、`inconclusive`），新增：

`asset.invalid_manifest` · `asset.digest_mismatch` · `asset.unsigned`（Policy 要求签名时）· `asset.quarantined` · `asset.kind_mismatch` ·
`compose.unresolvable{missing[]}` · `compose.effect_conflict` · `compose.requirements_unmet{missing[]}` ·
`ep.not_qualified` · `ep.policy_stale` · `ep.report_rejected` ·
`service.not_published` · `service.version_retired` · `quota.exceeded{dimension}`（含 `dimension: sampling`）· `plan.effect_not_allowed{category}` ·
`tenant.suspended` · `evolution.immutable_boundary`（优化器越权）·
`not_runnable.environment{missing[]}`（触发条件增：Lock 中任一资产 `withdrawn` / `quarantined` / `revoked`；无满足 Plan 数据驻留的 Runtime）。
错误码表达**可判定失败类**，不把内部堆栈塞进模型可见面。

---

## 4. 能力资产模型

### 4.1 资产判定与八种 kind

**资产** = 可复用、可分发、可被 Agent Definition 引用之物。判定标准三条同时满足：有稳定身份与版本；内容可 digest；
可脱离某次 Execution 存在。

| kind | 没有它则无法 | 必检 spec 要点 | MVP |
|---|---|---|---|
| `prompt` | 把可版本化的系统 / 角色 / 模板片段跨 Agent 复用，并约束变量完整性 | `role`、`content_ref`、`variables[]`（可要求 `integrity >= trusted_user`）、`slot`、`cache_hint: prefix \| volatile` | R1 |
| `skill` | 按 Agent Skills 标准做渐进披露的程序知识包 | `skill_dir`（含 `SKILL.md`）、`allowed-tools[]`；`declares.effects[]` 必须覆盖 | R1 |
| `mcp_server` | 把外部工具面钉在 `tools_snapshot` 上，检测供应链漂移 | `transport`（stdio / streamable_http）、`launch` 或 `url`、`tools_snapshot`+digest、每工具 `default_effect_category`、`credentials[]: SecretRef`；HTTP 传输增 `auth: none \| oauth2` | R1 |
| `model` | 钉 Provider + 模型身份 + 参数上界，声明 `direct` / `via_ep` | `provider`、`model_id`、`params_bounds`、`access`、`pricing_ref`（计量换算提示，非权威） | R1 |
| `memory_provider` | 多租户命名空间的记忆引擎接入与准入回报 | `engine`、`endpoint/launch`、`scoping`（per-tenant / per-activity）、`admission_hook` | R3（R1 可无） |
| `knowledge` | 只读、内容寻址的资料包（与可变 Memory 分开） | `content_ref`、`integrity_default`（缺省 `verified_data` 需 Verification 引用，否则 `untrusted_external`）、`freshness` | 可选 |
| `harness` | 声明 ABI、控制点、原生工具、`loop_schema`、prompt 槽、skill 加载方式 | `runtime_abi`、`control_points[]`、`native_effect_tools[]`（**字段必在，可空数组**）、`loop_schema`、`prompt_slots[]`、`skill_loading`（必须能只加载 name+description）、`tool_surface`、`install`/`bundle_digest` 规则；可选声明位 `cache_alignment`、`session_isolation`、`session_park`、`tool_search`；`adapter?` | R1 |
| `agent` | 把上述引用收成可发布的组合（Agent Definition） | 见 §5.2 | R1 |

**身份格式**：`<kind>:<namespace>/<name>@<version>`，例如 `skill:acme/trend-research@1.2.0`。同一 `version` 不同 `digest` 是两个版本。
Loop Profile 登记键：`loop:<namespace>/<name>@<version>`（**不是** kind 枚举值）。

### 4.2 不是资产的东西（与 redefinition 的关键差异）

| 事物 | 形态 | 理由 |
|---|---|---|
| **Loop** | Loop Profile：`{harness_ref, params}` 内联于 Definition，或命名 Profile（`profile.put` 写登记表，不强制 pack/push） | "是配置不是代码"；几行 YAML 走 OCI 生命周期是组合噪声；digest 化配置已满足追溯 |
| **单个 tool** | `mcp_server.tools_snapshot` 条目 / `skill.allowed-tools` / `harness.native_effect_tools[]`；非 MCP 的本地 / HTTP 动作打成最小 `mcp_server`（或 harness 原生工具） | 双源（`tool` + snapshot）必漂移；MCP 与 Skills 已是工具与技能的标准容器 |
| **Runtime 实例** | 运行平面对象：`runtime.register` 产生，**不进 OCI 仓库、不被 Definition 引用** | 不可分发、不可被引用；P0-4/P0-1 评审裁定（I12） |
| EP / Meter / Trust | 运行对象 / 事件投影 / 后置治理对象 | 不做成 kind |
| 一次装配出的上下文、Memory 条目、Artifact、Service / Tenant / Plan | 见 §3.2 | — |

Loop Profile 最小 params（跨 Harness 建议子集，各 Harness 经 `loop_schema` 扩展）：`max_steps` · `stop_conditions[]` ·
`retry{max, backoff}` · `delegation{enabled, max_concurrent, child_budget_ratio}` · `compaction{trigger}` ·
`human_in_loop{when}`（要输入，不是审批）。基座只按目标 Harness 的 `loop_schema` 校验，不解释语义；不匹配 → `asset.kind_mismatch`。
**Q23 处置**：跨 Harness 最小子集即上述字段；不做统一 loop 语言。

### 4.3 统一 manifest 最小字段

```yaml
apiVersion: fysun.io/v1alpha2
kind: skill                          # 八值之一
metadata:
  name: acme/trend-research          # namespace/name
  version: 1.2.0                     # SemVer
  digest: sha256:…                   # 身份的一部分；fysund 登记时重算比对
  publisher: principal:acme
  source: { oci: … }                 # 或 repo + revision
  license: Apache-2.0
  signatures: []                     # 可选；Policy 可要求
spec: { }                            # 按 kind（§4.1）
declares:
  effects: []                        # 必填（可空）；缺字段 → asset.invalid_manifest
  requires: { assets: [], control_points: [], environment: {} }
  provides: { tools: [], control_points: [] }
  compatibility: { harness: [], model: [] }   # 声明不是证据
# evidence[] 仅 fysund 追加（Verification 引用、conformance 结果、消费反馈折叠）
```

规则：manifest 不含任何 secret 值槽（SecretRef 三规则继承：每次操作重解析、描述面无值槽、空值即未配置）；
`declares.effects[]` 覆盖该资产引入的每个工具名，未覆盖者运行时按 `external_mutation` 并在 health 标未分类；
`metadata.digest` 不一致 → `asset.digest_mismatch`。

### 4.4 版本、digest、来源与供应链

- 引用用版本范围；Lock **只**存精确 version + digest；`lock_digest` = 三层规范序列化后再哈希，可重放（Q22：canonical digest 算法 R1 必须先定；
  media type 临时 `application/vnd.fysun.manifest.v1+json`；签名方案候选 Sigstore/Cosign，R1 Policy 缺省不要求、namespace 允许列表兜底）。
- `withdrawn`：不得进入**新** Lock；已有 Lock 与运行中 Execution 不杀。`revoked` / `quarantined`：新 Lock 拒绝；含它的 ServiceVersion 自动 `suspended`。
- MCP `tools/list` 与 `tools_snapshot` 漂移 → EP 回报 → 该 server 工具升为 `external_mutation` 直至重分类（T2 / T12）。
- 打包：`fysun asset pack/push/register/publish`；Loop Profile：`fysun profile put loop …` 写登记表。`fysund` 只存登记与 digest，不存内容（负例：存内容本体 → 失败）。

### 4.5 生命周期与发现

```text
draft ──publish（Policy 评估；可要求证据门）──▶ published ──deprecate──▶ deprecated ──withdraw──▶ withdrawn
  │                                               │
  └──────────── quarantine / revoke ──────────────┴──▶ 级联 Lock 与 ServiceVersion
```

`asset.search{kind?, labels?, capability_text?, provides.tools?, compatibility?}` 返回 manifest 摘要 + evidence 摘要（按层）+ health
（未分类工具、依赖被隔离、证据全部来自已失格 Runtime、过期证据）。**发现不隐含授权：可见 ≠ 可用**，可用由 Tenant Scope 与 Grant 决定。

### 4.6 资产负例

1. Agent Definition 引用无 digest 或 digest 重算不一致的资产 → 失败。
2. 登记 `kind = tool | loop | runtime` → 失败（`asset.kind_mismatch`）。
3. `skill.allowed-tools` 中工具未出现在 `declares.effects[]` 且 EP 未按 `external_mutation` 处理 → 失败。
4. `harness` 缺 `native_effect_tools` 字段（空数组合法）却 `runtime/qualified` → 失败。
5. Loop params 不符 `loop_schema` 却进入 Lock → 失败。
6. `revoked` digest 进入新 Lock，或其 ServiceVersion 仍 `stable` → 失败。
7. `fysund` 存资产内容本体 → 失败。
8. 仅 profile 变更却丢弃旧 Lock 的 `inherited` 标记（应保留并降权，而非当新 Verification）→ 失败。
9. manifest 或 `tools_snapshot` 中出现可放明文的凭据值槽 → 失败。

---

## 5. 组合：Agent Definition 与分层 Lock

### 5.1 Capability 契约

Capability 仍是行为契约，字段不变：`behavior`、`quality`（Skill Contract (P,O,A,V,F)，V 为 checklist 且每项标 `verifier_kind`）、
`bounds`（成本、时间、权限、风险；`tool_effects[]` 由组合汇总生成后**由发布者确认并可收窄**，`capability.register` 校验其覆盖组合中全部工具）、
`verification`。两处变化（相对 v0.7.0）：`implementation` 字段删除——Capability 由一个或多个 Agent Definition 实现，关系记录在
Definition 侧 `spec.capabilities[]`，证据分别归属；Capability 是契约对象，**不是资产 kind**。

### 5.2 Agent Definition（`kind = agent`）

```yaml
kind: agent
metadata: { name: acme/trend-writer, version: 3.1.0, digest: …, publisher: … }
spec:
  capabilities: [{ ref: capability:acme/trend-research@2, role: primary }]
  harness: { ref: "harness:deepseek/dsh@>=0.7.3 <0.8" }
  loop:    { profile: "loop:acme/careful@1", overrides: { max_steps: 40 } }   # 或完全内联 { params: {…} }
  model:   { ref: "model:openai/gpt-x@*", access: via_ep, fallback: ["model:anthropic/claude-y@*"] }  # fallback 是绑定顺序，不是运行时路由
  prompts: [{ ref: "prompt:acme/persona@1.4.0", slot: system, cache_hint: prefix }]
  skills:  [{ ref: "skill:acme/trend-research@1.2.0" }]
  mcp_servers: [{ ref: "mcp_server:acme/search@^2", tools_allow: [search.query] }]
  memory:  { ref: "memory_provider:memos/default@1", scope: tenant }
  knowledge: [{ ref: "knowledge:acme/brand-kb@2026.09" }]
  subagents:
    - { ref: "agent:acme/illustrator@1", via: handoff }
    - { service: "a2a://partner.example/summarizer", via: a2a }
  requirements:
    control_points: []
    reject_native_tools: [shell]        # 拒绝含无沙箱 shell 的 Harness
    environment: { offline: false }
  variants:
    edge:
      model: { ref: "model:local/qwen-small@*", access: direct }
      harness: { ref: "harness:fysun/edge-rs@^0.2" }
      mcp_servers: []
  policy_defaults: { ref: "policy:acme/default@3" }   # 产品缺省 Policy，部署 Policy 可覆盖
```

规则：`harness` 必选；`model` 必选（或由所选变体给出）；`loop` 缺省 = 该 Harness 的缺省 Profile，仍计算 `profile_digest`。
`overrides` 只能改 Loop params、prompt 变量缺省、`tools_allow`，**不能改 `declares.effects[]` 的类别**。
`tools[].enabled: false` 与 `mcp_servers[].tools_allow` 是工具面收窄的声明来源（F9 静态部分，resolve 时算入 Lock）。
`subagents[]` 是委派白名单；运行时 Handoff 只能指向白名单内目标或 Policy 另行允许者。
变体是 spec 局部覆盖，**不得改 `capabilities[]`**；变体不能满足契约 → 指向另一个收窄的 Capability，禁止静默降质。
**Definition 不引用 Runtime id**（I12）。

### 5.3 解析与约束

```text
compose.resolve(agent_ref, variant?, environment?)
  1. 展开引用 → 范围内最高 published → digest
  2. 检查约束（下表）；汇总 effects / requires
  3. 切分三层 digest（contract / substrate / profile）
  4. 计算 tool_surface（已收窄）
  5. 产出不可变 Lock；agent/resolved
compose.match(lock, node?) → runnable{runtime_ids[]} | not_runnable{missing[]}
```

| 约束 | 失败码 |
|---|---|
| 每个引用可解析到 `published` 版本（`deprecated` 告警；`withdrawn`/`quarantined`/`revoked` 不允许） | `compose.unresolvable{missing[]}` |
| `loop` params ⊨ `harness.loop_schema`；Profile `harness_ref` 匹配 | `asset.kind_mismatch` |
| 全部 `requires.control_points` ⊆ `harness.control_points`；`reject_native_tools ∩ native_effect_tools = ∅` | `compose.requirements_unmet` |
| 同名工具跨资产取 `max_severity`；Definition `declares.effects` ⊆ 汇总集（只能剔除） | `compose.effect_conflict` |
| `prompts[].slot ∈ harness.prompt_slots`；`memory.scope=tenant` 时 provider 支持 per-tenant | `compose.requirements_unmet` |
| manifest 与引用无 secret 值槽 | `asset.invalid_manifest` |

### 5.4 分层 Lock 与证据继承（对评审 P0-2 / P1-1 的裁定）

Lock 不可变，含三个 digest：

| 层 | 覆盖 | 变更时证据 |
|---|---|---|
| `contract_digest` | Capability 契约 + 确认后的 `tool_effects[]` + 工具面收窄 | **不继承**，全量证据门 |
| `substrate_digest` | Harness + MCP 快照 + model 身份 + memory/knowledge digest | **不继承** |
| `profile_digest` | Prompt、Loop Profile、允许的覆盖、变量缺省 | 旧 Lock 证据标记 **`inherited`**，健康查询降权展示；**不**自动当作本 Lock 的 Verification |

规则：

- `lock_digest = H(contract_digest, substrate_digest, profile_digest)`；Service 发布与 Activity 创建钉 `lock_digest`；升级 = 新 Lock。
- **证据继承按层判定**：`contract` / `substrate` 任一变更 → 该层证据从零、全量回归；仅 `profile` 变更 → 未改 digest 层的资产级证据
  可差分继承（复制引用而非改写，继承集合须可审计）；**资格化证据（Harness / Runtime 实例 / EP）不跨 id 转移**（继承 v0.7.0 原意，
  09 §9 负例 1 改写为"换 Harness 不得继承资格化证据，但未改 digest 的 skill 证据必须仍可查询"）。
- **Q30 处置**：`verifier_kind = deterministic` 的证据可自动继承；`model_judge` / `human` 项按 Policy 决定重跑或继承（缺省重跑进入回归集）。
- `compose.diff(lock_a, lock_b)` 从"人工提示"升级为**系统判定输入**：必须返回 `change_class ∈ {contract, substrate, profile}`，
  供证据继承与审批展示。禁止数值型折损（`confidence_discount` 之类）进入内核。
- Overlay（`this_activity` / `this_execution` 的 `lock_overrides_digest`）不能降低 `effect_category`、不能启用 Definition 已禁用的工具。

### 5.5 Adjustment 作用点

| 范围 | 作用 | 扩权？ |
|---|---|---|
| `this_execution` / `this_activity` | Lock 级覆盖，记 `lock_overrides_digest` | 启用新工具 / 放宽 `reject_native_tools` / 加预算 → Approval |
| `capability_default` | Agent Definition 补丁版本 + 重新解析 | 同上 |
| 优化器提交 | 只能改 profile 层（prompt / loop params / 少数覆盖） | 改 `tool_effects` 类别或加工具 → 拒绝（`evolution.immutable_boundary`） |

直接生效、留痕、可回退（P6）；`predicted_effect` 可选、评估对照、`evidence_gate` 继承 v0.7.0 10 §1。

### 5.6 组合负例

1. Lock 中出现版本范围而非精确 digest → 失败。
2. `compose.resolve` 对缺失控制点的 Harness 返回 `runnable` → 失败。
3. Definition `declares.effects[]` 把某工具类别降到低于汇总集 → 失败。
4. Handoff 目标不在 `subagents[]` 且无 Policy 允许却进入 `working` → 失败。
5. 变体解析后 `capabilities[]` 与主 spec 不一致 → 失败。
6. Adjustment 启用被 Definition 禁用的工具而未产生 Approval → 失败（F9）。
7. 仅 profile 变更却把旧 Lock 证据计为新契约 / 基座的 pass → 失败（I13）。

---

## 6. Runtime 基座与执行点

### 6.1 Harness → Runtime：登记与资格化

```text
harness 资产（published）──install 到 Node──▶ runtime.register(descriptor)      # 运行平面对象，不是 asset.kind
   ──▶ runtime.qualify(runtime_id) ──describe()──▶ 形状校验 · describe_digest 重算 · bundle_digest
        · 与 harness 资产 manifest 互证 · Policy 评估
   ──▶ runtime/qualified{runtime_id, harness_asset_id, control_points[], native_effect_tools[],
                         describe_digest, bundle_digest, policy_decision}
```

- `describe()` 五字段字面继承：`{type, version, control_points[], native_effect_tools[], describe_digest}`。
  新增校验：`describe()` 的声明必须与该 Harness 资产 manifest 的 `provides.control_points[]` / `native_effect_tools[]` 一致，
  不一致 → 拒绝资格化（资产声明与实例声明互证；负例：不一致却 qualified → 失败）。
- 资格化纪元与失格原因继承四种并新增第五种 `harness_asset_revoked`（Harness 资产版本被 `asset/revoked`）。
- 多 Runtime 并存、资格化证据不跨 id 转移、精确钉版与 CI 漂移预警继承 v0.7.0 07 §1.4 / §7；资格化恒由中心完成。
- `start_execution` 大声失败清单继承：缺 required 控制点、未资格化或已失格、Policy 拒绝 `native_effect_tools`、
  Lock 含被隔离 / 撤销资产、适配器前置条件不满足 → `not_runnable.environment{missing[]}`，**不降级运行**。

### 6.2 Runtime ABI 最小面

| 方向 | 方法 | 义务 |
|---|---|---|
| OS→RT | `describe()` | 五字段 |
| | `start_execution(execution, lock, bindings, grants_summary, inputs)` | 携带 Lock（资产 digest 集）与**结构化绑定**（Lock、Grant 摘要、Activity.continuity、Memory 检索引用，各带 label）；**不携带渲染好的上下文**——Harness 按 Lock 自行装配；大声失败清单见 §6.1 |
| | `resume_execution(execution, checkpoint_ref, watermarks)` | Effect 水位 + Approval 水位；仅在 OS 完成对账与重新武装后调用 |
| | `cancel(execution, reason)` / `health()` | 必须 |
| | `deliver_input(execution, input)` | **可选**：Harness 若支持进程外输入则实现；否则以 `execution.observe` 轮询或由适配器投递。OS 不再规定投递管道 |
| | `checkpoint(execution)` | 可返回 `unsupported` |
| | `park(execution)` / `thaw(snapshot)` | 控制点 `session_park`（可选）：长任务 HITL 释放进程；可返回 `unsupported` |
| RT→OS | `os.report_progress` / `os.report_failure`（脱敏 ≤ 4096 B + `diagnostics_ref`）/ `os.request_resource` / `os.propose_adjustment` / `os.submit_handoff` | 继承 |
| | `os.observe(execution)` | 查询 Approval 结果（仅结果枚举）、待投递输入、Grant 摘要、预算余量、生效 EP——**替代** `fysun.await_approval` / `fysun.pending_inputs` 系统工具 |
| | `os.submit_candidate` / `os.report_receipt` / `os.report_observed` | 仅当 Harness 自身作为执行点（`harness_adapter` 类 EP）时使用 |

**不得调用清单继承**：Runtime 不能签发 Grant、推进 Execution 权威状态、决定 Approval、改写标签、写权威账本
（`mint_grant` / `commit_effect` / `complete_activity` / `write_ledger` / `mark_verified` / `decide_approval` 不存在于 ABI）。
`execution.observe` 返回 Approval 的**结果枚举**，不含 `decided_by` / `reason` / `presentation`（审批事实不进模型可见面，T11）。

**传输**：本机 stdio JSON-RPC（R1；Windows named pipe 为 S6 spike）；跨机 gRPC / HTTP JSON-RPC 为开放问题 Q1，不在本说明拍板。
拒绝把 Arrow / FlatBuffers 前置为内核义务。

### 6.3 控制点与声明位

六个控制点继承 v0.7.0 05 §3：`context_injection`（语义改为"OS 提供**结构化视图**，Harness 负责渲染"；每次注入在会话账本写一条带
`context_view.digest` 的事件）、`checkpoint`、`fork`、`compaction_hook`、`session_link`、`subagent_delegation`
（只有 Handoff 状态迁移与 deliverables 入治理账本）。

新增**声明位**（非强制，`unsupported` 合法）：

| 声明位 | 语义 | 资格化后果 |
|---|---|---|
| `cache_alignment` | Harness 保证稳定前缀装配（身份 → 工具 schema 索引 → 静态 skill 元数据 → 租户/会话准静态 → 轨迹 → 易变值） | **不是资格化硬条件**。声明该位的 Harness 可跑前缀漂移探针；失败只清该位并告警（`runtime.cache_alignment_violated`），不拒绝最小面（对评审 P1-3 硬失败方案的拒绝裁定） |
| `session_isolation` | 会话级租户隔离 | `shared` Runtime 池化的前提（R3 缺省 `dedicated`，`shared` 延后至 Q24 有隔离证明） |
| `session_park` | 支持 `park`/`thaw` 快照 | 可选控制点；不承诺 KV cache / 模型权重迁移 |
| `tool_search` | 支持只把工具名 / 描述索引放进上下文，schema 按需加载 | OS 相应提供工具索引而非全量 schema |

OS 附带 `prefix_order[]` 绑定顺序建议（不渲染文本）；`prompt` 资产 `cache_hint` 区分 `prefix` / `volatile`；度量
`cache_read / input_tokens`（provider 回传），基线先测、阈值后定。

### 6.4 执行点（Enforcement Point，EP）

执行点**站在数据面上**、按 `fysund` 下发的 Policy 与 Grant 强制边界、并把事实回报 `fysund` 的组件。`fysund` 自身不是执行点。

| `kind` | 位置 | 强制什么 | 回报什么 | 参考实现 / MVP |
|---|---|---|---|---|
| `gateway` | 服务入口 | 租户鉴权、配额、速率、Plan 允许的 `effect_category`、Endpoint 协议 | `invocation/*`、`meter/recorded` | `fysun-gateway`；R2 |
| `mcp_guard` | Harness 与工具 / MCP server 之间 | `tools/list` 不可区分过滤；`tools/call` 分级 F3；MCP `sampling/createMessage` 防御；下游凭据注入 | `action-intent/*`、`effect/*`、`audit/observed{path: guard}`、`resource/metered` | `fysun-mcp-guard`（**独立进程，可选**；不再链进 `fysund`——ADR-0012"不嵌入"保留、"不前置"按 EP 框架重开）；R2（或由 adapter 覆盖） |
| `hooks` | Harness 进程内 hook | 对原生工具 deny / abstain（**只能拦不能记**，不产生 Effect / receipt / 标签；`fysund` 不可达时退出码 2 fail-closed） | `action-intent/rejected{path: hook}`、`audit/observed{path: hook, verdict: abstain}` | `fysun hook pre-tool-use`；可选 |
| `harness_adapter` | Harness 内部拦截点 | 原生工具 persist + receipt（如 DSH `approval/request` 拦截 + `tools/post-execute`）、控制点义务 | 同 `mcp_guard`，`path: adapter` | `fysun-dsh`；R1 可用最小集 |
| `provider_proxy` | Harness 与模型 Provider 之间 | 凭据注入、Token / 费用计量、模型允许列表 | `resource/metered` | 可选；对外 token 配额场景见 §8.5（Q26） |
| `external_gateway` | 第三方网关（如 agentgateway） | 接受 Policy 快照（Cedar）并按 EP 契约回报 | 同 `mcp_guard` 子集 | 延后评估（R3 后 spike S7）；不嵌入 `fysund` |

**无 EP 的 Runtime**：只有生命周期控制、预算（以时间与调用数计）、账本；F3 / F7 / F9 声明为 `not_covered`；
Policy 可拒绝把含非 `local_read` 效果的 Lock 绑到它。

### 6.5 EP 契约与 Policy 下发

```text
ep.describe() -> {kind, version, capabilities[], describe_digest}
                 # capabilities: filter_tools / persist_intent / receipt / labels / flags / secret_inject / meter
ep.policy_pull(scope) -> {policy_version, digest, bundle}   # 版本化 Cedar 快照 + Grant 有效集投影 + Lock 派生工具面
ep.report(events[])                                         # 批量回报（可微批）
ep.resolve_secret(SecretRef, execution_id) -> value         # 只对 qualified EP、按 Grant；每次重解析
ep.health()
```

- **资格化**：`ep.register` → `ep.qualify`（`describe_digest` 重算、Policy 评估、能力位检查）→ `ep/qualified`（准入类）；
  能力位变化 / Policy 换版 → `ep/disqualified`。未资格化 EP 的回报被拒（`ep.report_rejected` + `ledger/conflict{kind: ep_report_rejected}`），
  其 `resolve_secret` 被拒（I8）。
- **Policy 下发为主，approve 回中心**（D5）：EP 拉取快照本地评估，回报带 `policy_version`；`fysund` 在 `policy/revised` 后推送失效通知，
  EP 在 `policy_stale_ttl` 内未刷新则自行 fail-closed（只允许 `local_read`）。`outcome = approve`、任何 Grant 扩张、Memory 准入中需人的部分
  **回中心**：EP 提交候选，`fysund` 签发 Approval，EP 以 `os.observe` 语义等待或返回 `needs_human(approval_id)`。
  **EP 不可决定 Approval、不可签发 Grant、不可本地写 Effect 终态**（I1）。
- **同一 Policy 文本、同一 `policy_version`、同一输入 → `fysund` 与 EP 评估 `outcome` 必须相同**（I9；Cedar 实体扩展按 R0 修订的
  ADR-0009 封闭表，EP 禁止自造 context 键、禁止本地补丁）。
- **回报核对**：`fysund` 对每条回报校验 `ep_id` 资格化、`policy_version` 在允许窗口、幂等键唯一（`conflict.idempotency`）、Grant 未过期。
- 一个 Execution 可同时有多个 EP（gateway + mcp_guard + hooks）；**同一工具调用只能由一个 EP 产生 Effect**（重复 → `conflict.idempotency`）。
- EP 崩溃恢复时先回报未回报事件再接受新调用；`fysund` 对"有 `dispatch-started` 无 receipt"的 Effect 走 outcome-unknown 对账（继承）。
  `authorized` 未分发 Effect 的 TTL 由 EP 契约定义（并入原 Q21 → Q25）。
- **`fysund` 热路径禁令**：不转发 `tools/call`、不代理模型请求、不持有下游 MCP 会话、不渲染上下文。与数据面的全部交互是：
  下发（`policy_pull`）、接收回报（`report`）、解析 secret（`resolve_secret`）、签发 Approval / Grant（I2 负例）。

### 6.6 F3 分级热路径（对 ADR-0008 D5 的显式继承；评审 P0-1/P0-3 的裁定）

对每次 `tools/call` / 原生工具候选，按**合成后**的 `effect_category`（三源合一：资产 `declares.effects[]` ∪ 目录缺省 ∪ Policy 覆盖；
取 `max_severity`；未分类 → `external_mutation`）分三档：

| 档 | 类别 | 流程 |
|---|---|---|
| 快路径 | `local_read` | EP 内存 Cedar + Grant + Scope + 来源标签检查 →（与其它只读**并发**）分发 → `audit/observed{path, effect_category: local_read}`（可异步微批刷盘，经 EP 环形缓冲补送；补送失败 → Execution `waiting{reason: audit_backlog}`，不得静默丢）。**不建 Effect、不持久化 ActionIntent** |
| 完整协议（轻量持久） | `external_read` | 完整 F3：EP 本地记录 ActionIntent（可组批写，**不阻塞同步 fsync**）→ Policy（通常 `allow`）→ 分发 → receipt → 回报。**不得免持久化**——账本必须能回答"这条 `untrusted_external` 来自哪次外部读取"（H-2 承诺不变） |
| 完整协议（同步持久） | `local_mutation` / `external_mutation` / `credential_use` / `physical_action` | EP 本地 **persist-before-dispatch**（同步 WAL + 幂等键 + `policy_version` + `ep_id`）再分发 → receipt → 回报；`fysund` 以回报重建 Effect 投影 |

并行规则：同一轮内多个 `local_read`（及已 `authorized` 且 Policy `allow` 的 `external_read`）允许并发；任何 mutation /
`credential_use` / `physical_action` 与未决 Approval 保持串行；同一批次含 mutation 时先完成 read 再 persist mutation。
**明确拒绝**：把 `external_read` 放进无持久化快路径（与已采纳 ADR-0008 D5 / H-2 直接冲突）；
对每次 `tools/call` 同步回 `fysund` RPC（把 EP 做成新的 `fysund` 热路径）。

关键负例：`local_read` 无 `audit/observed` → 失败；`external_read` 只有 `audit/observed` 无 `action-intent/recorded` → 失败；
EP 把 `external_read` 标成 `local_read` 以走快路径 → 失败（类别合成三源合一，EP 不能单方降级——T13 变体）。

### 6.7 Activity / Execution 生命周期、恢复与重新武装

继承 v0.7.0 02 §2.2–2.3、03 §6：状态机、`waiting{reason}`、`input_required`、审批容量与限流
（`max_pending_approvals` / `approval_sla` / `approval_rate_limit` / `consecutive_rejections_alert`）、Handoff 与 A2A TaskState 对照。

变化：

- `Execution.runtime_binding` 增 `lock_digest`、`harness_asset_id`、`ep_ids[]`；`Execution.tenant_id`、`Activity.tenant_id`、`Activity.service_ref?`。
- 调度：`fysund` 拥有触发（人工、周期、事件、webhook、Service 调用）与 Runtime 选择（`compose.match` × Policy × 负载）；
  池化缺省 `dedicated`，`shared` 要求 `session_isolation` 且延后（Q24）。`session_mode: per_call | per_context` 时 gateway 可持有
  execution lease 以减少每条消息的控制 RPC（lease 失效回 `fysund`）。
- **恢复顺序（固定）**：账本投影 → Policy 版本校验 → Runtime 资格化校验 → fencing epoch 递增 → outcome-unknown 对账 →
  Approval 水位核对 → **重新武装** → `resume_execution` / `thaw`。含非 `local_read` 类别的 Activity 恢复后缺省
  `paused{reason: rearm_required}`，须 Principal `activity.resume` 或 Policy `auto_resume`；武装事实由账本事件 `activity/resumed` 承载。
- **`session_park`（可选）**：Activity 进入长时间 `input_required` / `waiting{approval}` 时，Runtime 可 `park` 交出
  `ExecutionSnapshot`（对话摘要、未完成 step、待输入标记 + `lock_digest`；**不含**权重 / KV cache）并释放进程；
  快照入 Artifact；唤醒走重新武装 + `thaw` / `resume_execution`；`thaw` 到错误 `lock_digest` 拒绝（I14）。
  这是 Temporal 式 durable wait 的模式借鉴，不引入 Temporal 依赖、不重写账本。

### 6.8 多租户隔离（运行侧）

- 每个 Activity / Execution 有 `tenant_id`；Grant 绑定 `(principal, scope)`，scope 含 tenant 前缀；跨租户 Handoff 必须经 Service 调用（A2A），
  不能直接引用对方 Execution。
- Memory / Knowledge：`memory.scope = tenant` 时 provider 必须支持命名空间隔离；跨租户 Memory 读取一律拒绝。
- Secret：`SecretRef` 归属 tenant scope；EP `resolve_secret` 校验 Execution 的 tenant。
- 租户数据驻留：Plan `data_residency` 约束 Runtime 选择（Node 标签）与 memory provider 选择。

### 6.9 Runtime / EP 负例

1. `describe()` 声明与 Harness 资产 manifest 不一致却 `runtime/qualified` → 失败。
2. `start_execution` 载荷含渲染文本上下文而非 Lock + 结构化视图 → 失败。
3. 未资格化 EP 的 `ep.report` 被接受 → 失败。
4. EP 本地写出 `approval/decided`、`grant/issued` 或 `effect/verified` → 失败（I1）。
5. `policy_stale_ttl` 超期的 EP 放行非 `local_read` 候选 → 失败。
6. 同一工具调用由两个 EP 各产生一个 Effect → 失败。
7. `shared` 池化下 Harness 无 `session_isolation` 仍被绑定多租户 → 失败。
8. `fysund` 进程出现在工具调用或模型请求转发链 → 失败（I2）。
9. 每次 `tools/list` 都 RPC 到 `fysund` → 失败（架构测试）。
10. `local_read` 被强制同步 WAL 才返回结果 → 失败（快路径语义破坏）。
11. `thaw` 到错误 `lock_digest` 仍恢复 → 失败（I14）。

---

## 7. 治理（横切）

### 7.1 Principal / Scope / Tenant

- Principal 类型：`User` / `Tenant` / `Execution` / `Runtime` / `EnforcementPoint` / `ExternalAgent` / `Publisher`。
  **Execution 是可授权主体**——委派出去的工作拥有独立身份与预算；Approval 的决定者必须是人或其授权系统，不能是 Runtime / EP / 外部 Agent。
- **Tenant = Principal + Scope**（治理对象，不是服务资产，不与 Service 平级）；系统级对象（Harness 资产、Policy 模板）在 `system` scope；
  跨 scope 可见需显式 Grant 或 `tenant_visibility = public`。多主体缺省：R1 单租户合法，但对象模型、事件字段 `tenant_id`、Scope 前缀
  从 R1 就存在（关闭 v0.7.0 Q5）。

### 7.2 Grant

- 继承字段与三条衰减不等式（`scope ⊆ 父`、`expiry ≤ 父`、`budget ≤ 父剩余`；负例向量）；新增 `root_of: plan_ref?`
  （由 Plan 派生的根 Grant）与 `ep_scope?`（限定只能由哪些 EP 使用）。补偿必须**新 Grant + 新 ActionIntent**，拒绝复用被补偿 Effect 的原 Grant。
- EP 从 `ep.policy_pull` 得到 Grant 有效集投影；EP 不能签发。Escrow 子 Grant 与边缘规则继承 ADR-0010（§10.5）。

### 7.3 Policy（Cedar 子集，版本化下发）

- 对象、版本化、Cedar 子集、`outcome` 四值（`allow / deny / modify / approve`，唯一 Policy 输出）、注解表达约定
  （`@fysun_outcome` / `@fysun_obligation` / `@fysun_rule_id`）、Cedar schema 校验全部继承 ADR-0003/0009。
- **实体模型扩展（须先修订 ADR-0009，R0 工作项）**：principal 增 `Tenant`、`EnforcementPoint`、`Publisher`；resource 增
  `Asset{kind, namespace, signed, publisher}`、`Service`、`Endpoint`、`Plan`；action 增 `Asset::"publish"`、`Service::"publish"`、
  `Service::"invoke"`、`EP::"qualify"`、`Runtime::"qualify"`、`Plan::"attach"`；context 增 `tenant_id`、`plan_ref`、`ep_kind`、`ep_id`、
  `lock_digest`、`asset_signed`、`asset_namespace`（继承 `native_effect_tools`、`offline` 等固定键）。新增键保持封闭表，禁止 EP 自造。
- **决策位置**：`fysund` 评估资产发布、Service 发布、Runtime / EP 资格化、Grant 签发、Approval 创建、Memory 准入；
  EP 本地评估 Effect 候选与 `tools/list` 过滤。同一 `policy_version`、同输入两处同 `outcome`（I9）。
- Policy 至少回答（在 v0.7.0 03 §7 清单上增）：允许哪些 namespace / registry 的资产、是否要求签名、哪些 kind 可由租户自带（BYO）、
  Runtime 池化策略、`policy_stale_ttl`、审批路由、`canary` 上限、数据驻留约束、计量可信要求（Q26）。

### 7.4 Approval

- 结果集封闭 `approved_once / rejected / cancelled / unavailable`（唯一非终态 `pending`）、配对事件 `approval/asked` / `approval/decided`
  （任一无法落账以 `unavailable.approval` 拒绝整个请求）、请求不复制参数（`subject_ref` 引用）、`options` 决定 `derived` 不决定 `result`、
  只有 `approved_once` 派生 Grant / Adjustment、到期 `on_expiry` 缺省 `deny`——全部继承 ADR-0011。
- `subject` 增 `asset_publish`、`service_publish`、`plan_attach`（当 Policy 对这些准入产出 `approve`）。
- **审批路由**（服务场景"谁批"，Plan / Policy 决定）：消费者数据上的外部 mutation → 租户 `approval_contacts`（运营者兜底）；
  Grant 扩张 / 扩权 Adjustment → 发布者 + 租户双签（`quorum: 2`）；Memory 准入 → 租户；供应链隔离解除 → 运营者。
  `decided_by` 必须在路由允许集合内，否则 `unavailable`。
- 审批面仍只由 `fysund` 提供（Web UI / CLI / `fysund` 发起的 elicitation）；gateway 与 EP 只透传 `needs_human(approval_id)`；
  Harness 文本与自绘卡片不构成审批面（T11）；审批事实不进入模型可见面。

### 7.5 来源与完整性标签

- `label{source_principal, integrity, origin_ref, flags?}`；四级 `integrity`；**工具结果一律 `untrusted_external`**（无例外），
  唯一升级路径 Verification / Memory 准入；`flags ⊆ {authority_claim, secret_shaped}`；标签只降不升；Runtime 不能改写——全部继承 ADR-0004。
- 赋予责任：EP（工具结果、外部 Artifact）、Harness 控制点（context_view 片段）、`fysund`（Lock、Grant 摘要等结构部分为 `trusted_system`）；
  `knowledge` 资产按 `integrity_default`。
- 资产层新增：`prompt.variables[]` 可要求 `integrity >= trusted_user`，Harness 装配不满足 → EP / 控制点回报
  `context/assembled{violations[]}`，Policy 决定 `deny` / `approve`。
- MCP Sampling 兼容窗口内的采样请求视为 `untrusted_external` 候选（§8.6）。

### 7.6 Effect 协议与 Verification

- ActionIntent → Effect → Receipt → Verification 状态机、幂等键（canonical 参数 digest；同键同参数重放、显式同键不同参数
  `conflict.idempotency` 硬拒绝）、outcome-unknown 只能 reconcile 或 quarantine——继承，**执行位置在 EP**（§6.6），`fysund` 核对投影。
- Verification 继承（`verifier_kind`、`independence`、`calibration_ref`、`inconclusive ≠ pass`；`human` 是一级 verifier 类型，
  `approved_once` 决定即该项 Verification）；证据归属见 §5.4 与 §9.1。
- 最低 verifier 要求可按 Service / Plan 设定（例如对外发布类动作至少 `deterministic` + `human`）。

### 7.7 资产供应链治理

| 威胁 | 机制 |
|---|---|
| 恶意 / 篡改资产 | digest 重算；签名要求（Policy）；namespace / registry 允许列表；`asset/quarantined` / `asset/revoked` 级联到 Lock 与 ServiceVersion |
| MCP server 工具语义漂移 | `tools_snapshot` 与实际 `tools/list` 比对（EP 回报）→ 待重新分类；漂移期间该 server 工具按 `external_mutation` |
| Harness 声明不实 | `describe()` 与 manifest 互证；conformance 只对经 EP 的面宣称；声明不实不在承诺范围 |
| Loop 参数越界 | `loop.params ⊨ harness.loop_schema`；预算仍由 Grant 兜底 |
| 提示词注入经 `knowledge` / `prompt` 资产 | 资产 `integrity_default`；`prompt.variables` 来源约束；`flags` 分类 |
| 租户自带资产越权 | BYO 只能覆盖允许槽位；**效果类别不可降级——校验在 `fysund`（compose / adjustment），EP 再强制一次**（defense in depth，Q29 处置） |

### 7.8 可观测性

- OTLP 导出继承；trace 关联键增 `tenant_id`、`service_id`、`invocation_id`；`fysun.*` 为自有命名空间与权威 SLO 键。
- **`gen_ai.*` 处置（Q15 收窄）**：OTel GenAI 语义约定截至 2026-08 仍全部 Development 且无 tagged release——导出可带 `gen_ai.*`
  （钉仓库 + commit SHA / schema URL），但**账本 schema 与 SLO 不得把未稳定语义当强制契约**；核心键走 `fysun.*`；
  准备 collector 改名层。拒绝"强制稳定 `gen_ai.*`"。
- 度量增：每 Service 调用成功率与时延、每租户用量、EP 回报延迟与拒绝率、`policy_stale` 次数、资产隔离次数、审计微批补送成功率。

---

## 8. Agent as a Service（服务平面）

### 8.1 角色与对象层级

角色：发布者（Publisher）/ 运营者（Operator）/ 消费者·租户（Tenant）/ 审核者（Reviewer）；同一 Principal 可兼多角色，
单机个人部署三者合一。服务平面**一级对象只有 Service**：

| 名称 | 层级 | 说明 |
|---|---|---|
| Service | 服务一级 | `agent_ref`、`versions[]`、`endpoints[]`、`plans[]`、`agent_card_ref`、`status`、`sla_declared?`、`tenant_visibility` |
| `versions[]` | Service 字段 | 钉 `lock_digest` + rollout 阶段；版本单位是 Lock，同一 Lock 不同 Endpoint 配置不是新版本 |
| `endpoints[]` | Service 绑定 | `protocol ∈ a2a \| mcp \| http`、`url`、`auth`（tenant token / mTLS / OAuth）、`exposes`；由 gateway 暴露，无独立 `endpoint.*` CRUD |
| Tenant | 治理对象 | Principal + Scope；资产 / Memory / Activity 的归属 |
| Plan | 附着 | Tenant×Service 约定：`quota{tokens, calls, effects{category}, runtime_seconds, concurrent_executions, approvals, sampling}`、`rate`、`allowed_effect_categories[]`、`approval_routing`、`data_residency?`、`period`；`plan/attached` 为准入事件；**派生该租户对该 Service 的根 Grant** |
| Meter | 事件投影 | `meter/recorded` + `meter.query` 投影；无独立记账表、无 `meter.create` |

### 8.2 发布（含 EP 覆盖门——不变量 I6）

```text
compose.resolve → Lock
service.publish(service, lock_digest, endpoints[], plans[])
  ├─ Policy 评估（发布者可发布哪些 effect_category、是否要求签名 / 证据门）
  ├─ EP 覆盖检查（I6）：该 Lock 的非 local_read 工具面被至少一个 qualified EP 覆盖；
  │   否则只允许 draft / staged，或仅对纯 local_read Lock 发布
  ├─ 可选 evidence_gate（回归集 Verification 全 pass 才允许标 stable）
  ├─ 生成并签名 Agent Card（A2A 1.0）：skills[] 由 capabilities[] 派生；security schemes 由 Endpoint.auth 派生
  └─ service/published{service_id, version, lock_digest, endpoints[]}（准入类）
service.rollout(staged | canary{percent} | stable | rolled_back)   # canary 按租户或调用比例路由，必须可审计
```

- **发布资格门**是两份 2026-09-12 评审一致要求的 P0 修正（原 redefinition 草案允许 R3 在 EP 闭合前把 mutation 服务标成
  `published`）：把带 `external_mutation` 工具的 Agent Card 公开出去而 PEP 仍是"指定"状态，等于把无守卫的 mutation 面标成服务。
  Personal 单租户允许绕过 gateway（CLI `activity.create` 直达，Tenant = owner、Plan = unlimited），**不允许绕过 EP 覆盖门**。
- Lock 中任一资产 `quarantined` / `revoked` → 相关 ServiceVersion 自动 `suspended`，事件级联。
- 消费者可在 Plan 中钉版本（`pin: version`），否则跟随 `stable`；回滚 = `service.rollout(prev, stable)`，运行中 Execution 不迁移。

### 8.3 发现与调用

- `service.search` 返回摘要 + Agent Card + 分层证据 + health；**MCP facade**：`protocol = mcp` 的 Endpoint 把 Service 暴露为一个
  MCP server，每个 Capability 一个工具（输入 = `behavior.input` schema），长任务经 MCP Tasks / A2A 异步。发现不隐含授权。
- 调用路径（热路径不含 `fysund` 转发）：

```text
消费者 → fysun-gateway（鉴权、Plan 配额本地快照评估）
       → fysund：invocation/received；create_or_wake Activity；从 Plan 根 Grant 衰减签发 Grant；compose.match 选 Runtime
       → Runtime.start_execution(lock, 结构化绑定)
       → Runtime ↔ 工具侧 EP（本地快照，F3 分级）
       → 回报与进度回 fysund；gateway 映射 A2A TaskState
```

一次调用 = 一个 Activity（或唤醒既有长期 Activity，取决于 `session_mode`）；A2A `contextId` ↔ `Activity.id` / Handoff 链根；
`input_required` ↔ 用户输入、`auth_required` ↔ Approval。gateway 是水平可扩展的无状态组件（状态在 `fysund` 与其本地队列）。

### 8.4 身份、委托、长任务、HITL

- **身份**（D8）：Tenant 凭据（token / mTLS / OAuth）终止于 gateway；Execution 是可授权 Principal；Handoff / 外部 A2A 调用签发衰减
  Grant 即委托链（ASI03）。**SPIFFE 不进 R0–R3 内核**；作为 EP / Runtime 跨机身份的可选后端延后评估。禁止长寿命静态 tenant token
  作为多租户生产缺省。
- **MCP HTTP 授权**（Q P1-8）：`mcp_server` 资产增 `auth: none | oauth2`；gateway / `mcp_guard` 按 MCP Authorization 规范做
  OAuth 2.1 + RFC 9728 PRM 发现 + RFC 8707 resource indicator audience 绑定；错 audience → 拒绝（负例）。
  SecretRef 只持 refresh / client 凭证，不把下游 token 写进账本。stdio 传输仍用环境凭据（规范允许），不套该规范。
- **委托链**：跨租户只能经 Service 调用；外部 Artifact 一律 `untrusted_external`（ASI07）。
- **长任务**：Activity 可无终点；长时间 HITL 用 `session_park` 释放 Runtime（§6.7）；恢复必再武装。
- **HITL**：审批路由见 §7.4；gateway 只透传 `needs_human(approval_id)`；Harness 文本不构成审批（T11 / ASI09）。

### 8.5 配额、计量与预算

- Plan → 根 Grant（预算维度含 `sampling`）；Execution Grant 从根衰减；超限 → `quota.exceeded{dimension}`，
  Execution `waiting{reason: budget}` 或 `failed`，**不得再 `execution/started`**。
- 计量来源：gateway（calls、并发）、工具侧 EP（effects）、`fysund`（时长、审批）、可选 `provider_proxy`（tokens）。
  **Harness 自报 token 标 `untrusted_external`，缺省不计入配额、只能作对照**；凡 Plan 含 token 配额或对外账单导出的 Service，
  `provider_proxy`（或等价可信计量 EP）为资格化前提（Q26 处置）。仅 Harness 自报 token 导致他户 `quota.exceeded` → 失败（N-12）。
- **不做计费**：`meter/recorded` 可经 OTLP / 批量导出给外部计费；价格只在 `model.pricing_ref` 作换算提示。

### 8.6 协议锚点（2026-09 现行版钉版）

| 标准 | 钉版与要点 | 用途 | 边界 |
|---|---|---|---|
| **MCP 2026-07-28** | 无状态核心、可缓存 `list`、扩展框架、授权加固；**Sampling（SEP-2577）/ Roots / Logging 弃用**（新实现 SHOULD NOT 采用，≥12 个月兼容窗）；**Authorization：OAuth 2.1、RFC 9728 MUST、RFC 8707 resource indicator、DCR 弃用 / CIMD SHOULD**；Tasks 在扩展框架下 | 工具面协议：`mcp_server` 资产接口、`mcp_guard` EP 拦截面（`tools/list` / `tools/call` / `resources` / `prompts` / tasks / elicitation）、Service MCP facade | 不用于 `fysund` ↔ Runtime 控制通道（那是 Runtime ABI） |
| **Sampling 防御（R2 义务）** | 新实现默认不声明 sampling capability；**缺省拒绝 `sampling/createMessage`**；兼容窗口内（旧协议版本 server）：采样请求视为 `untrusted_external` 候选、计入 Execution token 预算、剥离 `trusted_system` / Secret / Slot-0 身份提示、Policy 缺省 deny；超限 → `quota.exceeded{dimension: sampling}` | `mcp_guard` EP 义务 | 对应 OWASP ASI01/ASI02/ASI06 |
| **A2A 1.0**（AAIF Growth Stage，2026-08-27） | Card 字段按 1.0：`supportedInterfaces[]`（JSON-RPC / gRPC / HTTP+JSON）、`protocolVersion` 下放到 interface、`capabilities.extendedAgentCard`；`A2A-Version: 1.0`；Card 由 `fysund` 生成并签名 | Service 主 Endpoint 协议、Agent Card、跨租户 / 跨基座调用、TaskState 映射 | 基座内部 Handoff 走账本与 Runtime ABI |
| **Agent Skills** | `SKILL.md` 开放格式；三级渐进披露（name/description → 正文 → 资源）；OS 强制按需加载契约（`skill_loading` 必须能只加载 name+description） | 唯一 skill 包格式 | 不自造包格式 |
| **OCI artifact** | 全部 kind 的打包分发；digest 即身份；任意 OCI 兼容仓库 | `fysun asset pack/push`；Lock digest | 不用于运行时镜像编排 |
| **Cedar（子集）** | Policy 语言；下发到 EP 的快照格式；实体扩展走 ADR-0009 修订 | 治理横切 | 不用于 Plan / 计量表达 |
| **OpenTelemetry** | OTLP 导出；`gen_ai.*` 钉 commit（Development 状态）；`fysun.*` 自有命名空间 | 可观测性 | 账本 schema 不依赖未稳定约定 |
| **OpenAPI** | HTTP Endpoint 描述，由 Capability `behavior` schema 派生 | HTTP Endpoint | — |
| **hooks.json（Claude Code / Codex / DSH 兼容）** | `hooks` 类 EP 接入面 | 原生工具尽力阻断 | 不作安全边界宣称 |

### 8.7 OWASP Agentic Top 10（2026）对照（覆盖声明，不另造分类学）

| ID | 风险 | 控制机制 |
|---|---|---|
| ASI01 目标劫持 | 注入改写目标 | 来源标签；`prompt.variables` 完整性约束；`knowledge.integrity_default`；F2 |
| ASI02 工具滥用 | 合法工具被链式误用 | 绑定期工具面收窄；F9；未分类 = `external_mutation`；sampling 预算 |
| ASI03 身份与权限滥用 | 委托链过宽、旧凭据 | Execution Principal；Grant 只衰减；Secret 每次重解析；跨租户禁直连 Execution |
| ASI04 供应链 | 第三方 skill / MCP / harness | digest、签名 Policy、snapshot 漂移检测、quarantine / revoke 级联 |
| ASI05 意外代码执行 | Agent 生成代码 | 原生工具须声明；无 EP 则 `not_covered`；沙箱不在内核 |
| ASI06 记忆 / 上下文投毒 | 脏记忆污染后续 | Memory 准入；工具结果 `untrusted_external`；租户命名空间 |
| ASI07 不安全 A2A | 伪造 / 篡改委派 | 签名 Agent Card；外部产物 untrusted；内部 Handoff 走账本 |
| ASI08 级联失败 | 单点扩成全局 | fencing；预算；审批限流；canary；revoke 停新流量 |
| ASI09 HITL 利用 | 骗过审批人 | 审批面只在 `fysund`；`subject_ref` 不复制参数；Harness 文本无效 |
| ASI10 Rogue | 漂移超范围 | 资格化纪元；F9；优化器 Immutable Zone；quarantine |

### 8.8 服务负例

1. 未 `service/published` 的 Lock 可被 Endpoint 调用 → 失败。
2. 租户 A 读到租户 B 的 Memory / Artifact / Secret → 失败。
3. Plan 禁止的 `effect_category` 被 EP 放行 → 失败。
4. `quota.exceeded` 后仍 `execution/started` → 失败。
5. Agent Card `skills[]` 与 Lock `capabilities[]` 不一致 → 失败。
6. 资产 `revoked` 后 ServiceVersion 仍 `stable` 并接受调用 → 失败。
7. gateway 本地 `approval/decided` 或转发 Harness 文本为审批决定 → 失败。
8. canary 流量比例与 `service/rollout` 事件不一致（不可审计）→ 失败。
9. **无 qualified EP 覆盖 mutation 工具面的 Lock 被 `service/published` 且 Endpoint 可调用 → 失败（I6 / N-11）**。
10. Card 顶层仍用 0.3 的 `url` + `preferredTransport` 且无 `supportedInterfaces` → 失败。
11. MCP access token 错 audience（resource indicator 不匹配）被接受 → 失败。
12. sampling 兼容窗口内把 `trusted_system` 或 Secret 回传给 server → 失败（N-05）。

---

## 9. 演进与自进化

### 9.1 三种变化与证据分层

| 变化 | 载体 | 检查 |
|---|---|---|
| 调整（Adjustment） | Lock 级覆盖 / Definition 补丁版本 | 触及扩权、新外部动作、预算上调、设备安全时走 Policy；其余直接生效、留痕、可回退 |
| 资产升级 | 新资产版本 → 新 Lock → 新 ServiceVersion | 解析约束 + 可选证据门 + rollout |
| 契约变化 | Capability 新版本或新 Capability（谱系 `derived_from / forked_from / evolved_from`） | 契约不兼容默认派生新 Capability；Service 需新发布 |

证据归属四层 + 服务层：Verification → Capability（契约层）/ Agent Definition（组合层）/ Lock（三层 digest 各自）/ 各资产版本
（按 `attributable_to` 标注）；`downstream_outcome` → Service；conformance 结果 → harness / Runtime 实例 / EP。
配置声明与 `compatibility` 不算证据。质量表示沿用 Skill Contract (P,O,A,V,F)，V 的 checklist 项标 `verifier_kind`；
`inconclusive ≠ pass`；部署门需要 `human` 或已校准 `model_judge`（α 阈值：探索 ≥0.67、安全 ≥0.70、部署门控 ≥0.80，供参考不在规范写死）。

### 9.2 发布证据门与 canary

- `service.publish` 与 `service.rollout(stable)` 可由 Policy 要求 `evidence_gate`：对回归集跑一次，Verification 全部 `pass` 且无 `not-run`。
  **门看本 Lock 的非 `inherited` 证据**，或 Policy 明确允许"仅 profile 变更 + 回归集通过"（I13）。
- `canary` 阶段自动累积 Lock 级证据；Policy 可设"canary 期间 Verification 通过率 ≥ 阈值且人工介入率 ≤ 阈值"作为进入 `stable` 的前置。
- 影子评估：用已 `verified` 的脱敏输入重放（同一 Lock 契约 / 基座，仅换 profile）；未达 Policy 阈值不得 `stable`。
- 回滚 = `service.rollout(prev, stable)`；优化器失败不得留下半发布 Lock；**租户 A 的 Adjustment 不传播到租户 B**
  （除非发布者显式 promote 到 Definition 默认）。
- **SWE-bench Verified 不得作为对外证据门的回归集**（公开榜污染有一手陈述）；回归集用自有 Execution + 租户可选贡献的脱敏基准。

### 9.3 自进化最小闭环（Immutable Zone / 优化器插件）

内核提供**证据 → 归属 → Adjustment → 门控发布 → 回滚**；不内置 ACE / GEPA / DGM。优化器是可插拔生产者：

```text
Execution 结束 / 失败 / 低分 Verification
  → trajectory.export(execution_id)（脱敏轨迹：会话账本引用 + 治理事件 + checklist 结果；不含 Secret）
  → attribution[]：prompt | skill | loop | tool_surface | harness | model | 环境（人填或优化器提议、人/Policy 确认）
  → optimizer.propose(lock, evidence) → Adjustment[]（只许 profile 层）
  → 安全包络检查（下表）→ 证据门（回归集 + 可选影子）→ service.rollout canary → 指标 → stable 或 rollback
```

| 可变（Mutable Zone） | 不可变（除非 Approval + 新 substrate/contract） |
|---|---|
| Prompt 正文与 few-shot；在已允许槽位内换 prompt 版本 | `bounds.tool_effects[]` 升级 |
| Loop params（步数、重试、compaction 阈值） | 新增工具 / MCP server / 放宽 `reject_native_tools` |
| 检索 top-k、记忆抽取阈值 | Grant、配额、Plan `allowed_effect_categories` |
| 模型绑定顺序（fallback） | 设备安全、secret 范围 |

优化器插件边界：`optimizer.describe() → {targets: [prompt, loop_profile], needs: [trajectory, verification]}`；**不是 PDP**；
不能 `grant.issue`；不能标 Verification `pass`；不能读 Secret 明文；不得进入 `tools/call`、模型请求或 `compose.resolve` 热路径；
**禁止 Execution 内反射改 prompt**；登记为普通进程或 `harness` 资产（非新 kind）。越权 → `evolution.immutable_boundary`（I11 / N-07）。
内核提供 `trajectory.export`（脱敏）与 `adjustment.apply`。DSPy / GEPA 类优化器产出的是新 prompt / loop Profile 资产，经现有
`service.rollout` 晋升——GEPA 未在 FysunOS 回归集上验证 Pareto 改进前不写入内核承诺。

### 9.4 消费反馈

反馈只以 Verification 形式进入系统：`downstream_outcome`（后续 Activity 结果反推）、租户人工判定（`human` verifier，
`independence = different_principal`）。不设评分 / 点赞对象；反馈驱动的调整仍是 Adjustment，归属发布者或租户（BYO 槽位），不自动传播。

### 9.5 演进负例

1. 契约 / 基座未变，系统把旧 Verification 从发现界面抹掉（应保留 `inherited` 并降权）→ 失败。
2. `inherited` 证据被当作新 substrate 的 `evidence_gate` pass → 失败。
3. 优化器把 `external_read` 改成更低类别，或新增工具 / MCP server，未经 Approval → 失败（`evolution.immutable_boundary`）。
4. 影子集未跑过的 `stable` rollout（当 Policy 开启证据门时）→ 失败。
5. 独立于 Verification 的"评分"对象入账本 → 失败。
6. 租户 A 的反馈自动生效于租户 B → 失败。
7. 优化器进程出现在 `tools/call` / 模型请求路径 → 失败。

---

## 10. 系统架构、账本与恢复

### 10.1 平面拓扑

```text
消费者  A2A / MCP / HTTP / CLI
        │
        ▼
┌─ 服务入口 EP（fysun-gateway）──── 数据面 ──────────────────────────────┐
│  鉴权 · 配额 · 计量事件 · 调用 → activity.create_or_wake                │
└──────────────┬─────────────────────────────────────────────────────────┘
               │ 仅创建/唤醒（控制面 RPC，非每步 tool call）
               ▼
┌─ fysund 控制面 ────────────────────────────────────────────────────────┐
│  Catalog：登记 / resolve / 分层 Lock                                    │
│  Runtime 控制：qualify · 调度 · Activity/Execution · 恢复/再武装         │
│  Governance：Grant · Policy PDP · Approval · 唯一权威账本写入            │
│  Service 目录：publish / rollout / Plan→根 Grant                        │
└───────┬───────────────────────────────┬────────────────────────────────┘
        │ start/resume + 结构化绑定      │ policy_pull / 核对回报 / secret
        ▼                               ▼
┌─ Runtime（Harness 进程）─┐    ┌─ 工具侧 EP（mcp_guard / adapter / hooks）─┐
│  装配上下文 · 跑 loop     │    │  工具面过滤 · F3 分级 · sampling 防御      │
│  按需加载 skill           │◄──►│  本地评估 Policy 快照 · 微批回报           │
└──────────────────────────┘    └──────────────────────────────────────────┘
                                          │
                                          ▼
                              工具 / MCP server / Provider
```

### 10.2 组件与命名

| 名称 | 含义 |
|---|---|
| `fysund` | 控制面 daemon（Rust）：Catalog、Runtime 控制、PDP、Approval、Service 目录、唯一权威账本写入者。**不在数据面热路径** |
| `fysun` | CLI：`asset pack/push/register/publish`、`profile put`、`agent compose/resolve`、`service publish`、`runtime qualify`、`hook pre-tool-use` |
| `fysun-gateway` | 入口执行点（`kind=gateway`）：A2A server / MCP facade / HTTP；租户鉴权、配额、计量、把调用变成 Activity/Execution。独立进程，可水平扩展 |
| `fysun-mcp-guard` | 工具侧参考执行点（`kind=mcp_guard`），**独立进程、可选**；`tools/list` 过滤、`tools/call` 分级 F3、sampling 防御、Effect 回报；不再是 `fysund` 内组件（ADR-0012"不嵌入"保留、"不前置"按 EP 框架重开） |
| `fysun-dsh` | DeepSeek Harness 适配器：Harness 资产打包、Runtime ABI、控制点、原生工具执行点（`kind=harness_adapter`） |
| `fysund-edge` | 边缘节点精简 daemon（同一账本模型 + 本地账本的 EP 宿主；escrow、离线规则继承） |
| 资产仓库 | 任意 OCI 兼容制品仓库；`fysund` 只存登记与 digest |

仓库形态建议（继承 redefinition 开发计划 §5，可微调一次）：`schemas/`（JSON Schema draft 2020-12）、`conformance/`（按 EP 组合分列的声明式向量）、
`crates/`（fysun-types / fysun-schema / fysun-ledger / fysun-policy / fysun-assets / fysun-compose / fysun-runtime-abi / fysun-ep-contract / fysun-conformance）、
`apps/`（fysund / fysun / fysun-gateway / fysun-mcp-guard）、`adapters/fysun-dsh`（TS，独立 workspace）。
Rust 2021 + 钉 rust-toolchain；禁止第三个运行时进入 TCB（`fysun-dsh` 除外）。**CI 架构测试：`fysund` 二进制不得链接 MCP client 转发路径。**

### 10.3 event-sourced 治理账本

- 事件唯一真源；状态是投影；模型 chunk、KV cache、Loop step 计数、传感器高频采样不入账本。会话账本在 Harness，
  以 `execution_id` / `session_id` 关联，**关联不合并**。
- SQLite WAL 单机 + 编号迁移（多节点一致性 Q12 后置）；replay 两次投影 digest 必须稳定（金样测试）。
- 事件族见 §3.4；EP 回报必须带 `ep_id` + `policy_version`；核对失败 → `ep.report_rejected` + `ledger/conflict`。
- **治理账本 ⊆ `fysund`；EP 本地 WAL 是 pending replica**（I1）：EP 先本地持久化再分发再回报，`fysund` 以回报重建 Effect 投影；
  无回报 Effect 不得 `verified`；`dispatch-started` 无 receipt 走 outcome-unknown（继承）。

### 10.4 恢复模型

固定顺序（§6.7）：投影 → Policy 版本 → Runtime 资格化 → fencing → outcome-unknown 对账 → Approval 水位 → 重新武装 →
`resume_execution` / `thaw`。EP 崩溃：先回报未回报事件再接受新调用（Q25 的 R2 义务）。崩溃后不是一律重跑；
外部结果未知先对账；不承诺无条件 exactly-once。

### 10.5 边缘（继承 v0.7.0 11 §3 / ADR-0010，R3 之后）

`fysund-edge` = 带本地账本的 EP 宿主：Node 归属唯一（home）、事件 `node_id` / `origin_seq`（M0 起字段必在）、escrow 子 Grant
（`budget ≤ parent.remaining` 预扣）、离线只允许本地 Policy 预授权类别、`approve` 一律阻塞且 edge 不能决定 Approval、
联网后按 `origin_seq` 推送合并（`ledger/merged`）、冲突封闭枚举 `revoked_offline / policy_retired / home_violation / clock_skew`。
**R0–R3 不做边缘**；事件通用字段自 R1 起保留。

---

## 11. API 面

### 11.1 四组管理 API + 两份契约

应用 ↔ `fysund` 只有四组管理 API；Runtime ABI 与 EP 契约是控制面↔数据面的 RPC（不是 REST 资源）；消费者调用走
A2A / MCP / HTTP（gateway），不进本表。

```text
Catalog     资产 + 契约 + 组合 + 命名 Profile + Adjustment
Runtime     Node / Runtime 实例 / EP / Activity / Execution / 观察面
Service     Service 目录 / Tenant / Plan 附着 / meter 查询
Governance  Policy / Grant 观察 / Approval / Secret / Memory / replay
契约        Runtime ABI（§6.2）· EP 契约（§6.5）
```

### 11.2 方法表

**Catalog**

```text
asset.register / publish（准入类）/ deprecate / withdraw / quarantine / revoke
asset.describe / search / health / evidence            # 描述面无值槽、无内容本体
capability.register / describe / search / health        # tool_effects[] 由组合汇总后确认
agent.define · compose.resolve · compose.match · compose.diff   # diff 标注 contract/substrate/profile
profile.put(loop, …) / profile.describe                 # 命名 Loop Profile，非 kind
adjustment.apply / revert / promote / describe          # 包络检查见 §9.3
trajectory.export(execution_id)                         # 脱敏轨迹，供优化器
```

**没有** `asset.register(kind=runtime|loop|tool)`。

**Runtime**

```text
node.register / list / describe
runtime.register / qualify / list / health              # 实例，不是资产
ep.register / qualify / list / health
activity.create（钉 lock_digest，可带 tenant）/ start / pause / resume / cancel / describe / list
activity.park(activity_id)                              # 请求 Runtime park；不支持则忽略并保持进程
execution.observe        # 状态、waiting.reason、Approval 结果枚举、预算、生效 EP；不含 decided_by/reason 文本
execution.deliver_input  # 入队或 ABI deliver_input
handoff.* · artifact.* · verification.*                  # 观察面
```

**Service**

```text
service.create / publish / rollout / suspend / retire   # 版本单位 = Lock
service.describe / search / health / agent_card         # Card 由 fysund 签名
tenant.register / suspend / close / describe / usage
plan.define / attach（准入类）/ detach                   # 可 pin 版本
meter.query / export                                    # 投影；无 meter.create
```

**Governance**

```text
policy.load / revise / retire / describe(policy_id?) / explain(event_ref) / simulate(input)   # simulate：下发前干跑
grant.list / describe / revoke                          # 无公开 grant.issue（签发只由内部流程）
approval.list / describe / decide / batch_decide        # decide 校验路由允许集合
memory.list / retract / describe
secret.register / rotate / revoke                       # 只返回 SecretRef
system.status / replay / export
```

### 11.3 API 负例

1. 任一管理 API 的描述面返回可放明文的值槽 → 失败。
2. `grant.issue` 作为公开方法存在 → 失败。
3. `execution.observe` 返回 Approval 的 `decided_by` / `reason` / `presentation` → 失败。
4. Catalog 返回资产内容本体而非引用 → 失败。

---

## 12. 威胁模型与 conformance

### 12.1 资产与信任边界（相对 v0.7.0 的变化）

资产表继承 v0.7.0 13 §1，增：**资产目录与 Lock**（被投毒即组合失真）、**Service 目录与 Agent Card**（被伪造即冒充服务）。
信任边界变化：`fysund` 仍是唯一 TCB，但守卫从 `fysund` 内组件改为**独立 EP**；TCB 边界改写为
"权威面 = `fysund`；数据面强制点 = qualified EP（其快照与回报受 `fysund` 核对）"；凭据注入位置从"`fysund` 代理"改为
"只在 qualified EP 注入"（威胁模型中 Provider 代理图随之更新）。新增边界：下游 MCP server（HTTP）的 OAuth audience、
MCP sampling 反向通道、EP 冒充。

### 12.2 攻击向量 T1–T15

T1–T11 全部继承 v0.7.0 13 §3（间接 prompt injection、恶意实现包、权限扩张、凭据外泄、账本篡改 / 重放、验证欺骗、
记忆投毒、物理动作越界、Runtime 越权推进状态、DoS / 审批轰炸、审批面伪造），机制不变、落点改到 EP。新增：

| ID | 向量 | 机制 | 负例 |
|---|---|---|---|
| T12 资产供应链 | 恶意 / 篡改 / 未签名资产；`revoked` 仍在 Lock | digest 重算、签名 Policy、namespace 允许列表、quarantine/revoke 级联 | 未签名资产在 Policy 要求签名时被 `asset/published` → 失败；`revoked` 资产在 Lock → 失败 |
| T13 EP 冒充 / 失格回报 | 未资格化 EP 回报被接受；`policy_version` 超窗回报被接受；EP 本地决定 Approval；**EP 把 `external_read` 标成 `local_read` 走快路径** | EP 资格化、`policy_version` 窗口核对、I1、类别三源合一不可单方降级 | `ep.report_rejected` 路径全部负例；类别降级 → 失败 |
| T14 跨租户泄露 | Memory / Artifact / Secret / Activity 跨 Scope 可见；`shared` 池化会话串扰 | Scope 过滤、命名空间、`dedicated` 缺省 | 跨租户读取 → 失败；无 `session_isolation` 被 shared 绑定 → 失败 |
| T15 服务面滥用 | 配额绕过、canary 不可审计、Card 与 Lock 不一致、未发布 Lock 被调用、**无 EP 覆盖的 mutation 服务对外** | I6 发布门、计量可信、`service/rollout` 事件审计 | N-11；A5 各负例 |

### 12.3 明确不防（继承 + 重申）

模型权重后门；宿主 OS 被攻破；用户自己批准恶意动作；沙箱逃逸（沙箱 = containment）；完整 CaMeL 级信息流（Q13 延后）；
MCP server 自身实现漏洞；**Harness 自持模型凭据外泄**（不在 F5 范围；部署者可把 base URL 指向 `provider_proxy` 使其进入范围与计量）；
**无 EP 的 Runtime 上原生工具产生的外部效果**（`not_covered`，如实声明而非过度承诺）。

### 12.4 不变量负例索引（N-01–N-12，进入修订后 ADR / EP 契约 / 路线图验收）

| ID | 情境 | 期望 |
|---|---|---|
| N-01 | pack RuntimeInstance 为 OCI artifact | `asset.kind_mismatch` |
| N-02 | 仅 profile 变更却丢弃同 contract 证据 | 必须保留 `inherited` |
| N-03 | 声明 `cache_alignment` 的 Harness 把每轮时间戳写入 Slot0 | `runtime.cache_alignment_violated`（清位 + 告警） |
| N-04 | `local_read` 无 `audit/observed`，或 `external_read` 无 ActionIntent | 失败（ADR-0008 原文） |
| N-05 | sampling 兼容窗口带回 `trusted_system` 或 Secret | EP 拒绝 |
| N-06 | sampling / 模型调用超 Execution 预算 | `quota.exceeded{dimension: sampling}` |
| N-07 | 进化提议提升 `effect_category` 或新增工具 | `evolution.immutable_boundary` |
| N-08 | 影子集未通过即 `stable` | `evidence_gate` 失败 |
| N-09 | `thaw` 到错误 `lock_digest` | 拒绝 resume |
| N-10 | 租户 A 凭据调用租户 B Service | gateway 拒绝 |
| N-11 | 无 mutation EP 覆盖的 Lock 被 `service/published` | 失败 |
| N-12 | 仅 Harness 自报 token 导致他户 `quota.exceeded` | 失败 |

v0.7.0 的 `NV-T<n>-<序号>` 向量全部保留并按 EP 组合重登记；hooks 路径"只拦不记"定位不变（不计入 conformance 主路径）。

### 12.5 conformance 按 EP 组合分列

状态用语四类继承（`specified / implementation available / test executed / implemented`）；本说明期全部 `specified`。

| Profile 名 | EP 组合 | F3 / F5 / F7 / F9 范围 | 最早宣称阶段 |
|---|---|---|---|
| `ep.none.lifecycle` | 无 EP | 永不宣称 F3/F7/F9；仅生命周期与账本 | R1 |
| `ep.headless.adapter-min` | `harness_adapter` 最小（persist_intent + receipt + filter_tools） | 该 adapter 覆盖的工具面 | R2 |
| `ep.mcp-guard.gateway` | `mcp_guard` + `gateway` | MCP 工具面；原生工具 `not_covered`；secret 经 `resolve_secret` | R2 |
| `ep.mcp-guard.hooks` | `mcp_guard` + `hooks` | MCP 工具面 + 原生工具阻断（尽力，只拦不记） | R2/R3 |
| `ep.dsh.adapter` | `fysun-dsh`（全控制点） | MCP + 原生工具经拦截路径 | R3 后 |
| `ep.dsh.adapter+proxy` | + `provider_proxy` | + 模型凭据与可信 token 计量 | R3 后可选 |

一致性负例（继承 ADR-0007 §D2）：任一 Runtime / EP 组合的 `native_effect_tools[]` 非空而 conformance 对 F3/F7/F9 标注未限定范围 → 失败；
把 hooks 路径描述为"安全边界"→ 失败；出现四类之外的状态用语 → 失败。

---

## 13. 路线图与验收

### 13.1 R0–R3（可判定退出条件）

原则不变：第一个闭环要小、真、可判定；不先全量回写 `docs/design`；本说明 + 采纳后的 ADR 为实现规格，实现中回填。

| 阶段 | 做成什么 | 可验证退出 |
|---|---|---|
| **R0 规格与 ADR 工程** | 所有者采纳本说明 D1–D8 + §0.2 修正；ADR-0013 按差异改写（八 kind、Runtime 非 kind、D3 改写、负例 1 改"无登记不能引用八类资产；Loop 须可追溯 Profile id 或内联 digest"）；**ADR 修订**：ADR-0002 §D2 supersede 为"必经的是 EP 契约"；ADR-0007 §D3 hooks = hooks EP、§D5 投递路径废止（`os.observe` 替代）；ADR-0012 增补（独立进程允许、不嵌入保留）；**ADR-0009 增补实体 / context 封闭表**；ADR-0008 保留且 EP 契约必须引用 D5；sources.md MCP 钉版更新 2026-07-28；术语表以本说明 + §3/§4 为准 | grep：无"十一种资产"、无"`kind=runtime` 为可打包资产"、无"`fysund` 是 harness 的 MCP server"；定义句与本说明 §1.1 一致；资产 kind 表无 `runtime`；EP 契约出现 `local_read` / `external_read` 分档；MCP 页写明 Sampling 缺省拒绝 |
| **R1 登记·锁定·跑通** | 八种 manifest schema；`pack/push/register/publish`；Loop Profile put；分层 `compose.resolve`（base 先行、overlay API 可薄）；一个 Harness（薄包装或 `fysun-dsh`）实现最小 ABI；Activity/Execution + 恢复再武装；账本 replay digest 稳定；`describe()` 与 harness 资产互证；Policy 载入 + `policy.simulate`；**最小 EP（adapter-min）** | §4.6 负例 1–7；§5.6 负例 1、3、7；闭环：prompt+skill+mcp_server+harness+loop(Profile)+model → agent.define → resolve → match → run → 独立验证 → 杀进程恢复；`fysund` 不在 tool/模型转发链；`runtime/qualified` 带 `harness_asset_id`；`asset/published` 带 `policy_decision` |
| **R2 执行点·入口服务** | EP 契约完整（describe / policy_pull / report / resolve_secret / health）；`mcp_guard` 或 adapter 覆盖 MCP 工具面；**F3 分级**（local_read 快路径 + external_read 组批 + mutation 同步 WAL）；sampling 防御；`fysun-gateway` 至少一个 A2A Endpoint + 签名 Agent Card（1.0 形状）；单 Tenant 可调用；`os.observe` 替代系统工具；EP 恢复协议（Q25） | §6.9 负例 1–10；§8.8 负例 1、5、7、11、12；N-03–N-06；sampling 不可窃取 Secret；只读不强制同步 WAL；Policy 过期 fail-closed；度量管道存在（无强制阈值） |
| **R3 多租户·演进·第二 Harness** | Tenant / Plan / meter；canary 可审计；租户隔离（`dedicated` 缺省）；Adjustment + 分层证据 + 证据门 + `compose.diff{change_class}`；一个优化器插件（可假实现）只能提交 profile；第二 Harness 跑同一契约变体；可选 `session_park`；发布门 I6 全面生效 | §8.8 负例 2、3、4、6、8、9、10；§9.5 负例全部；N-02、N-07–N-12；两租户 Memory / 用量分离；profile 微调保留 `inherited`；越权 Adjustment 被拒；`thaw` 锁校验 |

**R3 之后的工作项（不进前四个门）**：Trust 对象 ADR（Q8/Q27，任何跨基座 `tenant_visibility = public` 前置）；边缘 `fysund-edge`
（继承 ADR-0010）；Personal 三合一应用；`fysun-dsh` 全控制点；`external_gateway`（agentgateway）评估 spike；`shared` 池化（Q24）；
SPIFFE 身份后端评估；完整 CaMeL（Q13）。

### 13.2 度量（先基线后阈值；R1 打点，R2 末基线报告，10 次以上闭环后再定阈值）

daemon 启动耗时 / 空闲内存；调度延迟（trigger → `start_execution`）；资产登记到可解析步骤数；新 Harness 打包到
`runtime/qualified` 步骤数；`compose.resolve` 时延；Service 发布到首个调用步骤数；EP `tools/call` pre p50/p95（按 `effect_category` 分位）；
EP 回报延迟与拒绝率；**前缀缓存命中率**（`cache_read / input_tokens`）；**审计快路径占比**（`audit/observed / (audit/observed + action-intent/recorded)`）；
每租户用量分离正确率；人工介入率、审批不可用率、回滚率、验证独立性比例（继承）；`runtime/disqualified` 按 reason；
**`fysund` CPU 在工具调用期间 ≈ 0**（热路径禁令的性能表达）。

### 13.3 Spike（R0 末–R1 初；默认时间盒 3–5 工作日；S1/S2/S4/S5 为 R2 开工阻断项）

| ID | 问题 | 通过标准 |
|---|---|---|
| S1 | DSH headless 是否含独立于 bundle 的原生工具（Q16） | 清单写入 harness manifest；有则 R2 EP 须覆盖或 Policy 拒绝非 `local_read` |
| S2 | DSH JSON-RPC / sdk-minimal 能否承载 Runtime ABI | 能 `start`/`cancel` 或 ≤200 行包装 |
| S3 | OCI artifact 以 manifest 为 config 的 media type（Q22） | pack/push/pull digest 稳定 |
| S4 | cedar-policy crate 在 EP 进程评估同一 snapshot | 同输入同 outcome（否则禁止 EP 本地"简化评估器"） |
| S5 | CognitiveOS 抽取地图（公开仓 vs 本地 `6f562bf`） | 抽取清单获 owner 签字 |
| S6 | Windows named pipe vs stdio 作为本机 ABI | CI 与开发机各一条路径，否则 R1 限 stdio |
| S7 | agentgateway 作 `external_gateway` 的差距 | 差距表；默认不依赖 |
| S8 | A2A 1.0 Agent Card 签名与 gateway 暴露 | Card 字段能从 capabilities 派生 |
| S9 | 前缀漂移探针（`cache_alignment` 声明位） | 同一 Lock 开关对比 cache 命中（基线） |

### 13.4 证据纪律与非声明

继承 v0.7.0 12 §5（四类状态用语、`not-run` 永不等于 pass、外部基准只说明方向）与 §6。本说明期间：全部条目 `specified`；
不声明任何资产已登记、任何 Harness 已资格化、任何 Service 已发布、任何性能数字。CI 门（R1 起）：
schema 一致性 grep、定义句 / 热路径禁词、replay 金样、cargo-deny / 许可证分类、conformance 向量。

---

## 14. 开放问题（合并清单；编号只增不复用）

| ID | 题 | 处置 / 建议姿态 |
|---|---|---|
| Q1 | ABI / EP 跨机传输 | 保留：本机 stdio JSON-RPC（R1）；跨机 gRPC vs HTTP JSON-RPC R3 前定；Arrow / 零拷贝拒绝 |
| Q2 | Secret Store 后端 | 保留：R1 本机加密文件 / OS keyring；生产后端后置 |
| Q3 | 审批 Web UI 范围 | 保留：最小审批页优先；不做聊天 UI |
| Q4 | `model_judge` 校准集与阈值 | 保留：R3 后；`human` 为主 |
| Q5 | 多主体 Scope | **关闭**：Tenant 核心；R1 单租户合法、R3 硬隔离 |
| Q6 | 许可证 | 保留：R0 决策（推荐 Apache-2.0）；未定前不发布二进制 |
| Q7–Q10 | （v0.7.0 已关闭） | Cedar 子集 / Intent 改名 / 边缘一致性 / agentgateway——分别由 ADR-0009 / 0006 / 0010 / 0012 关闭 |
| Q8（原 Q11/Q27） | Trust 对象 | R3 之后、任何跨基座 `public` 发布前置；单独立 ADR |
| Q12 | 多节点一致性 | 保留：R3 后；R1–R3 单节点 |
| Q13 | CaMeL 级信息流 | 保留：延后；标签 + 引用监视器不宣称值级 IFC |
| Q14 | Provider 路由 | **关闭**：不做路由；`fallback` 是绑定顺序 |
| Q15 | `gen_ai.*` 关系 | **收窄**：钉 commit / schema URL；`fysun.*` 为权威 SLO；并存 + 改名层 |
| Q16 | DSH headless 原生工具 | S1 spike，R2 前查证 |
| Q17–Q20 | `modify` 表达式 / `obligations[]` / `escalate` 目标 / `downstream_outcome` 窗口 | 保留：继承 v0.7.0 行为，R4 前不扩展 |
| Q21 | `approval_result_ttl` | **并入 Q25**（EP 契约 `authorized` TTL） |
| Q22 | manifest 规范化与签名 | canonical digest 算法 R1 必须先定；media type 临时值；签名 R1 可不要求（namespace 允许列表兜底）、R4 可要求；候选 Sigstore/Cosign |
| Q23 | loop_schema 通用子集 | **建议关闭**：跨 Harness 最小集 = `max_steps` + `stop_conditions[]`（+ Loop Profile 最小 params）；其余各 Harness 声明 |
| Q24 | `shared` 池化隔离证明 | R3 缺省 `dedicated`；`shared` 需注入测试负例后再开 |
| Q25 | EP WAL 与回报一致性 | 升级为 I1–I3 + EP 契约恢复协议；R2 交付；`authorized` TTL 显式配置 |
| Q26 | 计量可信度 | 对外多租户 token 配额**要求** `provider_proxy` 或等价可信计量 EP；自报缺省不计入 |
| Q28 | gateway A2A 能力位 | A2A 1.0 Card schema；能力位如实声明、禁止虚报 |
| Q29 | BYO 边界 | 允许 prompt / skill / model 凭据槽位；效果类别不可降级在 `fysund` 校验、EP 再强制一次 |
| Q30 | Overlay 证据校准 | `deterministic` 证据可自动继承；`model_judge` / `human` 项按 Policy 重跑或继承（缺省重跑） |
| Q31 | MCP 2026-07-28 无会话下下游会话持有 | per-request 凭据 + 可选应用层任务 id（EP 契约实现细节） |
| Q32 | optimizer 插件权限边界 | 缺省只许 profile 层（prompt / loop params）；改其他 Harness 绑定缺省否 |
| Q33 | `tool` 独立 kind | **关闭**：不保留（随 §4.2） |

## 15. 待所有者决策项（推荐值已给出，推荐 ≠ 已采纳）

| # | 问题 | 本说明推荐 | 影响 |
|---|---|---|---|
| 1 | 是否采纳本说明（= proposal D1–D8 + §0.2 评审修正）作为 v0.8 工作规格 | 采纳后再写生产代码；R0 完成文档与 ADR 闸门 | 全局 |
| 2 | R1 首个 Harness | DSH headless 薄包装；`fysun-dsh` 并行 spike（S1/S2） | R1/R2 |
| 3 | Personal 是否绕过 gateway | 允许绕过 gateway（Tenant 仍在）；**不可绕过 EP 覆盖门（I6）** | R3 |
| 4 | R2 工具侧 EP | 保留第一方 `fysun-mcp-guard`（独立进程）为参考 EP；hooks + adapter 为第二组合 | R2 |
| 5 | Loop Profile | R1 即提供 `profile.put` 命名登记；内联亦合法 | R1 |
| 6 | 证据门默认 | Policy 决定；对外 `stable` 建议默认开 | R3 |
| 7 | MCP 规范钉版 | R2 即钉 2026-07-28（Sampling 缺省拒绝、OAuth 2.1） | R2 |
| 8 | `session_park` | 保持可选 `unsupported`（R3 不强制） | R3 |
| 9 | 许可证 | Apache-2.0（与 CognitiveOS 抽取最省事） | R0 |

---

## 附录 A · 2026 外部锚点（检索日 2026-09-12；一手来源为准）

| 锚点 | 事实 | 对本设计 |
|---|---|---|
| MCP 2026-07-28 | 无状态核心、可缓存 `list`、扩展框架、授权加固；Sampling / Roots / Logging 弃用（SEP-2577，≥12 个月窗口） | 工具协议钉版；EP Sampling 防御 |
| MCP Authorization | OAuth 2.1；RFC 9728 PRM MUST；RFC 8707 resource indicator；DCR 弃用 / CIMD SHOULD | `mcp_server.auth`、gateway / EP audience 校验 |
| A2A 1.0（AAIF Growth Stage，2026-08-27） | Card `supportedInterfaces[]`、`extendedAgentCard`、签名 Card、企业多租户 | Service 主协议、Card 形状 |
| Agent Skills | `SKILL.md` 开放格式、三级渐进披露 | `skill` 资产与 `skill_loading` 契约 |
| AWS Bedrock AgentCore | 控制面 / 数据面分离；Gateway 用 Cedar 在工具调用点 default-deny（GA 2026-03） | 控制面 / 数据面分离是工业默认；EP + Cedar 同构（不照搬产品边界） |
| Prompt / KV prefix caching（Anthropic / OpenAI 文档；arXiv:2601.06007） | 长程工具循环成本主因是前缀未命中与工具串行 | `cache_hint` / `prefix_order[]` / `cache_alignment` 能力位；并行只读 |
| 自进化（ACE arXiv:2510.04618；GEPA arXiv:2507.19457；DGM arXiv:2505.22954；综述 arXiv:2507.21046 / 2607.13104） | 演化发生在上下文 / prompt / 脚手架；生产应离线编译一次、当制品发布 | 优化器为离线插件；Immutable Zone；证据 → Adjustment → 门控 |
| SWE-bench Verified 污染（OpenAI 官方陈述；arXiv:2512.10218） | 公开榜不再衡量前沿编码 | 拒绝公开榜作对外证据门；容器化可复现独立 verifier |
| Temporal durable execution | HITL durable wait、工具并行、I/O 进 Activity | `session_park` 模式借鉴；不引入依赖 |
| SPIFFE/SPIRE | 短寿命工作负载身份工业标准 | 后置可选身份后端；不进 R0–R3 内核 |
| OTel GenAI semantic-conventions-genai | 2026-06 迁独立仓库；全部 Development；schema URL 曾为 TODO | `gen_ai.*` 钉 commit；`fysun.*` 为权威 |
| OWASP Agentic Top 10（2026，ASI01–ASI10） | Agentic 风险分类学 | §8.7 覆盖映射，不另造分类学 |
| Durable execution / 工业实践（LangGraph、Restate 等） | journal / 幂等 / receipt 共识 | F3、恢复模型印证 |
| Progent / CaMeL（arXiv:2504.11703 / 2503.18813） | 单调收窄、标签 + 引用监视器 | F9 与来源标签的学术根据（数字只说明方向） |
| CAP（arXiv:2512.04123） | HITL、评估、可靠性是生产常态 | Approval 一级对象、SLA 投影维度 |

## 附录 B · 与 v0.7.0 的逐篇处置（R0 回填 `docs/design` 时的操作底稿）

| v0.7.0 | 关系 | 去向 |
|---|---|---|
| 00 术语 | 改写 | 合并本说明 §3–§5 术语；kind 八值；增 change_class / park / EP / Tenant |
| 01 定义与原则 | **重写** | 本说明 §1–§2 |
| 02 对象模型 | 扩展 | 四层实体 + 分层 Lock + 服务对象；Capability 去 `implementation` |
| 03 架构、账本与恢复 | **重写分层与拓扑**；账本与恢复继承 | 本说明 §10 + §6.7 |
| 04 Capability API | 扩展 | 本说明 §11（四组 + 两契约） |
| 05 Runtime ABI | 修订 | `start_execution` 不带渲染文本；`os.observe`；`deliver_input` 可选；`park/thaw` |
| 06 MCP 守卫规范 | **降级** | "`mcp_guard` EP 参考规范"，从属 §6.4；独立进程 |
| 07 Runtime 接入 | §1/§2/§6/§7 继承；§3–§5 废止 | 投递路径、系统工具、F5 部署指引删除或移入适配器文档 |
| 08 DSH 适配器 | 降级 | `fysun-dsh` = Harness 资产 + Runtime ABI + `harness_adapter` EP |
| 09 迁移清单 | 缩小 | Effect 协议进参考 EP；Personal 三合一应用 |
| 10 演进 | 扩展 | 本说明 §9（双轨：人工 Adjustment + 受限离线进化插件） |
| 11 边缘 | 继承 | `fysund-edge` = 带本地账本的 EP 宿主 |
| 12 MVP 计划 | **重写** | 本说明 §13（R0–R3） |
| 13 威胁模型 | 修订 | T12–T15 + Sampling/OAuth；conformance 按 EP 组合 |

---

*本说明由 `docs/redefinition`、`docs/proposal-v0.8`、两份 2026-09-12 评审与 v0.7.0 现行设计综合而成；
采纳路径见 §15 决策 1 与 §13.1 R0。未在本说明中重述的 v0.7.0 细节（对象字段全表、事件携带字段规则、
Cedar 语法子集、恢复负例全文、迁移清单明细）在其采纳前继续以现行文档为准，R0 回填时并入。*
