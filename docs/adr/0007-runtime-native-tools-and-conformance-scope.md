# ADR-0007 · Runtime 原生工具、资格化与 conformance 范围

- 状态：accepted
- 日期：2026-09-08
- 决策者：owner
- 背景来源：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md) §I.1 决策 5、6、9、12，§I.2 H-3，§0 G1/G2/G3/G6、C2-7；外部调研：DeepSeek Harness `docs/subsystems/scope.md`（scope 过滤）、`docs/tool-execution-pipeline.md`（能力位预检）、Claude Code / Codex hooks.json PreToolUse 文档
- supersedes：无（新增）。ADR-0002 §D1 的 `describe()` 返回形状与 §D5 的 Runtime 归类由本 ADR 扩展；ADR-0002 正文已于 v0.7.0 同步修订。

## 背景

ADR-0002 把 Runtime 边界定为"MCP 守卫 + 最小生命周期 RPC + 可选控制点"，并把只实现最小面的 Runtime 速记为 Level-1。
v0.6.0 文档在此之上有四处空白（蓝图 §0 G1/G2/G3/G6；以下篇章编号为 v0.6.0 旧编号）：

- **G1**：旧 09 §6.1（现 13 §6）的 conformance 表对 F3（外部效果先记录再执行）、F5（secret 隔离）、F7（能力不自扩权）、F9（权限单调收窄）一律标"满足"，
  不区分 Runtime 类型。但 Claude Code、Codex、Goose、OpenCode、DSH（含 headless 模式）都自带文件系统、shell、网络等**不经 MCP 的原生工具**，
  这些工具产生的外部效果根本不经过 MCP 守卫；对它们宣称 F3/F7/F9 "满足"是过度声明。
- **G2**：01 §4 的 F5 表述"请求经 daemon 代理"默认对所有 Runtime 成立。现实是 MCP-only harness 自持模型凭据、直连 Provider，`fysund` 看不到这些请求。
- **G3**：旧 04 §2.2（现 05 §2）的 `start_execution(context_view, …)` 与 `deliver_input(approval_result | message | …)` 是 OS → Runtime 的推送调用，
  但一个只有 MCP 客户端的 harness 没有任何"接收推送"的入口；文档没有说明 context_view 如何进入模型输入、Approval 决定如何送达。
- **G6**：已安装 Runtime 的登记只在旧 05 §5 出现且专指 DSH；其他 Runtime 如何登记、如何资格化、何时失效，没有定义。

同时，C2-7 指出 03 §1、旧 05 §6（现 07 §3）、旧 08 M2a（现 12 §3）把 Claude Code / Codex / Goose / OpenCode / DSH headless 并列为 Level-1 例子，
而 M2a 验收写成"任选现成 harness"，缺少可判定基线。

外部调研提供了可借用的形状：DeepSeek Harness 的工具流水线在工具体执行前对能力位做预检并**大声失败**（不降级）；其 scope 机制过滤工具面时
使被过滤工具对模型不可区分；其 `guard/` 包是单调的 loop 卫生层而不是安全守卫。Claude Code 与 Codex 的 hooks.json `PreToolUse` command hook
提供了一条对原生工具**只能阻断、不能记账**的路径：命令退出码 2 即阻断本次工具调用。

owner 决策（蓝图 §I.1 决策 5、6、9、12）：要求 Runtime 如实声明原生工具；Policy 据此拒绝资格化或降级 conformance；conformance 表按 Runtime
类型分列；hooks 路径作为"Level-1b，仅阻断，不计入 conformance 主路径"的补充；F5 范围限定；M2a 基线固定为 DSH headless；Runtime 推广为通用
Resource 类型并引入资格化纪元。本 ADR 记录这些决策的可判定形式。

## 决策

### D1 · `describe()` 必含 `native_effect_tools[]` 与 `describe_digest`

- 最小生命周期 RPC 的 `describe()` 返回形状定为
  **`describe() -> {type, version, control_points[], native_effect_tools[], describe_digest}`**。
  这一签名在 [00 §7](../design/00-terminology.md)、05 §2、07 §1、ADR-0002 §D1 与本 ADR 五处字面一致。
