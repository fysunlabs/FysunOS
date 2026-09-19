# ADR-0002 · 协议优先的 Runtime 边界

- 状态：accepted
- 日期：2026-09-07
- 决策者：owner
- 背景来源：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) §2.4、§4.1–§4.4、§7.1、§8.2
- supersedes：ADR-0001 §D2（部分）：
  "FysunOS 通过一个叠在 `dsh-base` 之上的 `fysun` bundle 替换凭据、审批、持久化、subagent 四个 seam 的
  provider 并订阅工具流水线事件" 与 "Cordis 通过语言无关的 Composition ABI 以 adapter 方式服务 TS Runtime"
  两句。D2 中"DSH 以隔离子进程进程外接入、不 fork、不 vendor、不进程内链接"与"FysunOS 不再自建 loop、
  prompt 装配、工具流水线、会话日志、compaction、subagent 传输、sandbox、聊天 UI、skills、终端、LSP"保留有效。
  ADR-0001 §D8 不受影响。ADR-0001 正文不改。

## 背景

ADR-0001 §D1 承诺"让同一能力在任意 Agent 实现、Runtime 与环境中持续、受控、可恢复地运行"。v0.5.0 的接入
形态（05）依赖 DSH 的 7 个私有 seam / 扩展点（`ctx.credentials`、`ctx.approval`、`ctx.sessionPersistence`、
`ctx.subagents`、`tools/pre-execute` waterfall、`agent/pre-step`、`sdk` profile）。审查报告 §2.4 指出：

- 任何非 DSH 的 harness 要成为 Runtime，必须重新实现一套等价的 seam 适配；而 Claude Code、Codex、OpenClaw、
  Gemini CLI、Goose、OpenCode 等都没有这些 seam，却都有 MCP 客户端与（多数）Agent Skills 加载；
- 05 §6 自己承认"每个新 Runtime 独立资格化；DSH 的证据不可转移"；
- 结论：v0.5.0 的承诺在实现路径上等价于"任何愿意实现 fysun bundle 的 Runtime"，承诺不成立。

报告 §4.1 给出 2026-09 的协议格局：工具调用与工具目录（MCP 2025-11-25，AAIF 治理）、长任务（MCP Tasks，
实验、正移入扩展）、需人提供输入（MCP elicitation；A2A `INPUT_REQUIRED` / `AUTH_REQUIRED`）、委派
（A2A v1.0）、实现包（Agent Skills，约 40 个产品支持）已有收敛标准；上下文注入、检查点、fork、compaction、
组件生命周期没有标准。v0.5.0 把两类职责放在同一层抽象里并统一用自定义 JSON-RPC 表达。

报告 §4.4 指出 Composition ABI（mount / unmount / replace / inspect / dependency_status / health / dispose /
quarantine）解决的是进程内动态组合，而 FysunOS 的边界是进程外、权威与非权威之间；三个后端里只有 Cordis
adapter 是真实需求且完全位于 TS Runtime 一侧。

报告 §7.1 指出 08 的 M2 完全等于"fysun bundle + DSH 子进程管理"，M3–M5 全部依赖 M2，而 DSH 处于
developer preview 并明言破坏性变更。

owner 已确认方向：OS ↔ Runtime 边界采用协议优先。本 ADR 记录该方向的可判定决策。

## 决策

### D1 · Runtime ABI 三块结构

v0.5.0 的 Harness ABI 更名为 **Runtime ABI**，由三块组成：

| 块 | 内容 | 谁必须实现 |
|---|---|---|
| MCP 守卫 | `fysund`（组件 `fysun-mcp-guard`）以 MCP 中间层身份位于 harness 与工具之间 | 所有 Runtime 必经 |
| 最小生命周期 RPC | `describe() -> {type, version, control_points[]}`、`start_execution`、`resume_execution`、`cancel`、`deliver_input`、`checkpoint()`（可返回 `unsupported`）、`health()`；语言无关 JSON-RPC over stdio（MVP） | 所有 Runtime 必须实现 |
| 可选控制点（能力位） | `context_injection`、`checkpoint`、`fork`、`compaction_hook`、`session_link`、`subagent_delegation` | 按 Runtime 声明 |

