# ADR-0008 · Effect 类别分类与只读判定

- 状态：accepted
- 日期：2026-09-08
- 决策者：owner
- 背景来源：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md) §I.1 决策 8，§I.2 H-2，§0 G4/G7；CognitiveOS `core/crates/cognitive-kernel/src/effects.rs`（幂等冲突与补偿语义）；MCP 2025-11-25 规范 tool annotations（`readOnlyHint` 等）
- supersedes：无（新增）。ADR-0002 §D2 "只读工具走审计快路径"与 ADR-0003 §D4 "action = Effect 类别 + 工具名"由本 ADR 给出可判定定义；两份 ADR 正文已于 v0.7.0 同步修订。

## 背景

v0.6.0 文档在至少八处引用"Effect 类别"（以下为 v0.6.0 旧编号：02 §2.6 `action`、02 §2.12 评估输入、03 §7 Policy 问题清单、旧 04 §2.1 只读判定、ADR-0003 §D4/§D5、
ADR-0005 §D6、旧 07 §2.4 离线规则），却没有任何一处定义它的取值与来源（蓝图 §0 G4）。后果是：

- Policy "哪些 Effect 类别可自动执行、哪些需 Approval"无法书写——Cedar 策略需要一个封闭的 action 分组；
- 旧 04 §2.1（现 06）"只读工具走审计快路径，是否只读由 Policy 判定"把每个工具的只读性判定推给部署者逐条书写，MVP 不可行；
- 审计快路径"仍产生审计事件"（02 §2.6、旧 04 §2.1）但事件族无对应条目（G7），快路径实际上不留痕；
- 03 §7 / 旧 07 §2.4（现 11 §2.4）的"离线期间只允许已预授权的 Effect 类别"没有可预授权的枚举。

MCP tool annotations（`readOnlyHint`、`destructiveHint`、`idempotentHint`、`openWorldHint`）是 server 自报的提示，MCP 规范明确它们
不可信、client 不得据此做安全决定。owner 决策（蓝图 §I.1 决策 8）：类别来源三源合一——Capability 契约必填、Resource 目录登记缺省、Policy 可
覆盖；未分类一律按最高常见严重度 `external_mutation` 处理（fail-closed）；annotations 只作输入。H-2 进一步决定 `external_read` 不走快路径。

本 ADR 同时收编两条 Effect 错误语义（借鉴 CognitiveOS `effects.rs`）：同幂等键不同参数的硬拒绝，与补偿必须使用新 Grant。它们在 v0.6.0 中
缺失，属于同一"Effect 协议可判定化"范围。

## 决策

### D1 · 六类别与严重度序

`effect_category` 是封闭枚举，按严重度升序：

| 类别 | 含义 | 例子 |
|---|---|---|
| `local_read` | 读取本机、本 Scope 内、无副作用、结果不出本机的观察 | 读文件、列目录、查询本地索引、读取账本投影 |
| `external_read` | 读取外部系统，可能留下访问痕迹或消耗外部配额，但不改变外部状态 | HTTP GET、搜索 API、读取远端邮箱 |
| `local_mutation` | 改变本机 / 本 Scope 内状态，可由本机手段回退 | 写文件、修改本地配置、本地 git commit |
| `external_mutation` | 改变外部系统状态或向外部发送内容 | 发帖、发邮件、写远端数据库、git push |
| `credential_use` | 使用、导出、轮换凭据，或以凭据换取更高权限 | OAuth 授权、`secret.import`、调用需要额外授权的接口 |
| `physical_action` | 驱动执行器产生物理世界效果 | 阀门、电机、锁具 |

- 严重度序 `local_read < external_read < local_mutation < external_mutation < credential_use < physical_action` 用于 D2 的合成规则与 Policy
  的比较运算（Cedar 中以父组 `Action::"category:<effect_category>"` 表达，[ADR-0009](0009-cedar-subset-and-entity-model.md) §D1）。
- 序仅表达"更保守"的方向，不表达风险量化；不引入分数。
- 一个工具只有一个类别。同一工具在不同参数下严重度不同（如 `git` 既能 `status` 也能 `push`）时，按其**最严重**用法分类，或在 Capability 契约中
  拆为多个工具名（如 MCP server 暴露 `git_status` 与 `git_push`）。

### D2 · 三源合一与 `max_severity` 合成

类别有三个来源，合成规则固定：

