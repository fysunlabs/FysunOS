# 00 · 术语与对象命名对照表

- 状态：设计稿 v0.7.0（2026-09-08）
- 变更依据：[ADR-0001](../adr/0001-founding-decisions.md) ~ [ADR-0006](../adr/0006-terminology-action-intent.md)（v0.7.0 修订）与 [ADR-0007](../adr/0007-runtime-native-tools-and-conformance-scope.md) ~ [ADR-0012](../adr/0012-mcp-guard-self-implementation.md)；实施基准为 [v0.7.0 全量重构蓝图](../research/20260908-v0.7.0-refactor-blueprint.md)
- 用途：全仓库术语、对象名、状态名、事件名、字段名、错误码、组件名的唯一基准。其他设计文档与本表冲突时，先修正冲突方；本表变更须同步 CHANGELOG。事件族全集只在本表 §4 维护，其他篇章只引用。

## 0. 文档地图与文件名（v0.7.0）

`docs/design` 共 14 篇，编号顺序为"基础（00–03）→ 接口（04–06）→ Runtime 接入（07–08）→ 迁移 / 演进 / 边缘（09–11）→ 计划与保证（12–13）"。
文件名沿用 `NN-kebab-title.md` 风格；本表是文件名的定稿，其他篇章的交叉引用以本表编号为准。

| 编号 | 文件名 | 标题 | 一句话范围 | 与 v0.6.0 的关系 |
|---|---|---|---|---|
| 00 | `00-terminology.md` | 术语与对象命名对照表 | 全仓库唯一术语基准；改名对照；Level 速记禁令；事件族全集 | 保留改写（文件名不变） |
| 01 | `01-definition-and-principles.md` | 定义与设计原则 | 定义、对外坐标、Capability 定义、P1–P8、F1–F9、不做什么、五概念 | 保留改写（文件名不变） |
| 02 | `02-object-model.md` | 对象模型 | 一级对象、状态机、字段、横切标签、AOS 映射 | 保留改写（文件名不变） |
| 03 | `03-architecture.md` | 系统架构、账本与恢复 | 四子系统、进程拓扑、双账本、event-sourced 账本（事件族只引用本表 §4）、恢复模型（含重新武装）、Policy 模型、可观测性 | 保留改写（文件名不变） |
| 04 | `04-capability-api.md` | Capability API 与错误语义 | 应用 ↔ OS 的全部方法；Agent Skills → Capability 映射；错误码全表 | 拆分自旧 `04-abi.md` §1、§3、§2.8 |
| 05 | `05-runtime-abi.md` | Runtime ABI：最小生命周期 RPC、控制点与 A2A | `describe` / `start` / `resume` / `cancel` / `deliver_input` / `checkpoint` / `health`；六控制点义务；Runtime → OS `os.*`；不得调用清单；`session_link`；A2A 外部 Handoff | 拆分自旧 `04-abi.md` §2.2–2.7 |
| 06 | `06-mcp-guard.md` | MCP 守卫规范（`fysun-mcp-guard`） | `tools/list` 过滤与不可区分性；`tools/call` pre / exec / post；Effect 类别三源合一；审计快路径与 `audit/observed`；结果标签与 `flags`；tasks / elicitation；守卫系统工具；下游 server 凭据与会话；hooks 阻断路径；agentgateway 关闭 | 拆分自旧 04 §2.1 + 旧 02 §2.6 准入流程 + 旧 03 §3 守卫段；新建 |
| 07 | `07-runtime-onboarding.md` | Runtime 接入与资格化 | Runtime 作为通用 Resource 类型；`native_effect_tools[]`；资格化纪元；MCP-only 直接接入（context_view 渲染与 `deliver_input` 投递路径）；hooks.json 阻断路径；边缘 Runtime；通用钉版与 CI 漂移预警 | 拆分自旧 `05-runtime-and-dsh-integration.md` §1 后半、§5、§6；新建 |
| 08 | `08-dsh-adapter.md` | DSH 参考适配器 `fysun-dsh` | DSH 定位与不吸收项、七组件归类、`approval/request` waterfall 拦截、DSH 原生工具治理路径、headless 模式、DSH 钉版、`guard/` 包消歧、Cordis 映射（附录 A） | 拆分自旧 05 §1–5 + 旧 04 附录 A |
| 09 | `09-migration-inventory.md` | 迁移清单 | CognitiveOS / Personal / DSH / Cordis 各迁什么；来源标签先例更正；Agent Teams 收窄；新建项 | 保留改写（原 `06-migration-inventory.md`） |
| 10 | `10-evolution.md` | 演进 | 三种变化、规则、机制、质量目标表示法、技能库生命周期对照 | 拆分自旧 `07-evolution-and-edge.md` §1 |
| 11 | `11-edge-and-offline-ledger.md` | 边缘与离线账本一致性 | 边缘场景、部署形态、物理 Effect 扩展、明确边界、可移植性三层；§3 离线账本一致性（ADR-0010） | 拆分自旧 07 §2 + 新建 §3 |
| 12 | `12-mvp-plan.md` | MVP 计划 | 边界、闭环、M0–M5（M2a / M2b）验收、边缘阶段、度量、证据纪律（四类状态用语）、统一开放问题清单 | 保留改写（原 `08-mvp-plan.md`） |
| 13 | `13-threat-model.md` | 威胁模型与 conformance 声明 | 资产、信任边界、T1–T11、不防、负例索引、按 Runtime 类型分列的 conformance 表 | 保留改写（原 `09-threat-model.md`） |

