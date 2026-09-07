# 05 · Runtime 接入与 DSH 适配器

- 状态：设计稿 v0.6.0（2026-09-07）
- 变更依据：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) 与 ADR-0002 ~ ADR-0006
- 前置：[04 ABI](04-abi.md)
- 决策依据：[ADR-0002](../adr/0002-protocol-first-runtime-boundary.md)（协议优先 Runtime 边界；DSH 为第一个 Level-2 参考适配器；supersedes ADR-0001 §D2 部分）
- 基线：DeepSeek Harness 本机只读副本 `cd5ef8148158c3a752a658978873241fdf8e2bbc`，vendored
  Cordis 4.0.1（见 [来源登记](../research/sources.md)）

## 1. 为什么 DSH 是第一个 Level-2 参考适配器而不是 OS、也不是唯一 Runtime

DSH 的设计哲学（其文档原文）："不存在需要打补丁的特权内核"、"一切皆插件"、"模型可见即已
记录"（append-only session log 是真源）、能力以 seam（Definition / Provider / Consumer）表达
可替换性、profile / bundle / patch 逐层叠加。它的安全立场克制而诚实：未经审计、沙箱不保证
隔离、审批一次性且 turn 级、动态 Cordis 包是临时实验且 VM 不是安全边界、schedule 仅为
session-local 提醒、goal 恢复后需人工 re-arm。

这恰好把 FysunOS 的职责边界勾出来了：

| 重建版概念 | DSH 现状 | 结论 |
|---|---|---|
| 决策核心（loop、prompt、工具选择、继续/停止） | 完整 | **直接用** |
| Runtime（执行引擎及适配） | 完整：subagent 六种 provider、sandbox、subprocess、terminal、e2b、Web UI、sdk/acp | **直接用** |
| Resource | 有 seam（`ctx.llm`、`ctx.credentials`、`ctx.skills`、token meter），进程本地，凭据由 provider 持有 | 换 provider，接到 OS |
| Activity / Execution（跨会话、跨进程、跨 Runtime） | goal 同会话且需人工 re-arm；schedule session-local；Teams 身份是 Root SessionId、凭据是"exact live Agent" | **OS 拥有** |
| Capability 契约与可移植性 | seam 是进程内实现可替换；无跨 Runtime 契约、无环境匹配 | **OS 拥有** |
| 权威底线（Grant、ActionIntent/Effect、独立验证、secret 隔离、Approval、Policy） | 无，且明确不做 | **OS 拥有** |

把权威层放进 DSH 进程，会让它与模型驱动的工具执行、可自我修改的动态 Cordis 包同居——
Cordis 论文把"自演化 harness 运行中替换自身组件"列为动机场景。这与 F1–F5 直接冲突。
DSH 又处于 developer preview、明言破坏性变更、单语言 Node；把 OS 身份绑在它上面等于放弃
边缘 Runtime 的可能。

v0.5.0 通过 7 个 DSH 私有 seam / 扩展点接入（`ctx.credentials`、`ctx.approval`、
`ctx.sessionPersistence`、`ctx.subagents`、`tools/pre-execute` waterfall、`agent/pre-step`、
`sdk` profile）。这使 [01 §1.1](01-definition-and-principles.md) "同一能力在任意 Agent 实现中运行"
的承诺在实现路径上等价于"愿意实现 fysun bundle 的 Runtime"——而 Claude Code、Codex、Goose、
OpenCode、OpenClaw、Gemini CLI 都没有这些 seam，却都有 MCP 客户端。协议优先后，承诺变为可判定：
任何实现 Runtime ABI 最小面（MCP 客户端 + 最小生命周期 RPC）的 Runtime。

因此：**DSH = 第一个 Level-2 参考适配器；FysunOS = 权威 + 活动 + 能力；任何 MCP 兼容 harness
都可作为 Level-1 Runtime 接入。**

## 2. 接入形态：`fysun-dsh` 适配器

技术形态仍是叠在 `dsh-base` 之上的 bundle，配一个 `fysun` profile；组件按 Runtime ABI 三块归类：

