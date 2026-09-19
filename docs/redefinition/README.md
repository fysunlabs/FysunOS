# FysunOS 重定义：Agent 能力管理（提示词、Skills、工具、MCP、Harness、Loop）和 Runtime 控制的运行基座

- 状态：**提案草案 v0.8.0-draft**（2026-09-11）。本文件夹是对 FysunOS 定位的重新定义，独立于现行设计文档
  （[docs/design](../design/00-terminology.md)，v0.7.0）；在 [ADR-0013 草案](adr-0013-draft-repositioning.md) 被采纳前，
  现行文档不因本文件夹改动。
- 前置阅读：[v0.7.0 定位审查结论](10-roadmap.md#1-为什么重定义)（本文件夹 10 §1 摘录）。
- 术语以 [00 术语表](00-terminology.md) 为本文件夹唯一基准；与现行 `docs/design/00` 冲突处以本文件夹为准，并在 00 §1 登记对照。

## 一句话定义

> **FysunOS 是 Agent 能力管理（提示词、Skills、工具、MCP、Harness、Loop）和 Runtime 控制的运行基座。**
> 它把提示词、Skills、工具、MCP server、Harness、Loop 策略统一为可登记、可版本化、可组合、可计量的**能力资产**并加以管理；
> 把 Runtime 的接入、资格化、绑定、生命周期与执行边界置于统一**控制**之下；在这个基座上，由资产组合成的 Agent 能在任意资格化 Runtime 上
> 可靠运行，并作为服务被发布、发现、调用、治理与演进，服务 Agent as a Service 生态。

两项职责与一个目标：

- **能力管理（Capability Management）**：一切可复用之物都是带 manifest 的能力资产——Prompt、Skill、Tool、MCP server、Harness、Loop、Model、
  Memory provider、Knowledge，以及由它们组合成的 Agent。基座负责登记、版本、来源、兼容、效果声明、组合解析与证据。基座不写 loop、不写 prompt，
  它让这些东西成为可管理、可替换、可追溯的资产。
- **Runtime 控制（Runtime Control）**：Runtime 是 Harness 的资格化实例。基座控制它的接入、`describe()` 资格化、与 Agent 的绑定、Activity / Execution
  生命周期、预算与取消、恢复与重新武装、执行边界（Grant ∩ Policy ∩ 工具面）。控制与决策在 `fysund`，数据面拦截在**执行点**；`fysund` 不在热路径。
- **目标：Agent as a Service 生态**：Agent 的对外形态是服务——有租户、有配额、有计量、有版本、有 Agent Card、有证据；
  消费者经 A2A / MCP / HTTP 调用，不关心它由哪个 Harness 在哪个 Runtime 上跑。服务平面建立在前两项职责之上，不替代它们。

## 与 v0.7.0 的关系

- **继承**：event-sourced 治理账本与唯一权威写入者；Capability 契约（行为 / 质量 / 边界 / 验证）；Activity / Execution 生命周期与恢复模型；
  Grant / Policy（Cedar 子集）/ Approval（封闭结果集）/ 来源标签 / Verification；Runtime 资格化纪元与 `native_effect_tools[]`；A2A / MCP / Agent Skills 标准优先。
- **改变**：Harness 与 Loop 从"工程术语 / 不做"升为一级资产种类；Runtime 定义为"Harness 的资格化实例"；Capability 的实现从单一 Skills 目录扩展为
  **Agent Definition（资产组合）**；新增服务平面（Service / Tenant / Plan / Meter / Endpoint）；多主体 Scope 从开放问题变为核心；
  执行点（Enforcement Point）替代"`fysund` 内联守卫是必经之路"。
- **废止**：`fysund` 作为 harness 唯一 MCP server 的强制形态；`deliver_input` 三条投递管道与 `fysun.await_approval` / `fysun.pending_inputs` 作为 OS 规范；
  `fysund` 渲染 context_view 进 harness 启动输入；Provider 代理作为 F5 前提。它们降级为某个执行点或适配器的实现细节。

## 文档地图（骨架）

| 编号 | 文档 | 回答的问题 | 状态 |
|---|---|---|---|
| — | [ADR-0013 草案 · 重定位为能力与 Runtime 基座](adr-0013-draft-repositioning.md) | 决策、否决项、后果、负例 | 草案 |
| 00 | [术语表](00-terminology.md) | 本文件夹的对象名、种类名、状态名、事件名；与 v0.7.0 的改名对照 | 草案 |
| 01 | [定位与原则](01-positioning.md) | 原点回顾、新定义、"基座"四件事、平面划分、对外坐标、原则 P1–P11、底线 F1–F9（含由谁执行）、不做什么 | 草案 |
| 02 | [能力资产模型](02-asset-model.md) | 统一 manifest；十一种资产的专有字段；版本 / digest / 来源 / 许可证；依赖与兼容；效果声明；生命周期 | 草案 |
| 03 | [组合：Agent Definition 与 Capability 契约](03-composition.md) | 资产如何组合成 Agent；解析与锁定；环境变体；子 Agent 引用；证据归属 | 草案 |
| 04 | [Runtime 基座](04-runtime-substrate.md) | Harness → Runtime 的安装、登记、资格化；最小 Runtime ABI；Activity / Execution；执行点契约；Policy 下发；账本；多租户隔离 | 草案 |
| 05 | [Agent as a Service](05-agent-as-a-service.md) | 角色；Service / Endpoint / Tenant / Plan / Meter；发布、发现、调用、计量、版本与回滚、跨租户审批、联邦 | 草案 |
| 06 | [治理](06-governance.md) | Principal / Scope（多租户）；Grant；Policy 实体模型扩展；Approval；来源标签；Verification；资产供应链治理；可观测性 | 草案 |
| 07 | [生态与互操作](07-ecosystem-and-interop.md) | MCP / A2A / Agent Skills / OCI / OTel 的用法与边界；资产打包与分发；发布者—运营者—消费者流程；联邦 | 草案 |
| 08 | [API 面](08-api-surface.md) | 六组 API（Registry / Composition / Runtime / Service / Governance / 执行点契约）与错误码 | 草案 |
| 09 | [演进与证据](09-evolution-and-evidence.md) | 证据归属；Adjustment；发布证据门；消费反馈；弃用与迁移 | 草案 |
| 10 | [路线图与开放问题](10-roadmap.md) | 为什么重定义；与 v0.7.0 逐篇继承 / 改写 / 废止关系；R0–R5 里程碑；开放问题；待所有者决策项 | 草案 |
| 11 | [开发计划](11-development-plan.md) | 把 00–10 落到可执行工作包：复用边界、仓库形态、R0–R5 验收、spike、conformance、风险与追溯 | 草案 |

阅读顺序：ADR-0013 草案 → 01 → 00 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11。

## 命名（暂定）

| 名称 | 含义 |
|---|---|
| `fysund` | 基座 daemon（Rust）：资产登记、组合解析、Runtime 管理、服务目录、Policy 决策、Approval、唯一权威账本写入者。**不在数据面热路径** |
| `fysun` | CLI：`fysun asset pack/push/register`、`fysun agent compose/resolve`、`fysun service publish`、`fysun runtime qualify`、`fysun hook pre-tool-use` |
| `fysun-gateway` | AaaS 入口执行点：A2A server / MCP facade / HTTP；租户鉴权、配额、计量、把调用变成 Activity / Execution。独立进程，可水平扩展 |
| `fysun-mcp-guard` | 工具侧参考执行点（独立进程，可选）：`tools/list` 过滤、`tools/call` pre / post、Effect 回报；不再是 `fysund` 内组件 |
| `fysun-dsh` | DeepSeek Harness 适配器：把 DSH 登记为 Harness 资产、实现 Runtime ABI 与控制点、作为原生工具执行点 |
| `fysund-edge` | 边缘节点精简 daemon（继承 v0.7.0 11 §3） |
| 资产仓库 | 任意 OCI 兼容制品仓库；资产以 OCI artifact 打包分发，`fysund` 只存登记与 digest |
