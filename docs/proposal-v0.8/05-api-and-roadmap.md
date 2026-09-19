# 05 · API 面与路线图

- 状态：**提案草案**
- 日期：2026-09-12
- 本文件不替代现行 [`docs/redefinition`](../redefinition/README.md) 直至被采纳。
- 未在此列出的方法语义继承 v0.7.0 [04](../design/04-capability-api.md) / [05](../design/05-runtime-abi.md) 中仍有效的部分。

## 1. 结论：四面 + 两份契约

应用 ↔ `fysund` 只有四组管理 API。Runtime ABI 与 EP 契约是控制面↔数据面的 RPC，**不是** REST 资源。消费者调用走 A2A / MCP / HTTP（gateway），不进本表。

```text
Catalog     资产 + 契约 + 组合 + 命名 Profile + Adjustment
Runtime     Node / Runtime 实例 / EP / Activity / Execution / 观察面
Service     Service 目录 / Tenant / Plan 附着 / meter 查询
Governance  Policy / Grant 观察 / Approval / Secret / Memory / replay
契约        Runtime ABI · EP 契约
```

相对 redefinition：Registry∪Composition → Catalog；去掉独立 Endpoint/Meter 资源；`runtime.register` 离开 Registry。ABI/EP 见 [03](03-agent-and-runtime.md) §5 与 [01](01-kernel.md) §3，此处不重复方法表。

## 2. Catalog

| 方法 | 说明 |
|---|---|
| `asset.register / publish / deprecate / withdraw / quarantine / revoke` | publish 为准入类 |
| `asset.describe / search / health / evidence` | 描述面无值槽、无内容本体 |
| `capability.register / describe / search / health` | 契约；`tool_effects[]` 由组合汇总后确认 |
| `agent.define` · `compose.resolve` · `compose.match` · `compose.diff` | diff 标注 contract/substrate/profile |
| `profile.put(loop, …)` / `profile.describe` | 命名 Loop Profile，非 `asset.kind` |
| `adjustment.apply / revert / promote / describe` | 包络检查见 [04](04-aaas-and-evolution.md) B3 |
| `trajectory.export(execution_id)` | 脱敏轨迹，供优化器；不含 Secret |

没有 `asset.register(kind=runtime|loop|tool)`。

## 3. Runtime

| 方法 | 说明 |
|---|---|
| `node.register / list / describe` | |
| `runtime.register / qualify / list / health` | 实例，不是资产 |
| `ep.register / qualify / list / health` | |
| `activity.create / start / pause / resume / cancel / describe / list` | `create` 钉 `lock_digest`；可带 `tenant` |
| `activity.park(activity_id)` | 请求 Runtime `park`；不支持则忽略并保持进程 |
| `execution.observe` | 状态、waiting.reason、Approval **结果枚举**、预算、生效 EP；不含 `decided_by` / `reason` 文本 |
| `execution.deliver_input` | 入队或 ABI `deliver_input` |
| `handoff.*` · `artifact.*` · `verification.*` | 观察面 |

## 4. Service

| 方法 | 说明 |
|---|---|
| `service.create / publish / rollout / suspend / retire` | 版本单位 = Lock |
| `service.describe / search / health / agent_card` | Card 由 `fysund` 签名 |
| `tenant.register / suspend / close / describe / usage` | |
| `plan.define / attach / detach` | attach 准入类；可 `pin` 版本 |
| `meter.query / export` | 投影；无 `meter.create` |

无 `endpoint.*` CRUD。Endpoint 随 `service.publish` 的字段提交。

## 5. Governance

`policy.load / revise / retire / describe / explain / simulate`  
`grant.list / describe / revoke`（无公开 `grant.issue`）  
`approval.list / describe / decide / batch_decide`（校验路由集合）  
`memory.list / retract / describe`  
`secret.register / rotate / revoke`（只返回 SecretRef）  
`system.status / replay / export`

## 6. 错误码原则

- 稳定前缀：`asset.*` · `compose.*` · `ep.*` · `service.*` · `quota.*` · `plan.*` · `tenant.*` · `not_runnable.*` · `denied.*` · `conflict.*` · `unavailable.*` · `needs_human` · `evolution.*`
- 错误码表达**可判定失败类**，不把内部堆栈塞进模型可见面。
- 新增：`evolution.immutable_boundary`（优化器越权）；`quota.exceeded{dimension:sampling}`。
- 继承 redefinition / v0.7.0 已列码，不重复抄表。

负例：描述面出现明文 secret；公开 `grant.issue`；`execution.observe` 返回审批理由文本；Catalog 返回资产字节本体。

## 7. 里程碑 R0–R3

