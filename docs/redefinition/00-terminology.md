# 00 · 术语表（重定义提案）

- 状态：提案草案 v0.8.0-draft（2026-09-11）
- 本文件夹内对象名、种类名、状态名、事件名、错误码、组件名的唯一基准。未在此列出的术语沿用现行 [docs/design/00](../design/00-terminology.md)。

## 1. 与 v0.7.0 的改名 / 升降级对照

| v0.7.0 | 重定义 | 依据 |
|---|---|---|
| Harness = 工程术语，不是规范对象 | `harness` 为一级资产种类 | ADR-0013 §D2 |
| Loop = 决策核心的控制策略，OS 只握预算 | `loop` 为一级资产种类（策略配置），Harness 消费；Loop **状态**仍不入账本 | ADR-0013 §D2 |
| Runtime（已安装）= Resource 类型 | Runtime = Harness 资产在某 Node 的**资格化实例**；仍是资产种类 `runtime` | ADR-0013 §D3 |
| `Capability.implementation` = Skills 目录引用 | `Capability` 由 **Agent Definition** 实现；Skills 目录是 `skill` 资产 | 01 §5 |
| Resource（模型 / 工具 / MCP server / 记忆 / 知识 / 设备） | 可复用者升为资产种类（`model` / `tool` / `mcp_server` / `memory_provider` / `knowledge`）；设备、账号、上下文源仍为 Resource | 02 §1 |
| `fysun-mcp-guard` = `fysund` 内组件，harness 唯一 MCP server | 独立进程的**执行点**之一，可选 | ADR-0013 §D4 |
| `deliver_input` 三条投递路径、`fysun.await_approval` / `fysun.pending_inputs` | 不再是 OS 规范；Approval 结果为 Runtime 可查询状态（`execution.observe`）；投递由 Harness 适配器 / 执行点自行实现 | ADR-0013 后果 |
| Provider 代理 = F5 前提 | 可选执行点（`provider_proxy`）；F5 改为"只向资格化执行点解析" | 01 §7 |
| Q5 多主体 Scope（开放） | **Tenant** = Principal + Scope + Plan，核心对象 | ADR-0013 §D6 |
| — | 新增：`Asset` / `Manifest` / `Agent Definition` / `Lock` / `Service` / `ServiceVersion` / `Endpoint` / `Tenant` / `Plan` / `Meter` / `Enforcement Point` | 本提案 |

## 2. 一级对象

| 对象 | 一句话 | 归属平面 | 详见 |
|---|---|---|---|
| **Asset** | 带 manifest 的可复用之物；`kind` 封闭枚举；有版本、digest、来源、许可证、效果声明、证据 | 资产 / 组合 | 02 |
| **Manifest** | 资产的机器可读描述；通用字段 + 按 `kind` 的专有字段 | 资产 / 组合 | 02 §2 |
| **Capability** | 行为契约：`behavior` / `quality` / `bounds`（含 `tool_effects[]`）/ `verification` | 资产 / 组合 | 03 §1 |
| **Agent Definition** | 实现 Capability 的资产组合：引用 + 覆盖 + 需求 + 环境变体；本身是 `agent` 资产 | 资产 / 组合 | 03 |
| **Lock** | Agent Definition 某版本解析后的锁定包：全部资产的精确版本 + digest + 绑定 | 资产 / 组合 | 03 §4 |
| **Runtime** | Harness 资产在某 Node 的安装实例；经 `runtime.qualify` 取得资格化纪元 | 运行 | 04 §1 |
| **Activity / Execution** | 长期活动 / 一次运行（继承 v0.7.0 02 §2.2–2.3） | 运行 | 04 §3 |
| **Enforcement Point（执行点，EP）** | 承担数据面拦截并回报事实的组件：`gateway` / `mcp_guard` / `hooks` / `harness_adapter` / `provider_proxy` / `external_gateway`；须资格化 | 运行 | 04 §4 |
| **Node** | 账本归属节点（`central` / `edge` / `cloud`）（继承） | 运行 | 04 §7 |
| **Service** | Agent Definition 的对外发布形态；有 `versions[]`、`endpoints[]`、`plans[]`、状态与 Agent Card | 服务 | 05 §2 |
| **ServiceVersion** | Service 的某个版本，钉住一个 Lock；有 rollout 状态 | 服务 | 05 §2 |
| **Endpoint** | 服务的调用入口：`protocol ∈ a2a \| mcp \| http`，由 `fysun-gateway` 暴露 | 服务 | 05 §2 |
| **Tenant** | 消费主体 = Principal + Scope + Plan；资产、Memory、Activity 的归属单位 | 服务 / 治理 | 05 §6 |
| **Plan** | 租户对某 Service 的用量约定：配额、速率、允许的 `effect_category`、审批路由；派生 Grant | 服务 | 05 §7 |
| **Meter** | 计量记录：`tokens` / `calls` / `effects{category}` / `runtime_seconds` / `approvals`；事件 `meter/recorded` | 服务 | 05 §7 |
| **Principal / Scope / Grant / Policy / Approval / ActionIntent / Effect / Verification / Handoff / Adjustment / Memory / Artifact / Event** | 继承 v0.7.0 02，字段变更见 06 | 治理 / 运行 | 06 |

## 3. 资产种类（封闭枚举）

