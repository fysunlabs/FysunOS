# ADR-0002 · 协议优先的 Runtime 边界

- 状态：accepted（v0.7.0 修订）
- 日期：2026-09-07
- 决策者：owner
- 背景来源：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) §2.4、§4.1–§4.4、§7.1、§8.2
- supersedes：ADR-0001 §D2（部分）：
  "FysunOS 通过一个叠在 `dsh-base` 之上的 `fysun` bundle 替换凭据、审批、持久化、subagent 四个 seam 的
  provider 并订阅工具流水线事件" 与 "Cordis 通过语言无关的 Composition ABI 以 adapter 方式服务 TS Runtime"
  两句。D2 中"DSH 以隔离子进程进程外接入、不 fork、不 vendor、不进程内链接"与"FysunOS 不再自建 loop、
  prompt 装配、工具流水线、会话日志、compaction、subagent 传输、sandbox、聊天 UI、skills、终端、LSP"保留有效。
  ADR-0001 §D8 不受影响。ADR-0001 §D2 正文已于 v0.7.0 按本 ADR 修订。
- 修订：v0.7.0（2026-09-08）由 [ADR-0007](0007-runtime-native-tools-and-conformance-scope.md)（`describe()` 扩展、原生工具）、
  [ADR-0008](0008-effect-category-and-read-only-determination.md)（只读判定）、[ADR-0011](0011-approval-result-set-and-approval-surface.md)
  （DSH 拦截路径）、[ADR-0012](0012-mcp-guard-self-implementation.md)（关闭 §D7）承接；见文末修订历史。

## 背景

ADR-0001 §D1 承诺"让同一能力在任意 Agent 实现、Runtime 与环境中持续、受控、可恢复地运行"。v0.5.0 的接入
形态（05）依赖 DSH 的 7 个私有 seam / 扩展点（`ctx.credentials`、`ctx.approval`、`ctx.sessionPersistence`、
`ctx.subagents`、`tools/pre-execute` waterfall、`agent/pre-step`、`sdk` profile）。审查报告 §2.4 指出：

- 任何非 DSH 的 harness 要成为 Runtime，必须重新实现一套等价的 seam 适配；而 Claude Code、Codex、OpenClaw、
  Gemini CLI、Goose、OpenCode 等都没有这些 seam，却都有 MCP 客户端与（多数）Agent Skills 加载；
- 旧 05 §6（v0.5.0 编号）自己承认"每个新 Runtime 独立资格化；DSH 的证据不可转移"；
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
| 最小生命周期 RPC | `describe() -> {type, version, control_points[], native_effect_tools[], describe_digest}`、`start_execution`、`resume_execution`、`cancel`、`deliver_input`、`checkpoint()`（可返回 `unsupported`）、`health()`；语言无关 JSON-RPC over stdio（MVP） | 所有 Runtime 必须实现 |
| 可选控制点（能力位） | `context_injection`、`checkpoint`、`fork`、`compaction_hook`、`session_link`、`subagent_delegation` | 按 Runtime 声明 |

- `describe()` 的 `native_effect_tools[]` 如实列出 Runtime 内置、不经 MCP 守卫、可产生外部效果的工具；`describe_digest` 是资格化纪元的基准。
  声明不实的 Runtime 不在承诺范围（[ADR-0007](0007-runtime-native-tools-and-conformance-scope.md) §D1/§D7）。
- `fork_execution` 从最小 RPC 移除，升为控制点 `fork`；RPC 名保留为控制点扩展调用 `runtime.fork_execution`。
- `checkpoint()` 同时是最小 RPC（未支持时返回 `unsupported`）与控制点能力位 `checkpoint`（声明支持）。
- Runtime → OS 调用（`os.request_context`、`os.submit_candidate`、`os.observe_effect`、`os.request_resource`、
  `os.report_checkpoint`、`os.report_progress`、`os.request_handoff`、`os.report_failure`、`os.propose_memory`、
  `os.propose_adjustment`、`os.report_receipt`——最后一条为适配器拦截路径专用：对经 `os.submit_candidate` 准入、由 Runtime 自行执行工具体的 Effect
  回填 receipt，[05 §4](../design/05-runtime-abi.md)）保留为控制点适配器的内部语义；只实现最小面的 Runtime 经 MCP 守卫隐式完成。
- Runtime 不得调用（不存在于 ABI 中）：`mint_grant`、`commit_effect`、`complete_activity`、`write_ledger`、
  `mark_verified`、`decide_approval`。