- `native_effect_tools[]`：Runtime **内置、不经 MCP 守卫**、可产生外部效果（文件写入、shell、网络、凭据使用等）的工具清单。
  每项至少含工具名与缺省 `effect_category`（[ADR-0008](0008-effect-category-and-read-only-determination.md)）。
  只读的原生工具（如本地文件读取）可以不列，但列出不算错。
- `describe_digest`：对 `{type, version, control_points[], native_effect_tools[]}` canonical 编码的 digest，是资格化纪元（D7）的基准。
- 声明责任在 Runtime（或其薄包装进程、适配器）：**声明不实的 Runtime 不在 FysunOS 的承诺范围**——这是 01 §1.1 可判定限定的一部分，
  与 ADR-0002 §D1 "对不可控制的内部机制，Runtime 必须如实声明"同一条原则。
- `fysun-dsh` 的 `describe()` 必须如实列出 DSH 原生工具（fs、shell 等）于 `native_effect_tools[]`，**不为空**（蓝图 §I.2 H-3 选项 B：
  保留 DSH 原生工具，经 `approval/request` 拦截路径治理，见 ADR-0002 §D5 与 [08](../design/08-dsh-adapter.md) §2）。
- 负例：`describe()` 返回缺 `native_effect_tools` 字段（而非空数组）却产生 `runtime/qualified` → 失败；`describe()` 缺 `describe_digest`
  → 不得 `runtime/qualified`。

### D2 · Policy 可据 `native_effect_tools[]` 拒绝资格化或降级 conformance

- `runtime.qualify(resource_id)` 流程：调用 `describe()` → 把 `native_effect_tools[]` 与 `control_points[]` 送入 Policy 评估
  （context 组新增 `native_effect_tools: Set<String>`，[ADR-0009](0009-cedar-subset-and-entity-model.md) §D1）→
  `allow` 则写 `runtime/qualified{control_points[], native_effect_tools[], describe_digest, bundle_digest}`；`deny` 则拒绝并返回原因，
  不产生 `runtime/qualified`。资格化只接受 `allow` / `deny`：`Approval.subject` 是封闭集合、没有"Runtime 资格化"一项，Policy 对
  `Action::"runtime/qualify"` 产出 `approve` 或 `modify` 时一律按 `deny` 处理（fail-closed）并在 `rejected(reason)` 中注明；人工把关的方式是先不放行、
  确认后换版 Policy 再 `runtime.qualify`（07 §1.2、04 §1.6）。
- Policy 有两种处置：(a) **拒绝资格化**——含特定原生工具（如无沙箱的 shell）的 Runtime 在该 Scope 内不允许运行；
  (b) **允许但降级 conformance**——Runtime 可运行，但 conformance 声明中 F3/F7/F9 的范围如实限定为"经 MCP 守卫的工具面"。
- [13](../design/13-threat-model.md) §6 的 conformance 表按 Runtime 类型分四列：声明全部六个控制点（`fysun-dsh`）/ MCP-only（含
  `native_effect_tools`）/ MCP-only + hooks 阻断路径（速记 Level-1b）/ 边缘 Runtime。
  状态用语只用 `specified / implementation available / test executed / implemented` 四类，不再出现"满足 / 部分"。
- MCP-only 列的 F3/F7/F9 注明"限经守卫的 MCP 工具面；原生工具不在范围"；`fysun-dsh` 列注明"MCP 工具面经守卫；DSH 原生工具经
  `approval/request` 拦截 + `tools/post-execute` receipt 路径；依赖 `fysun` profile 禁止 DSH approval policy `never`"。
- Capability `requirements` 可声明"拒绝含特定 `native_effect_tools` 的 Runtime"；Capability 子系统的环境匹配据此过滤。
- 负例：一个 Runtime 的 `native_effect_tools[]` 非空，而 conformance 声明对其 F3 标为 `implemented` 且未限定范围 → 失败（文档一致性检查）。

### D3 · hooks.json PreToolUse 阻断路径 = Level-1b，只能 deny / abstain，不计入 conformance 主路径

- 对支持 hooks.json（Claude Code / Codex / DSH 兼容格式）的 harness，部署者可配置 `PreToolUse` command hook 指向 CLI 子命令
  **`fysun hook pre-tool-use`**：命令读取 harness 传入的工具调用载荷（工具名、参数）→ 调 `fysund` 本地 RPC → Policy 评估 →
  `deny` 时退出码 2（harness 阻断本次调用），其余退出码 0（放行 = abstain）。