原则：第一个闭环要小、真、可判定。不先全量回写 `docs/design`。本文件夹 + 采纳后的 ADR 为实现规格；实现中再回填 design。

| 阶段 | 做成什么 | 可验证退出 |
|---|---|---|
| **R0 规格** | 所有者采纳本提案 D1–D8（或列出修改）；ADR-0013 按 [00](00-decisions.md) 差异改写后进入 `docs/adr`；术语表以本 README + 02 为准 | grep：无「十一种资产」「`kind=runtime` 为可打包资产」「`fysund` 是 harness 的 MCP server」；定义句与 README 一致 |
| **R1 登记·锁定·跑通** | 八种 manifest schema；`pack/push/register/publish`；Loop Profile put；分层 `compose.resolve`；一个 Harness（薄包装或 `fysun-dsh`）实现最小 ABI；Activity/Execution + 恢复再武装；账本 replay digest 稳定；`describe()` 与 harness 资产互证 | [02](02-assets.md) §8.1–7；[03](03-agent-and-runtime.md) §8.1,3,7；闭环：prompt+skill+mcp_server+harness+model → agent.define → resolve → match → run → 独立验证 → 杀进程恢复；`fysund` 不在 tool/模型转发链 |
| **R2 执行点·入口服务** | EP 契约；`mcp_guard` 或 adapter 覆盖 MCP 工具面；F3 分级；sampling 防御；`fysun-gateway` 至少一个 A2A Endpoint + 签名 Agent Card；单 Tenant 可调用；`os.observe` 替代系统工具 | [01](01-kernel.md) §9；[04](04-aaas-and-evolution.md) A5.1,5,7；sampling 不可窃取 Secret；只读不强制同步 WAL；Policy 过期 fail-closed |
| **R3 多租户·演进·第二 Harness** | Tenant/Plan/meter；canary 可审计；租户隔离；Adjustment + 分层证据 + 证据门；一个优化器插件（可假实现）只能提交 profile；第二 Harness 跑同一契约的变体；可选 `park` | [04](04-aaas-and-evolution.md) A5.2–4,6,8 与 B5；两租户 Memory/用量分离；profile 微调保留 `inherited`；越权 Adjustment 被拒 |

相对 redefinition R0–R5：不把「Trust 对象 / 边缘账本 / Personal 全量 / 全控制点 DSH」绑进前四个门。它们是 R3 之后的工作项，不承诺本提案内核未包含的对象。

不做进 R0–R3：SPIFFE、自造零拷贝传输、数值置信度折损、OS 内置 ACE、推测执行调度器、`shared` 池化的形式化隔离证明。

## 8. 开放问题（带入实现，不阻塞 R0）

| ID | 问题 |
|---|---|
| Q1 | ABI/EP 跨机传输：gRPC vs HTTP JSON-RPC |
| Q2 | Secret Store 后端 |
| Q3 | 审批 Web UI 范围 |
| Q4 | `model_judge` 校准集与替代人工的阈值 |
| Q5 | canonical digest 算法与签名（Sigstore vs 自签） |
| Q6 | Harness 自报 token 在多租户是否计入；`provider_proxy` 是否成为计量前提 |
| Q7 | MCP 2026-07-28 无状态远程 vs 本机 stdio 会话：EP 双模式如何声明 |
| Q8 | 跨基座 Trust 对象（原 Q11/Q27）——单独 ADR，不进 R3 退出条件 |

v0.7.0 Q5 多主体：本提案关闭为「R3 必做，R1 单租户」。Q14 路由：不做。

## 9. 待所有者拍板（≤8）

1. **是否以本文件夹替换 redefinition 作为 v0.8 工作规格**（采纳后才改 `docs/design` / ADR-0013 正文；本轮仍不改那些文件）。
2. **R1 首个 Harness**：DSH headless 薄包装（快、控制点少）还是 `fysun-dsh`（慢、原生工具可记）。
3. **Personal 是否允许绕过 gateway**（本提案：允许，Tenant 仍在）。
4. **R2 工具侧 EP**：第一方 `fysun-mcp-guard`，还是 hooks + adapter，外部网关后置。
5. **Loop Profile**：R1 即提供命名 `profile.put`，还是 R1 只允许 Agent Definition 内联。
6. **证据门默认**：对外 `stable` 是否强制回归集（本提案：Policy 决定，建议对外默认开）。
7. **MCP 2026-07-28**：R2 gateway/MCP facade 是否必须声明该版本，或 R2 钉 2025-11-25、R3 再升。
8. **`session_park`**：R3 必做还是保持可选 `unsupported`（本提案：可选）。
