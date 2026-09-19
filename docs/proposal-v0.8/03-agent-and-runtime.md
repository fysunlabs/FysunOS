# 03 · Agent Definition 与 Runtime

- 状态：**提案草案**
- 日期：2026-09-12
- 本文件不替代现行 [`docs/redefinition`](../redefinition/README.md) 直至被采纳。
- 决策：[00](00-decisions.md) D2–D5、D7。

## 1. 结论

Agent 是 **Agent Definition**（`kind=agent`）：一组资产引用 + 覆盖 + 需求 + 环境变体，声明它实现哪些 Capability 契约。解析得到分层 Lock。运行时把 Lock 绑到**已资格化的 Runtime 实例**（Harness 在某 Node 上的安装）。OS 不装配 prompt、不选工具、不跑 loop；它提供结构化绑定、裁剪后的工具面、生命周期与恢复。

## 2. Agent Definition

```yaml
kind: agent
metadata: { name: acme/trend-writer, version: 3.1.0, digest: … }
spec:
  capabilities: [{ ref: capability:acme/trend-research@2, role: primary }]
  harness: { ref: "harness:deepseek/dsh@>=0.7.3 <0.8" }
  loop:    { profile: "loop:acme/careful@1", overrides: { max_steps: 40 } }
  # loop 也可完全内联: { params: { max_steps: 40, stop_conditions: […] } }
  model:   { ref: "model:openai/gpt-x@*", access: via_ep, fallback: ["model:anthropic/claude-y@*"] }
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
    reject_native_tools: [shell]
    environment: { offline: false }
  variants:
    edge:
      model: { ref: "model:local/qwen-small@*", access: direct }
      harness: { ref: "harness:fysun/edge-rs@^0.2" }
      mcp_servers: []
  policy_defaults: { ref: "policy:acme/default@3" }
```

规则：

- `harness` 必选；`model` 必选（或由所选变体给出）；`loop` 缺省 = 该 Harness 的缺省 Profile，仍计算 `profile_digest`。
- `overrides` 只能改 Loop params、prompt 变量缺省、`tools_allow`。不能改 `effect_category`。
- `fallback` 是绑定顺序，不是运行时路由。
- `subagents[]` 是委派白名单；Handoff 只能指向名单或 Policy 另行允许者。
- 变体是 spec 局部覆盖，**不得**改 `capabilities[]`。变体不能满足契约 → 应指向另一个收窄的 Capability，禁止静默降质。

解析约束失败码：`compose.unresolvable` / `compose.requirements_unmet` / `compose.effect_conflict` / `asset.kind_mismatch`。控制点缺失、`reject_native_tools` 命中、memory tenant 隔离不支持、prompt slot 不在 `prompt_slots`——一律大声失败。

## 3. 解析与分层 Lock

```text
compose.resolve(agent_ref, variant?, environment?)
  1. 展开引用 → 范围内最高 published → digest
  2. 检查约束；汇总 effects / requires
  3. 切分三层 digest（契约 / 基座 / 调优）
  4. 计算 tool_surface（已收窄）
  5. 产出不可变 Lock；agent/resolved
compose.match(lock, node?) → runnable{runtime_ids[]} | not_runnable{missing[]}
```

Service 发布与 Activity 创建都钉 `lock_digest`。升级 = 新 Lock。`compose.diff` 必须标出变更落在哪一层，供证据继承与审批展示。

子 Agent：内部 Handoff 走账本 + Runtime ABI；外部走 A2A。子 Execution 的 Grant 从父衰减（scope/expiry/budget 三条不等式）。不设多 Agent 子系统（ADR-0001 §D6）。

## 4. Harness → Runtime

```text
harness 资产 published
  → 安装到 Node
  → runtime.register(descriptor)          # 运行平面对象，不是 asset.kind
  → runtime.qualify → describe() 与 harness manifest 互证
  → runtime/qualified{runtime_id, harness_asset_id, control_points[],
                      native_effect_tools[], describe_digest, bundle_digest}
```

多 Runtime 并存；资格化证据不跨实例转移；精确钉版与 CI 漂移预警继承 v0.7.0 07。资格化恒由中心完成。

## 5. 最小 Runtime ABI