`prompt` · `skill` · `tool` · `mcp_server` · `harness` · `loop` · `model` · `memory_provider` · `knowledge` · `runtime` · `agent`

资产 id 形态：`<kind>:<namespace>/<name>@<version>`，例如 `skill:acme/trend-research@1.2.0`、`harness:deepseek/dsh@0.7.3+cd5ef81`；
每个版本有唯一 `digest`（OCI artifact digest 或目录 canonical digest）。同一 `version` 不同 `digest` 是两个资产版本（延续 v0.7.0 07 §7）。

## 4. 状态集

| 对象 | 状态 |
|---|---|
| Asset（版本） | `draft` / `published` / `deprecated` / `withdrawn` / `quarantined` |
| Agent Definition（版本） | 同 Asset；另有 `resolved`（存在 Lock）投影 |
| Runtime | 资格化纪元：有生效 `runtime/qualified`，或最近 `runtime/disqualified{reason}`（继承） |
| Enforcement Point | `registered` / `qualified` / `disqualified{reason}` |
| Service | `draft` / `published` / `suspended` / `retired` |
| ServiceVersion | `staged` / `canary{percent}` / `stable` / `rolled_back` / `retired` |
| Tenant | `registered` / `active` / `suspended` / `closed` |
| Activity / Execution / Approval / Effect / Handoff / Verification / Adjustment / Memory | 继承 v0.7.0 00 §3 |

## 5. 事件族（新增与变更；其余继承 v0.7.0 00 §4）

新增：

- `asset/registered{asset_id, kind, version, digest, source, publisher}` · `asset/published` · `asset/deprecated{successor?}` · `asset/withdrawn{reason}` ·
  `asset/quarantined{reason}` · `asset/revoked{reason}`（供应链撤销，级联到 Lock 与 Service）
- `agent/defined{asset_id, capabilities[]}` · `agent/resolved{asset_id, lock_digest, bindings_digest}`
- `ep/registered{ep_id, kind}` · `ep/qualified{ep_id, kind, describe_digest, policy_version}` · `ep/disqualified{ep_id, reason}` ·
  `ep/policy-pulled{ep_id, policy_version}`
- `service/published{service_id, version, lock_digest, endpoints[]}` · `service/rollout{service_id, version, stage}` · `service/suspended` · `service/retired`
- `tenant/registered` · `tenant/suspended` · `plan/attached{tenant_id, service_id, plan}` · `plan/detached`
- `meter/recorded{tenant_id, service_id, execution_id, dimension, amount, policy_version?}`
- `invocation/received{tenant_id, service_id, endpoint, correlation_id}` · `invocation/completed{state}`

变更：

- 准入类事件（携带 `policy_decision`）增 `asset/published`、`service/published`、`ep/qualified`、`plan/attached`。
- `effect/*`、`action-intent/*`、`audit/observed` 增 `ep_id`（哪个执行点回报）与 `policy_version`（执行点本地评估所用快照）。
- `runtime/qualified` 增 `harness_asset_id`（该 Runtime 是哪个 Harness 资产版本的实例）。
- 通用字段增 `tenant_id`（可空：系统级事件）。

## 6. 错误码（新增；其余继承 v0.7.0 00 §8）

`asset.invalid_manifest` · `asset.digest_mismatch` · `asset.unsigned`（Policy 要求签名时）· `asset.quarantined` · `asset.kind_mismatch` ·
`compose.unresolvable{missing[]}` · `compose.effect_conflict` · `compose.requirements_unmet{missing[]}` ·
`ep.not_qualified` · `ep.policy_stale` · `ep.report_rejected` ·
`service.not_published` · `service.version_retired` · `quota.exceeded{dimension}` · `plan.effect_not_allowed{category}` · `tenant.suspended` ·
`not_runnable.environment{missing[]}`（继承，触发条件增"Lock 中任一资产 `withdrawn` / `quarantined` / `revoked`"）

## 7. 组件

| 名称 | 含义 |
|---|---|
| `fysund` | 基座 daemon；唯一权威写入者；PDP；Approval；不在数据面 |
| `fysun` | CLI |
| `fysun-gateway` | 入口执行点（`kind = gateway`） |
| `fysun-mcp-guard` | 工具侧参考执行点（`kind = mcp_guard`），独立进程，可选 |
| `fysun-dsh` | DSH Harness 适配器：Harness 资产打包、Runtime ABI、控制点、原生工具执行点（`kind = harness_adapter`） |
| `fysund-edge` | 边缘节点 daemon（继承） |
| 资产仓库 | 任意 OCI 兼容制品仓库 |

## 8. 禁令与定义句

- **唯一正式定义句**："FysunOS 是 Agent 能力管理（提示词、Skills、工具、MCP、Harness、Loop）和 Runtime 控制的运行基座。"
  README、01 §2.1、ADR-0013 §D1 字面一致；其他文档引用时不改写、不缩略；"操作系统"不再用于定义句。
- `Level-1 / Level-2 / Level-1b` 速记禁令继承（v0.7.0 00 §1.3）；本提案不再使用这些速记，统一以 `control_points[]`、`native_effect_tools[]`、执行点 `kind` 表达。
- "运行基座"不得被写成"运行时（runtime）"：Runtime 是被控制的对象，不是基座本身。