旧文件名 → 新文件名（供重命名与交叉引用替换）：`04-abi.md` → 拆为 `04-capability-api.md` / `05-runtime-abi.md` / `06-mcp-guard.md`；
`05-runtime-and-dsh-integration.md` → 拆为 `07-runtime-onboarding.md` / `08-dsh-adapter.md`；`06-migration-inventory.md` → `09-migration-inventory.md`；
`07-evolution-and-edge.md` → 拆为 `10-evolution.md` / `11-edge-and-offline-ledger.md`；`08-mvp-plan.md` → `12-mvp-plan.md`；`09-threat-model.md` → `13-threat-model.md`。
开放问题清单统一登记于 12 §7，13 与各 ADR 只引用。

## 1. 改名对照

### 1.1 v0.5.0 → v0.6.0

| v0.5.0 用法 | v0.6.0 用法 | 依据 |
|---|---|---|
| `Intent`（效果级请求记录） | `ActionIntent` | ADR-0006；避免与 AOS / MCP / A2A 中"顶层目标"义的 intent 冲突 |
| 事件族 `intent/*` | `action-intent/recorded` / `action-intent/rejected` / `action-intent/replayed` | 同上 |
| Harness ABI | **Runtime ABI** | ADR-0002 |
| Composition ABI（公共接口） | 降为内部接口："内部适配器生命周期"（v0.6.0 在 04 附录 A；v0.7.0 移至 08 附录 A） | ADR-0002 |
| DSH = 默认参考 Runtime；`fysun` bundle | DSH = 参考适配器 `fysun-dsh`（v0.6.0 称"第一个 Level-2 参考适配器"，v0.7.0 改称见 §1.2） | ADR-0002 |
| `fysun-tool-gate`（DSH tools 流水线订阅） | 作为独立组件取消；MCP 工具面由 MCP 守卫取代 | ADR-0002 |
| 部署策略 = 本地文件 | **Policy** 版本化账本对象（Cedar 子集） | ADR-0003 |
| `needs_human` 仅为错误码 | `Approval` 一级对象；`needs_human(approval_id)` 保留为错误码 | ADR-0005 |
| Handoff 状态 `issued / accepted / in_progress / delivered` | `submitted / working / completed`（对齐 A2A v1.0），`verified` 为 FysunOS 叠加态 | 02 §2.8 |
| 事件 `handoff/issued`、`handoff/accepted`、`handoff/delivered` | `handoff/submitted`、`handoff/working`、`handoff/completed` | 同上 |
| Activity 等待态只有 `waiting` | `waiting`（等触发）与 `input_required`（等人）分开 | ADR-0005 |
| 安全包络作为 Effect 字段 | 安全包络为绑定到设备 Resource 的 Policy 对象；Effect 只引用 | 11 §2.3 |
| 里程碑 M2 | M2a（MCP 守卫 + MCP-only 接入）/ M2b（`fysun-dsh` 适配） | 12 §3 |

"Harness"一词保留为工程术语（01 §7：一次 Execution 中围绕决策核心装配出的支撑组合），不再出现在任何接口名中。

### 1.2 v0.6.0 → v0.7.0

| v0.6.0 用法 | v0.7.0 用法 | 依据 |
|---|---|---|
| Level-1 / Level-2 可出现于组件描述 | 仅叙述速记；标识符禁令（§1.3）；新增 Level-1b（hooks 阻断） | 蓝图决策 12；ADR-0007 |
| `fysun-dsh` = "DSH 的 Level-2 参考适配器" | "DSH 参考适配器，声明全部六个控制点" | 蓝图决策 12 |
| `describe() -> {type, version, control_points[]}` | `describe() -> {type, version, control_points[], native_effect_tools[], describe_digest}` | ADR-0007 |
| Execution `waiting`（无属性） | `waiting{reason: trigger \| approval \| input}` | 蓝图决策 12；ADR-0011 §D9 |
| Approval 状态 `pending / approved / denied / modified / expired` | `pending / approved_once / rejected / cancelled / unavailable` | ADR-0011 |
| Approval 五个事件（请求 / 批准 / 拒绝 / 修改 / 过期） | `approval/asked` \| `approval/decided{result}` | ADR-0011 |
| Approval `presentation` 含参数 | `subject_ref{action_intent_id?, params_digest?, object_ref?}`（`effect_candidate` 填前两项，其他 subject 填 `object_ref`），不复制参数 | ADR-0011 |
| 六控制动作为 Policy 映射目标 | `policy_decision.outcome` 四值为唯一 Policy 输出；六动作为 Activity 子系统内部词汇 | 蓝图决策 12；ADR-0003 §D2 |
| Memory 写入被表述为一种（内部的）Effect，走审计快路径 | `memory/admitted` 准入事件，不走 Effect 状态机、不需 receipt | 蓝图决策 12 |
| 工具结果标签三处例外 | 一律 `untrusted_external` | 蓝图决策 10 / H-5；ADR-0004 §D1 |
| `label{source_principal, integrity, origin_ref}` | 增 `flags?: [authority_claim, secret_shaped]` | 借鉴 AKP；ADR-0004 §D1 |
| "Effect 类别"未定义 | `effect_category ∈ local_read \| external_read \| local_mutation \| external_mutation \| credential_use \| physical_action` | ADR-0008 |
| — | `Capability.contract.bounds.tool_effects[]`（必填） | ADR-0008 |
| `policy.describe()` | `policy.describe(policy_id?)` | 蓝图决策 12；ADR-0003 §D1 |
| 已安装 Runtime 的登记只针对 DSH（旧 05 §5） | 通用 Resource 类型"Runtime（已安装）"；`runtime.list` / `runtime.qualify`；资格化纪元 `runtime/disqualified` | 蓝图决策 12；ADR-0007 §D7 |
| — | `audit/observed` 事件 | 蓝图决策 12 / G7；ADR-0008 §D5 |
| — | `execution/waiting{reason}` 事件 | C1-6；ADR-0011 §D9 |
| — | 边缘：`Node`、`node_id`、`origin_seq`、`escrow`、`ledger/merged \| ledger/conflict`、`node/registered \| online \| offline` | ADR-0010 |
| — | 错误码 `conflict.idempotency`、`unavailable.approval` | ADR-0008 §D6、ADR-0011 §D2 |
| `fysun-mcp-guard`（agentgateway 采用形态悬而未决） | 自实现；与 DSH `guard/` 包无关 | ADR-0012 |
| — | CLI 子命令 `fysun hook pre-tool-use`；守卫系统工具 `fysun.await_approval`、`fysun.pending_inputs` | ADR-0007 §D3 / §D5 |
| `Activity.constraints{max_pending_approvals, approval_sla}` | 增 `approval_rate_limit`、`consecutive_rejections_alert` | ADR-0011 §D6 |
| Activity 恢复"不强制人工 re-arm" | 含非 `local_read` 类别时缺省 `paused{reason: rearm_required}`，Policy `auto_resume` 可覆盖 | H-7；ADR-0011 §D8 |
| `Handoff.deliverables[]` | 增 `summary ≤ 4096B` 脱敏规则；`Execution.error_report` 同 | 借鉴 DeepSeek Harness |
| F5 "secret 隔离" | "secret 隔离（OS 管理的 secret 与经 `fysund` 代理的 Provider）" | 蓝图决策 6；ADR-0007 §D4 |
| 威胁向量 T1–T10 | 增 T11 审批面伪造 | H-4 |
| 08 开放问题 + 09 开放问题两处 | 统一清单于 12 §7 | H-9 |
| conformance 状态用语"满足 / 部分" | `specified / implementation available / test executed / implemented` | 借鉴 CognitiveOS |
| `fysun-dsh` "不再订阅 `tools/post-execute`" | 订阅 `tools/post-execute` 作为 DSH 原生工具 receipt 采集点 | H-3 选项 B；ADR-0002 §D5 |