- 这条路径的能力边界：**只能 deny 或 abstain**。它不持久化 ActionIntent、不产生 Effect、没有 receipt、不打来源标签、不能 `modify`
  参数、不能触发 Approval（harness 的 hook 是同步阻断，无等待语义）。
- 账本记录：deny → `action-intent/rejected{reason: denied.policy, path: hook}`（准入类，带 `policy_decision`）；abstain →
  `audit/observed{path: hook, verdict: abstain, …}`（非准入类）。除此之外 hooks 路径**不得**产生任何其他准入类事件。
- 速记 **Level-1b** = MCP 客户端 + 最小生命周期 RPC + hooks.json PreToolUse 阻断。它是叙述性速记，与 Level-1 / Level-2 一样禁止进入任何
  标识符（[00 §1](../design/00-terminology.md)）。
- conformance：hooks 路径**不计入主路径**。13 §6 的"MCP-only + hooks"列只能把 hooks 的贡献描述为"对原生工具的 Policy 阻断（尽力）"，
  不得据此把 F3/F7/F9 的范围扩大到原生工具。理由：hook 由 harness 进程调用，harness 可被配置绕过、hook 失败时 harness 的缺省行为不受
  `fysund` 控制、且 hook 看不到工具结果。
- 负例：hooks 路径的 deny 被记为 `action-intent/rejected` 以外的准入类事件（如 `effect/authorized`）→ 失败；设计文档或 conformance 声明
  把 hooks 路径描述为"安全边界"或据此对原生工具宣称 F3 → 失败；`fysun hook pre-tool-use` 在 `fysund` 不可达时退出码为 0 → 失败
  （必须 fail-closed，退出码 2）。

### D4 · F5 在 MCP-only Runtime 下的范围限定与部署指引

- F5 "secret 隔离"的范围定为：**OS 管理的 secret 与经 `fysund` 代理的 Provider**。在此范围内 F5 的表述不变：明文只进 Secret Store 与
  批准的非日志输入路径，Runtime 只见 `SecretRef`。
- harness **自持的模型凭据**（其配置文件或环境变量中的 API key，直连 Provider）不在 F5 范围；[13](../design/13-threat-model.md) §4
  "明确不防"登记"harness 自持模型凭据外泄"。
- 部署指引（07 §5）：把 harness 的 OpenAI 兼容 base URL 指向 `fysund` 的 Provider 代理端点，使模型请求进入 F5 范围并计量；
  这是部署者选择，不是资格化前提。
- 00 §6 底线表 F5 名称改为"secret 隔离（范围：OS 管理的 secret 与经 `fysund` 代理的 Provider）"。
- 负例：设计文档对 MCP-only Runtime 宣称"所有模型请求经 `fysund` 代理"而未附部署前提 → 失败。

### D5 · MCP-only Runtime 的 context_view 渲染与 `deliver_input` 投递路径

MCP-only harness 没有接收 OS 推送的入口；以下路径定义 `start_execution` 与 `deliver_input` 在此类 Runtime 下如何落地（详见 07 §3）：

- **context_view 渲染**：薄包装进程（实现最小生命周期 RPC 的一方）在 `start_execution` 时把 context_view 渲染进 harness 的启动输入
  （system prompt 片段、初始用户消息或等价入口，按 harness 而异）；同时 `fysund` 以 MCP `resources/read`（URI
  `fysun://context/<execution_id>`）与 `prompts/get` 暴露同一视图，供 harness 按需再读。`context/assembled{labels_digest}` 照常记录。
  conformance 声明"**仅初始投递，不保证每步刷新**"——每步刷新需要 `context_injection` 控制点。
- **`deliver_input(approval_result)`**：优先路径为 MCP Tasks（若 harness 支持 `tasks/*`，`needs_human` 的 `tools/call` 以 task 形式
  挂起，决定到达后 `tasks/result` 返回）；否则守卫**阻塞** `tools/call` 直到决定到达或 Policy `approval_wait_timeout` 超时；超时返回
  `needs_human(approval_id)` 错误，harness 可调用守卫系统工具 `fysun.await_approval(approval_id)`（`local_read`，长轮询）继续等待。
  决定到达而无开放通道时，Effect 停在 `authorized` 直到 Runtime 重新提交同幂等键候选（命中 `action-intent/replayed`）或轮询，受 Policy
  `approval_result_ttl` 约束，超期按 Approval `on_expiry` 处理。
