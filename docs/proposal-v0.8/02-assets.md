# 02 · 资产模型

- 状态：**提案草案**
- 日期：2026-09-12
- 本文件不替代现行 [`docs/redefinition`](../redefinition/README.md) 直至被采纳。
- 决策：[00](00-decisions.md) D2/D7。

## 1. 结论

资产 = 可复用、可分发、可被 Agent Definition 引用、有稳定身份与内容 digest、可脱离某次 Execution 存在。八种 `kind`，一个统一 manifest。Loop Profile 可登记但不占 `kind`。Runtime 实例不是资产。

`fysund` 只存登记与 digest；内容在 OCI（或目录 canonical digest）。登记时重算 digest，不一致 → `asset.digest_mismatch`。

## 2. 种类表（每种必须回答「没有它就无法完成 X」）

| kind | 没有它则无法 | 谁写入 | 谁读取 | 热路径？ | MVP |
|---|---|---|---|---|---|
| `prompt` | 把可版本化的系统/角色/模板片段在多个 Agent 间复用，并约束变量完整性 | 发布者 | Harness 装配；Lock | 否（正文由 Harness 拉） | R1 |
| `skill` | 按 Agent Skills 标准做渐进披露的程序知识包 | 发布者 | Harness（先元数据后正文） | 元数据可在启动上下文；正文按需 | R1 |
| `mcp_server` | 把外部工具面钉在 `tools_snapshot` 上，检测供应链漂移 | 发布者 | EP 过滤；Harness MCP client | 连接在热路径；登记否 | R1 |
| `model` | 钉 Provider+模型身份+参数上界，并声明 `direct` / `via_ep` | 发布者 | Harness；计量换算提示 | 推理在热路径；资产否 | R1 |
| `memory_provider` | 多租户命名空间的记忆引擎接入与准入回报 | 运营者/发布者 | Runtime 绑定 | 检索可在热路径 | R3（R1 可无） |
| `knowledge` | 只读、内容寻址的资料包（与可变 Memory 分开） | 发布者 | Harness 检索 | 否 | 可选 |
| `harness` | 声明 ABI、控制点、原生工具、`loop_schema`、prompt 槽、skill 加载方式 | 发布者/运营者 | 资格化；`compose.match` | 进程在热路径；资产否 | R1 |
| `agent` | 把上述引用收成可发布的组合（Agent Definition） | 发布者 | resolve / Service | 否 | R1 |

**不是资产**

| 事物 | 形态 | 理由 |
|---|---|---|
| Loop | Profile：`{harness_ref, params}` 内联或 `profile:loop/<ns>/<name>@<ver>` | 无它则无法追溯步数/停止策略——所以要有 **digest 化配置**，不需要 OCI `kind` |
| 单个 tool | `mcp_server` 快照条目 / skill `allowed-tools` / harness 原生 | 第三套身份必漂 |
| Runtime 实例 | 运行平面对象 | 不可分发、不可被 Definition 引用 |
| 一次装配出的上下文 | Harness 会话账本 | 不可复用 |
| Memory 条目 / Artifact | 治理对象 | 按次 Execution 产生 |
| Service / Tenant / Plan | 服务或治理 | 见 [04](04-aaas-and-evolution.md) |

Capability 仍是**契约对象**（`behavior/quality/bounds/verification`），不是 `kind`。`implementation` 字段不存在；实现关系写在 Agent Definition `spec.capabilities[]`。

## 3. 统一 manifest 最小字段

```yaml
apiVersion: fysun.io/v1alpha2
kind: skill                          # 八值之一
metadata:
  name: acme/trend-research          # namespace/name
  version: 1.2.0                     # SemVer
  digest: sha256:…                   # 身份的一部分
  publisher: principal:acme
  source: { oci: … }                 # 或 repo+revision
  license: Apache-2.0
  signatures: []                     # Policy 可要求
spec: { }                            # 按 kind
declares:
  effects: []                        # 必填（可空）；缺字段 → invalid_manifest
  requires: { assets: [], control_points: [], environment: {} }
  provides: { tools: [], control_points: [] }
  compatibility: { harness: [], model: [] }   # 声明不是证据
# evidence[] 仅 fysund 追加
```

规则：无 secret 值槽；`declares.effects[]` 覆盖该资产引入的每个工具名；未覆盖者运行时按 `external_mutation` 并在 health 标未分类（继承 [ADR-0008](../adr/0008-effect-category-and-read-only-determination.md)）。

## 4. `spec` 要点（只列机器要检查的）