| 方向 | 方法 | 义务 |
|---|---|---|
| OS→RT | `describe()` | 五字段 |
| | `start_execution(execution, lock, bindings, grants_summary, inputs)` | 无渲染文本；缺控制点/未资格化/Lock 含隔离资产 → 大声失败 |
| | `resume_execution(…, checkpoint_ref, watermarks)` | Effect + Approval 水位 |
| | `cancel` / `health` | 必须 |
| | `deliver_input?` | 可选；否则 `execution.observe` |
| | `checkpoint?` | 可 `unsupported` |
| | `park?` / `thaw?` | 控制点 `session_park`；可 `unsupported` |
| RT→OS | `os.report_progress/failure`（脱敏 ≤4096B） | 必须 |
| | `os.request_resource` / `os.propose_adjustment` / `os.submit_handoff` | |
| | `os.observe` | 查 Approval、待输入、Grant 摘要、预算——替代系统工具 `await_approval` |
| | `os.submit_candidate` / `report_receipt` / `report_observed` | 仅当 Runtime 自身是 `harness_adapter` EP |

不得调用：签发 Grant、推进 Execution 权威状态、决定 Approval、写账本、改标签。

控制点继承六个：`context_injection`（OS 给结构化视图）、`checkpoint`、`fork`、`compaction_hook`、`session_link`、`subagent_delegation`。新增声明位（非强制）：`cache_alignment`、`session_isolation`、`session_park`、`tool_search`（Harness 支持只把工具索引放进上下文，schema 按需）。

传输：本机 stdio JSON-RPC（R1）；跨机 gRPC 或 HTTP 为开放问题，不在本文件拍板。

## 6. 性能机制（OS 提供约束，Harness 执行）

这些机制的目标是降低无效 token、避免每步控制面往返、让长任务可恢复。OS **不**实现 compaction 算法。

| 机制 | 何时发生 | OS 做什么 | Harness 做什么 |
|---|---|---|---|
| 工具面裁剪 | `compose.resolve` + `start_execution` | 写入 Lock 与 EP 快照 | 只展示过滤后的 `tools/list` |
| 渐进披露 | 启动与按需 | 要求 `skill_loading` 能先给 name+description | 正文与 `references/` 仅在命中时加载（Agent Skills 标准） |
| 工具检索 | 若声明 `tool_search` | 提供工具名/描述索引，不把全量 JSON Schema 塞进绑定 | `defer_loading` 类行为；未声明则一次加载裁剪后的 schema |
| 上下文预算 | 绑定 | 提供 `prefix_order[]` 与各片段 label；不渲染 | 稳定前缀（身份、工具索引、静态 skill 元数据）在前，易变（时间戳、本轮输入）在后 |
| Compaction | 控制点 | 触发策略可来自 Loop Profile；OS 不改写会话账本 | 实现折叠；折叠产物若回 OS 必须带 label |
| Checkpoint | 控制点 / 最小 RPC | 记双水位 | 返回 checkpoint_ref |
| 恢复与再武装 | 崩溃 / 换进程 | [01](01-kernel.md) §5 顺序 | `resume` / `thaw` |
| 并行 tool call | 热路径 | EP 允许只读批次并发；mutation 与只读之间设屏障（同一批次含 mutation 则先完成 read 再 persist mutation） | 按模型并行发出；OS 不调度推测执行 |
| 代码态工具 | Harness 能力 | 工具面仍受 EP 过滤；中间结果可不进模型上下文 | 可选；内核不实现沙箱 |

`cache_alignment` **不是**资格化硬条件：MCP-only 薄包装往往做不到稳定前缀。声称该位的 Harness，资格化可跑前缀漂移探针；失败只清该位，不拒绝最小面。

## 7. Adjustment 作用点

| 范围 | 作用 | 扩权？ |
|---|---|---|
| `this_execution` / `this_activity` | Lock 级覆盖，记 `lock_overrides_digest` | 启用新工具 / 放宽 native 拒绝 / 加预算 → Approval |
| `capability_default` | Agent Definition 补丁版本 + 重新解析 | 同上 |
| 优化器提交 | 只能改 profile 层（prompt / loop params / 少数覆盖） | 改 `tool_effects` 类别或加工具 → 拒绝 `evolution.immutable_boundary` |

直接生效、留痕、可回退（P6）；证据规则见 [04](04-aaas-and-evolution.md)。

## 8. 负例

1. Lock 含版本范围而非 digest。
2. 变体解析后 `capabilities[]` 与主 spec 不同。
3. `start_execution` 带渲染文本。
4. Handoff 目标不在白名单且无 Policy 却 `working`。
5. 未声明 `session_isolation` 的 Runtime 被 `shared` 多租户绑定。
6. 优化器 Adjustment 降低 `effect_category` 或新增 MCP server 而未走 Approval。
7. `describe()` 与 harness 资产不一致却 `runtime/qualified`。