### 1.3 Level 速记禁令

**Level-1 / Level-2 / Level-1b 仅为叙述性速记，不是等级制度**：

- Level-1 = MCP 客户端 + 最小生命周期 RPC（"MCP-only Runtime"）；
- Level-2 = Level-1 + 至少一个可选控制点（"声明控制点的 Runtime"）；
- Level-1b = MCP 客户端 + 最小生命周期 RPC + hooks.json PreToolUse 阻断（**仅阻断，不计入 conformance 主路径**，ADR-0007 §D3）。

禁令：这些速记**不得出现在任何对象名、字段名、组件名、事件名、错误码、Cedar 实体与属性名中**；账本中不存在 Level 字段。
环境匹配与资格化依据的是 `control_points[]` 与 `native_effect_tools[]`，不是 Level 标签。叙述中优先使用描述性表达（"MCP-only Runtime"、
"声明全部六个控制点的 Runtime"、"hooks 阻断路径"）。验收方式：对 README、`docs/design`、`docs/adr` 做反引号内含 Level 速记的文本搜索，命中数须为零。

## 2. 一级对象

| 对象 | 归属子系统 | 一句话 | 定义位置 |
|---|---|---|---|
| Capability | Capability | 可复用、可配置、可演进的行为能力；契约与实现分离；`bounds.tool_effects[]` 必填 | 02 §2.1 |
| Activity | Activity | 有终点或持续的工作目标；状态而非调度器；`target` 不可变；恢复后需重新武装 | 02 §2.2 |
| Execution | Activity | Activity 的一次运行；绑定 Capability 快照、Runtime（含资格化引用）、Resource | 02 §2.3 |
| Principal / Scope | Authority | 授权主体与范围；Execution 是可被授权的主体 | 02 §2.4 |
| Grant | Authority | 只能衰减的授权，带预算与撤销 epoch；可作 escrow 子 Grant 下发边缘 | 02 §2.5 |
| **ActionIntent** → Effect → Receipt → Verification | Authority | 外部效果协议：先记录再执行，独立验证；`effect_category` 决定路径 | 02 §2.6、§2.7 |
| Handoff | Activity | 委派与交接事实；外部形态为 A2A Task；只有状态迁移与 deliverables 入治理账本 | 02 §2.8 |
| Resource | Resource | 模型、Secret、上下文源、记忆 / 知识、工具 / 适配器、实现包、设备、其他 Agent、**已安装 Runtime** | 02 §2.9 |
| Adjustment | Capability | 轻量演进载体；可选预测与评估对照 | 02 §2.10 |
| Approval | Authority | 人在环审批；结果集封闭且 fail-closed；权限扩张的唯一合法路径 | 02 §2.11 |
| Policy | Authority | 版本化部署策略对象（Cedar 子集）；每个准入决定引用其版本；唯一输出 `outcome` 四值 | 02 §2.12 |
| Memory | Resource | 经准入的 durable 知识；写入为准入事件不走 Effect 状态机 | 02 §2.13 |
| Event / Projection / Artifact | Authority | 账本真源 / 投影 / 内容寻址产物 | 02 §2.14 |
| 来源 / 完整性标签（横切） | Resource 赋予 | 每个上下文片段、工具结果、Artifact 携带的 `label`（含 `flags`） | 02 §2.15 |
| **Node（边缘节点）** | Authority | 账本归属节点；边缘离线合并单位 | 02 §2.16、11 §3 |

不进入对象模型：角色、团队、数字员工、Project、Routine、业务流程、Agent Shell、聊天消息、Prompt 正文、Skill 正文、工具 schema、模型流式 chunk、
**Plan**（决策核心内部规划）、**Loop 状态**（armed / step 计数 / 继续策略）、**Agent Teams 的 roster / mailbox / task DAG**（留在 Runtime 会话账本，
经 `session_link` 关联）。

