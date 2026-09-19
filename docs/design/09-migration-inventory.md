# 09 · 迁移清单

- 状态：设计稿 v0.7.0（2026-09-08）
- 变更依据：[v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md)；[ADR-0001](../adr/0001-founding-decisions.md) ~ [ADR-0006](../adr/0006-terminology-action-intent.md)（v0.7.0 修订）与 [ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) ~ [ADR-0012](../adr/0012-mcp-guard-self-implementation.md)
- 前置：[00 术语表](00-terminology.md)、[02 对象模型](02-object-model.md)
- 基线：CognitiveOS `agent-kernel@6f562bf1bdb86582955de36eb93e51126573f89c`（分支
  `personal/P14-T04-member-join`；公开仓库 `agentkernel/cognitive-os` 的对应 revision 待查证）；DSH `cd5ef8148158c3a752a658978873241fdf8e2bbc`；
  Cordis：上游 cordis 4.0.0-rc.7（DSH vendored；DSH 清单版本 4.0.1）；上游最新 v4.0.0-rc.9。见 [来源登记](../research/sources.md)。

FysunOS 不是 CognitiveOS 的原地重构。迁移原则：

1. **迁语义与经验证的机制，不迁目录结构。** 迁入的代码要按 FysunOS 对象模型改名与重组。
2. **凡 DSH 已有且不涉及权威的，不迁。** Personal 自研的 Runtime 片段整体放弃。
3. **业务对象回到应用层。** Project / Member / Routine / 数字员工在 FysunOS 上作为应用实现。
4. **合同与负例迁入时不得削弱。** 变更需在 FysunOS 的正式流程中重新登记。
5. **有标准的部分用标准，不迁自定义等价物。** 工具面走 MCP，外部委派走 A2A，实现包走 Agent Skills。

本篇中标注"一手资料确认"的条目，是 v0.7.0 重构时对上游仓库原文核对后的结论；核对时未能确定精确 revision 的写"待查证"，不臆造。

## 1. 从 CognitiveOS core 迁入

| 迁入 | 参考位置（迁移时按实际代码审阅） | 落到 FysunOS |
|---|---|---|
| ActionIntent / Effect 协议：persist-before-dispatch、幂等键（canonical 参数 digest）、fencing epoch、`OUTCOME_UNKNOWN` 只能 reconcile 或 quarantine。CognitiveOS 中的效果级请求记录名为 `Intent`，迁入后改名 `ActionIntent`（[ADR-0006](../adr/0006-terminology-action-intent.md)），本行保留对照供代码比对 | `core/crates/cognitive-kernel/src/effects.rs` | Authority 子系统（[02 §2.6](02-object-model.md)） |
| Effect 错误语义（一手资料确认，`effects.rs` 七属性模型）：**同一幂等键、不同参数 = 幂等冲突，硬拒绝**，不去重；**补偿需要新的授权 Grant 与新的请求记录，拒绝复用原 Grant** | `core/crates/cognitive-kernel/src/effects.rs` | 02 §2.6 错误语义、[06 §4](06-mcp-guard.md)；错误码 `conflict.idempotency`（00 §8）；[ADR-0008](../adr/0008-effect-category-and-read-only-determination.md) §D6 / §D7 |
| 确定性 replay 与投影 digest 稳定性 | `core/crates/cognitive-kernel/src/replay.rs` | 治理账本（改为 event-sourced 唯一真源，[03 §5](03-architecture.md)） |
| 恢复固定顺序与 checkpoint epoch 校验 | `core/crates/cognitive-kernel/src/recovery.rs` | Activity 子系统恢复模型（补 Approval 水位与重新武装，[03 §6](03-architecture.md)） |
| 上下文 authorize-before-rank、预算过滤、显式损失、digest 绑定；上下文条目信任等级 `LoadedContextItemTrustLevel {Control, Authoritative, Verified, Untrusted}` 与 `UntrustedInput` 分区角色（一手资料确认：只覆盖上下文装配阶段，未沿参数流传播到 Effect） | `core/crates/cognitive-kernel/src/context.rs` | Resource 子系统上下文装配服务（补来源标签赋予）；信任等级只作为 `label` 的上下文层先例，取值集不直接迁入（见下段） |
| Capability Grant 衰减、Lease / CAS 围栏语义 | `cognitive-kernel` / `cognitive-domain` | Grant 对象（02 §2.5） |
| `SecretRef` 与 Secret Store 边界；用户指向的凭据导入边界 | AXIOMS A5 / ADR-0055 | Resource 子系统 Secret Store（补 SecretRef 三规则，02 §2.9） |
| 负例向量与符合性 runner 的方法论（向量与实现分离、`not-run` 永不等于 pass）；**四类状态用语 `specified / implementation available / test executed / implemented`**（一手资料确认：README 与 `core/conformance/README.md`） | `core/conformance/`、`core/tests/golden/`、`core/conformance/README.md` | FysunOS conformance（重新登记，不直接复制向量内容）；状态用语进入 [12 §5](12-mvp-plan.md) 证据纪律与 [13 §6](13-threat-model.md) conformance 表 |
| AKP 入站事件的 `authority_claim` 与 `secret_shaped` 标志（一手资料确认） | `core/crates/cognitive-akp/src/deepseek_harness.rs` | `label.flags`（02 §2.15）；MCP 守卫 post 阶段对工具结果分类（06 §4）；AKP 协议本身不迁（见下） |
| 公理 A1–A6、A8 | `docs/governance/AXIOMS.md` | 底线 F1–F6、F8（[01 §4](01-definition-and-principles.md)） |