- **Level-1 / Level-2 只是文档速记，不是等级制度**：Level-1 = MCP 客户端 + 最小生命周期 RPC；Level-2 =
  Level-1 + 至少一个可选控制点；Level-1b = Level-1 + hooks.json PreToolUse 阻断（仅阻断，不计入 conformance 主路径，ADR-0007 §D3）。
  账本记录 `runtime/qualified{control_points[], native_effect_tools[], describe_digest, bundle_digest}`，不记录 Level。
  速记**不得出现在任何对象名、字段名、组件名、事件名、错误码、Cedar 实体与属性名中**（[00 §1](../design/00-terminology.md) 禁令）。
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

- 守卫结果语义为 `Pass` / `Reject` / `Mutated`，借用 agentgateway `mcpGuardrails` 的三值语义，守卫自实现
  （[ADR-0012](0012-mcp-guard-self-implementation.md)）。`Mutated` 对应 `policy_decision.outcome = modify`。
- 只有 `effect_category = local_read` 的工具走审计快路径（产生 `audit/observed` 事件，不持久化 ActionIntent / Effect）；
  `external_read` 走完整协议。只读判定与类别来源见 [ADR-0008](0008-effect-category-and-read-only-determination.md)。
- **不可区分性**（借鉴 DeepSeek Harness scope 过滤）：被 `tools/list` 过滤掉的工具在 `tools/call` 中被调用 → 必须 `Reject`，且对外返回的
  MCP 错误**与调用一个不存在的工具完全相同**（错误码、消息形状、耗时特征不泄露"该工具存在但被过滤"）；账本内记
  `action-intent/rejected{reason: denied.grant | denied.policy}`。
- 负例：被过滤工具的 `tools/call` 返回与未知工具可区分的错误 → 失败；账本出现对应的 `effect/authorized` → 失败；
  一次非快路径 `tools/call` 无配对的 `action-intent/recorded` → 失败；一次 `local_read` 快路径调用无 `audit/observed` → 失败。

### D3 · 外部 Handoff 形态 = A2A

- 当 Handoff 目标是外部 Agent（非本 `fysund` 管理的 Execution）时，`fysund` 作为 A2A client 发起 Task。
- Handoff 状态直接映射 A2A v1.0 TaskState（对照表见
  [00-terminology](../design/00-terminology.md) §3），并叠加 FysunOS 专有的 `verified` 态：父 Execution 或独立
  Verifier 对 deliverables 的验证。`completed` 不含验证语义。
- Handoff 记录 `context_id`（= Activity id 或 Handoff 链根），对应 A2A `contextId`；终态不重开，细化走同
  `context_id` 的新 Handoff。
- FysunOS 的 Capability 以签名 Agent Card 对外暴露为 A2A server 属于多主体场景，**非 MVP**；它是
  开放问题 Q5（多主体 Scope 何时激活，[12 §7](../design/12-mvp-plan.md)）的第一个触发点。
- 外部 A2A Task 返回的 Artifact 一律 `untrusted_external`（不存在"可信 Agent Card"例外，[ADR-0004](0004-provenance-labels-and-monotonic-privilege.md) §D1 修订）。
- 负例：一个外部 A2A Task 的 `TASK_STATE_COMPLETED` 直接导致 `handoff/verified` → 失败。

### D4 · 实现包 = Agent Skills

- `Capability.implementation` 引用一个 Agent Skills 目录（digest 固定）。FysunOS **不定义**自己的实现包格式。
- `SKILL.md` 的 `allowed-tools` 是 Grant 请求的初始上界：Execution 请求的 Grant 只能是它的子集，之后只能再收窄。
- `compatibility` 映射到 `Capability.requirements`。
- 渐进披露的三层（元数据 → 正文 → 按需资源）由 Resource 子系统的上下文装配承担，不进入对象模型。
- DSH 的 skills 加载格式若与 Agent Skills 不同，以 Agent Skills 为准（[08 §5](../design/08-dsh-adapter.md)）。
- 负例：一个 Execution 获得的 Grant 包含 `allowed-tools` 之外的工具 → 失败。

### D5 · DSH 降为声明全部六个控制点的参考适配器

- DSH 不再是"默认参考 Runtime"，而是**声明全部六个控制点的参考适配器**，适配器名 `fysun-dsh`；技术形态仍为叠在
  `dsh-base` 之上的 bundle + `fysun` profile，隔离子进程、不 fork、不 vendor、不进程内链接（ADR-0001 §D2
  保留部分）。bundle 启动时校验 `dsh_version` 与 `schema_digest`，不匹配拒绝启动（[08 §6](../design/08-dsh-adapter.md)）。
- 子组件重新归类：`fysun-tool-gate` **作为独立组件取消**——MCP 工具面由 DSH 把 `fysund` 配为 MCP server 即受守卫；
  `fysun-credentials`、`fysun-approval`、`fysun-session-link`、`fysun-context`、`fysun-subagent` 为控制点适配器；
  `fysun-rpc` 为最小生命周期 RPC 的 DSH 实现。