- **`deliver_input(message | event | handoff_result)`**：由薄包装进程在 harness 的 turn 边界投递（作为下一条用户消息），或由 harness 经守卫
  系统工具 `fysun.pending_inputs()`（`local_read`）主动拉取。conformance 声明"**尽力、下一 turn 边界**"。
- 守卫系统工具 `fysun.await_approval` / `fysun.pending_inputs` 出现在 `tools/list`，不受 Capability `allowed-tools` 限制但受 Policy 控制
  （[06](../design/06-mcp-guard.md) §7）。
- 负例：MCP-only Runtime 的 conformance 声明含"每步上下文注入"或"实时投递" → 失败；`approval_wait_timeout` 超时后守卫返回成功而非
  `needs_human` → 失败。

### D6 · `start_execution` 大声失败，不得降级运行

以下任一条件成立时 `start_execution` 必须返回 `not_runnable.environment{missing[]}` 并**不启动** Runtime 会话：

- Capability `requirements.control_points` ⊄ Runtime 已资格化的 `control_points[]`；
- Runtime 没有当前有效的 `runtime/qualified`（从未资格化，或已 `runtime/disqualified` 且未重新资格化）；
- Policy 在本 Scope 下拒绝该 Runtime 的 `native_effect_tools[]`；
- 适配器前置条件不满足（如 `fysun-dsh` 检测到 DSH approval policy 为 `never`、bundle 的 `dsh_version` / `schema_digest` 不匹配）。

`missing[]` 列出未满足项。借鉴 DeepSeek Harness 能力位预检的做法：预检失败即终止，不"尽力运行再看"。

- 负例：缺 required 控制点的 Runtime 被 `start_execution` 成功启动并以"部分支持"运行 → 失败；已 `disqualified` 的 Runtime
  `start_execution` 成功 → 失败。

### D7 · Runtime 为通用 Resource 类型；资格化纪元与 `runtime/disqualified`

- 已安装 Runtime 是 Resource 的一种类型"Runtime（已安装）"（[02 §2.9](../design/02-object-model.md)），从 v0.6.0 只针对 DSH 的"已安装 Runtime"
  登记项推广为通用类型。登记字段：类型、版本、来源、`bundle_digest`（安装包 / bundle 的 digest；无 bundle 的 harness 记其可执行文件或
  安装清单 digest）、`describe()` 快照（`control_points[]`、`native_effect_tools[]`、`describe_digest`）、资格化纪元。
- Capability API：`runtime.list()` 返回已登记 Runtime 及资格化状态；`runtime.qualify(resource_id) -> qualified{control_points[],
  native_effect_tools[]} | rejected(reason)`。
- **资格化纪元**：一次 `runtime/qualified` 对应一个 `(describe_digest, bundle_digest)` 组合。任一 digest 变化（Runtime 升级、配置改变、
  bundle 替换）、Policy 撤销该 Runtime 的允许、或适配器前置条件失效 → 写 `runtime/disqualified{resource_id, reason: describe_changed |
  bundle_changed | policy_revoked | adapter_precondition_failed}`；此后 `start_execution` 按 D6 拒绝，直到重新 `runtime.qualify`。
- 正在运行的 Execution 不因 `runtime/disqualified` 被强制终止；是否让其继续、暂停或取消由 Activity 子系统按 Policy 决定，
  但新的 `start_execution` / `resume_execution` 一律按 D6 拒绝。
- 多个 Runtime 并存（同类型不同版本、不同类型）按 Resource id 区分；资格化证据不跨 Resource id 转移（延续 ADR-0002 后果段）。
- 负例：`describe_digest` 变化后账本无 `runtime/disqualified` 而 `start_execution` 成功 → 失败；`runtime/qualified` 缺
  `native_effect_tools[]` 或 `describe_digest` → 失败。

### D8 · M2a 基线 = DSH headless 模式（登记查证项）