**来源标签：部分先例、扩展新建。** 来源 / 完整性标签 `label` 在 CognitiveOS 有**上下文层部分先例**——`cognitive-kernel/src/context.rs` 的
`LoadedContextItemTrustLevel {Control, Authoritative, Verified, Untrusted}` 在上下文装配阶段给每个条目分级，并以 `UntrustedInput` 分区角色隔离不可信输入；
但它**仅覆盖上下文装配，未沿参数流传播到 Effect**，也不为工具结果与 Artifact 打标签。FysunOS 在其上**扩展新建**（[02 §2.15](02-object-model.md)、
[ADR-0004](../adr/0004-provenance-labels-and-monotonic-privilege.md)）：标签沿参数流进入 Effect 候选、工具结果一律 `untrusted_external`、增加 `flags`、
Policy 可引用标签。**Approval、Policy 版本化（Cedar 子集）、Memory 记录**在 CognitiveOS 中无对应机制，不是迁移项，按 02 新建（清单见 §6）。

**不迁**：双内核 / 七层 / 三平面白皮书结构；六族资源作为"必修课"；企业多租户与联邦默认路径；
现有 `core/specs` 的全部 schema 原样（按新对象模型重新建模）；AKP 作为唯一适配协议的地位
（被 Runtime ABI 取代：MCP 守卫 + 最小生命周期 RPC；可借鉴其 envelope 设计与入站标志）；
`cognitive-akp/src/harness.rs` 的 `LoopDriver` 把 loop 边界放在内核内（一手资料确认）——FysunOS **不照搬**，loop 属 Runtime 会话账本，
OS 不持有 Loop 状态（[01 §6](01-definition-and-principles.md)、02 §3、[ADR-0001](../adr/0001-founding-decisions.md) §D2）。

## 2. 从 CognitiveOS Personal 迁入

| 迁入 | 参考位置 | 落到 FysunOS |
|---|---|---|
| Rust daemon 作为唯一权威写入者 + SQLite WAL 存储 | `personal/apps/kernel-server`、`personal/crates/cognitive-store` | `fysund` |
| 独立 Verification 执行器（不接受 Agent 或 executor 自报完成） | `personal/apps/kernel-server/src/personal/verification_executor.rs` | Authority 子系统 Verifier（补 `verifier_kind` / `independence` 记录，02 §2.7） |
| Provider 代理（凭据只在 daemon 注入） | Personal Provider proxy | Resource 子系统；只覆盖经 `fysund` 代理的请求，harness 自持凭据不在 F5 范围（[07 §5](07-runtime-onboarding.md)） |
| DSH 进程外 stdio broker 的经验（含 stale bearer 等故障记录） | `personal/crates/cognitive-runtime/src/hosted_dsh_broker.rs`、`docs/bug/dsh-*.md` | `fysun-rpc` 与 Runtime 管理（[08 §2](08-dsh-adapter.md)） |
| 动态工具生命周期（描述符、可用性、隔离） | `personal/crates/cognitive-runtime/src/dynamic_tool_ecosystem.rs` | Resource 目录 + `resource/*` 生命周期事件（`registered / bound / unavailable / quarantined`）；目录可登记工具缺省 `effect_category`（06 §3） |
| SQLite 编号迁移机制（只追加不改写） | handbook `store-and-migrations` | 治理账本 schema 管理 |
| Control Plane 中"活动 / 证据 / 审计 / Installed Agent"视图的产品经验 | ADR-0056 / ADR-0059 | Web UI（daemon 静态服务）；Approval 呈现面为第一优先级，且审批面只由 `fysund` 提供（02 §2.11） |
| 安装、user-service、daemon status 的运维经验 | `personal/deploy`、handbook | `fysun` CLI 与部署 |