- **DSH 原生工具治理路径**（v0.7.0，蓝图 §I.2 H-3 选项 B）：DSH 原生 fs / shell 等工具保留，不经 MCP 守卫；`fysun-approval` 在 DSH
  `approval/request` waterfall **拦截、不 `next()`、全部转给 `fysund`**，该拦截点即 DSH 原生工具的 **ActionIntent 持久化点**
  （persist-before-dispatch 成立：拦截发生在工具体执行前）；`fysun-dsh` **订阅 `tools/post-execute` 作为 receipt 采集点**。
  原生工具的 `tools/list` 过滤由 `fysun` profile 的 `tools.restrict` 承担，不可区分性由 DSH scope 机制保证（M2b 验收项）。
  `fysun-dsh` 的 `describe()` 必须如实把 DSH 原生工具列入 `native_effect_tools[]`（不为空）。
  前置条件：`fysun` profile 必须禁止 DSH approval policy `never`（DSH 的 `never` 在服务内部先于 waterfall 执行，会绕过拦截）；
  违反则 `fysun-dsh` 拒绝启动或 `runtime/disqualified{reason: adapter_precondition_failed}`
  （[ADR-0011](0011-approval-result-set-and-approval-surface.md) §D5）。
- M2a 的 MCP-only 基线 Runtime 固定为 **DSH headless 模式**（[ADR-0007](0007-runtime-native-tools-and-conformance-scope.md) §D8）；
  Codex / Claude Code / Goose / OpenCode / OpenClaw / Gemini CLI 等列为**候选**（只要有 MCP 客户端即可接入，但均含原生工具，须按
  ADR-0007 §D1 声明），也可继续作为 DSH 内的委派目标。
- 负例：代码中出现 `fysun-tool-gate` 组件，或设计文档把它列为现役组件（而非"作为独立组件取消"）→ 失败；`fysun-dsh`
  未实现 `describe()` 或 `native_effect_tools[]` 为空 → 不得 `runtime/qualified`；`fysun-approval` 对 `approval/request` 调用 `next()`
  → 失败；DSH approval policy 为 `never` 时 `fysun-dsh` 正常启动 → 失败；一次 DSH 原生工具调用无 ActionIntent 而被放行 → 失败。

### D6 · Composition ABI 降为内部适配器生命周期

- Composition ABI 从公共接口降为：(a) `fysund` 内部适配器 trait；(b) `fysun-dsh` 内部对 Cordis 生命周期的映射。
  文档位置为 [08 附录 A](../design/08-dsh-adapter.md)（"内部适配器生命周期（原 Composition ABI）"；v0.7.0 由旧 04 附录 A 移至 08 附录 A）。
- 公共接口从三组减为两组：Capability API（应用 ↔ OS）与 Runtime ABI（Runtime ↔ OS）。
- 原 Composition ABI 末尾的两条 OS 语义规则保留，改由账本事件表达：
  - 组件消失不得绕过 Grant 撤销与 Effect 对账：`resource/unavailable` 或 `resource/quarantined` 必须触发关联
    Grant 的 `grant/revoked` 与未决 Effect 的对账（`effect/outcome-unknown` → `effect/reconciled` 或
    `effect/quarantined`）；
  - 组件图不是信任图：`resource/registered` 不隐含任何 Grant。
- 新增事件 `resource/quarantined` 承接原 `quarantine` 语义。
- 负例：一个 Resource 进入 `unavailable` 后，其绑定 Grant 仍可 `grant/consumed` → 失败。

### D7 · agentgateway 的采用形态（已由 ADR-0012 关闭：自实现）

- v0.6.0 本节把 `fysun-mcp-guard` 是否采用 agentgateway（库嵌入 / 独立进程前置 / 仅借用语义自实现）列为旧 08 开放问题 10（现 12 §7 Q10）。
- v0.7.0 由 [ADR-0012](0012-mcp-guard-self-implementation.md) 关闭：**`fysun-mcp-guard` 自实现**，仅借用 `Pass` / `Reject` / `Mutated`
  三值语义与 Cedar 策略格式（[ADR-0003](0003-policy-object-and-language.md)）；不作为库嵌入、不进程前置、不 fork、不 vendor。
  F1（唯一权威写入者）不变。

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
- **fork agentgateway 后修改语义。**否决理由同 ADR-0001 §D2 对 DSH 的处理：不 fork、不 vendor（形态选择已由 ADR-0012 关闭为自实现）。

## 后果

