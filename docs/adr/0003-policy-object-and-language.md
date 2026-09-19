# ADR-0003 · Policy 对象与策略语言

- 状态：accepted（v0.7.0 修订）
- 日期：2026-09-07
- 决策者：owner
- 背景来源：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) §2.2、§3.2、§3.4、§5.1、§6.2、§7.3、§8.2
- supersedes：无（新增）。ADR-0001 §D7"部署策略声明、能力不自扩权"保留有效；本 ADR 补充其载体与语言。
- 修订：v0.7.0（2026-09-08）由 [ADR-0009](0009-cedar-subset-and-entity-model.md)（子集范围与实体模型）、[ADR-0008](0008-effect-category-and-read-only-determination.md)
  （`effect_category`）承接；见文末修订历史。

## 背景

ADR-0001 §D7 决定"不内置统一风险等级模型；部署环境声明什么动作可自动执行、何时需人工；OS 执行边界"。
v0.5.0 中这个声明的载体是本地文件：03 §7 的部署策略为"本地文件"，Capability API 为 `policy.load(path)`。

审查报告 §3.2 指出三个缺口：

- 策略无版本、无 digest；准入决定（`effect/authorized` 等）不引用策略版本；账本无法回答"这条 Effect 当时为什么
  被放行"，也无法在策略变更后区分"变更前的决定"与"变更后的决定"。
- F6 要求"契约与负例不为实现让路"，但策略本身没有进入可追溯范围；策略可以被静默改动而不留痕。
- 策略的表达形式未定义。AOS 6.7 要求策略决定含规则版本、结果、理由、obligations；Amazon Bedrock AgentCore
  Policy 与 agentgateway 均采用 Cedar 并在每次工具调用前评估。

报告 §3.7 进一步要求策略能引用来源标签（[ADR-0004](0004-provenance-labels-and-monotonic-privilege.md)），
§3.4 要求策略能为 Effect 类别设定最低 verifier 要求，§6.2 要求安全包络成为绑定设备 Resource 的 Policy 对象。
这些都需要 Policy 是一个有结构、可版本化、可静态分析的对象，而不是一个自由格式文件。

## 决策

### D1 · Policy 为版本化账本对象

- `Policy{id, version, digest, source, loaded_at, rules_ref, status}` 是 Authority 子系统的一级对象。
  `rules_ref` 指向内容寻址的 Artifact（策略正文）；`source` 记录来源（本地路径、Capability API 调用者、
  下发者 Principal）。
- 事件：`policy/loaded`（首次载入）、`policy/revised`（同 `id` 新 `version`）、`policy/retired`。
- `version` 只增不改；同一 `(id, version)` 的 `digest` 不可变。`status ∈ {active, retired}`。
- 同一时刻一个 Scope 内有效的 Policy 集合由投影给出；Activity 通过 `activity/policy-bound` 绑定具体版本。
- Capability API：`policy.load(source) -> {policy_id, version, digest}`、`policy.describe(policy_id?)`——无参返回当前 Scope 全部 `active`
  Policy 的列表 `[{policy_id, version, digest, rules_summary}]`，有参返回单个——、`policy.explain(event_ref) -> policy_decision`。
  （v0.7.0 修订：v0.6.0 的 `policy.describe() -> {version, digest, rules_summary}` 只能描述一个 Policy，与"一个 Scope 内有效的 Policy 集合"不匹配。）
- 负例：同一 `(id, version)` 出现两个不同 `digest` → 失败；一次 `policy.load` 后账本无 `policy/loaded` 或
  `policy/revised` 事件 → 失败。

### D2 · 每个准入类事件携带 `policy_decision`

- 准入类事件（`effect/authorized`、`action-intent/rejected`、`approval/asked`、`grant/issued`、
  `memory/admitted`、`memory/rejected`、`runtime/qualified`）必须携带
  `policy_decision{policy_version, rule_id, outcome, obligations[]}`。`audit/observed`（`local_read` 快路径审计事件）与
  `approval/decided` **不是**准入类事件，不携带 `policy_decision`。
- `outcome ∈ {allow, deny, modify, approve}`，取 AOS 的四值：`approve` = 需要 Approval，`fysund` 据此创建
  Approval 对象并向 Runtime 返回 `needs_human(approval_id)`；`modify` 对应 MCP 守卫的 `Mutated`，改写后的
  参数记入 ActionIntent。**`outcome` 四值是 Policy 的唯一输出**；`proceed / proceed_with_warning / retry / fallback / escalate / terminate`
  六个控制动作是 Activity 子系统的内部策略词汇，不由 Policy 产出（`escalate → approve`、`terminate → deny + cancel` 为其映射）。