**不迁**：Pi adapter；conversation projection（ADR-0058 的 Path B transcript）；自研 loop / 工具 /
会话片段；Project / Member / Routine / Conversation / Vault 作为 OS 对象（回到应用层）；
调度服务中与 Routine 耦合的部分（Activity triggers 重新实现）。

## 3. 从 DeepSeek Harness 复用（不迁入 OS；作为 Runtime 直接使用或经 `fysun-dsh` 适配）

| 直接使用 | 说明 |
|---|---|
| agent loop、system-prompt 装配、tools 执行流水线 | 决策核心。MCP 工具面经 MCP 守卫的 `tools/list` / `tools/call` 提供（[06](06-mcp-guard.md)）；DSH 原生 fs / shell 等工具保留，经 `approval/request` waterfall 拦截 + `tools/post-execute` receipt 路径治理（[08 §2](08-dsh-adapter.md)），并如实列入 `describe()` 的 `native_effect_tools[]` |
| session log、fork / resume / compaction、crash repair | Runtime 会话账本 |
| subagent 六种 provider（in-process spawn / fork、ACP、Codex、Claude Code、dsh-sdk） | 委派执行载体；其中 Codex / Claude Code 等亦可作为 MCP-only Runtime **候选**直接接入（[07 §3](07-runtime-onboarding.md)）——候选 MCP-only Runtime 均含原生工具，接入时须按 [ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) §D1 如实声明 `native_effect_tools[]` |
| sandbox / subprocess / terminal / e2b / fs seam | 执行环境 |
| skills（以 Agent Skills 规范为准）、plan mode、goals（同会话）、jobs、workflow 脚本 | 决策核心辅助能力 |
| Web 对话 UI、sdk / acp / headless profile | 入口与驱动通道。headless 模式是 M2a 的 MCP-only 基线（[12 §3](12-mvp-plan.md)；查证项"DSH headless 是否有独立于 bundle 的原生工具"登记于 12 §7）。DSH 自身的审批卡片在 `fysun` profile 下失效，审批面只由 `fysund` 提供（02 §2.11） |

