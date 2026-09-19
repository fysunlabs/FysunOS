# FysunOS v0.8 精简设计提案

- 状态：**提案草案**
- 日期：2026-09-12
- 本文件夹**不替代**现行 [`docs/redefinition`](../redefinition/README.md)，直至所有者采纳。
- 现行设计仍为 [`docs/design`](../design/00-terminology.md)（v0.7.0）与已采纳 ADR。本提案是可独立阅读的替代方案，不是对 redefinition 的补丁。

## 一句话定义

> **FysunOS 是 Agent 能力管理与 Runtime 控制的运行基座：登记并组合可版本化的能力资产，资格化 Runtime，在执行点强制边界，使 Agent 作为服务被发布、调用、计量与演进。**

它管理的能力包括提示词、Skills、工具（经 MCP）、Harness 及其 Loop 策略；Loop 是 Harness 的配置，不是与 Harness 平级的一级资产。基座不实现 loop、不装配 prompt、不在热路径转发工具或模型请求。

## 与 v0.7.0 / redefinition 的关系

| 版本 | 角色 | 本提案对它做什么 |
|---|---|---|
| v0.7.0 [`docs/design`](../design/00-terminology.md) | 现行设计 | **继承**账本、F1–F9、Grant∩Policy∩工具面、Approval 封闭结果集、Activity/Execution、Cedar 子集、协议优先、资格化纪元。**废止** `fysund` 内联 MCP 守卫 / Provider 代理 / 上下文渲染 / 三条投递管道（与 redefinition 同向）。 |
| v0.8-draft [`docs/redefinition`](../redefinition/README.md) | 被复检的重定位草案 | **吸收**双职责（能力管理 + Runtime 控制）、`fysund` 退出热路径、执行点、Agent Definition、服务平面。**改写**资产种类、服务对象层级、API 面、锁定与证据、演进闭环、路线图。逐篇对照见 [CHANGELOG-vs-redefinition.md](CHANGELOG-vs-redefinition.md)。 |

不采纳 redefinition 的原因一句话：方向对，对象膨胀。十一类平级资产、五类平级服务对象、六组 API，把配置、实例、计量事件都抬成一级，热路径与自进化因此变钝。

## 文档地图

| 文件 | 回答 |
|---|---|
| [00-decisions.md](00-decisions.md) | 拍板决策、否决项、相对 ADR-0013 草案的差异 |
| [01-kernel.md](01-kernel.md) | 内核最小集、控制面/数据面、执行点、资格化、账本、Activity/Execution、热路径 |
| [02-assets.md](02-assets.md) | 压缩后的资产模型、manifest、种类表、锁定、效果声明 |
| [03-agent-and-runtime.md](03-agent-and-runtime.md) | Agent Definition、环境变体、Harness→Runtime、ABI、性能机制 |
| [04-aaas-and-evolution.md](04-aaas-and-evolution.md) | 服务平面 + 自进化最小闭环 |
| [05-api-and-roadmap.md](05-api-and-roadmap.md) | API 面、R0–R3、开放问题、待拍板项 |
| [CHANGELOG-vs-redefinition.md](CHANGELOG-vs-redefinition.md) | 给所有者的逐篇差异说明 |

阅读顺序：本 README → 00 → 01 → 02 → 03 → 04 → 05。CHANGELOG 供对照，不是设计正文。

## 对象速查（只读本提案时）

| 一级 | 平面 | 一句话 |
|---|---|---|
| Asset / Manifest | 资产 | 八种可登记可分发之物 |
| Capability | 资产 | 行为契约，不是 kind |
| Agent Definition / Lock | 资产 | 组合与三层钉死 |
| Runtime 实例 / Node / EP | 运行 | 资格化后的进程与拦截点 |
| Activity / Execution | 运行 | 长期工作 / 一次 run |
| Service | 服务 | 对外发布形态 |
| Principal / Tenant / Scope / Grant / Policy / Approval | 治理 | 谁、在哪、能做什么、谁批 |
| Event | 治理 | 账本真源 |

Loop Profile、Endpoint、Plan、Meter 不是上表一级。

## 命名

| 名称 | 含义 |
|---|---|
| `fysund` | 控制面 daemon（Rust）：登记、解析锁定、资格化、PDP、Approval、唯一权威账本写入者。**不在数据面热路径** |
| `fysun` | CLI |
| `fysun-gateway` | 入口执行点：A2A / MCP facade / HTTP；鉴权、配额、把调用变成 Activity |
| `fysun-mcp-guard` | 工具侧参考执行点（独立进程，可选） |
| `fysun-dsh` | DeepSeek Harness 适配器（Harness 资产 + Runtime ABI + `harness_adapter` 执行点） |
| `fysund-edge` | 边缘精简 daemon（继承 v0.7.0 11 §3） |
| 资产仓库 | 任意 OCI 兼容制品仓库；`fysund` 只存登记与 digest |

## 相对 redefinition 的概念变更表

