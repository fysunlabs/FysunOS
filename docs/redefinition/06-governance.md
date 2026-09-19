# 06 · 治理（横切）

- 状态：提案草案 v0.8.0-draft（2026-09-11）
- 决策依据：[ADR-0013 草案](adr-0013-draft-repositioning.md) §D5、§D6、§D8
- 继承：v0.7.0 [ADR-0003](../adr/0003-policy-object-and-language.md)（Policy）、[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)（来源标签与 F9）、[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md)（Effect 类别）、[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md)（Cedar 子集）、[ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md)（Approval）、[13 威胁模型](../design/13-threat-model.md)

治理对象基本继承 v0.7.0；本文只写**变化**与在新平面中的落点。未提及的规则以现行文档为准。

## 1. Principal / Scope / Tenant

- Principal 类型：`User` / `Tenant` / `Execution` / `Runtime` / `EnforcementPoint` / `ExternalAgent` / `Publisher`（User 或组织的发布身份）。
- Scope 带 tenant 前缀；系统级对象（Harness 资产、Policy 模板）在 `system` scope；跨 scope 可见需要显式 Grant 或 `tenant_visibility = public`。
- Approval 决定者必须是人或其授权系统，不能是 Runtime、EP 或外部 Agent（继承）。

## 2. Grant

- 继承字段与三条衰减不等式；新增 `root_of: plan_ref?`（由 Plan 派生的根 Grant）与 `ep_scope?`（限定只能由哪些 EP 使用该 Grant）。
- 有效权限集 = Grant ∩ Policy 允许集 ∩ 工具面（Lock 派生）；F9 收窄自动、扩张只经 `approved_once`（继承）。
- EP 从 `ep.policy_pull` 得到 Grant 有效集投影；EP 不能签发。

## 3. Policy

- 对象、版本化、Cedar 子集、`outcome` 四值、注解表达约定、schema 校验继承。
- **实体模型扩展**（ADR-0009 §D1 增补）：
  - principal 增 `Tenant`、`EnforcementPoint`、`Publisher`；
  - resource 增 `Asset`（属性 `kind`、`namespace`、`signed`、`publisher`）、`Service`、`Endpoint`、`Plan`；
  - action 增 `Asset::"publish"`、`Service::"publish"`、`Service::"invoke"`、`EP::"qualify"`、`Runtime::"qualify"`、`Plan::"attach"`；
  - context 增 `tenant_id`、`plan_ref`、`ep_kind`、`ep_id`、`lock_digest`、`asset_signed: Bool`、`asset_namespace`、`native_effect_tools`（继承）、`offline`（继承）。
- **决策位置**：`fysund` 评估资产发布、Service 发布、Runtime / EP 资格化、Grant 签发、Approval 创建、Memory 准入；EP 本地评估 Effect 候选与 `tools/list` 过滤。
  同一 Policy 文本、同一 `policy_version`，两处评估结果必须一致（负例：同输入不同 outcome）。
- Policy 至少回答的问题在 v0.7.0 03 §7 基础上增：允许哪些 namespace / registry 的资产、是否要求签名、哪些 `kind` 的资产可由租户自带、
  Runtime 池化策略（`dedicated` / `shared`）、`policy_stale_ttl`、审批路由、`canary` 上限、数据驻留约束。

## 4. Approval

- 结果集 `approved_once / rejected / cancelled / unavailable`、配对事件、不复制参数、审批面只由 `fysund` 提供、重新武装——全部继承。
- `subject` 增 `asset_publish`、`service_publish`、`plan_attach`（当 Policy 对这些准入产出 `approve`）。
- 审批路由（05 §10）由 Plan / Policy 决定；`decided_by` 必须在路由允许集合内，否则 `unavailable`。
- EP 与 gateway 只透传 `needs_human(approval_id)`；Runtime 以 `os.observe` 查询结果。

## 5. 来源与完整性标签