```text
dsh --profile fysun
  = dsh-base
  + fysun-dsh 适配器
      (a) MCP 守卫路径
      │     DSH 的 MCP 客户端配置：fysund（fysun-mcp-guard）为唯一 MCP server
      │     （v0.5.0 的 fysun-tool-gate 取消）
      (b) Level-2 控制点适配器
      ├── fysun-credentials     替换 ctx.credentials provider（Secret / Provider 代理）
      ├── fysun-approval        替换 ctx.approval 应答方（Approval 应答 → deliver_input(approval_result)）
      ├── fysun-session-link    旁挂 ctx.sessionPersistence provider（session_link）
      ├── fysun-context         订阅 agent/pre-step（context_injection）
      ├── fysun-subagent        新增 ctx.subagents provider（subagent_delegation → Handoff）
      (c) 最小生命周期 RPC
      └── fysun-rpc             经 sdk profile 实现最小生命周期 RPC（OS → Runtime）
```

| 类别 | 组件 | 说明 |
|---|---|---|
| (a) MCP 守卫路径 | （无适配器组件） | DSH 把 `fysund` 配为唯一 MCP server；`tools/list` / `tools/call` 经 MCP 守卫；`fysun-tool-gate` **取消** |
| (b) Level-2 控制点适配器 | `fysun-credentials` | `ctx.credentials` provider 只返回 `SecretRef`；`ctx.llm` 的 Provider 请求经 daemon 代理，明文只在 daemon 注入 |
| | `fysun-approval` | 应答 DSH 的 `approval/request`：按 Grant 与 Policy 可自动应答的直接应答；需人工的返回 `needs_human(approval_id)` 并等待 `deliver_input(approval_result)`；**无策略即 fail-closed** |
| | `fysun-session-link` | 实现 `session_link` 控制点：提交 `session_id / execution_id / causation_id / fencing_epoch` 与会话水位；compaction 前后报告水位（`compaction_hook`） |
| | `fysun-context` | 实现 `context_injection` 控制点：在 `agent/pre-step` 注入 OS 装配的受控上下文视图，片段带 `label` |
| | `fysun-subagent` | 实现 `subagent_delegation` 控制点：委派产生 Handoff 与子 Execution（可仍由 DSH in-process / ACP / Codex provider 实际执行） |
| (c) 最小生命周期 RPC | `fysun-rpc` | `describe / start_execution / resume_execution / cancel / deliver_input / checkpoint / health` 的 DSH 实现；`checkpoint` 与 `fork` 控制点经 DSH 的 session fork / resume 实现 |

DSH 扩展点与 ABI 的对应：

| DSH 扩展点 | FysunOS 接入 | 对应 ABI |
|---|---|---|
| `tools/pre-execute → tools/execute → tools/post-execute`（waterfall） | **不再订阅**；工具经 MCP 守卫：`tools/list` 过滤工具面，`tools/call` pre / 执行 / post 对应 ActionIntent → Effect → Receipt | MCP 守卫（04 §2.1） |
| `approval/request`（waterfall） | 由 `fysun-approval` 应答；需人工时等待 `deliver_input(approval_result)`；无策略即 fail-closed | Approval / 最小 RPC |
| `ctx.credentials` seam | provider 只返回 `SecretRef`；Provider 请求经 daemon 代理 | `os.request_resource` |
| `ctx.sessionPersistence` seam | 旁挂 provider，提交关联字段与会话水位 | `session_link` / `os.report_checkpoint` |
| `ctx.subagents` provider | "由 FysunOS 调度"的委派实现，协作事实落 OS 账本 | `subagent_delegation` / `os.request_handoff` |
| `agent/pre-step`（waterfall） | 注入受控上下文视图（authorize-before-rank、显式损失、digest 绑定、`label`）；改写已领取消息 | `context_injection` / `os.request_context` |
| `ctx.sandbox` seam | 视为 containment，不视为安全边界；Effect 协议不因沙箱而放松 | — |
| `dsh --profile sdk` / `acp` | 进程外驱动通道；`fysund` 以子进程方式启动 DSH，经 JSON-RPC 调用最小生命周期 RPC | 最小生命周期 RPC（04 §2.2） |

`fysun-dsh` 在 `runtime.describe()` 中声明的 `control_points[]`：`context_injection`、`checkpoint`、
`fork`、`compaction_hook`、`session_link`、`subagent_delegation`（六个全部）。

DSH 保持原样运行其 loop、prompt、compaction、session log、Web UI。FysunOS 不改写 DSH 源码；
升级 DSH 时维护的是这个适配器与一组适配测试。

## 3. 双账本关联