| kind | 必检 spec |
|---|---|
| `prompt` | `role`、`content_ref`、`variables[]`（可要求 `integrity`）、`slot`、`cache_hint`（`prefix` / `volatile`） |
| `skill` | `skill_dir`（含 `SKILL.md`）、`allowed-tools[]`；effects 必须覆盖 |
| `mcp_server` | transport、launch 或 url、`tools_snapshot`+digest、每工具 `default_effect_category`；凭据只 `SecretRef` |
| `model` | `provider`、`model_id`、`params_bounds`、`access: direct\|via_ep` |
| `memory_provider` | `scoping` 是否 per-tenant / per-activity；`admission_hook` |
| `knowledge` | `content_ref`、`integrity_default`、`freshness` |
| `harness` | `runtime_abi`、`control_points[]`、`native_effect_tools[]`（字段必在，可空数组）、`loop_schema`、`prompt_slots[]`、`skill_loading`（必须能只加载 name+description）、`tool_surface`、`install`/`bundle_digest` 规则；可选 `cache_alignment`、`session_isolation`、`session_park` |
| `agent` | 见 [03](03-agent-and-runtime.md) |

**身份格式。** `<kind>:<namespace>/<name>@<version>`，例如 `skill:acme/trend-research@1.2.0`。同一 `version` 不同 `digest` 是两个版本。Loop Profile id：`loop:<namespace>/<name>@<version>`（**不是** `kind` 枚举值，只是登记键）。

**Loop Profile 最小 params（跨 Harness 建议子集，Harness 可扩展）。**  
`max_steps` · `stop_conditions[]` · `retry{max, backoff}` · `delegation{enabled, max_concurrent, child_budget_ratio}` · `compaction{trigger}` · `human_in_loop{when}`（要输入，不是审批）。基座只按目标 Harness 的 `loop_schema` 校验，不解释语义。不匹配 → `asset.kind_mismatch`。

**为何不把 Prompt/Model 再降级为「仅内联」。** 人格包与模型目标要在多个 Agent 间复用并单独弃用；若只能内联，供应链撤销无法精确级联。允许内联的是 Loop 与少量 overrides，因为它们几乎总是 per-agent。

## 5. 依赖、锁定、digest、来源

- 引用用版本范围；Lock **只**存精确 version + digest。
- 分层 digest 定义见 [00](00-decisions.md) D7。`lock_digest` = 三层规范序列化后再哈希，可重放。
- `withdrawn`：不得进入**新** Lock；已有 Lock 与运行中 Execution 不杀。`revoked` / `quarantined`：新 Lock 拒绝；含它的 Service version 自动非 stable。
- 供应链：namespace/registry 允许列表；Policy 可要求签名（`asset.unsigned`）；MCP `tools/list` 与快照漂移 → EP 回报 → 该 server 工具升为 `external_mutation` 直至重分类。
- 打包：`fysun asset pack/push/register/publish`。Loop Profile：`fysun profile put loop …` 写登记表，不强制 pack。

## 6. 效果声明汇总

Agent Definition `bounds.tool_effects[]` = 组合内各资产 `declares.effects[]` ∪ Harness `native_effect_tools[]`，同名取 `max_severity`。Definition 只能剔除工具，不能降级类别。Policy 为来源 (c)。未声明工具在 EP fail-closed 为 `external_mutation`。

工具面收窄的声明来源只有：Definition 禁用、`mcp_servers[].tools_allow`、Grant、Policy。这是 F9 的静态部分，在 resolve 时算入 Lock。

## 7. 生命周期

```text
draft ─publish(Policy, 可选证据门)▶ published ─deprecate▶ deprecated ─withdraw▶ withdrawn
                                      └─ quarantine / revoke ▶ 级联 Lock 与 Service
```

版本不可变。Capability 契约不兼容 → 新 Capability id（谱系继承 v0.7.0）。发现接口返回摘要 + evidence + health；可见 ≠ 可用。

## 8. 负例

1. Agent Definition 引用无 digest 的资产。
2. 登记 `kind=tool|loop|runtime`。
3. `skill.allowed-tools` 未出现在 `declares.effects[]` 且 EP 未按 `external_mutation`。
4. Harness 缺 `native_effect_tools` 字段却 `runtime/qualified`。
5. Loop params 不符 `loop_schema` 却进入 Lock。
6. `revoked` digest 进入新 Lock，或其 Service version 仍 `stable`。
7. `fysund` 存资产内容本体。
8. 仅 profile 变更却丢掉旧 Lock 的 `inherited` 标记（应保留并降权，而非当新 Verification）。
