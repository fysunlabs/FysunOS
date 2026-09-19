# ADR-0009 · Cedar 子集范围与实体模型

- 状态：accepted
- 日期：2026-09-08
- 决策者：owner
- 背景来源：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md) §C.2 ADR-0009、§F §7（关闭旧 08 开放问题 7）；[ADR-0003](0003-policy-object-and-language.md) §D3/§D4；Cedar 策略语言与 Cedar schema（JSON）规范
- supersedes：无（新增）。落实 ADR-0003 §D3 "子集范围待定"与 §D4 评估输入形状；ADR-0003 正文已于 v0.7.0 同步修订。关闭旧 08 开放问题 7。

## 背景

ADR-0003 选定 Cedar 子集为策略语言，但把"子集范围（允许的 Cedar 语法与实体类型）"留为旧 08 开放问题 7（现 12 §7 Q7），原则是"MVP 只收录 D4 的评估输入
能表达的部分，不收录需要外部数据查询的扩展"。v0.6.0 之后有三件事使范围必须现在定下：

- [ADR-0008](0008-effect-category-and-read-only-determination.md) 定义了六值 `effect_category`，Policy "哪些类别可自动执行"需要 action 分组；
- [ADR-0007](0007-runtime-native-tools-and-conformance-scope.md) 要求 Policy 能据 `native_effect_tools[]` 拒绝资格化，
  [ADR-0010](0010-edge-offline-ledger-consistency.md) 要求 Policy 能表达离线预授权，二者都是新的 context 键；
- `policy_decision.outcome` 是四值（`allow / deny / modify / approve`），而 Cedar 原生只产 allow / deny；不定义映射约定，`approve` 与 `modify`
  就无法由 Cedar 策略产出，只能在实现里硬编码——违背 ADR-0003 "不自造策略语言"的本意。

[ADR-0012](0012-mcp-guard-self-implementation.md) 关闭了 agentgateway 复用评估，与 CEL 的互操作项随之失去必要性。

## 决策

### D1 · 实体模型

Cedar 评估的四组输入（ADR-0003 §D4）落为以下实体类型与 context 固定键：

| 组 | Cedar 实体 | 说明 |
|---|---|---|
| principal | `User`、`Execution`、`Runtime`、`ExternalAgent` | `Execution` 是可被授权主体（02 §2.4）；`in` 关系表达 Grant 链（子 Execution `in` 父 Execution）；`Runtime` 用于资格化评估；`ExternalAgent` 用于 A2A 侧 |
| action | `Action::"<server>/<tool>"`；父组 `Action::"category:<effect_category>"`（六个）与 `Action::"capability:<capability_id>"` | 工具级 action `in` 其类别组与所属 Capability 组；守卫系统工具为 `Action::"fysun/await_approval"` 等，`in Action::"category:local_read"`；资格化评估的 action 为 `Action::"runtime/qualify"` |
| resource | `Scope`、`Device`、`Account`、`McpServer` | `Device` 承载安全包络 Policy 绑定；`McpServer` 承载下游 server 级规则（如整 server 禁用） |
| context | 固定键（下表） | 由 `fysund` 在每次评估时构造；Runtime 不能提供或改写 |

context 固定键：

| 键 | 类型 | 来源 |
|---|---|---|
| `effect_category` | `String`（六值之一） | ADR-0008 合成结果 |
| `params_digest` | `String` | canonical 参数 digest |
| `min_integrity` | `String` | 全部参数值 `label.integrity` 中最低者 |
| `untrusted_params` | `Set<String>` | `integrity = untrusted_external` 的参数名集合 |
| `result_flags` | `Set<String>` | post 阶段结果分类：`authority_claim`、`secret_shaped`（ADR-0004 §D1 修订）；pre 阶段为空集 |
| `budget_remaining` | `Long` | 当前 Grant 剩余预算（主维度） |
| `reversibility` | `String` | 可逆性声明（工具目录或物理 `reversibility_class`） |
| `runtime_type` | `String` | `describe().type` |
| `control_points` | `Set<String>` | `runtime/qualified` 中的 `control_points[]` |
| `native_effect_tools` | `Set<String>` | `runtime/qualified` 或 `describe()` 中的 `native_effect_tools[]` 工具名 |
| `assurance_level` | `String` | `assurance.level`（若有；缺省 `unknown`） |
| `offline` | `Bool` | 评估节点当前是否离线（ADR-0010） |
| `pending_approvals` | `Long` | 所属 Activity 当前未决 Approval 数 |

