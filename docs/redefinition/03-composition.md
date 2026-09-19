# 03 · 组合：Agent Definition 与 Capability 契约

- 状态：提案草案 v0.8.0-draft（2026-09-11）
- 决策依据：[ADR-0013 草案](adr-0013-draft-repositioning.md) §D1、§D2
- 前置：[00 术语表](00-terminology.md)、[02 能力资产模型](02-asset-model.md)

本文回答"资产如何组合成一个 Agent、契约在哪里、解析与锁定怎么做、环境差异怎么表达、证据归谁"。

## 1. Capability 契约（继承并收口）

Capability 仍是行为契约（v0.7.0 02 §2.1），字段不变：`behavior`、`quality`（Skill Contract 骨架 P/O/A/V/F，V 为 checklist 并标 `verifier_kind`）、
`bounds`（成本、时间、权限、风险；`tool_effects[]`）、`verification`。两处变化：

- `implementation` 字段删除；Capability 由一个或多个 **Agent Definition** 实现，关系记录在 Agent Definition 侧（`spec.capabilities[]`），
  证据分别归属（§6）。
- `bounds.tool_effects[]` 不再手写，而是由组合汇总生成后**由发布者确认并可收窄**（02 §6）；`capability.register` 仍校验其覆盖组合中全部工具。

Capability 面向使用者仍只回答四个问题：做什么、怎样算做好、依靠什么做、在什么边界内做。

## 2. Agent Definition（`kind = agent`）

```yaml
apiVersion: fysun.io/v1alpha1
kind: agent
metadata: { name: acme/trend-writer, version: 3.1.0, digest: …, publisher: principal:acme, license: … }
spec:
  capabilities:                       # 实现哪些契约
    - ref: capability:acme/trend-research@2
      role: primary
  harness: { ref: "harness:deepseek/dsh@>=0.7.3 <0.8", adapter: "fysun-dsh" }
  loop:    { ref: "loop:acme/careful@1", overrides: { max_steps: 40 } }
  model:   { ref: "model:openai/gpt-x@*", access: via_ep, fallback: [ "model:anthropic/claude-y@*" ] }   # fallback 是绑定顺序，不是运行时路由
  prompts:
    - { ref: "prompt:acme/persona@1.4.0", slot: system }
    - { ref: "prompt:acme/style-guide@2", slot: developer }
  skills:  [ { ref: "skill:acme/trend-research@1.2.0" }, { ref: "skill:acme/content-pack@0.9" } ]
  tools:   [ { ref: "tool:acme/publish-xhs@1", enabled: false } ]      # 显式禁用 = 从工具面剔除
  mcp_servers: [ { ref: "mcp_server:acme/search@^2", tools_allow: [ search.query ] } ]
  memory:  { ref: "memory_provider:memos/default@1", scope: tenant }
  knowledge: [ { ref: "knowledge:acme/brand-kb@2026.09" } ]
  subagents:                            # 可委派目标：其他 Agent Definition 或外部 Service
    - { ref: "agent:acme/illustrator@1", via: handoff }
    - { service: "a2a://partner.example/summarizer", via: a2a }
  requirements:                         # 汇总后可再加严
    control_points: [ ]
    reject_native_tools: [ shell ]      # 拒绝含无沙箱 shell 的 Harness
    environment: { offline: false }
  variants:                             # 环境变体（§5）
    edge:
      model: { ref: "model:local/qwen-small@*", access: direct }
      harness: { ref: "harness:fysun/edge-rs@^0.2" }
      mcp_servers: [ ]
  policy_defaults: { ref: "policy:acme/default@3" }   # 产品缺省 Policy，部署 Policy 可覆盖
declares:
  effects: [ ]                          # 由 compose.resolve 汇总填充；发布者只能收窄
  requires: { … }
evidence: [ ]
```

规则：

- Agent Definition 本身是资产：有版本、digest、发布、弃用、隔离。
- 每一项引用都是**资产引用**，可带版本范围；`overrides` 只能覆盖目标资产 manifest 允许覆盖的字段（Loop 的 `params`、prompt 的 `variables` 缺省值、
  mcp_server 的 `tools_allow`），不能改 `declares.effects[]` 的类别。
- `harness` 必选、`loop` 可选（缺省用 Harness 自带缺省 Loop，仍需登记为 `loop` 资产以便追溯）、`model` 必选（或由变体给出）。
- `tools[].enabled: false` 与 `mcp_servers[].tools_allow` 是**工具面收窄**的声明来源，执行点据此过滤 `tools/list`（F9 的静态部分）。
- `subagents[]` 是可委派目标白名单；运行时 Handoff 只能指向白名单内目标或 Policy 另行允许者。