- `fork_execution` 从最小 RPC 移除，升为控制点 `fork`；RPC 名保留为控制点扩展调用 `runtime.fork_execution`。
- `checkpoint()` 同时是最小 RPC（未支持时返回 `unsupported`）与控制点能力位 `checkpoint`（声明支持）。
- Runtime → OS 调用（`os.request_context`、`os.submit_candidate`、`os.observe_effect`、`os.request_resource`、
  `os.report_checkpoint`、`os.report_progress`、`os.request_handoff`、`os.report_failure`、`os.propose_memory`、
  `os.propose_adjustment`）保留为 Level-2 控制点的内部语义；Level-1 经 MCP 守卫隐式完成。
- Runtime 不得调用（不存在于 ABI 中）：`mint_grant`、`commit_effect`、`complete_activity`、`write_ledger`、
  `mark_verified`、`decide_approval`。
- **Level-1 / Level-2 只是文档速记，不是等级制度**：Level-1 = MCP 客户端 + 最小生命周期 RPC；Level-2 =
  Level-1 + 至少一个可选控制点。账本记录 `runtime/qualified{control_points[]}`，不记录 Level。
  Capability.requirements 可以要求特定控制点，Capability 子系统据此做环境匹配。
- 负例：一个 Runtime 声明 `control_points = []` 却被账本记为"Level-2"→ 失败（账本中不允许出现 Level 字段）；
  一个未声明 `checkpoint` 的 Runtime 收到 `checkpoint()` 后返回非 `unsupported` 的成功 → 失败。

### D2 · MCP 守卫语义

harness 把 `fysund` 视为唯一的 MCP server，或 `fysund` 前置在 harness 配置的所有 MCP server 之前。

| MCP 方法 | 守卫行为 |
|---|---|
| `tools/list` | 按 Grant 与 Policy 过滤并改写工具面。看不见就调不了，这是"能力不得自扩权"（F7）的机械化 |
| `tools/call` pre | = `submit_candidate`：ActionIntent 持久化、Policy 评估、Grant 检查、来源标签检查（[ADR-0004](0004-provenance-labels-and-monotonic-privilege.md)） |
| `tools/call` 执行 | Effect dispatch |
| `tools/call` post | Receipt 回填；对结果打来源标签 |
| `tasks/*` | 长任务与 `outcome_unknown` 的标准表达；`effect/reconciled` 对应 `tasks/result` |
| `elicitation/create` | Approval 的传输载体之一（client 支持时；[ADR-0005](0005-approval-and-evidence-gated-evolution.md)） |

- 守卫结果语义为 `Pass` / `Reject` / `Mutated`，与 agentgateway `mcpGuardrails` 相同。`Mutated` 对应
  `policy_decision.outcome = modify`。
- 只读工具走审计快路径（仍产生事件，不产生外部 Effect 的 receipt 等待）。
- 负例：被 `tools/list` 过滤掉的工具在 `tools/call` 中被调用 → 必须 `Reject`（`denied.grant` 或 `denied.policy`），
  账本不得出现对应的 `effect/authorized`；一次 `tools/call` 无配对的 `action-intent/recorded` → 失败。

### D3 · 外部 Handoff 形态 = A2A

- 当 Handoff 目标是外部 Agent（非本 `fysund` 管理的 Execution）时，`fysund` 作为 A2A client 发起 Task。
- Handoff 状态直接映射 A2A v1.0 TaskState（对照表见
  [00-terminology](../design/00-terminology.md) §3），并叠加 FysunOS 专有的 `verified` 态：父 Execution 或独立
  Verifier 对 deliverables 的验证。`completed` 不含验证语义。