- 未列出的键不得出现在策略中；策略引用未知键 → `policy.load` 拒绝（D4）。
- 新增 context 键需修订本 ADR 并同步 Cedar schema。
- 负例：策略引用 `context.tool_name`（不在表中）被 `policy.load` 接受 → 失败；Runtime 在 `os.submit_candidate` 中携带的任何 context 覆盖被采纳
  → 失败。

### D2 · 语法子集

允许：

- 策略效果 `permit` / `forbid`；
- scope 约束：`principal == E`、`principal in E`、`principal is T`、`principal is T in E`；`action == A`、`action in A`、`action in [A, …]`；
  `resource == E`、`resource in E`、`resource is T`、`resource is T in E`；
- `when` / `unless` 条件子句；
- 运算符：`== != < <= > >= && || !`、`in`、`has`、`is`、`like`（通配符字符串匹配）、`.contains()`、`.containsAll()`、`.containsAny()`；
  `if … then … else`；
- 字面量：`Bool`、`Long`、`String`、`Set`、`Record`、实体 UID；
- 策略注解（`@key("value")`，用于 D3）。

不收录：

- 策略模板（`?principal` / `?resource` 槽位与 link）——版本化 Policy 对象要求正文自足，模板实例化会产生账本外状态；
- 扩展类型 `ip` / `decimal` / `datetime` 及其方法——MVP 评估输入中没有需要它们的键；到期与时间窗由 `fysund` 在 Cedar 之外按 Grant / Approval
  字段处理；
- 实体 tags（Cedar 4.x 的 `.hasTag()` / `.getTag()`）——实体属性已足够，tags 会增加 schema 自由度；
- 任何需要在评估时查询外部数据的扩展。

负例：仓库中出现 FysunOS 自定义的策略语法定义 → 失败（延续 ADR-0003 §D3）；使用模板或扩展类型的策略被 `policy.load` 接受 → 失败。

### D3 · `outcome` 四值的表达约定

Cedar 评估只产 allow / deny。`policy_decision.outcome` 的四值由**策略注解**承载：

| 注解 | 含义 |
|---|---|
| `@fysun_outcome("approve")` | 该 `permit` 匹配时结果为 `approve`（需要 Approval） |
| `@fysun_outcome("modify")` | 该 `permit` 匹配时结果为 `modify`，改写内容由 `@fysun_obligation` 给出 |
| `@fysun_obligation("<obligation>")` | 准入后义务（可多条）；`modify` 的改写表达也在此，本轮只支持枚举形式（见待后续） |
| `@fysun_rule_id("<id>")` | 写入 `policy_decision.rule_id`；缺省为 Cedar policy id |

无 `@fysun_outcome` 注解的 `permit` 表示 `allow`。`forbid` 不接受 `@fysun_outcome`。

评估规则（固定，`fysund` 实现）：

1. 任一 `forbid` 匹配 → `outcome = deny`；
2. 否则，匹配的 `permit` 中任一带 `@fysun_outcome("approve")` → `approve`；
3. 否则，任一带 `@fysun_outcome("modify")` → `modify`（多条 `modify` 的 obligations 合并；冲突改写 → `deny` 并记原因）；
4. 否则，存在匹配的 `permit` → `allow`；
5. 无任何匹配 → `deny`（Cedar 缺省拒绝，与 fail-closed 一致）。

`policy_decision.rule_id` 记录决定性的那条策略（`deny` 时为匹配的 `forbid`；`approve` / `modify` 时为带该注解的 `permit`；`allow` 时为任一匹配的
`permit`，多条时按 policy id 字典序取首条并在 `obligations[]` 中列出其余）。