- `label{source_principal, integrity, origin_ref, flags?}` 与四级 `integrity`、工具结果一律 `untrusted_external`、唯一升级路径（Verification / Memory 准入）、`flags` 分类——继承。
- 赋予责任：EP（工具结果、外部 Artifact）、Harness 控制点（context_view 片段）、`fysund`（Lock、Grant 摘要、Activity.continuity 等结构部分为 `trusted_system`；
  `knowledge` 资产按 `integrity_default`）。
- 资产层：`prompt.variables[]` 可要求 `integrity >= trusted_user`，Harness 装配时不满足 → 由 EP 或控制点回报 `context/assembled{violations[]}`，Policy 决定 `deny` / `approve`。

## 6. Effect 协议与 Verification

- ActionIntent → Effect → Receipt → Verification 状态机、幂等键、`conflict.idempotency`、补偿新 Grant、outcome-unknown 对账——继承，但**执行位置在 EP**（04 §4.2）。
- Verification 继承（`verifier_kind`、`independence`、`calibration_ref`、`inconclusive ≠ pass`）；证据归属见 03 §6。
- 最低 verifier 要求可按 Service / Plan 设定（例如对外发布类动作至少 `deterministic` + `human`）。

## 7. 资产供应链治理

| 威胁 | 机制 |
|---|---|
| 恶意 / 篡改资产 | digest 重算；签名要求（Policy）；namespace / registry 允许列表；`asset/quarantined` / `asset/revoked` 级联到 Lock 与 ServiceVersion |
| MCP server 工具语义漂移 | `tools_snapshot` 与实际 `tools/list` 比对（EP 回报）→ 待重新分类；漂移期间该 server 的工具按 `external_mutation` |
| Harness 声明不实 | `describe()` 与 manifest 互证；conformance 只对经 EP 的面宣称；声明不实不在承诺范围 |
| Loop 参数越界 | `loop.params ⊨ harness.loop_schema`；预算仍由 Grant 兜底 |
| 提示词注入经 `knowledge` / `prompt` 资产 | 资产 `integrity_default`；`prompt.variables` 来源约束；`flags` 分类 |
| 租户自带资产越权 | BYO 只能覆盖允许槽位；效果类别不可降级；Policy 评估 `asset_namespace` |

## 8. 威胁向量增补（在 T1–T11 基础上）

- **T12 资产供应链**：吸收原 T2 实现包部分并扩展到全部 `kind`；负例：未签名资产在 Policy 要求签名时被发布；`revoked` 资产仍在 Lock。
- **T13 执行点冒充 / 失格回报**：未资格化 EP 回报被接受；`policy_version` 超窗回报被接受；EP 本地决定 Approval。
- **T14 跨租户泄露**：Memory / Artifact / Secret / Activity 跨 Scope 可见；`shared` 池化下会话串扰。
- **T15 服务面滥用**：配额绕过、`canary` 不可审计、Agent Card 与 Lock 不一致、未发布 Lock 被调用。

## 9. 可观测性

- OTLP 导出继承；trace 关联键增 `tenant_id`、`service_id`、`invocation_id`；`fysun.*` 命名空间继承，与 `gen_ai.*` 关系仍为开放问题。
- 度量增：每 Service 调用成功率与时延、每租户用量、EP 回报延迟与拒绝率、`policy_stale` 次数、资产隔离次数。

## 10. 负例（R1 / R4）

1. 同一 Policy 版本、同一输入，`fysund` 与 EP 评估 `outcome` 不同 → 失败。
2. `decided_by` 不在审批路由允许集合却产生 `approved_once` → 失败。
3. `prompt.variables` 要求 `trusted_user` 的槽位被 `untrusted_external` 填充且无 `context/assembled{violations[]}` → 失败。
4. Policy 要求签名时未签名资产 `asset/published` → 失败。
5. 跨租户 Memory 读取成功 → 失败。