- Handoff 记录 `context_id`（= Activity id 或 Handoff 链根），对应 A2A `contextId`；终态不重开，细化走同
  `context_id` 的新 Handoff。
- FysunOS 的 Capability 以签名 Agent Card 对外暴露为 A2A server 属于多主体场景，**非 MVP**；它是
  08 开放问题 5（多主体 Scope 何时激活）的第一个触发点。
- 负例：一个外部 A2A Task 的 `TASK_STATE_COMPLETED` 直接导致 `handoff/verified` → 失败。

### D4 · 实现包 = Agent Skills

- `Capability.implementation` 引用一个 Agent Skills 目录（digest 固定）。FysunOS **不定义**自己的实现包格式。
- `SKILL.md` 的 `allowed-tools` 是 Grant 请求的初始上界：Execution 请求的 Grant 只能是它的子集，之后只能再收窄。
- `compatibility` 映射到 `Capability.requirements`。
- 渐进披露的三层（元数据 → 正文 → 按需资源）由 Resource 子系统的上下文装配承担，不进入对象模型。
- DSH 的 skills 加载格式若与 Agent Skills 不同，以 Agent Skills 为准（05 §4）。
- 负例：一个 Execution 获得的 Grant 包含 `allowed-tools` 之外的工具 → 失败。

### D5 · DSH 降为第一个 Level-2 参考适配器

- DSH 不再是"默认参考 Runtime"，而是**第一个 Level-2 参考适配器**，适配器名 `fysun-dsh`；技术形态仍为叠在
  `dsh-base` 之上的 bundle + `fysun` profile，隔离子进程、不 fork、不 vendor、不进程内链接（ADR-0001 §D2
  保留部分）。
- 子组件重新归类：`fysun-tool-gate` **取消**（DSH 把 `fysund` 配为 MCP server 即可）；`fysun-credentials`、
  `fysun-approval`、`fysun-session-link`、`fysun-context`、`fysun-subagent` 为 Level-2 控制点适配器；
  `fysun-rpc` 为最小生命周期 RPC 的 DSH 实现。
- Codex / Claude Code / Goose / OpenCode / OpenClaw / Gemini CLI 等可作为 Level-1 直接接入（只要有 MCP
  客户端），也可继续作为 DSH 内的委派目标。
- 负例：代码中出现 `fysun-tool-gate` 组件，或设计文档把它列为现役组件（而非"已取消"）→ 失败；`fysun-dsh`
  未实现 `describe()` → 不得 `runtime/qualified`。

### D6 · Composition ABI 降为内部适配器生命周期

- Composition ABI 从公共接口降为：(a) `fysund` 内部适配器 trait；(b) `fysun-dsh` 内部对 Cordis 生命周期的映射。
  文档位置为 04 附录 A（"内部适配器生命周期（原 Composition ABI）"）。
- 公共接口从三组减为两组：Capability API（应用 ↔ OS）与 Runtime ABI（Runtime ↔ OS）。
- 原 Composition ABI 末尾的两条 OS 语义规则保留，改由账本事件表达：
  - 组件消失不得绕过 Grant 撤销与 Effect 对账：`resource/unavailable` 或 `resource/quarantined` 必须触发关联
    Grant 的 `grant/revoked` 与未决 Effect 的对账（`effect/outcome-unknown` → `effect/reconciled` 或
    `effect/quarantined`）；
  - 组件图不是信任图：`resource/registered` 不隐含任何 Grant。
- 新增事件 `resource/quarantined` 承接原 `quarantine` 语义。
- 负例：一个 Resource 进入 `unavailable` 后，其绑定 Grant 仍可 `grant/consumed` → 失败。

### D7 · agentgateway 复用形态待评估

- `fysun-mcp-guard` 是否复用 agentgateway（Rust、AAIF、Apache-2.0）有三种形态：作为库嵌入 `fysund` /
  作为独立进程前置 / 仅借用语义自实现。形态选择列为 08 开放问题 10，在 M2a 前决定。