## 3. 状态机

| 对象 | 状态集 | 说明 |
|---|---|---|
| Capability.status | `draft` / `active` / `withdrawn` | 不变 |
| Activity.state | `created` / `active` / `waiting` / `input_required` / `paused` / `blocked` / `completed` / `cancelled` | `waiting` = 等触发；`input_required` = 等人（Approval 未决或需要用户输入）；`paused` 带 `reason`（`user` / `rearm_required` / …）——进程重启后，Capability `tool_effects` 含非 `local_read` 类别的 Activity 缺省恢复为 `paused{reason: rearm_required}`，须 `activity.resume` 或 Policy `auto_resume`；纯只读 Activity 直接 `waiting`；`blocked` 携带原因（如 `consecutive_rejections`、`revoked_offline`、`policy_retired`、quarantine 待对账）与恢复动作 |
| Execution.state | `starting` / `running` / `waiting` / `checkpointed` / `finished` / `failed` / `cancelled` | `waiting` 带 `reason: trigger \| approval \| input`；等 Approval 时 `waiting{reason: approval}`，其所属 Activity 为 `input_required` |
| Handoff.state | `submitted` / `working` / `input_required` / `auth_required` / `completed` / `verified` / `failed` / `rejected` / `cancelled` | 对齐 A2A v1.0 TaskState；`verified` 为 FysunOS 叠加态；`working` 可带 `phase: accepted \| in_progress` 作为属性而非状态 |
| Effect.state | `authorized` / `dispatch_started` / `receipt` / `outcome_unknown` / `reconciled` / `quarantined` / `compensated` | 不变；`local_read` 工具不进入 Effect 状态机（审计快路径） |
| Approval.state | `pending` / `approved_once` / `rejected` / `cancelled` / `unavailable` | `pending` 唯一非终态；`modify(params)` 折入 `approved_once` 并记改写后 `params_digest`；到期按 `on_expiry` 折入 `rejected`（`deny`，缺省）或保持 `pending`（`keep_waiting` / `escalate`）；不存在 `expired` |
| Policy.status | `active` / `retired` | 版本只增不改 |
| Memory.state | `proposed` / `admitted` / `rejected` / `superseded` / `retracted` | 矛盾记录不合并，标记 `conflict_with` |
| Adjustment | `applied` / `reverted` / `promoted`，附 `evaluated_against.result` | 见 §5 |
| Verification.verdict | `pass` / `pass_with_warning` / `fail` / `inconclusive` | `inconclusive` 不得解释为通过 |
| Node | `registered` / `online` / `offline` | 中心节点恒 `online`；边缘节点 `offline` 期间按 11 §3 离线规则运行 |

### Handoff ↔ A2A v1.0 TaskState 对照

| A2A TaskState | Handoff.state |
|---|---|
| `TASK_STATE_SUBMITTED` | `submitted` |
| `TASK_STATE_WORKING` | `working` |
| `TASK_STATE_INPUT_REQUIRED` | `input_required` |
| `TASK_STATE_AUTH_REQUIRED` | `auth_required` |
| `TASK_STATE_COMPLETED` | `completed` |
| （无） | `verified`（FysunOS 叠加：父 Execution 或独立 Verifier 对 deliverables 的验证） |
| `TASK_STATE_FAILED` | `failed` |
| `TASK_STATE_REJECTED` | `rejected` |
| `TASK_STATE_CANCELED` | `cancelled`（FysunOS 全仓库统一用英式拼写） |

## 4. 事件族（v0.7.0 完整集；全仓库唯一全集）

```text
principal/created | principal/revoked
scope/defined | scope/revised
node/registered | node/online | node/offline

capability/registered | capability/revised | capability/derived
capability/evidence-recorded | capability/withdrawn

activity/created | activity/activated | activity/waiting | activity/input-required
activity/paused | activity/resumed | activity/blocked | activity/completed | activity/cancelled
activity/policy-bound | activity/triggered

execution/started | execution/waiting | execution/checkpointed | execution/resumed | execution/forked
execution/finished | execution/failed | execution/cancelled

handoff/submitted | handoff/working | handoff/input-required | handoff/auth-required
handoff/completed | handoff/verified | handoff/failed | handoff/rejected | handoff/cancelled

grant/issued | grant/attenuated | grant/consumed | grant/revoked

action-intent/recorded | action-intent/rejected | action-intent/replayed
effect/authorized | effect/dispatch-started | effect/receipt
effect/outcome-unknown | effect/reconciled | effect/quarantined | effect/compensated
audit/observed

verification/requested | verification/reported | verification/inconclusive

approval/asked | approval/decided

policy/loaded | policy/revised | policy/retired

memory/proposed | memory/admitted | memory/rejected | memory/superseded | memory/retracted

adjustment/applied | adjustment/predicted | adjustment/evaluated
adjustment/reverted | adjustment/promoted

context/assembled

resource/registered | resource/bound | resource/metered | resource/unavailable | resource/quarantined
runtime/attached | runtime/qualified | runtime/disqualified | runtime/health | runtime/detached

ledger/merged | ledger/conflict
```

v0.6.0 → v0.7.0 事件变更：删除 Approval 的五个旧事件（请求 / 批准 / 拒绝 / 修改 / 过期）；新增 `approval/asked`、`approval/decided`、`audit/observed`、
`execution/waiting`、`runtime/disqualified`、`node/registered | online | offline`、`ledger/merged`、`ledger/conflict`。

携带字段规则：

- **通用字段**：`seq`、`time`、`principal`、`causation_id`、`correlation_id`、**`node_id`**（产生事件的节点）、**`origin_seq`**（该节点本地单调序；
  中心节点恒 `= seq`）；与 Execution 相关的事件另带 `execution_id`、`fencing_epoch`。