- 四值在 Cedar 中的表达约定（注解 `@fysun_outcome`）见 [ADR-0009](0009-cedar-subset-and-entity-model.md) §D3。
- `obligations[]` 是准入后必须履行的附加义务（例如"结果须经 `human` verifier"、"打上 `verified_data`
  之前不得进入 Memory"），由 `fysund` 记录并在后续事件中核对。
- `policy_version` 必须能在账本中解析到一个 `policy/loaded` 或 `policy/revised` 事件。
- 负例：任一准入类事件缺 `policy_decision` → 失败；`policy_decision.policy_version` 指向账本中不存在的版本
  → 失败；`outcome = approve` 却无对应 `approval/asked` → 失败；`policy_decision.outcome` 出现四值以外的取值 → 失败。

### D3 · 策略语言采用 Cedar 子集（MVP），不自造

- Policy 正文使用 Cedar 策略语言的一个子集。FysunOS 不自造策略 DSL。
- 选 Cedar 而非 CEL 的理由：
  - Cedar 是授权语言而非通用表达式语言，策略可静态分析——能回答"这套策略是否允许某类动作"，与 F7
    "能力不得自扩权"及 F9（[ADR-0004](0004-provenance-labels-and-monotonic-privilege.md)）的可判定表述一致；
  - 有形式化验证与自动推理工具，Rust 实现可用，与 `fysund` 技术栈一致（ADR-0001 §D4）；
  - AgentCore Policy 与 agentgateway 已采用，是有工业先例的授权语言；MCP 守卫自实现（[ADR-0012](0012-mcp-guard-self-implementation.md)）
    后借用其策略格式，便于对照测试，不构成复用承诺。
- 子集范围（允许的 Cedar 语法与实体类型）见 [ADR-0009](0009-cedar-subset-and-entity-model.md)（旧 08 开放问题 7 已关闭）；
  确定原则不变：只收录 D4 的评估输入能表达的部分，不收录需要外部数据查询的扩展。
- 负例：仓库中出现 FysunOS 自定义的策略语法定义 → 失败；Policy 正文无法被 Cedar 解析器接受 → `policy.load` 拒绝。

### D4 · 策略评估输入

策略评估至少包含以下四组输入：

| 组 | 内容 |
|---|---|
| principal | 请求的 Principal（Execution 是可被授权主体）及其 Grant 链 |
| action | `effect_category`（六值，[ADR-0008](0008-effect-category-and-read-only-determination.md)）+ 工具名（MCP `tools/call` 的工具标识） |
| resource | Scope、设备 Resource、账号 / 外部主体标识、下游 MCP server |
| context | 参数摘要、**来源标签**（参数值的 `label.integrity`、`untrusted_params`）、结果 `flags`（post 阶段）、预算余量、可逆性、Runtime 类型与 `control_points[]`、**`native_effect_tools[]`**、`assurance` 级别（若有）、**`offline`**（边缘节点是否离线）、未决 Approval 数 |

- 来源标签进入 context 是 [ADR-0004](0004-provenance-labels-and-monotonic-privilege.md) §D3 的前提。
- 实体类型与 context 固定键的精确形状见 [ADR-0009](0009-cedar-subset-and-entity-model.md) §D1；评估输入的构造进入 06（守卫从 `tools/call`
  构造）与 05（控制点适配器经 `os.submit_candidate` 构造）；Runtime 不能提供或改写 context。
- 负例：一条策略引用了评估输入之外的字段 → `policy.load` 拒绝，而不是运行时静默为空。

### D5 · Policy 可表达的规则类别

Policy 至少能表达以下规则，这些规则在 v0.5.0 中散落在注释与设计叙述里，v0.6.0 起必须由 Policy 对象承载：

| 规则类别 | 示例 | 对应文档 |
|---|---|---|
| 最低 verifier 要求 | 发布类 Effect 至少需要 `deterministic` + `human` 两类 Verification | 02 §2.7 |
| 来源标签规则 | 外部发送类 Effect 的收件人参数含 `untrusted_external` → `approve` | ADR-0004 |
| 审批容量与限流 | 与 `Activity.constraints.max_pending_approvals`、`approval_sla`、`approval_rate_limit`、`consecutive_rejections_alert` 联动的超限动作 | ADR-0005、ADR-0011 |
| Effect 类别覆盖 | 对某工具 / 某下游 server 的 `effect_category` 覆盖（唯一能降低类别的来源） | ADR-0008 |
| Runtime 资格化 | 含特定 `native_effect_tools` 的 Runtime 是否允许资格化 | ADR-0007 |
| 恢复后自动恢复 | `auto_resume`：含非 `local_read` 类别的 Activity 重启后是否免人工重新武装 | ADR-0011 §D8 |
| 离线保守规则 | `fysund-edge` 离线期间只允许本地 Policy 已预授权的 `effect_category`；需要 Approval 的一律阻塞；escrow 预算 | 11 §3、ADR-0010 |
| 安全包络 | 绑定设备 Resource 的 Policy 子类：`velocity_bounds`、`geofence`、`interface_allowlist`、`rate_limit`、`deadman`、`emergency_stop_ref`；Effect 记录 `envelope_ref` 与 `envelope_check` | 11 |
| `assurance` → `outcome` | 按 `effect_category` × `assurance.level` 映射到 `outcome` 四值；`proceed / retry / fallback / escalate / terminate` 等六控制动作为 Activity 子系统内部词汇，不是 Policy 输出 | 02 |