- M2a "MCP 守卫 + MCP-only 接入"的验收基线固定为 **DSH headless 模式**（`dsh --profile headless` 或等价），不再"任选现成 harness"。
  理由：与 M2b 的 `fysun-dsh` 共享同一上游钉版与 CI，避免为 M2a 引入第二个上游的漂移成本。
- 登记查证项（12 §7 开放问题）："需查证 DSH headless 是否有独立于 bundle 的原生工具"；查证结果决定 M2a 中 `native_effect_tools[]` 的
  实际内容，不影响本 ADR 的规则。
- Claude Code / Codex / Goose / OpenCode / OpenClaw / Gemini CLI 列为**候选**，标注"均含原生工具，接入时须按 D1 声明"；不作为 M2a 验收对象。

## 否决的替代方案

- **把原生工具视为已受控（维持 v0.6.0 的一律"满足"）。**这是过度声明：原生工具的外部效果不经守卫，F3/F7/F9 对其没有任何机制作用；
  与 F6（契约与负例不为实现让路）和 ADR-0002 "如实声明"原则直接冲突。
- **要求 harness 禁用全部原生工具作为资格化前提。**多数 harness 的原生工具不可完全禁用，或禁用后失去实用价值；这等于把"任意 Runtime"
  再次收窄为"愿意改造自己的 Runtime"，重蹈 v0.5.0 的覆辙。正确做法是如实声明 + Policy 决定 + conformance 分列。
- **以 hooks.json 作为 conformance 主路径。**hook 由 harness 调用、可被绕过、不能记账、看不到结果；把它当作主路径会让 F3（先记录再执行）
  的"记录"依赖非权威进程。只能作为补充阻断。
- **为原生工具伪造 Effect 记录（hook 阻断时同时写 `effect/authorized`）。**没有 receipt、没有 dispatch 控制的"Effect"违背 Effect 状态机
  语义，会污染对账与恢复。

## 后果

- 13 §6 conformance 表重写为按 Runtime 类型分列的四列表，状态用语四类。
- M2a 验收增项：被过滤工具错误形状不可区分；`start_execution` 缺控制点 / 未资格化大声失败；context_view 初始投递可见；
  `needs_human` 阻塞 → 超时 → `fysun.await_approval` 路径通过；`describe_digest` 变化 → `runtime/disqualified`；hooks 路径 deny 退出码 2 且
  账本 `audit/observed{path: hook}` 或 `action-intent/rejected`（可选项，不阻塞 M2a）。
- MCP-only Runtime 的 conformance 范围收窄并如实声明；Approval 触发率在此类 Runtime 下更高（原生工具不受控 → Policy 倾向更保守），
  是可接受代价。
- 事件族新增 `runtime/disqualified`、`audit/observed`；`runtime/qualified` 增三字段；错误码 `not_runnable.environment` 触发条件扩展；
  Capability API 新增 `runtime.list()`、`runtime.qualify()`；CLI 新增 `fysun hook` 子命令；守卫新增系统工具 `fysun.await_approval`、
  `fysun.pending_inputs`。
- 07 篇新建，容纳 Level-1 投递路径、hooks 路径、通用钉版与 CI 漂移预警；08 篇专述 `fysun-dsh`。

## 待后续 ADR 的问题

- DSH headless 原生工具清单的查证结果（12 §7）。
- MCP server card（SEP-1649 / SEP-1960）稳定后是否用于 Runtime 能力发现替代或补充 `describe()`（沿 ADR-0002）。
- 完整信息流控制（Level-3 控制点）继续开放（12 §7）。
- 多个 hooks 路径同时存在（harness 自带 + `fysun hook`）时的先后与冲突处置，待有第二个实例再定。

其余见 [12 MVP 计划 §7](../design/12-mvp-plan.md)。

## 修订历史

- v0.7.0（2026-09-08，WP6 收尾）：§D2 补"资格化只接受 `allow` / `deny`，Policy 产出 `approve` / `modify` 一律按 `deny` 处理（fail-closed）"。
  为什么：WP3 在 07 §1.2 依据 `Approval.subject` 封闭集（无"Runtime 资格化"项）作出该裁定，04 §1.6 与本 ADR 同步以保持三处字面一致。