## 3. 组合约束（解析时检查）

| 约束 | 失败码 |
|---|---|
| 每个引用可解析到 `published` 版本（`deprecated` 允许但告警；`withdrawn` / `quarantined` / `revoked` 不允许） | `compose.unresolvable{missing[]}` |
| `loop.harness_ref` 匹配所选 Harness；`loop.params ⊨ harness.loop_schema` | `asset.kind_mismatch` |
| 全部 `requires.control_points` ⊆ `harness.provides.control_points` | `compose.requirements_unmet{missing[]}` |
| `requirements.reject_native_tools ∩ harness.native_effect_tools = ∅` | `compose.requirements_unmet` |
| 同名工具在不同资产中的 `effect_category` 取 `max_severity`；Definition 的 `declares.effects` ⊆ 汇总集（只能剔除） | `compose.effect_conflict` |
| `prompts[].slot ∈ harness.prompt_slots` | `compose.requirements_unmet` |
| `memory.scope = tenant` 时 `memory_provider.scoping` 必须支持 per-tenant | `compose.requirements_unmet` |
| manifest 与引用中无 secret 值槽 | `asset.invalid_manifest` |

## 4. 解析与锁定（`compose.resolve`）

```text
agent.define(manifest) ──▶ agent/defined
compose.resolve(agent_ref, variant?, environment?)
  1. 展开引用 → 选版本（范围内最高 published）→ 取 digest
  2. 检查 §3 约束
  3. 汇总 declares.effects / requires
  4. 产出 Lock{ agent_ref, variant, assets[]{kind, ref, version, digest}, bindings{slots, overrides}, effects[], requirements, lock_digest }
  5. agent/resolved{asset_id, lock_digest, bindings_digest}
```

- **Lock 是不可变的**；同一 Agent Definition 版本 + 同一变体 + 同一解析时刻的资产目录状态 → 同一 `lock_digest`（可重放）。
- Service 发布、Activity 创建都钉住 `lock_digest`，不钉浮动范围；升级 = 新 Lock。
- `compose.match(lock, node?)`：对照已资格化 Runtime（`runtime.list`）与 Node 资源，返回 `runnable{runtime_ids[]}` / `not_runnable{missing[]}`；
  不设兼容等级，不静默降质（继承）。

## 5. 环境变体

`variants.<name>` 是对 `spec` 的**局部覆盖**（同一契约、不同资产绑定），解析时选定一个变体。典型：`desktop` / `cloud` / `edge`。
变体不改变 `capabilities[]`；若某变体无法满足契约（如 edge 缺少必要工具），发布者应把它声明为实现另一个收窄的 Capability，而不是静默降质。

## 6. 证据归属

- Verification 结果同时记入：Capability（契约层证据）、Agent Definition 版本（组合层证据）、Lock（精确组合证据）、以及各资产版本（资产层证据，按贡献标注：
  例如 `verifier_kind = deterministic` 的 schema 校验记到 `skill`，`downstream_outcome` 记到 `agent`）。
- `evidence` 摘要在发现接口按层展示；换 Harness / Model 后旧证据保留在旧 Lock，不迁移（延续"资格化证据不跨 id 转移"）。
- `asset.health(ref)`：未分类工具、依赖被隔离、证据全部来自已 `disqualified` 的 Runtime、反复失败的 checklist 项。

## 7. Adjustment 作用于组合

Adjustment（继承 v0.7.0 02 §2.10）的 `target` 扩展为：Agent Definition 的某个引用或覆盖（换 prompt 版本、改 Loop 参数、禁用某工具、换 Model）。
`scope = capability_default` 对应"产生 Agent Definition 新补丁版本并重新解析"；`this_activity` / `this_execution` 对应 Lock 级覆盖（记录 `lock_overrides_digest`，
不改 Lock 本身）。触及扩权（启用新工具、放宽 `reject_native_tools`、提高预算）走 Policy / Approval；其余直接生效、留痕、可回退。

## 8. 负例（R2）

1. Lock 中出现版本范围而非精确 digest → 失败。
2. `compose.resolve` 对缺失控制点的 Harness 返回 `runnable` → 失败。
3. Definition `declares.effects[]` 把某工具类别降到低于汇总集 → 失败。
4. Handoff 目标不在 `subagents[]` 且无 Policy 允许却进入 `working` → 失败。
5. 变体解析后 `capabilities[]` 与主 spec 不一致 → 失败。
6. Adjustment 启用被 Definition 禁用的工具而未产生 Approval → 失败（F9）。