| 概念 | 处置 | 本提案形态 |
|---|---|---|
| `prompt` / `skill` / `mcp_server` / `model` / `memory_provider` / `knowledge` / `harness` / `agent` | **保留**为可登记资产 | 八种 `kind`；见 [02](02-assets.md) |
| Capability 契约 | **保留** | 契约对象，不是资产 `kind`；由 Agent Definition 实现 |
| Agent Definition / Lock | **保留并改写** | Lock 分层：`contract` / `substrate` / `profile` |
| Enforcement Point | **保留** | 数据面拦截；资格化；Policy 快照本地评估 |
| Activity / Execution | **保留两级** | Activity = 长期工作；Execution = 一次 run |
| Grant / Policy / Approval / 来源标签 / Verification / F1–F9 | **保留** | 执行位置在 EP + `fysund` 核对，规则不变 |
| `loop` | **降级** | Harness 的 Loop Profile（可内联或命名引用），不走 OCI、不是 `kind` |
| `tool` | **降级** | 不独立成 `kind`；工具来自 `mcp_server.tools_snapshot`、`skill.allowed-tools`、`harness.native_effect_tools[]` |
| `runtime` 作为资产 `kind` | **删除** | Runtime 是运行平面实例：Harness × Node × 资格化纪元 |
| Service | **保留**为服务对象（非资产） | 唯一服务一级对象 |
| ServiceVersion | **降级** | `Service.versions[]` 记录，不是平级对象 |
| Endpoint | **降级** | `Service.endpoints[]` 绑定（protocol + url + auth） |
| Tenant | **降级为治理对象** | Principal + Scope；不是与 Service 平级的服务资产 |
| Plan | **降级** | Tenant×Service 的附着约定，派生根 Grant |
| Meter | **降级为事件** | `meter/recorded`；查询是投影，不是一级对象 |
| Registry API + Composition API | **合并** | Catalog 面 |
| 六组 API | **压缩** | Catalog / Runtime / Service / Governance + 两份契约（ABI、EP） |
| R0–R5 | **压缩** | R0–R3，退出条件可判定 |
| 分层 Lock + 调优层证据继承 | **新增** | 仅 profile 变更可继承证据；契约/基座变更必须重评 |
| Loop Profile / Optimizer 插件 | **新增** | 优化器是可插拔资产（`kind=harness` 的特例或外部进程），不是内核 |
| `park` / `thaw` | **新增**（可选控制点） | 长任务 HITL 释放 Runtime；不是强制 ABI |
| MCP sampling 防御 | **新增** | 执行点义务，R2 |
| 确定性前缀槽为资格化硬失败 | **拒绝** | 改为 Harness 声明的 `cache_alignment` 能力位；OS 给结构化绑定顺序，不渲染文本 |
| SPIFFE 作为内核身份 | **拒绝（R0–R3）** | Execution Principal + Tenant 凭据 + 可选 mTLS；SPIFFE 不进内核 |
| 自造传输（Arrow / FlatBuffers） | **拒绝** | MVP：本机 stdio JSON-RPC；跨机 gRPC/HTTP 为开放问题 |

## 2026 锚点（本次检索，2026-09-12）

| 锚点 | 事实 | 对本提案 | 来源 |
|---|---|---|---|
| MCP | 规范 **2026-07-28**：无状态核心、可缓存 `list`、扩展框架、授权加固 | 工具协议；EP 拦 `tools/list|call` 与 sampling；stdio 仍可用于本机 | [MCP blog 2026-07-28](https://blog.modelcontextprotocol.io/posts/2026-07-28/) |
| A2A | 2026-08-27 被 AAIF 接纳为 Growth Stage；Linux Foundation 伞下 | Service 主协议；Agent Card；Handoff ↔ TaskState | [A2A × AAIF](https://a2a-protocol.org/latest/blog/2026/08/27/a-new-chapter-for-a2a-joining-the-agentic-ai-foundation/) |
| Agent Skills | `SKILL.md` 开放格式；三级渐进披露（name/description → 正文 → 资源） | 唯一 skill 包格式；OS 强制按需加载的契约，不自造 | [agentskills.io/specification](https://agentskills.io/specification) |
| AWS AgentCore | 控制面 `bedrock-agentcore-control` vs 数据面 `InvokeAgentRuntime` | 控制面/数据面分离是工业默认，不是发明 | [AgentCore 接口](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/develop-agents.html) |
| Context engineering / 工具披露 | Anthropic：上下文是有限注意力预算；Tool Search + `defer_loading` + 代码态调用 | 工具面裁剪在绑定完成；热路径不把全量 schema 塞进每次推理 | [Context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)（2025-09-29）；[Advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use) |
| 自进化 | ACE（上下文 playbook）；GEPA（Pareto prompt）；DGM（改代码 + 基准门） | 内核只提供证据→Adjustment→门控；优化器可插拔 | [ACE arXiv:2510.04618](https://arxiv.org/abs/2510.04618)；[DGM arXiv:2505.22954](https://arxiv.org/abs/2505.22954)；综述 [arXiv:2507.21046](https://arxiv.org/abs/2507.21046) |
| OWASP Agentic Top 10 | 2026：ASI01–ASI10（目标劫持、工具滥用、身份、供应链、RCE、记忆投毒、A2A、级联、HITL 利用、rogue） | 映射到 Grant/EP/标签/Approval/供应链撤销；不另造威胁分类学 | [OWASP GenAI](https://genai.owasp.org/download/52117) |
| OTel GenAI | 2026-06 迁入独立仓库；属性仍为 **Development**，无稳定保证 | 导出 `gen_ai.*`（钉 commit）+ `fysun.*`；不把未稳定约定写进账本 schema | [OTel community #3414](https://github.com/open-telemetry/community/issues/3414)；状态综述 2026-08 |

GEPA 论文条目以综述引用为准（[AgentsPulse 综述](https://agentspulse.github.io/tutorials/self-evolving-agents-review-en/)）；独立 arXiv 编号本次未单独打开原文。
