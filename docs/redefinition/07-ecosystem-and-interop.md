# 07 · 生态与互操作

- 状态：提案草案 v0.8.0-draft（2026-09-11）
- 决策依据：[ADR-0013 草案](adr-0013-draft-repositioning.md) §D7
- 前置：[02 能力资产模型](02-asset-model.md)、[05 Agent as a Service](05-agent-as-a-service.md)

基座的价值取决于生态能否把现成的 Harness、工具、Skills、模型接进来并把 Agent 发出去。本文回答"用哪些标准、怎么用、边界在哪、三方角色怎么协作"。

## 1. 标准用法与边界

| 标准 | 在基座中的用途 | 不用于 | 备注 |
|---|---|---|---|
| **MCP** | `mcp_server` 资产的接口；工具侧 EP 的拦截面（`tools/list` / `tools/call` / `resources` / `prompts` / `tasks` / `elicitation`）；Service 的 MCP facade | `fysund` 与 Runtime 的控制通道（那是 Runtime ABI） | `fysund` 不再是 harness 的 MCP server |
| **A2A** | Service 的主要 Endpoint 协议；Agent Card 生成与签名；跨基座 / 跨租户调用；Handoff ↔ TaskState 映射 | 基座内部 Handoff 的传输（内部走账本与 Runtime ABI） | 对外暴露即多主体 |
| **Agent Skills** | `skill` 资产格式（`SKILL.md`、`allowed-tools`、`compatibility`） | 其他 `kind` 的格式 | 不自造包格式 |
| **OCI artifact** | 全部 `kind` 的打包与分发；digest 即身份；任意 OCI 仓库 | 运行时镜像编排（那是容器平台的事） | `fysun asset pack / push / register` |
| **Cedar（子集）** | Policy 语言；下发到 EP 的快照格式 | 计量与 Plan 的表达（那是 Plan 对象） | 继承 ADR-0009 |
| **OpenTelemetry** | trace / metrics 导出；`gen_ai.*` 钉 commit，`fysun.*` 自有命名空间 | 账本存储 | 继承 |
| **hooks.json（Claude Code / Codex / DSH 兼容）** | `hooks` 类 EP 的接入面 | 安全边界宣称 | 只拦不记 |
| **OpenAPI** | HTTP Endpoint 描述 | | 由 Capability `behavior` schema 派生 |

## 2. Harness 生态

| Harness | 接入形态 | 预期 `control_points[]` | 原生工具 | EP 组合 |
|---|---|---|---|---|
| DeepSeek Harness（`fysun-dsh`） | 适配器 bundle + profile；Runtime ABI 由适配器实现 | 全部六个 | 有，经适配器拦截 | `harness_adapter` (+ `mcp_guard`) |
| DSH headless | 薄包装实现 Runtime ABI | 空 | 待查证 | `mcp_guard` |
| Claude Code / Codex | 薄包装 + hooks.json | 空 | 有 | `mcp_guard` + `hooks` |
| Goose / OpenCode / OpenClaw / Gemini CLI | 薄包装 | 空 | 有 | `mcp_guard` |
| 边缘 Runtime（Rust / Python） | 本体实现 Runtime ABI | 按实现 | 通常无 | `mcp_guard`（在 `fysund-edge` 内） |
| 自研 / 第三方 SDK harness | 本体实现 Runtime ABI + 控制点 | 按实现 | 按声明 | 任意 |

接入义务对所有 Harness 一致：打包为 `harness` 资产（manifest 含 `control_points[]`、`native_effect_tools[]`、`loop_schema`、`prompt_slots[]`、`tool_surface`）；
实例经 `runtime.register` + `runtime.qualify`；`describe()` 与 manifest 互证。薄包装进程不在 TCB 内，其声明受"声明不实不在承诺范围"约束（继承）。

## 3. 资产打包与分发

```text
fysun asset pack ./trend-research      # 读取 manifest，计算 canonical digest，产出 OCI artifact（config = manifest，layers = 内容）
fysun asset push registry.example/acme/trend-research:1.2.0
fysun asset register oci://registry.example/acme/trend-research@sha256:…   # fysund 拉取 manifest、重算 digest、Policy 评估、asset/registered
fysun asset publish skill:acme/trend-research@1.2.0                         # asset/published（准入类；可要求证据门）
```

- `fysund` 只存登记与 digest；内容由 Runtime / EP 按 Lock 从仓库拉取（或由运营者预置本地缓存）；拉取后重算 digest，不一致 → `asset.digest_mismatch`，Execution 不启动。
- 私有仓库凭据以 `SecretRef` 登记，由 EP / Runtime 宿主解析。
- 仓库联邦：多个基座共享同一 OCI 仓库即共享资产；登记与证据各自维护，不同步。

## 4. 三方协作流程

```text
发布者                          运营者                             消费者（租户）
────                            ────                               ────
pack / push 资产                运行 fysund + gateway + EP
agent.define → compose.resolve  runtime.register / qualify
service.publish（staged）       policy.load（部署 Policy）
                                approval（发布审批，若 Policy 要求）
service.rollout（canary）       ep.qualify（gateway / mcp_guard）
                                                                   service.search → Agent Card
                                                                   plan.attach（订阅）
                                                                   A2A / MCP / HTTP 调用
                                approval（Effect 审批，按路由）      approval（按路由）
service.rollout（stable）       asset.quarantine（供应链事件）       tenant.usage / evidence
```

## 5. 兼容性与版本策略

- manifest `apiVersion` 独立演进；`fysund` 支持 N 与 N-1；Lock 记录解析时的 `apiVersion`。
- Runtime ABI 与 EP 契约带版本号；Harness / EP 的 `describe()` 声明支持的版本；不匹配 → 拒绝资格化，不降级。
- 资产 `deprecated → withdrawn` 至少一个发布周期；`revoked` 无宽限。

## 6. 与现有生态项目的关系

- **DeepSeek Harness / Cordis**：参考 Harness 与适配器；Cordis 语义只在适配器内部（继承 ADR-0001 §D2）。
- **agentgateway**：候选 `external_gateway` EP；接受 Cedar 快照与 EP 回报契约后可作为工具侧 EP；ADR-0012 的"不嵌入"结论不变，"不前置"结论重开。
- **记忆引擎（MemOS / Mem0 / Letta）**：`memory_provider` 资产；须支持 per-tenant 命名空间才能用于多租户 Service。
- **CognitiveOS / Personal**：Personal 作为第一个发布者 + 运营者 + 租户三合一的应用；CognitiveOS 的 Effect 协议进入参考 EP 实现。

## 7. 负例（R1 / R3）

1. 某 `kind` 采用了非 OCI 的自造分发格式 → 失败。
2. `fysund` 与 Runtime 的控制通道使用 MCP 而非 Runtime ABI → 失败。
3. Service 的 A2A Agent Card 未由 `fysund` 签名或 `skills[]` 与 Lock 不一致 → 失败。
4. Runtime 或 EP 拉取的资产内容 digest 与 Lock 不一致仍启动 Execution → 失败。