| 作为设计参考吸收（不复制代码） | 落到 FysunOS |
|---|---|
| "模型可见即已记录"不变量 | 写入 Runtime ABI 的 `session_link` 控制点义务（[05 §6](05-runtime-abi.md)） |
| seam 三角（Definition / Provider / Consumer） | Resource 与适配器的可替换性表达 |
| profile / bundle / patch 逐层叠加 | `fysun-dsh` 适配器的组织方式（08 §1） |
| Agent Teams 的 roster / mailbox（queued-minus-delivered）/ task DAG（revision CAS、`blockedBy` 无环、advisory writeScopes）；DSH 标注为实验特性（一手资料确认：`docs/subsystems/agent-team.md`） | **收窄**：只有 Handoff 状态迁移 + `deliverables` 进入治理账本；roster / mailbox / task DAG 留在 Runtime 会话账本，经 `session_link` 关联（02 §2.8、02 §3） |
| Ralph 交接（有界结构化报告：状态、摘要、证据、后续步骤、阻塞） | `Handoff.deliverables[]` 与 `Activity.continuity` 的形态 |
| Session 投影缓存的"冷读取阶梯"（缓存行 + 尾部回放） | 治理账本投影缓存策略 |
| scope 过滤的不可区分性（一手资料确认：`docs/subsystems/scope.md`——被 scope 过滤掉的工具不在 prompt 里且拒绝执行，与不存在的工具不可区分） | [06 §2](06-mcp-guard.md) `tools/list` 过滤：被过滤工具的 `tools/call` 返回与未知工具相同的 MCP 错误，账本内记真实原因；`fysun` profile 的 `tools.restrict` 对 DSH 原生工具承担同一职责（08 §2，M2b 验收项） |
| 凭据三规则（一手资料确认：`docs/subsystems/credentials.md`——每操作重解析、描述面无值槽、空值即未配置） | `SecretRef` 三规则（02 §2.9、00 §5）；下游 MCP server 凭据同规则（06 §8） |
| subagent provider 能力位在 `start()` 前校验，缺失则以 `UNSUPPORTED_CAPABILITY` 大声失败（一手资料确认：`docs/subsystems/subagent.md`） | [05 §2](05-runtime-abi.md) `start_execution` 大声失败：缺 required 控制点 / 未资格化 / Policy 拒绝 `native_effect_tools` → `not_runnable.environment{missing[]}`，不降级运行（ADR-0007 §D6） |
| goal 的 `armed / disarmed` 故意不持久化（一手资料确认：`docs/agent-lifecycle.md`） | [03 §6](03-architecture.md) 重新武装：FysunOS 同样要求恢复后重新武装，但武装事实由账本事件 `activity/resumed` 承载、可由 Policy `auto_resume` 预授权、可审计（02 §2.2） |
| `SubagentResult.diagnostic` 剥离工具输入 / 文件内容 / 环境值 / 凭据，上限 4096 字节（一手资料确认） | `Handoff.deliverables[].summary` 与 `Execution.error_report.message`（≤ 4096 字节，剥离栈迹、环境变量、宿主绝对路径；完整内容经 Artifact 引用，02 §2.3、§2.8） |
| `tools/post-execute` 作为工具流水线的 accept / block / replace 阶段（一手资料确认：`docs/tool-execution-pipeline.md`） | [08 §2](08-dsh-adapter.md)：`fysun-dsh` 订阅 `tools/post-execute` 作为 DSH 原生工具的 receipt 采集点 |
| hooks.json `PreToolUse` command hook（Claude Code / Codex / DSH 兼容格式；退出码 2 阻断） | [06 §9](06-mcp-guard.md)、[07 §4](07-runtime-onboarding.md) hooks 阻断路径：`fysun hook pre-tool-use`，只能 deny / abstain，不计入 conformance 主路径 |
| `packages/guard/` 是 loop 卫生插件（一手资料确认：`packages/guard/README.md`） | 命名消歧：与 `fysun-mcp-guard` 无关（00 §9、08 §5）；其判定对 OS 只是 Runtime 内部意见，不替代守卫（05 §5） |

| 不照搬 | 原因与 FysunOS 做法 |
|---|---|
| DSH 把 approval policy / sandbox mode / permission preset 以会话日志最后一条事件为真源（一手资料确认：`docs/subsystems/approval.md`、`docs/subsystems/permission-presets.md`） | 会话日志不是治理真源。FysunOS 的真源是版本化 Policy 对象，每个准入决定引用其版本（02 §2.12、[03 §7](03-architecture.md)）；`fysun` profile 禁止 DSH approval policy `never`（它在服务内部先于 waterfall 执行，会绕过拦截，08 §2） |
| DSH 的一次性 turn 级审批与自绘审批卡片 | 决定不入治理账本、不能派生 Grant、Runtime 可自行"批准"（T9 / T11）。FysunOS 以 Approval 一级对象承接，`fysund` 是唯一 answerer；`fysun-approval` 在 `approval/request` waterfall 拦截、不 `next()`、全部转 `fysund`（[ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md)） |
| Agent Teams 的全部协作事实入治理账本 | 见上表 Agent Teams 行：只有 Handoff 状态迁移与 `deliverables` 入账 |
| CognitiveOS `cognitive-akp/src/harness.rs::LoopDriver` 把 loop 边界放在内核内（一手资料确认；列于此处便于与 DSH loop 对照） | loop 属 Runtime；OS 只握预算、取消、恢复边界与重新武装事件（01 §6、02 §3） |

**不吸收为 OS 语义**：见 [08 §5](08-dsh-adapter.md)。

## 4. 从 Cordis 吸收（收窄为 `fysun-dsh` 适配器内部实现）

版本事实（一手资料确认）：DSH vendored 的上游 Cordis 版本为 **4.0.0-rc.7**（DSH `docs/rescope.md`），DSH 清单 `vendor/cordis/package.json` 的版本号为
4.0.1；上游最新 tag 为 **v4.0.0-rc.9**（对应 commit 待查证）。DSH 与 Cordis 为同一批作者，Cordis rc 变更直接传导到 DSH——这是 `fysun-dsh` 精确钉版与
CI 每日漂移预警的依据（[08 §6](08-dsh-adapter.md)）。理论基础见 arXiv:2608.25512。