| 来源 | 谁写 | 约束 |
|---|---|---|
| (a) Capability `contract.bounds.tool_effects[]: [{tool, effect_category}]` | Capability 作者（[02 §2.1](../design/02-object-model.md)） | **必填**；必须覆盖 `allowed-tools` 中每个工具；`capability.register` 缺整个字段 → 拒绝 |
| (b) Resource 目录登记的缺省类别 | 部署者登记 MCP server / 工具时（02 §2.9） | 可选；MCP tool annotations 只作为登记时的**建议输入** |
| (c) Policy 覆盖 | 部署者 | 可升可降；是**唯一**能降低类别的来源 |

- 合成：`effective = (c) if present else max_severity((a), (b))`；(a)(b) 中任一缺失取另一方；**(a)(b) 均缺 → `external_mutation`**。
- (a) 中未列出但出现在 `allowed-tools` 的工具：按 `external_mutation` 处理，且 `capability.health(id)` 标为"未分类工具"信号；这是 Capability
  契约缺陷，不是运行时错误。
- Capability 单方不能降低 (b) 已登记的类别（只能更保守），防止实现包（T2 供应链）把 `external_mutation` 工具声明成 `local_read` 走快路径。
- 下游 MCP server 发出 `notifications/tools/list_changed` 后守卫重新拉取 `tools/list` 并**重新合成**全部类别；工具语义变更后未重分类是 T2 负例。
- 负例：`readOnlyHint = true` 且 (a)(b) 均缺的工具被按 `local_read` 评估 → 失败；Capability 把目录登记为 `external_mutation` 的工具声明为
  `local_read` 后该工具走了快路径 → 失败。

### D3 · 未分类 → `external_mutation`（fail-closed）

- D2 的兜底不取最严重的 `physical_action`，因为后者要求包络 Policy 与物理扩展字段，对普通工具会造成 `not_runnable`；也不取更宽松的类别，
  因为那会让未分类工具绕过完整协议。`external_mutation` 是"走完整协议、缺省需要 Policy 明示才能自动执行"的最低保守点。
- 负例：任何未分类工具走审计快路径 → 失败（M1）。

### D4 · MCP tool annotations 仅作输入

- `readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint` 只在 (b) 登记时作为建议值呈现给部署者，不进入 D2 合成。
- 守卫在 `tools/list` 改写时可**保留**下游 annotations 原样透传给 harness（供模型参考），但守卫自身不读取它们做任何决定。
- 负例：守卫代码路径中出现以 annotations 为依据的分支 → 失败。

### D5 · 只有 `local_read` 走审计快路径；`external_read` 走完整协议

- 审计快路径（v0.6.0 称 `NoExternalCommitment`）**仅**对 `effective effect_category = local_read` 的工具开放：不持久化 ActionIntent / Effect，
  但必须产生事件 **`audit/observed{execution_id, tool, effect_category: local_read, params_digest, result_label, path: guard | hook | adapter, verdict?}`**，
  并照常做 Scope 检查、Grant 检查、`tools/list` 过滤与结果打标签。`path` 取值：`guard`（MCP 守卫）、`hook`（hooks 阻断路径，ADR-0007 §D3）、
  `adapter`（声明控制点的 Runtime 经适配器拦截点 / `os.submit_candidate` 提交的只读候选，如 `fysun-dsh` 对 DSH 原生只读工具的处理，[08 §2](../design/08-dsh-adapter.md)）。
- `external_read` 走完整协议（ActionIntent → Effect → Receipt）。Policy 通常对其 `allow`，因此实际开销主要是账本写入；理由是外部读取会留下访问
  痕迹、消耗配额、可能触发外部速率限制，且其结果是 T1 注入的主要载体——需要 Effect 记录来定位 `untrusted_external` 来源（蓝图 §I.2 H-2）。
- `audit/observed` 不是准入类事件（不带 `policy_decision`）；它与 Level-1 会话账本的关联面一起构成 MCP-only Runtime 的可审计面
  （[03](../design/03-architecture.md) §4）。
- 度量项"审计快路径占比"（`audit/observed` / (`audit/observed` + `action-intent/recorded`)）保留（12 §4）。
- 负例：`external_read` 工具产生 `audit/observed` 而无 `action-intent/recorded` → 失败；`local_read` 快路径调用无 `audit/observed` → 失败。

### D6 · 幂等冲突硬拒绝：`conflict.idempotency`

- ActionIntent 的幂等键 = canonical 参数 digest（ADR-0006 §D1）。当 Runtime 提交的候选携带**显式幂等键**（MCP `tools/call` 的 `_meta`
  或 `os.submit_candidate` 的 `idempotency_key`）且该键已存在、但 canonical 参数与已记录者不同 → 返回 **`conflict.idempotency`**，
  账本记 `action-intent/rejected{reason: conflict.idempotency}`；**不去重、不以任一方为准、不覆盖**。
