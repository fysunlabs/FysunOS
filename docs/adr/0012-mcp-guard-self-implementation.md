# ADR-0012 · `fysun-mcp-guard` 自实现，关闭 agentgateway 复用评估

- 状态：accepted
- 日期：2026-09-08
- 决策者：owner
- 背景来源：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md) §I.1 决策 7、§I.2 H-8；[ADR-0002](0002-protocol-first-runtime-boundary.md) §D7；旧 08 开放问题 10；agentgateway 1.3 `mcpGuardrails` 文档（Pass / Reject / Mutated 语义、Cedar 策略格式）
- supersedes：ADR-0002 §D7（agentgateway 采用形态悬而未决）——由本 ADR 关闭；ADR-0002 正文已于 v0.7.0 同步修订。关闭旧 08 开放问题 10。

## 背景

ADR-0002 §D7 把 `fysun-mcp-guard` 是否采用 agentgateway（Rust、AAIF、Apache-2.0）留为三选一的开放问题：作为库嵌入 `fysund` / 作为独立进程前置 /
仅借用语义自实现，并要求"在 M2a 前决定"。v0.6.0 文档在 README、00 §9、03 §3、旧 04 §2.1、旧 05 §7、旧 08 §7（v0.6.0 旧编号）六处重复这一悬而未决的表述。

v0.7.0 重构使守卫职责显著超出 agentgateway `mcpGuardrails` 的范围：Effect 类别三源合一与只读判定（[ADR-0008](0008-effect-category-and-read-only-determination.md)）、
ActionIntent 持久化与幂等冲突、结果标签与 `authority_claim` / `secret_shaped` 分类（[ADR-0004](0004-provenance-labels-and-monotonic-privilege.md) 修订）、
Approval 阻塞与 `fysun.await_approval` 系统工具（[ADR-0007](0007-runtime-native-tools-and-conformance-scope.md) §D5）、下游 MCP server 会话持有与
`SecretRef` 注入、`audit/observed` 快路径事件——这些都与 `fysund` 的账本写入紧耦合，任何形态的外部守卫都要么把账本写入拆到两个进程，
要么在守卫之外再做一遍同样的判定。

同时，DSH 自带一个名为 `guard/` 的包（loop 卫生插件：检测重复、空转等），与"MCP 守卫"名称接近但职责无关，需在术语上消歧。

owner 决策（蓝图 §I.1 决策 7）：仅借用 Pass / Reject / Mutated 语义与 Cedar 格式，`fysun-mcp-guard` 自实现；独立成本 ADR；关闭旧 08 Q10（现 12 §7 Q10）与
ADR-0002 §D7。

## 决策

### D1 · `fysun-mcp-guard` 为 `fysund` 内自实现组件

- `fysun-mcp-guard` 是 `fysund` 进程内的组件（Rust），实现 [06](../design/06-mcp-guard.md) 全部规范：`tools/list` 过滤与不可区分性、`tools/call`
  pre / 执行 / post、类别合成、审计快路径、结果标签与 flags、`tasks/*` 与 `elicitation/create`、守卫系统工具、下游 server 会话与凭据边界、
  hooks 路径的本地 RPC 端。
- **不以 agentgateway 为库依赖、不 fork、不 vendor、不作为独立进程前置**。`fysund` 的依赖清单中不出现 agentgateway crate。
- F1（唯一权威写入者）不变且更简单：守卫判定与账本写入在同一进程、同一事务边界内完成。

### D2 · 借用的仅是语义与格式

- 守卫结果语义 **`Pass` / `Reject` / `Mutated`** 三值，与 agentgateway `mcpGuardrails` 相同；与 `policy_decision.outcome` 的对应为
  `allow → Pass`、`deny → Reject`、`modify → Mutated`、`approve → 阻塞后按结果 Pass / Reject`。