| 吸收 | 落到 FysunOS |
|---|---|
| 可撤销效果（每个上下文变换携带逆操作，卸载时 LIFO 撤销） | 适配器内 dispose 语义；账本只记 `resource/unavailable \| quarantined` 与由此触发的 Grant 撤销、Effect 对账 |
| 反应式依赖（coeffect 规格驱动激活 / 停用 / 重载） | 适配器内 mount 排序；不进入账本 |
| loader 的配置 reconciliation、HMR | 适配器内部；OS 侧不实现 HMR |

内部 trait 参考见 [08 附录 A](08-dsh-adapter.md)（v0.7.0 由旧 04 附录 A 移至 08）。

**不做**：以 Cordis 为内核基底；把插件图当信任 / 委派图；fork 或 vendor Cordis；把上述任何一项作为 OS 公共接口暴露；
**把 `isolate(name)` 当作安全隔离**——`packages/core/src/context.ts` 的 `isolate(name)` 只是服务名的命名空间符号（一手资料确认），
不构成信任边界，不能替代 Grant / Policy / Scope。

## 5. 迁移顺序（与 MVP 计划对齐）

1. 先落对象模型与账本 schema（新建，不复制 `core/specs`）；事件通用字段自 M0 起含 `node_id` / `origin_seq`（中心恒自身，[11 §3](11-edge-and-offline-ledger.md)）；
2. 迁 ActionIntent / Effect（含幂等冲突硬拒绝与补偿新 Grant 规则）、replay、recovery、context 四个内核模块并按新对象改名；
3. 迁 Verifier 与 Provider 代理；
4. 以 DSH stdio broker 经验实现 `fysun-rpc`；MCP 守卫（`fysun-mcp-guard`，自实现）与 Runtime 资格化登记先于 DSH 适配器落地
   （对应 12 M2a → M2b；M2a 基线为 DSH headless）；
5. 其余按 [12 MVP 计划](12-mvp-plan.md) 的闭环需要逐项拉取。

每次迁入都在 [来源登记](../research/sources.md) 记录来源 revision 与路径，保证可追溯。

## 6. 新建项清单（不迁移；无来源或仅有部分先例）

以下项在 CognitiveOS、DSH、Cordis 中无对应机制（`label` 仅有上下文层部分先例），按 v0.7.0 设计稿新建：

| 新建项 | 定义位置 | 依据 |
|---|---|---|
| 来源 / 完整性标签 `label`（含 `flags`） | 02 §2.15 | ADR-0004；CognitiveOS 上下文层部分先例（`LoadedContextItemTrustLevel`），扩展新建 |
| Approval 一级对象 | 02 §2.11 | ADR-0005 |
| Approval 结果集与审批面 | 02 §2.11 | [ADR-0011](../adr/0011-approval-result-set-and-approval-surface.md) |
| Policy 对象（Cedar 子集） | 02 §2.12、03 §7 | ADR-0003、[ADR-0009](../adr/0009-cedar-subset-and-entity-model.md) |
| Memory 记录 | 02 §2.13 | 02 |
| MCP 守卫（`fysun-mcp-guard`，自实现） | [06](06-mcp-guard.md) | ADR-0002、[ADR-0012](../adr/0012-mcp-guard-self-implementation.md) |
| Effect 类别三源合一 | 06 §3 | [ADR-0008](../adr/0008-effect-category-and-read-only-determination.md) |
| hooks 阻断路径 | 06 §9 | [ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) |
| Runtime 资格化纪元 | [07 §1](07-runtime-onboarding.md) | ADR-0007 |
| A2A client（外部 Handoff） | [05 §7](05-runtime-abi.md) | ADR-0002 |
| Agent Skills 登记（来源 + digest + `allowed-tools` 上界 + `tool_effects[]` 映射） | [04 §2](04-capability-api.md)、07 §1 | ADR-0002、ADR-0008 |
| 边缘离线合并（Node、escrow、`ledger/merged` / `ledger/conflict`） | [11 §3](11-edge-and-offline-ledger.md) | [ADR-0010](../adr/0010-edge-offline-ledger-consistency.md) |
| OTLP 导出 | 03 §8 | 03 |