- 同键同参数 → 命中 `action-intent/replayed`，返回已有 Effect 的状态（不变）。
- 理由（CognitiveOS `effects.rs` 经验）：同键不同参数意味着 Runtime 或适配器有 bug，或有人在重放时篡改参数；静默去重会让账本记录的参数与
  实际执行的参数不一致，破坏 F3。
- 负例：同幂等键不同 canonical 参数的两次提交都返回成功 → 失败（M1）。

### D7 · 补偿使用新 Grant 与新 ActionIntent

- `effect/compensated` 对应的补偿动作是一次**新的** Effect：必须有自己的 ActionIntent（新幂等键）与自己的 Grant 检查；**拒绝复用原 Effect
  的 Grant**——原 Grant 可能已撤销、耗尽或到期，且补偿动作的类别可能高于原动作（如撤回一条已发帖子是 `external_mutation`）。
- 补偿 ActionIntent 的 `causation_id` 指向被补偿的 Effect；Policy 对补偿类候选可单独设规则（context 中 `reversibility` 与被补偿 Effect 的引用）。
- 负例：`effect/compensated` 的补偿 Effect 引用原 Grant 且原 Grant 已 `grant/revoked` 仍执行 → 失败；补偿 Effect 无独立 `action-intent/recorded`
  → 失败。

## 否决的替代方案

- **以 `readOnlyHint` 为只读依据。**MCP 规范自身声明 annotations 不可信；被投毒的 MCP server 可把任何工具标为只读绕过完整协议（T2）。
- **由 Runtime 自报类别。**Runtime 是非权威方且运行模型输出；自报类别等于让候选决定自己是否被审查（T9）。
- **由 Capability 单方决定（无目录缺省、无 Policy 覆盖）。**实现包是供应链威胁的主要载体（T2）；必须有部署者侧的下限与覆盖权。
- **`external_read` 白名单快路径。**owner 否决（H-2）：外部读取是注入载体且留下外部痕迹；账本必须能回答"这条 `untrusted_external` 内容来自哪次
  外部读取"。快路径的收益（少一次账本事务）不值这个代价。
- **未分类默认 `physical_action`。**会使所有未分类工具 `not_runnable`（缺包络 Policy）；未分类应该是"走完整协议"而非"不可运行"。
- **同幂等键不同参数时以后者为准（last-write-wins）。**破坏 F3 的记录一致性；隐藏 Runtime bug。

## 后果

- `capability.register` 增加 `tool_effects[]` 必填校验与覆盖检查；`capability.health` 增"未分类工具"信号（04 §1.1）。
- Cedar 实体模型：action 实体按类别分组为六个父组（ADR-0009 §D1）。
- 事件族新增 `audit/observed`；错误码新增 `conflict.idempotency`；`action-intent/rejected.reason` 取值增 `conflict.idempotency`。
- M1 验收增项：未分类工具按 `external_mutation` 评估（负例）；`conflict.idempotency`；补偿必须新 Grant + 新 ActionIntent（负例）；
  `audit/observed` 只对 `local_read`，`external_read` 走完整协议（负例）。
- 守卫（[06](../design/06-mcp-guard.md) §3–§5）承担类别合成、快路径判定与 `list_changed` 重分类。
- 边缘离线规则（[ADR-0010](0010-edge-offline-ledger-consistency.md) §D4）获得可预授权的类别枚举。

## 待后续 ADR 的问题

- 同一工具按参数拆分类别（如 `git` 子命令）是否需要守卫层的参数模式匹配，还是一律要求 MCP server 拆工具名。
- `credential_use` 与 `external_mutation` 的边界案例（如带写权限的 OAuth 授权同时是二者）是否需要多类别标注。
- `audit/observed` 的保留策略（账本体量）是否需要独立于治理事件的压缩规则。

其余见 [12 MVP 计划 §7](../design/12-mvp-plan.md)。

## 修订历史

- v0.7.0（2026-09-08，WP6 收尾）：§D5 `audit/observed.path` 取值由 `guard | hook` 扩为 `guard | hook | adapter`。为什么：WP3 在 08 §2 定义 DSH 原生只读工具
  经 `fysun-approval` 拦截点走审计快路径，该路径既非守卫也非 hook，00 §4 已登记第三值 `adapter`，本 ADR 同步以保持字面一致。对应蓝图：§I.2 H-3 选项 B。