- **准入类事件**（携带 `policy_decision{policy_version, rule_id, outcome, obligations[]}`）：`effect/authorized`、`action-intent/rejected`、
  `approval/asked`、`grant/issued`、`memory/admitted`、`memory/rejected`、`runtime/qualified`。`audit/observed` 与 `approval/decided` **不是**准入类。
- `approval/asked{approval_id, subject, subject_ref{action_intent_id?, params_digest?, object_ref?}, policy_decision, expiry}`。
- `approval/decided{approval_id, result: approved_once | rejected | cancelled | unavailable, option?, params_digest?, decided_by, derived[]}`；
  每个 `approval/decided` 必有且仅有一个配对的 `approval/asked`。
- `audit/observed{execution_id, tool, effect_category: local_read, params_digest, result_label, path: guard | hook | adapter, verdict?}`；
  `path = adapter` 表示候选经声明控制点的 Runtime 的适配器拦截点 / `os.submit_candidate` 提交（如 `fysun-dsh` 对 DSH 原生只读工具的处理，08 §2）。
- `execution/waiting{reason: trigger | approval | input}`。
- `activity/paused{reason: user | rearm_required | …}`；`activity/blocked` 带原因与恢复动作。
- `runtime/qualified{resource_id, control_points[], native_effect_tools[], describe_digest, bundle_digest}`；
  `runtime/disqualified{resource_id, reason: describe_changed | bundle_changed | policy_revoked | adapter_precondition_failed}`。
- `grant/issued` 若由 Approval 派生带 `approval_id`（只可指向 `result = approved_once` 的 Approval）；若为边缘预留预算带 `escrow?: {node_id}`。
  `adjustment/applied` 若由 Approval 派生带 `approval_id`（同一约束）。
- `action-intent/rejected.reason` 取值：`denied.policy` / `denied.grant` / `denied.budget` / `denied.fencing` / `conflict.idempotency` / `unavailable.approval`；
  hooks 路径产生的另带 `path: hook`。
- `adjustment/reverted` 带 `reason`；`context/assembled` 带 `labels_digest`。
- `ledger/merged{node_id, from_seq, to_seq, digest}`；`ledger/conflict{node_id, kind: revoked_offline | policy_retired | home_violation | clock_skew, refs[]}`。

## 5. 关键字段与取值