- 安全包络作为 Policy 子类意味着同一能力在不同设备上适用不同包络；包络声明的校准是部署者责任，"通过校验 ≠
  世界安全"。
- 负例：安全包络以 Effect 字段而非 Policy 引用形式出现 → 失败。

### D6 · F6 范围扩展到 Policy

- F6"契约与负例不为实现让路"的范围扩展为"契约、负例与 Policy 不为实现让路"：实现无法满足某条 Policy 时，
  修改的是实现或显式发布新 Policy 版本，不得在实现中绕过或弱化评估。
- 恢复流程（03 §6）在 `resume_execution` 前校验绑定的 Policy 版本仍存在且未 `retired`；已 `retired` 则
  Activity 进入 `blocked`，原因为 Policy 失效。
- 负例：一条被拒绝的 candidate 在不发布新 Policy 版本的情况下于重试后被放行 → 失败。

## 否决的替代方案

- **保留本地文件、不入账本。**无法回答"当时为何放行"，策略改动不留痕，F6 覆盖不到策略；与 AOS 6.7 的
  决策记录要求相悖。
- **自定义 DSL。**与 P8（不重复做已有的事）冲突；无现成静态分析工具。
- **CEL。**表达式语言而非授权语言，能写任意布尔表达式但难以静态回答"是否存在被允许的扩权动作"。
- **OPA / Rego。**生态偏云基础设施与 Kubernetes 准入；体积与部署形态不适合 owner-local 与 `fysund-edge`。

## 后果

- M1 验收新增：每个准入类事件携带 `policy_decision`（含 `policy_version`）；`policy.explain(event_ref)` 能
  从账本重建决定。
- 恢复时增加 Policy 版本校验步骤；Policy `retired` 成为 Activity `blocked` 的一种原因。
- Cedar 子集解析、schema 校验与评估器进入 `fysund` 依赖（Rust 实现可用）；`fysund-edge` 同样需要，用于离线保守规则与
  安全包络校验。
- Cedar 子集范围由 ADR-0009 落定（旧 08 开放问题 7 关闭）；许可证开放问题需注明 Cedar 为 Apache-2.0。
- 部署者获得可静态分析的策略：能在部署前回答"这套策略是否允许某类动作"，而不是在运行时才发现。
- 每次准入评估增加 Cedar 评估开销；策略规模在 owner-local 场景下很小，不作为性能风险处理。

## 待后续 ADR 的问题

- 多主体 / 多节点下 Policy 的分发与版本一致性（边缘离线一致性已由 [ADR-0010](0010-edge-offline-ledger-consistency.md) 落定，
  多主体部分继续开放）。
- `obligations[]` 的枚举与核对规则；`modify` 改写表达式的通用形式（ADR-0009 待后续）。

其余见 [12 MVP 计划 §7](../design/12-mvp-plan.md)。

## 修订历史

- v0.7.0（2026-09-08）：§D1 `policy.describe()` → `policy.describe(policy_id?)`（无参返回 Scope 内全部 `active` Policy 列表）；§D2 准入类事件
  清单更新（`approval/asked` 替代 v0.6.0 的 Approval 请求事件名；增 `runtime/qualified`；注明 `audit/observed` 与 `approval/decided` 非准入类），
  增"`outcome` 四值为唯一 Policy 输出、六控制动作为 Activity 内部词汇"，负例同步；§D3 子集范围改为"见 ADR-0009"，Cedar 选择理由去
  "采用 agentgateway 时策略格式无需转换"；§D4 `action` = `effect_category` + 工具名，`context` 增 `native_effect_tools[]`、`offline`、结果 `flags`、
  未决 Approval 数，评估输入形状引用 ADR-0009；§D5 表增 Effect 类别覆盖、Runtime 资格化、`auto_resume`、审批限流四行，"`assurance` → 控制动作"
  行改为"`assurance` → `outcome` 四值"，离线与包络行引用改指 11；否决段去 agentgateway 格式转换与复用表述；后果段与待后续段删 Cedar 子集项、
  边缘项，链接改指 12。为什么：蓝图 §0 C1-4（四值与六控制动作并存）、C1-8（`policy.describe` 与 Policy 集合不匹配）、G4（Effect 类别无定义）；
  §I.1 决策 7、8、12。对应蓝图：§C.1 ADR-0003 行。