- MVP 计划的 M2 拆为 M2a（MCP 守卫 + MCP-only 接入，基线 DSH headless）与 M2b（`fysun-dsh` 六控制点适配）。
  M2a 的验收：每个非快路径 `tools/call` 配对一个 Effect，`local_read` 快路径配对 `audit/observed`；`tools/list` 按 Grant ∩ Policy ∩
  `allowed-tools` 过滤且**被过滤工具的 `tools/call` 返回与未知工具相同的错误**（不可区分性负例）；`start_execution` 缺控制点 / 未资格化大声
  失败；F9 负例（[ADR-0004](0004-provenance-labels-and-monotonic-privilege.md)）。
- 边缘 Runtime 只需实现 MCP 客户端 + 最小生命周期 RPC；"Node 依赖不适合边缘"的论证（[07 §8](../design/07-runtime-onboarding.md)）仍然成立且
  不再构成阻碍。
- 引入对 MCP（2025-11-25 及后续）、A2A v1.0、Agent Skills 版本演进的跟随成本；MCP 2026 路线图中的
  agent 通信与 server card 若稳定，可能替代部分自定义 RPC。
- MCP 守卫为每次 `tools/call` 增加一次 pre/post 往返；`local_read` 的审计快路径缓解但不消除此开销。
- DSH 资格化证据不可转移到其他 Runtime；每个 Runtime 按 Resource id 独立 `runtime/qualified`，并受资格化纪元约束（ADR-0007 §D7）。
- ADR-0001 §D2 已按本 ADR 修订；维护 `fysun-dsh` 对钉住 revision 的适配与 CI 漂移预警的持续成本仍存在，只是不再是关键路径。
- 旧 08 开放问题 1 拆为两问：MCP 传输（stdio + Streamable HTTP，跟随 MCP）；最小生命周期 RPC（stdio JSON-RPC，
  MVP）。

## 待后续 ADR 的问题

- 对外 A2A server（签名 Agent Card）与多主体 Scope 的激活条件（12 §7）。
- MCP server card（SEP-1649 / SEP-1960）稳定后是否用于 Runtime 能力发现替代 `describe()`。
- 完整信息流控制是否需要 Level-3 控制点（报告附录 B；12 §7）。

其余见 [12 MVP 计划 §7](../design/12-mvp-plan.md)。

## 修订历史

- v0.7.0（2026-09-08）：§D1 `describe()` 返回增 `native_effect_tools[]`、`describe_digest`，`runtime/qualified` 增三字段，速记段增 Level-1b
  定义与标识符禁令引用；§D2 守卫改"借用语义、自实现"，只读快路径限定为 `local_read` 并产生 `audit/observed`，负例改为"被过滤工具的 `tools/call`
  必须返回与未知工具相同的 MCP 错误"，增"只读判定见 ADR-0008"；§D4 引用改指 08 §5；§D5 标题与正文去 Level 标识，M2a 基线 = DSH headless、
  其他 harness 列为候选，**"`fysun-tool-gate` 取消（DSH 把 `fysund` 配为 MCP server 即可）"修订为"`fysun-tool-gate` 作为独立组件取消；
  `fysun-approval` 在 `approval/request` waterfall 拦截作为 DSH 原生工具准入点，并订阅 `tools/post-execute` 作为 receipt 采集点"**，
  增 `native_effect_tools[]` 不为空、禁止 approval policy `never`、bundle 校验 `dsh_version` / `schema_digest`；§D6 附录 A 位置 → 08 附录 A；
  §D7 改"已由 ADR-0012 关闭：自实现"；否决段 agentgateway 项去"三者之间选择"；后果段 M2a 验收补不可区分性与大声失败、资格化纪元；
  待后续段删 agentgateway 项。为什么：蓝图 §I.1 决策 5（原生工具声明）、7（agentgateway 自实现）、9（M2a 基线）、11（DSH 审批拦截）、
  12（Level 禁令、Runtime 资格化纪元）；§I.2 H-3 选项 B（保留 DSH 原生工具并经拦截 + `tools/post-execute` 路径治理，因此 v0.6.0 "不再订阅
  `tools/post-execute`"不再成立）；§0 C2-7（候选清单）；借鉴 DeepSeek Harness scope 过滤的不可区分性。对应蓝图：§C.1 ADR-0002 行。
- v0.7.0（2026-09-08，WP6 收尾）：§D1 Runtime → OS 调用清单增 `os.report_receipt`。为什么：H-3 选项 B 使 `fysun-dsh` 订阅 `tools/post-execute` 采集 DSH
  原生工具 receipt，需要一条 Runtime → OS 的 receipt 回填调用（00 §7、05 §4、08 §2 已登记），本 ADR 的清单同步以保持字面一致。