| 字段 | 所在对象 | 取值 |
|---|---|---|
| `label` | 上下文片段、工具结果、Artifact、Effect 参数 | `{source_principal, integrity, origin_ref, flags?}` |
| `label.integrity` | 同上 | `trusted_user` / `trusted_system` / `verified_data` / `untrusted_external`；工具结果与外部 A2A Artifact 一律 `untrusted_external`，唯一升级路径为 Verification 或 Memory 准入 |
| `label.flags` | 工具结果（守卫 post 阶段分类） | 子集 of `authority_claim` / `secret_shaped`；Policy 可据此 `modify` / `deny` |
| `effect_category` | ActionIntent.action、`tool_effects[]`、Resource 目录、`audit/observed` | `local_read` / `external_read` / `local_mutation` / `external_mutation` / `credential_use` / `physical_action`（严重度升序）；来源三源合一（06 §3、ADR-0008） |
| `Capability.contract.bounds.tool_effects[]` | Capability | `[{tool, effect_category}]`，**必填**，覆盖 `allowed-tools` 每个工具；未列出的工具按 `external_mutation` 处理并由 `capability.health` 标"未分类" |
| `Capability.requirements` | Capability | Resource 类型与条件、`control_points[]`、环境要求；可声明拒绝含特定 `native_effect_tools` 的 Runtime |
| `policy_decision` | 准入类事件 | `{policy_version, rule_id, outcome, obligations[]}` |
| `policy_decision.outcome` | 同上 | `allow` / `deny` / `modify` / `approve`（`approve` = 需要 Approval）；**唯一 Policy 输出**；`proceed / proceed_with_warning / retry / fallback / escalate / terminate` 六控制动作为 Activity 子系统内部词汇（`escalate → approve`、`terminate → deny + cancel`） |
| `Execution.waiting.reason` | Execution | `trigger` / `approval` / `input` |
| `Execution.runtime_binding` | Execution | Runtime Resource id、类型与实例、`session_id`、`control_points[]`、`native_effect_tools[]`、`qualification_ref`（指向 `runtime/qualified`） |
| `Execution.error_report` | Execution | `{message ≤ 4096 字节, diagnostics_ref}`；`message` 剥离栈迹、环境变量、宿主绝对路径 |
| `Approval.subject` | Approval | `effect_candidate` / `grant_expansion` / `adjustment` / `handoff_accept` / `memory_admit` / `verification_override`；`handoff_accept` 仅在目标为外部 A2A Agent 且 Policy 要求、或目标 Capability `requirements` 超出父 Grant 时触发 |
| `Approval.subject_ref` | Approval、`approval/asked` | `{action_intent_id?, params_digest?, object_ref?}`；不复制参数 |
| `Approval.options` | Approval | `approve_once` / `approve_for_activity` / `approve_for_capability` / `deny` / `modify(params)` / `defer(until)`；决定 `derived`，不决定 `result` |
| `Approval.result` | Approval、`approval/decided` | `approved_once` / `rejected` / `cancelled` / `unavailable`；只有 `approved_once` 派生 Grant / Adjustment |
| `Approval.on_expiry` | Approval | `deny`（缺省 → `rejected`）/ `escalate` / `keep_waiting`（后两者保持 `pending`） |
| `Activity.constraints` 审批相关 | Activity | `max_pending_approvals`、`approval_sla`、`approval_rate_limit`、`consecutive_rejections_alert` |
| `Runtime.native_effect_tools[]` | Runtime（Resource）、`describe()`、`runtime/qualified` | Runtime 内置、不经 MCP 守卫、可产生外部效果的工具清单（含缺省 `effect_category`）；如实声明是资格化前提 |
| `Runtime.describe_digest` / `Runtime.bundle_digest` | Runtime（Resource）、`runtime/qualified` | `describe()` 快照 digest / 安装包或 bundle digest；任一变化 → `runtime/disqualified` |
| `Verification.verifier_kind` | Verification | `deterministic` / `model_judge` / `human` / `downstream_outcome` |
| `Verification.independence` | Verification | 集合，元素取自 `different_process` / `different_model` / `different_principal`；空集记为 `not_independent` |
| `Verification.calibration_ref` | Verification | judge 类引用校准集与 Krippendorff α；缺失记 `uncalibrated` |
| `Verification.verdict` | Verification | `pass` / `pass_with_warning` / `fail` / `inconclusive` |
| `assurance`（可选） | Effect 候选、Verification | `{level: high \| medium \| low \| unknown, basis[], dependence}`；`basis` 为证据引用，不是分数；Policy 可按 `effect_category` × `level` 映射到 `outcome` 四值 |
| `Adjustment.predicted_effect`（可选） | Adjustment | `{statement, metric?, direction?}`；为空时 `evaluated_against.result` 恒 `inconclusive`；给出但缺 `statement` → 拒绝；`approve_for_capability` 派生时自动 `statement = approval.reason` |
| `Adjustment.evaluated_against` | Adjustment | `{window, result: supported \| refuted \| inconclusive, verification_refs[]}` |
| `Adjustment.evidence_gate` | Adjustment | `{enabled, regression_set_ref, result}`；由 Policy 或用户开启 |
| `Handoff.deliverables[]` | Handoff | Artifact 引用 + 子 Execution Verification 引用 + `summary`（≤ 4096 字节，剥离栈迹、环境变量、宿主绝对路径；完整内容经 Artifact 引用） |
| `Memory.scope` | Memory | `principal` / `activity` / `capability` / `global` |
| `Handoff.context_id` | Handoff | = Activity id 或 Handoff 链根；对应 A2A `contextId` |
| `Policy` | Policy | `{id, version, digest, source, loaded_at, rules_ref, status}` |
| Policy 投递 / 对账参数 | Policy（部署设定，不是 Cedar 评估 context 键） | `approval_wait_timeout`（守卫阻塞 `tools/call` 等待 Approval 决定的上限，超时返回 `needs_human(approval_id)`，07 §3.4）；`approval_result_ttl`（决定到达而无开放通道时放行保持有效的上限，超期按 Approval `on_expiry` 处置，07 §3.4、12 §7 Q21）；`receipt_timeout`（适配器拦截路径下 `os.report_receipt` 的等待上限，超时 → `effect/outcome-unknown`，08 §2）；`auto_resume`（恢复后免人工重新武装，03 §6） |
| `SecretRef` | Resource（Secret）、Execution.resource_bindings、下游 MCP server 凭据 | 三规则：**每次操作重解析**（不缓存明文）、**描述面无值槽**（`resource.describe` / `tools/list` 中不存在可放明文的字段）、**空值即未配置**（不以空串或占位符表示） |
| `node_id` / `origin_seq` | 全部事件、Node | 节点标识 / 节点本地单调序；中心节点 `origin_seq = seq` |
| `Grant.escrow` | Grant（边缘预留预算） | `{node_id}`；`budget ≤ parent.remaining`，edge 离线期间只消耗 escrow |
| 安全包络（Policy 子类） | 绑定设备 Resource 的 Policy | `velocity_bounds`、`geofence`、`interface_allowlist`、`rate_limit`、`deadman`、`emergency_stop_ref`；Effect 记录 `envelope_ref` 与 `envelope_check` |
| 质量目标骨架 | Capability.contract.quality | Skill Contract `(P 前置条件, O 操作, A 产物, V 验证器, F 已知失败模式)`；V 为 checklist，每项标注 `verifier_kind` |

## 6. 底线 F1–F9

| ID | 名称 | 备注 |
|---|---|---|
| F1 | 唯一权威写入者 | |
| F2 | 概率组件只产候选（配套机制：来源标签） | |
| F3 | 外部效果先记录再执行 | MCP-only Runtime 下范围限于经守卫的 MCP 工具面，见 13 §6 |
| F4 | 完成由独立验证判定 | |
| F5 | secret 隔离（范围：OS 管理的 secret 与经 `fysund` 代理的 Provider） | harness 自持模型凭据不在范围（13 §4）；部署指引：harness 的 OpenAI 兼容 base URL 指向 `fysund` 代理 |
| F6 | 契约与负例不为实现让路（范围含 Policy） | |
| F7 | 部署策略声明、能力不自扩权 | MCP-only Runtime 下范围限于经守卫的 MCP 工具面，见 13 §6 |
| F8 | 未知外部改动受保护 | |
| **F9** | **Execution 内有效权限单调收窄**：有效权限集 = Grant ∩ Policy 允许集 ∩ 工具面；收窄自动生效；任何扩张必须经 Approval（`result = approved_once`），且扩张后仍 ⊆ 父 Grant | MCP-only Runtime 下范围限于经守卫的 MCP 工具面，见 13 §6 |

## 7. Runtime ABI

