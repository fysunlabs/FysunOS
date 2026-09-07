# 04 · 接口与 ABI

- 状态：设计稿 v0.1（2026-09-06）
- 前置：[02 对象模型](02-object-model.md)、[03 架构](03-architecture.md)

FysunOS 对外有三组接口。它们约定**外部行为**，不统一内部思考步骤；只有需要机器执行或跨
组件交接的部分才必须结构化，其余可以保留为自然语言与可引用资源。

| 接口 | 方向 | 使用者 |
|---|---|---|
| **Capability API** | 应用 ↔ OS | 产品、Agent Shell、SDK、CLI、UI |
| **Harness ABI** | Runtime ↔ OS | DSH、边缘 Runtime、其他执行引擎 |
| **Composition ABI** | 组件图 ↔ OS | Rust 原生后端、Cordis adapter、未来 WASM 后端 |

线协议一律语言无关（JSON-RPC over stdio / 本地 socket，MVP），所有跨边界对象带 `digest`。

## 1. Capability API

面向应用与人机入口，按对象分组：

### 1.1 Capability

```text
capability.register(contract, implementation, requirements)      -> capability_id
capability.revise(id, contract? , implementation?, requirements?) -> new version / derived id
capability.derive(from_id, reason, changes)                       -> new capability_id (lineage)
capability.find(query: natural language | structured filter)      -> [summary]
capability.describe(id)                                           -> full record incl. evidence
capability.match(id, environment)                                 -> runnable | adaptable(bindings) | not_runnable(missing[])
capability.withdraw(id)
```

### 1.2 Activity / Execution

```text
activity.create(capability_id, target, constraints, triggers?, policy_binding?) -> activity_id
activity.start(id) / pause(id) / resume(id) / cancel(id)
activity.trigger(id, event)
activity.observe(id)               -> state, current executions, blockers, budget, artifacts
activity.wait(id, condition, timeout)
execution.observe(id)              -> state, checkpoints, effects summary, handoffs
execution.cancel(id)
execution.fork(id, boundary?)      -> new execution_id
```

### 1.3 Handoff / Artifact / Verification（观察面）

```text
handoff.list(execution_id) / handoff.observe(id)
artifact.get(ref) / artifact.put(bytes, media_type) -> ref
verification.report(id)
```

### 1.4 Adjustment

```text
adjustment.apply(target, scope, change, effective_from?)  -> adjustment_id (auto risk_class)
adjustment.revert(id)
adjustment.promote(id, to_scope)    -> e.g. this_activity -> capability_default
adjustment.history(target)
```

### 1.5 Resource / Grant / Policy（管理面）

```text
resource.register(descriptor) / resource.list(filter) / resource.health(id)
secret.import(source, name) -> secret_ref    （只经批准的非日志路径；永不返回明文）
grant.list(principal, scope) / grant.revoke(id)
policy.load(path) / policy.describe()
```

### 1.6 系统

```text
system.status() / system.events(filter, cursor) / system.replay_check()
```

## 2. Harness ABI

Runtime 必须实现（被 OS 调用）与可以调用（调用 OS）的两组操作。

### 2.1 OS → Runtime（Runtime 必须实现）

```text
runtime.describe()                          -> type, version, supported control points (capability bits)
runtime.start_execution(execution, context_view, agent_core_config, grants_summary) -> session_id
runtime.resume_execution(execution, checkpoint_ref, effect_watermark)             -> session_id
runtime.fork_execution(execution, boundary)                                        -> session_id
runtime.deliver_input(execution, message | event | handoff_result)
runtime.checkpoint(execution)               -> checkpoint_ref
runtime.cancel(execution, reason)
runtime.health()
```

`supported control points` 是能力位，例如：`context_injection`、`tool_gate`、`approval_gate`、
`checkpoint`、`fork`、`compaction`、`subagent_delegation`。Capability.requirements 可以要求特定
控制点；Capability 子系统据此做环境匹配。**对不可控制的内部机制，Runtime 必须如实声明，
OS 不宣称全面管理。**

### 2.2 Runtime → OS（Runtime 可以调用）

```text
os.request_context(execution, need)               -> context_view (authorized, digest-bound, losses[])
os.submit_candidate(execution, candidate)         -> admitted(effect_id) | rejected(reason) | needs_human(request_id)
os.observe_effect(effect_id)                      -> state, receipt?
os.request_resource(execution, resource_request)  -> binding | denied
os.report_checkpoint(execution, checkpoint_ref, session_watermark)
os.report_progress(execution, summary)
os.request_handoff(execution, to, objective, inputs, expected) -> handoff_id
os.report_failure(execution, error, recoverable)
os.propose_memory(execution, candidate)           -> queued for admission
os.propose_adjustment(execution, change, scope)   -> adjustment_id | needs_human
```

**候选（candidate）**是统一的提交单元：工具调用、外部动作、记忆写入、能力调整、委派请求
都以候选进入 OS，由 OS 决定准入。**工具调用在 ABI 层的定义就是 Effect 候选。**

### 2.3 Runtime 不得调用（不存在于 ABI 中）

```text
mint_grant / commit_effect / complete_activity / write_ledger / mark_verified
```

### 2.4 会话账本关联义务

Runtime 在自己的会话账本中记录 OS 提供的 `execution_id`、`fencing_epoch`，并在每次
`report_checkpoint` 时提交会话水位；OS 在治理账本中记录 `session_id`。任何一方都可以从
自己的账本出发定位另一方。

## 3. Composition ABI

语言无关的组件生命周期接口，抽取自 Cordis 的可撤销效果 + 反应式依赖语义，由不同后端实现：

```text
mount(component, scope, dependencies, config)   -> revision
unmount(component, revision)
replace(component, new_revision)
inspect(scope)                                  -> component graph
dependency_status(component)                    -> satisfied | waiting(missing[]) | broken
health(component)
dispose(component)
quarantine(component, reason)
```

每次生命周期变化产出：

```json
{
  "component_id": "component://...",
  "revision": "sha256:...",
  "scope": "scope://...",
  "dependencies": [],
  "lifecycle": "active | inactive | quarantined",
  "disposal": "complete | partial | failed",
  "event_ref": "event://..."
}
```

| 后端 | 服务对象 | 说明 |
|---|---|---|
| Rust 原生后端 | `fysund` 自身的适配器、Verifier、策略模块 | 最小 mount/unmount/replace/dispose |
| **Cordis adapter** | TS Runtime（DSH 及其插件树） | 把 Context / Fiber / Service / dispose 映射到 ABI；Cordis 不被 fork、不成为内核依赖 |
| WASM 后端（后续） | 可移植能力实现 | 可选 |

规则：组件消失（例如 Cordis service 卸载）**不得绕过** OS 的 Grant 撤销与 Effect 对账；
组件图是依赖图，不是信任/委派图。

## 4. 错误与拒绝语义

所有接口的拒绝都带稳定错误码与可判定原因，至少区分：

- `denied.policy`（部署策略禁止）
- `denied.grant`（无授权或已撤销）
- `denied.budget`（预算耗尽）
- `denied.fencing`（陈旧 epoch）
- `not_runnable.environment`（环境不满足，附缺失清单）
- `needs_human`（策略要求人工确认，附请求 id）
- `unknown_outcome`（Effect 结果未知，进入对账）

拒绝不是失败测试；`not-run` 永不等于通过。