- 负例：两条 `permit` 同时匹配、一条带 `approve` 一条无注解，结果为 `allow` → 失败；`forbid` 带 `@fysun_outcome` 被 `policy.load` 接受 → 失败。

### D4 · Cedar schema 校验

- `fysund` 维护一份 Cedar schema（JSON 格式），声明 D1 的实体类型、action 分组与 context 键类型；schema 随 `fysund` 版本发布。
- `policy.load(source)` 在解析后以 schema 做校验：引用未知实体类型、未知属性、类型不匹配、使用 D2 不收录的语法 → 拒绝，返回定位信息，
  不产生 `policy/loaded`。
- schema 变更（新增 context 键、新增 action 组）走 `fysund` 版本发布，并要求既有 `active` Policy 重新校验；校验不通过的 Policy 在恢复时按
  ADR-0003 §D6 处理（Activity `blocked`，原因为 Policy 失效）。
- 负例：`policy.load` 接受了引用未知属性的策略、运行时该属性静默为空 → 失败（延续 ADR-0003 §D4 负例）。

### D5 · 与 ADR-0008 的衔接

- `Action::"category:<effect_category>"` 六个父组由 `fysund` 按 ADR-0008 合成结果为每个工具级 action 建立 `in` 关系；Policy 可只对类别组写规则
  （"`external_mutation` 一律 `approve`"）而无需枚举工具。
- `Action::"capability:<id>"` 父组由 Capability 的 `allowed-tools` ∩ `tool_effects[]` 建立，使 Policy 能表达"某 Capability 下的工具"。
- 资格化评估（[ADR-0007](0007-runtime-native-tools-and-conformance-scope.md) §D2）为 `principal is Runtime`、`action == Action::"runtime/qualify"`、
  `resource is Scope`，context 含 `native_effect_tools` 与 `control_points`。

## 否决的替代方案

- **完整 Cedar（含模板、扩展类型、tags）。**模板实例化产生账本外状态，与 Policy 版本化冲突；扩展类型与 tags 在当前评估输入下无需求，
  收录只会扩大 schema 与测试面。
- **以自定义 JSON 规则表达 `approve` / `modify`，Cedar 只管 allow / deny。**等于两套策略语言并存，`policy.explain` 无法用单一 `rule_id` 解释
  决定；与 ADR-0003 "不自造"冲突。
- **`approve` 用 `forbid` + 特殊注解表达。**Cedar 语义中 `forbid` 恒优先且不可被 `permit` 覆盖；把 `approve` 挂在 `forbid` 上会让"需要审批"
  变成不可能被更具体的 `permit` 放行的硬拒绝，与 Approval 的语义（放行路径）相反。
- **保留 CEL 互操作项。**agentgateway 复用已关闭（ADR-0012），CEL 互操作不再有需求方。

## 后果

- 旧 08 开放问题 7 关闭；剩余两项（`modify` 改写表达式形式、`obligations[]` 枚举全集）转为新的开放问题（12 §7）。
- `policy.load` 增 schema 校验步骤。
- Policy 正文的可静态分析性得到保证：能回答"这套策略在哪些 `effect_category` 下会产出 `approve`"、"是否存在对 `external_mutation` 无条件
  `allow` 的 `permit`"。
- 03 §7 Policy 模型的问题清单每一条都能落到 D1 的键；`fysund-edge` 使用同一 schema 的子集（`offline` 键为真）。
- 实现依赖：Cedar Rust crate（解析、schema 校验、评估）；策略注解读取由 `fysund` 自实现（Cedar 解析器保留注解）。

## 待后续 ADR 的问题

- `modify` 的改写表达式：本轮只允许枚举形式 `redact:<param>`、`clamp:<param>:<max>`（写在 `@fysun_obligation` 字符串中）；
  通用改写表达式待有需求再定。
- `obligations[]` 的枚举全集与核对规则（承 ADR-0003 待后续）。
- 多主体 / 多节点下 Policy 分发与版本一致性（承 ADR-0003，与 ADR-0010 §D7 一并开放）。

其余见 [12 MVP 计划 §7](../design/12-mvp-plan.md)。