| 块 | 内容 | 谁必须实现 |
|---|---|---|
| **MCP 守卫** | `fysund`（组件名 `fysun-mcp-guard`，借用 agentgateway `Pass` / `Reject` / `Mutated` 语义，自实现，ADR-0012）以 MCP 中间层身份位于 harness 与工具之间：`tools/list` 按 Grant ∩ Policy ∩ `allowed-tools` 过滤改写（被过滤工具不可区分）；`tools/call` pre = `submit_candidate`（ActionIntent 持久化、类别合成、Policy、Grant、来源标签）、执行 = Effect dispatch、post = Receipt 回填 + 结果标签与 `flags`；`local_read` 走审计快路径产生 `audit/observed`；`tasks/*` 表达长任务与 outcome-unknown；`elicitation/create` 为 `fysund` 发起的审批载体；守卫作为 MCP client 持有下游 server 会话与 `SecretRef` 注入（06） | 所有 Runtime 必经 |
| **最小生命周期 RPC** | `describe() -> {type, version, control_points[], native_effect_tools[], describe_digest}`、`start_execution`（大声失败：缺 required 控制点 / 未资格化 / Policy 拒绝 `native_effect_tools` → `not_runnable.environment{missing[]}`）、`resume_execution`、`cancel`、`deliver_input`、`checkpoint()`（可返回 `unsupported`）、`health()`；语言无关 JSON-RPC over stdio（MVP）（05 §2） | 所有 Runtime 必须实现 |
| **可选控制点**（能力位） | `context_injection`（每次注入须在 Runtime 会话账本产生带 `context_view.digest` 的事件）、`checkpoint`、`fork`、`compaction_hook`、`session_link`、`subagent_delegation`（只有 Handoff 状态与 deliverables 入治理账本）（05 §3） | 按 Runtime 声明；账本记录 `runtime/qualified{control_points[], native_effect_tools[], describe_digest, bundle_digest}` |
| **hooks.json（Level-1b）** | harness 的 PreToolUse command hook 指向 `fysun hook pre-tool-use`；Policy `deny` 时退出码 2 阻断；只能 deny / abstain，不产生 Effect、receipt、标签；deny → `action-intent/rejected{reason: denied.policy, path: hook}`（准入类）；abstain → `audit/observed{path: hook, verdict: abstain}`（非准入类）（06 §9、07 §4） | 可选补充，不计入 conformance 主路径 |
| **守卫系统工具** | `fysun.await_approval(approval_id)`（`local_read`，长轮询等待 Approval 决定）、`fysun.pending_inputs()`（`local_read`，拉取待投递的 `message / event / handoff_result`）；出现在 `tools/list`，不受 `allowed-tools` 限制但受 Policy 控制（06 §7、07 §3） | `fysund` 提供 |
| **A2A**（外部 Handoff 形态） | `fysund` 作为 A2A client 发起 Task；Handoff 状态直接映射 TaskState；外部 Artifact 一律 `untrusted_external`；对外暴露 A2A server（签名 Agent Card）为多主体场景，非 MVP（05 §7） | 外部 Agent |
| **Agent Skills**（实现包） | `Capability.implementation` 引用 Skills 目录（digest 固定）；`allowed-tools` 为 Grant 请求初始上界且每个工具须在 `bounds.tool_effects` 有类别映射；`compatibility` 映射到 `requirements`（04 §2） | 实现包作者 |

Runtime → OS 调用（控制点适配器的内部语义；MCP-only Runtime 经 MCP 守卫隐式完成）：`os.request_context`、`os.submit_candidate`、
`os.observe_effect`、`os.request_resource`、`os.report_checkpoint`、`os.report_progress`、`os.request_handoff`、
`os.report_failure`（≤ 4096 字节脱敏 + `diagnostics_ref`）、`os.propose_memory`、`os.propose_adjustment`、
`os.report_receipt(execution, effect_id, receipt)`（适配器拦截路径专用：对经 `os.submit_candidate` 准入、由 Runtime 自行执行工具体的 Effect 回填 receipt，
如 `fysun-dsh` 的 `tools/post-execute`；`fysund` 在返回 `admitted(effect_id)` 时记 `effect/dispatch-started`，无 receipt 到达 → `effect/outcome-unknown`，08 §2）。

Runtime 不得调用（不存在于 ABI 中）：`mint_grant`、`commit_effect`、`complete_activity`、`write_ledger`、`mark_verified`、`decide_approval`。
hooks 判定与 DSH 进程内 `guard/` 包判定对 OS 只是 Runtime 内部意见，不替代守卫。

MCP-only Runtime 的投递路径（07 §3）：context_view 由薄包装进程在 `start_execution` 时渲染进 harness 启动输入，并经 MCP `resources/read`
（`fysun://context/<execution_id>`）/ `prompts/get` 暴露同一视图，conformance 声明"仅初始投递"；`approval_result` 优先 MCP Tasks，否则守卫阻塞
`tools/call` 至决定或 Policy `approval_wait_timeout`，超时返回 `needs_human(approval_id)` 后 harness 经 `fysun.await_approval` 再等；
`message | event | handoff_result` 在 turn 边界投递或经 `fysun.pending_inputs` 拉取，conformance 声明"尽力、下一 turn 边界"。

## 8. 错误与拒绝码

`denied.policy` · `denied.grant` · `denied.budget` · `denied.fencing` · `not_runnable.environment` · `conflict.idempotency` · `unavailable.approval` ·
`needs_human(approval_id)` · `input_required(request_ref)` · `auth_required(approval_id)` · `unknown_outcome` · `inconclusive`

- `denied.grant` / `denied.policy`（被过滤工具）：被 `tools/list` 过滤的工具其 `tools/call` **对外返回与未知工具相同的 MCP 错误**（不可区分），
  账本内记 `action-intent/rejected{reason: denied.grant | denied.policy}` 保留真实原因；