- 策略格式为 Cedar（[ADR-0003](0003-policy-object-and-language.md)、[ADR-0009](0009-cedar-subset-and-entity-model.md)）；选 Cedar 的理由已不含
  "便于采用 agentgateway"，改为可静态分析与 Rust 实现可用（ADR-0003 §D3 修订）。
- 借用语义的意义在于：若未来出现兼容这两种约定的外部守卫产品，`fysund` 的策略与结果表达无需转换即可对照测试；这不是复用承诺。
- 负例：守卫结果出现三值以外（如 `Warn`、`Defer`）→ 失败；守卫结果与 `policy_decision.outcome` 对应关系出现表外映射 → 失败。

### D3 · 与 DSH `guard/` 包消歧

- DSH 的 `guard/` 包（`packages/guard`）是 **loop 卫生插件**——检测重复工具调用、空转、超长 turn 等，作用于 DSH 进程内的 loop 决策；
  它**不是**安全守卫，不在 TCB 内，其判定对 OS 只是 Runtime 内部意见（05 §5）。
- 全仓库使用"MCP 守卫"/ `fysun-mcp-guard` 指本 ADR 的组件；提到 DSH 的 `guard/` 时必须带"DSH `guard/` 包（loop 卫生插件）"限定。
- 负例：设计文档把 DSH `guard/` 包描述为 FysunOS 安全机制的一部分 → 失败。

### D4 · 关闭项

- 旧 08 开放问题 10（agentgateway 采用形态）关闭。
- ADR-0002 §D7 关闭，正文改为引用本 ADR。
- 旧 08 开放问题 7 中"与 CEL（agentgateway `mcpAuthorization`）的互操作"子项删除（ADR-0009 待后续段不再列）。
- README 许可证段"agentgateway 为 Apache-2.0"改为"Cedar 为 Apache-2.0"；sources.md 中 agentgateway 条目的用途改为"守卫结果语义与策略格式的
  参照，非复用候选"（WP4 / WP6 落实）。

## 否决的替代方案

- **作为库嵌入 `fysund`。**引入外部依赖的发布节奏（agentgateway 跟随 AAIF 与 Kubernetes 生态）；其 guardrails 抽象覆盖不到 ActionIntent 持久化、
  类别合成、快路径与系统工具，嵌入后仍需在其外围自实现大部分逻辑；TCB 扩大（引入其全部传输与路由代码）。
- **作为独立进程前置。**多一跳延迟；策略在前置进程与 `fysund` 各评估一次（前置只能做过滤改写，ActionIntent 持久化仍在 `fysund`），
  两处结果可能不一致；与 F1 "唯一权威写入者"的单进程语义重叠而无收益；下游 server 会话与 `SecretRef` 注入必须在 `fysund` 内，
  前置进程拿不到凭据也就做不了真正的守卫。
- **fork agentgateway 后修改语义。**已由 ADR-0002 否决（不 fork、不 vendor）。
- **完全不借用语义、自定义结果集。**三值语义已足够表达 `outcome` 映射，自定义只增加对照成本。

## 后果

- README、00 §9、03 §3、06 §1、07 §8 中关于 agentgateway 的悬而未决表述全部改为"自实现，借用 agentgateway 语义（ADR-0012）"；旧 05 §7
  代价表中 agentgateway 一行删除。
- `fysund` 依赖清单不含 agentgateway；许可证兼容性说明中去 agentgateway、保留 Cedar（Apache-2.0）。
- 12 §6 非声明删"agentgateway 已被复用或评估完成"。
- 06 负例增"`fysund` 依赖 agentgateway crate → 失败"。
- 守卫的全部行为由 06 规范约束，可独立于任何外部产品做 conformance 测试。

## 待后续 ADR 的问题

- 无新增。若未来 MCP 官方或 AAIF 发布标准化的守卫 / guardrail 协议，是否对齐其结果语义另议。

其余见 [12 MVP 计划 §7](../design/12-mvp-plan.md)。
