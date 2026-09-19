# 08 · API 面

- 状态：提案草案 v0.8.0-draft（2026-09-11）
- 前置：[00 术语表](00-terminology.md)；方法语义详见各篇
- 继承：v0.7.0 [04 Capability API](../design/04-capability-api.md)、[05 Runtime ABI](../design/05-runtime-abi.md) 中未被本文改动的方法

六组接口：Registry / Composition / Runtime / Service / Governance 为应用 ↔ `fysund` 的管理 API；Runtime ABI 与 EP 契约为 `fysund` ↔ 数据面组件的控制契约。
消费者面向的调用接口不在此列（A2A / MCP / HTTP，由 gateway 暴露，见 05）。

## 1. Registry API（资产）

| 方法 | 说明 |
|---|---|
| `asset.register(ref \| manifest) -> asset_id` | 拉取 manifest、重算 digest、Policy 评估；`asset/registered` |
| `asset.publish(asset_id)` / `asset.deprecate(asset_id, successor?)` / `asset.withdraw(asset_id, reason)` | 生命周期；`publish` 为准入类，可产生 `subject = asset_publish` 的 Approval |
| `asset.quarantine(asset_id, reason)` / `asset.revoke(asset_id, reason)` | 运营者供应链动作；级联 |
| `asset.describe(asset_id)` / `asset.search(query)` / `asset.health(asset_id)` | 描述面无值槽 |
| `asset.evidence(asset_id) -> evidence[]` | 按层（资产 / 组合 / Lock）返回 |
| `runtime.register(descriptor)` / `runtime.qualify(runtime_id)` / `runtime.list()` / `runtime.health(runtime_id)` | Runtime 是 `kind = runtime` 的资产；继承 v0.7.0 07 §1 |

## 2. Composition API（组合）

| 方法 | 说明 |
|---|---|
| `capability.register(contract)` / `capability.describe` / `capability.search` / `capability.health` | 契约；`tool_effects[]` 由组合汇总后确认 |
| `agent.define(manifest) -> asset_id` | Agent Definition 即 `kind = agent` 的资产；`agent/defined` |
| `compose.resolve(agent_ref, variant?, environment?) -> Lock` | 解析与锁定；`agent/resolved`；失败码 `compose.*` |
| `compose.match(lock_digest, node?) -> runnable{runtime_ids[]} \| not_runnable{missing[]}` | 环境匹配 |
| `compose.diff(lock_a, lock_b)` | 两个 Lock 的资产差异（升级审查用） |
| `adjustment.apply / revert / promote / describe` | 继承；`target` 扩展到 Definition 引用与 Lock 级覆盖 |

## 3. Runtime API（运行控制）

| 方法 | 说明 |
|---|---|
| `activity.create(lock_digest, target, constraints, triggers, policy_binding, tenant?) / start / pause / resume / cancel / describe / list` | 继承；`resume` 解除 `rearm_required` |
| `execution.observe(execution_id)` | 状态、`waiting.reason`、Approval 结果、待投递输入、预算余量、生效 EP |
| `execution.deliver_input(execution_id, input)` | 由 `fysund` 记账并交给 Runtime ABI `deliver_input`（若支持）或入待投递队列 |
| `handoff.describe / list` · `artifact.describe / fetch` · `verification.describe / list` | 观察面，继承 |
| `ep.register(descriptor)` / `ep.qualify(ep_id)` / `ep.list()` / `ep.health(ep_id)` | 执行点管理 |
| `node.register / list / describe` | 继承 |

## 4. Service API（AaaS）

| 方法 | 说明 |
|---|---|
| `service.create(agent_ref, tenant_visibility)` / `service.publish(service_id, lock_digest, endpoints[], plans[])` / `service.rollout(service_id, version, stage)` / `service.suspend` / `service.retire(version)` | 发布与版本 |
| `service.describe / search / health / agent_card(service_id)` | 发现 |
| `tenant.register / suspend / close / describe / usage(tenant_id, period)` | 租户 |
| `plan.define(service_id, plan)` / `plan.attach(tenant_id, service_id, plan_id, pin?)` / `plan.detach` | 订阅；`attach` 为准入类 |
| `meter.query(filter)` / `meter.export(period, format)` | 计量查询与导出 |

## 5. Governance API（治理）

| 方法 | 说明 |
|---|---|
| `policy.load(source)` / `policy.revise` / `policy.retire` / `policy.describe(policy_id?)` / `policy.explain(event_ref)` / `policy.simulate(input)` | 继承 + `simulate`（对给定 context 干跑，运营者验证下发前的效果） |
| `grant.list(principal, scope)` / `grant.revoke(grant_id)` / `grant.describe` | 观察与撤销；签发只由内部流程 |
| `approval.list / describe / decide / batch_decide` | 继承；`decide` 校验路由允许集合 |
| `memory.list / retract / describe` | 继承 |
| `secret.register(name, scope) -> SecretRef` / `secret.rotate` / `secret.revoke` | 只返回引用；描述面无值槽 |
| `system.status` / `system.replay(range)` / `system.export(range)` | 继承 |

## 6. Runtime ABI（`fysund` ↔ Runtime）

见 [04 §2](04-runtime-substrate.md)。方法：`describe` / `start_execution` / `resume_execution` / `cancel` / `health` / `deliver_input?` / `checkpoint?`；
回调：`os.report_progress` / `os.report_failure` / `os.request_resource` / `os.propose_adjustment` / `os.submit_handoff` / `os.observe`；
仅 `harness_adapter` 类 EP 使用：`os.submit_candidate` / `os.report_receipt` / `os.report_observed`。
传输：stdio JSON-RPC（本机）/ gRPC 或 HTTP JSON-RPC（跨机，R3 定）。

## 7. EP 契约（`fysund` ↔ 执行点）

见 [04 §4.2](04-runtime-substrate.md)。方法：`ep.describe` / `ep.policy_pull` / `ep.report` / `ep.resolve_secret` / `ep.health`；
`fysund` → EP 通知：`policy_invalidated{policy_version}`、`grant_revoked{grant_id}`、`execution_cancelled{execution_id}`。
传输：本机 stdio / Unix socket；跨机 gRPC（mTLS）。

## 8. 错误码（新增汇总；继承 v0.7.0 04 §3 全表）

| 错误码 | 触发 |
|---|---|
| `asset.invalid_manifest` / `asset.digest_mismatch` / `asset.unsigned` / `asset.quarantined` / `asset.kind_mismatch` | 登记、解析、拉取 |
| `compose.unresolvable{missing[]}` / `compose.effect_conflict` / `compose.requirements_unmet{missing[]}` | 解析 |
| `ep.not_qualified` / `ep.policy_stale` / `ep.report_rejected` | 执行点 |
| `service.not_published` / `service.version_retired` / `quota.exceeded{dimension}` / `plan.effect_not_allowed{category}` / `tenant.suspended` | 服务面 |
| `not_runnable.environment{missing[]}` | 继承；触发条件增 Lock 含被隔离 / 撤销资产、无满足 Plan 数据驻留的 Runtime |
| `needs_human(approval_id)` / `denied.*` / `conflict.idempotency` / `unavailable.approval` | 继承 |

## 9. 负例（R1–R3）

1. 任一管理 API 的描述面返回可放明文的值槽 → 失败。
2. `grant.issue` 作为公开方法存在 → 失败（签发只由内部流程）。
3. `execution.observe` 返回 Approval 的 `decided_by` / `reason` / `presentation` → 失败（审批事实不进模型可见面）。
4. Registry API 返回资产内容本体而非引用 → 失败。