- `not_runnable.environment{missing[]}`：环境不满足 `requirements`；触发条件含缺任一 required 控制点、Runtime 未资格化或已 `disqualified`、
  Policy 拒绝其 `native_effect_tools`、适配器前置条件不满足（如 DSH approval policy 为 `never`、bundle `dsh_version` / `schema_digest` 不匹配）；
- `conflict.idempotency`：同一幂等键、不同 canonical 参数；硬拒绝，不去重；
- `unavailable.approval`：`approval/asked` 或 `approval/decided` 无法提交，或审批面不可用；整个请求拒绝（fail-closed）；
- `needs_human(approval_id)`：Policy 要求人工确认，`fysund` 已创建 Approval；MCP-only Runtime 下的投递路径见 07 §3；
- `input_required(request_ref)`：Execution 或 Handoff 需要父方或用户提供输入（非授权问题）；对应 A2A `INPUT_REQUIRED`、MCP elicitation；
- `auth_required(approval_id)`：需要额外授权，映射为 `subject = grant_expansion` 的 Approval；对应 A2A `AUTH_REQUIRED`；
- `unknown_outcome`：Effect 结果未知，进入对账；不得假定成功；
- `inconclusive`：Verification 无法判定；不是通过。

## 9. 组件与命名

| 名称 | 含义 |
|---|---|
| `fysund` | 权威 daemon（Rust），唯一权威写入者；唯一审批面提供者（Web UI / CLI / `fysund` 发起的 elicitation） |
| `fysun` | CLI；子命令含 `fysun hook pre-tool-use`（hooks.json PreToolUse 阻断路径的命令端，调 `fysund` 本地 RPC，Policy `deny` 时退出码 2） |
| `fysun-mcp-guard` | MCP 守卫组件（`fysund` 内），自实现，借用 agentgateway `Pass` / `Reject` / `Mutated` 语义与 Cedar 格式（ADR-0012）；与 DSH 的 `guard/` 包（loop 卫生插件）无关 |
| `fysun-dsh` | DSH 参考适配器，声明全部六个控制点；技术形态为叠在 `dsh-base` 之上的 bundle + `fysun` profile，bundle 校验 `dsh_version` 与 `schema_digest`；子组件 `fysun-credentials`、`fysun-approval`（在 DSH `approval/request` waterfall 拦截、不 `next()`、全部转 `fysund`）、`fysun-session-link`、`fysun-subagent`、`fysun-context`、`fysun-rpc`；DSH 原生工具经 `approval/request` 拦截 + `tools/post-execute` receipt 路径治理；`describe()` 的 `native_effect_tools[]` 不为空；`fysun` profile 禁止 DSH approval policy `never` |
| `fysund-edge` | 边缘节点上的精简 daemon（同一账本模型），内含确定性包络校验器；持有 escrow Grant，离线按 11 §3 运行，联网后按 `origin_seq` 推送合并 |

## 10. Capability API 方法索引（累积）

`capability.register(contract, implementation: skills_dir_ref | ..., requirements)`（校验 `bounds.tool_effects[]` 必填）· `capability.revise` · `capability.derive` ·
`capability.find` · `capability.describe` · `capability.match` · `capability.health(id)`（含"未分类工具"信号）· `capability.withdraw` ·
`activity.create / start / pause / resume（解除 paused{reason: rearm_required}，产生 activity/resumed）/ cancel / trigger / observe / wait` ·
`execution.observe`（返回 `waiting.reason`）· `execution.cancel` · `execution.fork` ·
`handoff.list / observe` · `artifact.get / put` · `verification.report(id)` ·
`adjustment.apply(…, predicted_effect?) / revert / promote / history` ·
`approval.list(filter)` · `approval.observe(id)` · `approval.decide(id, option, params?) -> {result, …}` · `approval.batch_decide(batch_ref, option)` ·
`resource.register / list / health` · `secret.import(source, name) -> secret_ref` · `grant.list / revoke` ·
`policy.load(source) -> {policy_id, version, digest}` · `policy.describe(policy_id?)`（无参返回当前 Scope 全部 `active` Policy 列表 `[{policy_id, version, digest, rules_summary}]`，有参返回单个）· `policy.explain(event_ref) -> policy_decision` ·
`memory.list(filter)` · `memory.retract(id, reason)` ·
`runtime.list()` · `runtime.qualify(resource_id) -> qualified{control_points[], native_effect_tools[]} | rejected(reason)` ·
`node.*`（边缘节点登记与合并观察，非 MVP，11 §3）·
`system.status / events / replay_check`

完整签名与语义见 04。

## 11. 威胁向量编号

T1 间接 prompt injection · T2 恶意 / 被投毒实现包（含下游 MCP server 工具语义变更）· T3 权限扩张 · T4 凭据外泄 · T5 账本篡改 / 重放 / 陈旧 epoch · T6 验证欺骗 ·
T7 记忆投毒 · T8 物理动作越界 · T9 Runtime 越权推进状态 · T10 DoS / 预算耗尽（含审批轰炸）· **T11 审批面伪造**（Runtime 文本输出或自绘卡片冒充审批面；
机制：审批面只由 `fysund` 提供、DSH 卡片失效、审批事实不进模型可见面、`decide_approval` 不在 ABI）。
负例向量编号格式 `NV-T<n>-<序号>`（占位，实现阶段登记；归属见 13 §5）。

## 12. 来源缩写

报告中的 A1–A15（学术）、B1–B6（协议）、C1–C5（工业）、D1–D4（需求数据）编号仅在审查报告内使用；设计文档引用时写全名或 arXiv 号，不写编号。
蓝图中的决策编号（§I.1 决策 1–12、§I.2 H-1 … H-9、§0 C1-x / C2-x / G1–G7）可在 ADR 修订历史与 CHANGELOG 中引用，设计正文不引用。