```text
DSH session log（会话账本）                fysund 治理账本
  session/start ─── session_id ─────────▶ execution/started {session_id}
  tool/call ──────── candidate ─────────▶ action-intent/recorded → effect/authorized
  tool/result ◀───── receipt ───────────  effect/receipt
  approval/request ─ waterfall ─────────▶ approval/requested … approval/granted → deliver_input(approval_result)
  step/end ───────── watermark ─────────▶ execution/checkpointed {session_watermark}
  fork ───────────── parentSession ─────▶ execution/forked
  subagent spawn ─── delegation ────────▶ handoff/submitted + execution/started(child)
```

- DSH 的 session log 仍是模型所见与所答的真源；OS 不复制消息正文。
- OS 的账本是治理事实的真源；DSH 不能推进 Execution 状态，也不能决定 Approval。
- 跨账本 replay 测试：从两侧任一账本出发都能定位另一侧，并且 Effect 与 tool/result 一一配对
  （见 [03 §4](03-architecture.md#4-双账本)）。

## 4. 明确不从 DSH 吸收为 OS 语义的东西

- 把 Session 当全局真源（OS 有自己的 Activity/Execution）；
- 一次性 turn 级审批当 Grant（OS 的 Grant 有 principal/scope/budget/expiry/revocation）；
  也不当 Approval（OS 的 Approval 有 subject/options/expiry/derived）；
- session-local schedule 当调度（OS 的 Activity triggers 跨进程持久）；
- 动态 Cordis 包（`cordis_define/run`）当持久扩展——它只能是 `ephemeral-experiment`，
  不能直接生成 durable Capability 实现；
- Agent Teams 的 "exact live Agent" 对象身份当授权凭据（OS 使用 Execution 作为 Principal）；
- DSH 的 skills 加载格式当实现包格式——若与 Agent Skills 规范不同，以 Agent Skills 为准；
  `Capability.implementation` 只引用 Agent Skills 目录；
- DSH 内部工具注册表当能力目录——OS 的工具面由 MCP 守卫的 `tools/list` 决定。

## 5. 版本与供应链

- DSH 以**精确版本 + digest** 钉住，作为 OS 管理的 Installed Runtime 登记（版本、来源、健康、
  digest 可见）。
- 破坏性变更时先在隔离环境跑适配测试，再升级；适配器与 DSH 版本一一对应。
- Agent Skills 包以**来源登记 + digest + `allowed-tools` 上界**登记；DSH 侧加载的 skills 目录不
  绕过这一登记（供应链威胁见 [09 威胁模型](09-threat-model.md) T2）。
- DSH 与 Cordis 均为 MIT；FysunOS 不 vendor、不 fork。

## 6. 其他 Runtime

- **Level-1 直接接入**：Claude Code / Codex / Goose / OpenCode / OpenClaw / Gemini CLI 等，只要有
  MCP 客户端即可——把 `fysund` 配为其 MCP server，工具面与工具调用即受 MCP 守卫；最小生命周期
  RPC 可由 Runtime 本体实现，也可由一个薄包装进程提供（负责拉起 / 恢复 / 取消该 harness 并转发
  `deliver_input`）。
- 它们也可继续作为 DSH 内的委派目标（`fysun-subagent` 经 DSH 的 Codex / Claude Code / ACP
  provider 执行），此时被 OS 视为 Handoff 的执行载体，其内部状态留在远端进程，OS 只记录 Handoff
  与产物。
- **边缘 Runtime**（Rust/Python）：只需 MCP 客户端 + 最小生命周期 RPC；决策核心可以是本地小模型、
  规则模块或远端调用；`checkpoint()` 可返回 `unsupported`。
- 每个 Runtime 独立资格化（`runtime/qualified{control_points[]}`）；DSH 的证据不可转移到其他 Runtime。

## 7. 代价与对策

| 代价 | 对策 |
|---|---|
| 双账本的关联与 replay 复杂度 | 关联字段固定；跨账本 replay 测试进入 MVP 验收 |
| 进程外 broker 延迟 | 只读操作走审计快路径；Effect 协议只在 mutation 上 |
| MCP 守卫引入的每次 `tools/call` 往返 | 只读工具走审计快路径；同类候选批量提交与批量审批（`batch_ref`） |
| agentgateway 复用形态未定 | 无论哪种形态采用相同 Pass / Reject / Mutated 语义与 Cedar 策略格式（[08 开放问题 10](08-mvp-plan.md#7-开放问题待后续-adr)） |
| DSH 破坏性变更 | 精确钉版；适配测试；升级前隔离验证 |
| Node 依赖不适合部分边缘设备 | 这正是 OS ≠ DSH 的理由；边缘 Runtime 只需 MCP 客户端 + 最小 RPC |