- 无论哪种形态，守卫结果采用 `Pass` / `Reject` / `Mutated` 语义，策略采用 Cedar 格式
  （[ADR-0003](0003-policy-object-and-language.md)），以便未来切换。
- 若采用独立进程前置，F1（唯一权威写入者）不变：agentgateway 只做过滤与改写，ActionIntent 持久化与
  Effect 状态推进仍只由 `fysund` 执行。

## 否决的替代方案

- **继续以 DSH 私有 seam 为唯一接入形态。**"任意 Runtime"的承诺不成立（§2.4）；M2–M5 押在单一
  developer-preview 项目上（§7.1）；边缘 Runtime 必须携带 Node 依赖。
- **自定义 JSON-RPC 覆盖全部 Runtime ↔ OS 职责。**工具调用、长任务、人工输入、委派、实现包已有收敛标准；
  自定义协议等于要求每个 harness 为 FysunOS 写一次适配，与"不重复做 Runtime 已做好的事"（P8）冲突。
- **把 Composition ABI 保留为公共接口。**它服务的三个后端中只有 Cordis adapter 是真实需求且位于 TS 一侧；
  其真正的 OS 语义可由 Resource 事件与 Grant 撤销规则表达（§4.4）。
- **一次性全量采用 MCP Tasks 作为 Effect 长任务的唯一表达。**MCP Tasks 仍为实验特性且正移入扩展
  `io.modelcontextprotocol/tasks`（SEP-2663）；FysunOS 对 `tasks/*` 的支持为可选，`outcome_unknown` 的
  账本语义不依赖它。
- **fork agentgateway 后修改语义。**否决理由同 ADR-0001 §D2 对 DSH 的处理：不 fork、不 vendor；复用只
  在库嵌入、进程前置、借用语义三者之间选择。

## 后果

- 08 的 M2 拆为 M2a（MCP 守卫 + Level-1 接入任意一个现成 MCP 兼容 harness）与 M2b（`fysun-dsh` Level-2
  适配）。M2a 的验收：每个 `tools/call` 配对一个 Effect；`tools/list` 按 Grant 过滤且被过滤工具不可调用；
  F9 负例（[ADR-0004](0004-provenance-labels-and-monotonic-privilege.md)）。
- 边缘 Runtime 只需实现 MCP 客户端 + 最小生命周期 RPC；05 §7"Node 依赖不适合边缘"的论证仍然成立且不再构成
  阻碍。
- 引入对 MCP（2025-11-25 及后续）、A2A v1.0、Agent Skills 版本演进的跟随成本；MCP 2026 路线图中的
  agent 通信与 server card 若稳定，可能替代部分自定义 RPC。
- MCP 守卫为每次 `tools/call` 增加一次 pre/post 往返；只读工具的审计快路径缓解但不消除此开销。
- DSH 资格化证据不可转移到其他 Runtime；每个 Runtime 按 `control_points[]` 独立 `runtime/qualified`。
- ADR-0001 §D2 的两句被 supersede，但"维护一个与 DSH 版本一一对应的 bundle"的持续成本仍存在，只是不再
  是关键路径。
- 08 开放问题 1 拆为两问：MCP 传输（stdio + Streamable HTTP，跟随 MCP）；最小生命周期 RPC（stdio JSON-RPC，
  MVP）。

## 待后续 ADR 的问题

- agentgateway 复用形态（08 开放问题 10）。
- 对外 A2A server（签名 Agent Card）与多主体 Scope 的激活条件（08 开放问题 5）。
- MCP server card（SEP-1649 / SEP-1960）稳定后是否用于 Runtime 能力发现替代 `describe()`。
- 完整信息流控制是否需要 Level-3 控制点（报告附录 B）。

其余见 [08 MVP 计划 §7](../design/08-mvp-plan.md#7-开放问题待后续-adr)。
