# 变更日志

本仓库只有设计文档。版本号指设计文档集合的整体版本，与 git tag 一致。历史版本段保留写作当时的用语与篇章编号，不随后续术语变更改写
（当前用语以最新版本段与 [00 术语表](design/00-terminology.md) 为准）。

## v0.7.0（2026-09-08）

- 日期：2026-09-08
- 依据：[v0.7.0 全量重构蓝图](research/20260908-v0.7.0-refactor-blueprint.md)（唯一实施规格；其 §I 所有者最终决策优先于全文其他表述）
  与所有者两轮交互式决策。第一轮（蓝图 §I.1，方向性）：全量重构并允许拆分 / 重编号篇章；ADR-0001 ~ 0006 允许直接修订正文但必须带修订历史；
  v0.6.0 原样归档到 `docs/archive/v0.6.0/`；远期开放问题中只有边缘离线账本一致性纳入本轮正式设计；Runtime 须如实声明原生工具、conformance 按
  Runtime 类型分列、hooks.json 作为"仅阻断"补充；F5 范围限定；agentgateway 只借用语义、守卫自实现；Effect 类别三源合一；M2a 基线 = DSH headless；
  来源标签删除全部例外；DSH 审批由 `fysund` 唯一应答；十余项小项按推荐。第二轮（蓝图 §I.2，确认与新问题）：接受 00–13 十四篇文档地图；Approval
  结果集封闭为四值且 `options` 只决定 `derived`（H-1）；`external_read` 不走快路径（H-2）；保留 DSH 原生工具并经 `approval/request` 拦截 +
  `tools/post-execute` receipt 路径治理（H-3 选项 B）；新增 T11 审批面伪造（H-4）；删除 A2A 可信 Agent Card 例外（H-5）；恢复后重新武装（H-7）；
  agentgateway 关闭独立成 ADR-0012（H-8）；开放问题统一于 12 §7（H-9）。
- 修订 ADR（状态均为 `accepted（v0.7.0 修订）`，各自文末有 `## 修订历史`）：
  [ADR-0001](adr/0001-founding-decisions.md) §D2 按 ADR-0002 重写为"DSH 为声明全部六个控制点的参考适配器；Cordis 语义仅在适配器内部"，
  消除"bundle 逐版跟随 DSH 版本"、"Harness ABI"、`Intent/Effect` 三处过时表述；
  [ADR-0002](adr/0002-protocol-first-runtime-boundary.md) `describe()` 增 `native_effect_tools[]` / `describe_digest`，守卫改自实现并写入不可区分性，
  D5 改为"`fysun-tool-gate` 作为独立组件取消；`fysun-approval` 拦截 `approval/request` 并订阅 `tools/post-execute`"，D7 由 ADR-0012 关闭，
  Runtime → OS 清单增 `os.report_receipt`；
  [ADR-0003](adr/0003-policy-object-and-language.md) `policy.describe(policy_id?)`、准入类事件清单更新、`outcome` 四值为唯一 Policy 输出、
  `action` = `effect_category` + 工具名、子集范围移交 ADR-0009；
  [ADR-0004](adr/0004-provenance-labels-and-monotonic-privilege.md) 工具结果一律 `untrusted_external`、`label` 增 `flags`、hooks 路径无标签、
  D5 负例改"M1 登记、按 13 §5 归属验收"；
  [ADR-0005](adr/0005-approval-and-evidence-gated-evolution.md) 状态集与事件改为 ADR-0011 的封闭形式、`predicted_effect` 可选、`waiting.reason`、
  审批面只由 `fysund` 提供；
  [ADR-0006](adr/0006-terminology-action-intent.md) §D3 随 ADR 修订规则更新并把归档范围增至 v0.6.0。
- 新增 ADR（均 accepted）：
  [ADR-0007 Runtime 原生工具、资格化与 conformance 范围](adr/0007-runtime-native-tools-and-conformance-scope.md)（`native_effect_tools[]`、资格化纪元、
  hooks 阻断路径、F5 限定、MCP-only 投递路径、M2a 基线）、
  [ADR-0008 Effect 类别分类与只读判定](adr/0008-effect-category-and-read-only-determination.md)（六值、三源合一、快路径只对 `local_read`、幂等冲突、补偿新 Grant）、
  [ADR-0009 Cedar 子集范围与实体模型](adr/0009-cedar-subset-and-entity-model.md)（关闭旧 Q7）、
  [ADR-0010 边缘离线账本一致性](adr/0010-edge-offline-ledger-consistency.md)（关闭旧 Q9）、
  [ADR-0011 Approval 结果集、事件配对与审批面来源](adr/0011-approval-result-set-and-approval-surface.md)（含审批轰炸、`handoff_accept` 触发条件、重新武装）、
  [ADR-0012 `fysun-mcp-guard` 自实现](adr/0012-mcp-guard-self-implementation.md)（关闭旧 Q10 与 ADR-0002 §D7）。
- 全仓库术语变更（完整对照见 [00 §1.2](design/00-terminology.md)）：Level-1 / Level-2 / Level-1b 降为叙述速记并禁入一切标识符；
  `describe() -> {type, version, control_points[], native_effect_tools[], describe_digest}`；Approval 状态 `pending / approved_once / rejected /
  cancelled / unavailable`，事件只剩配对的 `approval/asked` / `approval/decided`，`presentation` 改引用 `subject_ref`；Execution `waiting{reason}`；
  `policy_decision.outcome` 四值为唯一 Policy 输出；Memory 写入为准入事件 `memory/admitted`；工具结果一律 `untrusted_external` 并增 `flags`；
  新增 `effect_category` 六值与 `Capability.contract.bounds.tool_effects[]`；`policy.describe(policy_id?)`；Runtime 推广为通用 Resource 类型
  （`runtime.list` / `runtime.qualify`、`runtime/disqualified`）；新增事件 `audit/observed`（`path: guard | hook | adapter`）、`execution/waiting`、
  `node/*`、`ledger/merged | conflict`；新增错误码 `conflict.idempotency`、`unavailable.approval`；`fysun-mcp-guard` 自实现；新增 CLI 子命令
  `fysun hook pre-tool-use` 与守卫系统工具 `fysun.await_approval` / `fysun.pending_inputs`；Runtime → OS 新增 `os.report_receipt`；
  Activity 恢复后缺省 `paused{reason: rearm_required}`；F5 名称限定范围；威胁向量增 T11；conformance 状态用语改为四类。

### 按文档

| 文档 | 来源 / 拆分关系 | 主要变更 |
|---|---|---|
| README | 改写 | 版本 v0.7.0；定位句加"如实声明原生工具"限定、DSH 改"声明全部六个控制点的参考适配器"；对外坐标改"供给（Trust 除外）"；conformance 链接指 13；索引改为 00–13 + 来源 + 审查报告 + 蓝图 + CHANGELOG + 归档 v0.6.0 / v0.5.0；命名表按 00 §9（`fysun-mcp-guard` 自实现、`fysun-dsh` 新描述、增 `fysun hook`）；许可证段去 agentgateway、改 Cedar |
| 00 术语与对象命名对照表 | 保留改写（文件名不变） | 新增 §0 文档地图与旧→新文件名映射；§1 拆为 1.1 / 1.2（v0.6.0 → v0.7.0 对照）/ 1.3 Level 速记禁令；§2 增 Node、Runtime 为 Resource；§3 Approval 五值、`waiting.reason`、`paused{rearm_required}`；§4 事件族为全仓库唯一全集（`approval/asked | decided`、`audit/observed`、`execution/waiting`、`runtime/disqualified`、`node/*`、`ledger/*`）；§5 增 `flags`、`effect_category`、`tool_effects[]`、`subject_ref`、`result`、`native_effect_tools[]`、两个 digest、`SecretRef` 三规则、边缘字段、Policy 投递 / 对账参数；§6 F5 限定与 F3/F7/F9 脚注；§7 `describe()` 五字段、hooks 与守卫系统工具行、`os.report_receipt`；§8 增两错误码与不可区分性说明；§9 组件表更新；§10 改为累积索引；§11 增 T11 |
| 01 定义与设计原则 | 保留改写 | §1.1 可判定限定补 `native_effect_tools[]` 如实声明；§1.4 Control & Governance Plane 的供给声明改为"供给（Trust 除外）"；P8 DSH 表述；§4 F5 改范围限定、F3/F7/F9 加 MCP-only 范围脚注、标签段引用改 12 §7；§6 增"不自建审批面以外的对话 UI"、"不持有 Loop 状态"；§7 Runtime 行补原生工具声明 |
| 02 对象模型 | 保留改写 | 总览图改写（Memory 写入为准入事件、Runtime 节点、Approval `result`、Node / escrow）；§2.1 `tool_effects[]` 必填；§2.2 审批限流与恢复后重新武装；§2.3 `waiting.reason`、`error_report`、`qualification_ref`；§2.6 准入流程重写（类别合成、`conflict.idempotency`、补偿新 Grant、快路径只对 `local_read`、`assurance` 只映射四值）；§2.8 deliverables 脱敏、Agent Teams 收窄、`handoff_accept` 触发条件；§2.9 SecretRef 三规则、删除只读本地源例外、Runtime（已安装）与下游 MCP server 行；§2.10 `predicted_effect` 可选；§2.11 Approval 结果集封闭与审批面；§2.12 评估输入更新；§2.13 写入不是 Effect；§2.14 `node_id` / `origin_seq`；§2.15 标签一律 `untrusted_external` 与 `flags`；新增 §2.16 Node；§3 / §4 更新 |
| 03 系统架构、账本与恢复（改题） | 保留改写 | 分层图与进程拓扑去 Level 标识、加 hooks 命令与下游会话；子系统职责增类别合成、审批限流、节点归属、Runtime 目录；§4 双账本增 Loop 状态 / mailbox、MCP-only 关联面；§5.1 删除事件族全文改引用 00 §4 并固定携带字段规则；§6 恢复模型增 Runtime 资格化校验与重新武装步骤及负例；§7 Policy 问题清单增类别覆盖、资格化、限流、`auto_resume`、投递 / 对账参数、边缘预算；§8 增 `audit/observed` 导出、`fysun.*` 两域区分、新增度量 |
| 04 Capability API 与错误语义（新文件名） | 拆分自旧 04 §1、§2.8、§3 | 接口总览改两组并引用 05 / 06；`capability.register` 校验 `tool_effects[]`；`activity.resume` 解除重新武装；`execution.observe` 返回 `waiting.reason`；`predicted_effect` 可选规则；`approval.decide` 返回 `result` 且决定只经本组接口或 `fysund` 发起的 elicitation 落账；`policy.describe(policy_id?)`；新增 `runtime.list` / `runtime.qualify`（`approve` / `modify` 按 `deny` fail-closed）、`node.*`；错误表增 `conflict.idempotency`、`unavailable.approval`，`not_runnable.environment` 触发条件扩展；新增 §4 负例 |
| 05 Runtime ABI（新建文件） | 拆分自旧 04 §2.2–2.7 | 三块结构与速记说明；`describe()` 五字段；`start_execution` 大声失败清单；`resume_execution` 两个水位；`deliver_input(approval_result)` 载荷限定 `{approval_id, result, params_digest, grants_summary?}`；控制点义务增 `context_injection` 会话事件与 `subagent_delegation` 收窄；`os.*` 增 `conflict(idempotency)`、`os.report_failure` 脱敏、`os.report_receipt`；不得调用清单补 hooks / DSH `guard/` 判定只是内部意见；§6 关联面加 `audit/observed`；§7 A2A 删除可信 Agent Card 例外 |
| 06 MCP 守卫规范（新建文件） | 拆分自旧 04 §2.1 + 旧 02 §2.6 准入流程 + 旧 03 §3 守卫段；§3、§5、§7、§8、§9 新建 | 定位（唯一 MCP server 或前置、自实现、与 DSH `guard/` 无关）；`tools/list` 过滤与不可区分性、`list_changed` 重分类；Effect 类别三源合一与 `max_severity` 合成；`tools/call` pre / 执行 / post 全流程；审计快路径与 `audit/observed`（`path` 三值）；`tasks/*` 与 `elicitation/create`；守卫系统工具；下游 MCP server 信任边界；hooks.json 阻断路径；负例集 |
| 07 Runtime 接入与资格化（新建文件） | 拆分自旧 05 §1 后半、§5、§6、§7；投递路径、hooks 路径、通用 Resource 类型与资格化纪元新建 | Runtime 作为 Resource 类型的登记字段；`runtime.qualify` 流程（`approve` / `modify` 按 `deny`）；资格化纪元四种 `disqualified.reason`；`native_effect_tools[]` 定义、声明责任与 Policy 两种处置；MCP-only 直接接入（候选清单与 M2a 基线、薄包装进程、context_view 渲染路径、`deliver_input` 三条投递路径、守卫系统工具、关联面）；hooks 路径接入与能力边界；F5 限定与部署指引；边缘 Runtime；通用钉版与 CI 漂移预警；代价与对策；负例索引 |
| 08 DSH 参考适配器 `fysun-dsh`（新建文件） | 拆分自旧 05 §1–5；附录 A 迁自旧 04 附录 A | 定位补 DSH / Cordis 同作者与 `isolate` 非隔离两条事实；组件按三块归类，`fysun-tool-gate` 作为独立组件取消；`fysun-approval` 拦截 `approval/request`、不 `next()`、全部转 `fysund`；新增 §2.3 DSH 原生工具治理路径（拦截点持久化 ActionIntent、`tools/post-execute` receipt、`tools.restrict` 过滤、禁止 approval policy `never`、`os.report_receipt`、`receipt_timeout`）与 §2.4 前置条件；§2.5 `describe()`（`native_effect_tools[]` 不为空）；§3 headless 模式为 M2a 基线并登记查证项；§4 双账本关联更新；§5 不吸收项增三条；§6 精确钉版、bundle 校验 `dsh_version` / `schema_digest`、CI 每日漂移预警；§7 `guard/` 包消歧；附录 A 增 Cordis 映射参考与 `isolate` 说明 |
| 09 迁移清单（新文件名） | 保留改写（原 06） | 基线按 sources.md 新表述；§1 增 `LoadedContextItemTrustLevel`、Effect 错误语义、四类状态用语、AKP 入站标志行，来源标签改"上下文层部分先例、扩展新建"，不迁项增 `LoopDriver`；§3 DSH 表去 Level 标识、Agent Teams 收窄、新增设计参考行（scope 不可区分性、凭据三规则、能力位预检、goal 不持久化、诊断剥离、`tools/post-execute`、hooks、`guard/`）与不照搬行；§4 Cordis 版本事实与 `isolate`；§6 新建项增五行 |
| 10 演进（新建文件） | 拆分自旧 07 §1 | `predicted_effect` 表述统一为 ADR-0005 §D5 修订版；`approve_for_capability` 派生自动 `statement`；V 规则 `human` 项以 `approved_once` 为准；健康查询增"未分类工具" |
| 11 边缘与离线账本一致性（新建文件） | 拆分自旧 07 §2；§3 新建 | §1 范围与承诺；§2 "任何 Level-1 Runtime" → "任何实现 Runtime ABI 最小面的 Runtime"，离线保守规则改引用 §3；新增 §3 离线账本一致性七小节（节点归属、escrow、离线行为含重新武装推导、合并、冲突封闭枚举、负例汇总、边缘阶段验收） |
| 12 MVP 计划（新文件名） | 保留改写（原 08） | 边界改 DSH headless 为 MCP-only 基线并增不可省略项；闭环第 3 / 6 / 7 步更新、新增第 10 步；§3 里程碑表按蓝图 §F 增删并新增边缘阶段行；§4 度量增三项；§5 证据纪律增四类状态用语；§6 非声明更新；§7 统一开放问题清单（Q7 / Q9 / Q10 关闭、Q8 删除、Q11–Q21 新增） |
| 13 威胁模型与 conformance 声明（新文件名） | 保留改写（原 09） | 信任边界增 `native_effect_tools` 与 hooks、下游 MCP server、harness 自持凭据行；T1 / T2 / T4 / T5 / T9 / T10 增负例，新增 T11 审批面伪造；§4 不防增两条；§5 索引更新；§6 conformance 表按四类 Runtime 分列、状态用语四类、`fysun-dsh` 列注明原生工具治理路径依赖；§6.2 增 `native_effect_tools[]` 声明义务与 `context_injection` 会话事件义务；§7 只引用 12 §7 |
| docs/adr/README.md | 改写 | 规则改为"accepted 的 ADR 可在版本重构中直接修订正文并带修订历史；推翻仍需新 ADR"；索引增 0007–0012；说明段记 ADR-0001 §D2 已按 ADR-0002 修订 |
| docs/research/sources.md | 修订 | Cordis 行更正为"上游 4.0.0-rc.7（DSH vendored；清单 4.0.1）；上游最新 v4.0.0-rc.9"；DSH 行用途去"默认 Runtime"；新增 CognitiveOS 公开仓库、`LoadedContextItemTrustLevel`、DeepSeek Harness v0.7.0 补登文档、Claude Code / Codex hooks 文档、Cordis `isolate` 与 rc.9、蓝图条目；agentgateway 条目改为"语义借用来源，非复用候选" |
| docs/archive/v0.6.0/（新建） | 归档 | v0.6.0 的 README、CHANGELOG、design 00–09、adr README + 0001–0006、sources.md 原样快照（21 个文件），`ARCHIVE.md` 记 git tag `v0.6.0`（commit `3e65b64`）与 v0.7.0 主要差别摘要 |
| docs/research/20260907-frontier-review-and-recommendations.md、20260908-v0.7.0-refactor-blueprint.md | 未修改 | 历史记录 / 实施基准，v0.7.0 完成后蓝图转为历史记录 |

### 与 v0.6.0 决策的差异

- **agentgateway 关闭**：v0.6.0 把 `fysun-mcp-guard` 是否复用 agentgateway 列为开放问题 Q10；v0.7.0 由 ADR-0012 关闭为自实现，仅借用 `Pass` / `Reject` / `Mutated` 语义与 Cedar 格式。
- **Level 禁令**：v0.6.0 允许 Level-1 / Level-2 出现在组件描述；v0.7.0 只保留为叙述速记，禁止进入任何对象名、字段名、组件名、事件名、错误码、Cedar 实体与属性名，并新增 Level-1b（hooks 阻断）速记。
- **Approval 结果集**：v0.6.0 `pending / approved / denied / modified / expired` 与五个事件；v0.7.0 封闭为 `approved_once / rejected / cancelled / unavailable`（`modify` 折入 `approved_once`，`expired` 折入 `on_expiry`），事件只剩配对的 `approval/asked` / `approval/decided`，请求不复制参数。
- **F5 限定**：v0.6.0 假定所有 Runtime 的模型请求经 daemon 代理；v0.7.0 把 F5 范围限定为"OS 管理的 secret 与经 `fysund` 代理的 Provider"，harness 自持凭据列入不防。
- **DSH 原生工具**：v0.6.0 把 `fysun-tool-gate` 作为独立组件取消的同时不再订阅 `tools/post-execute`，原生工具事实上不受治理；v0.7.0 保留 DSH 原生工具并经 `approval/request` 拦截（ActionIntent 持久化点）+ `tools/post-execute`（receipt 采集点）治理，`tools/post-execute` 从"不订阅"改为"订阅"。
- **conformance 声明**：v0.6.0 对 Level-1 Runtime 一律"满足"；v0.7.0 按四类 Runtime 分列，MCP-only 下 F3 / F7 / F9 限经守卫的 MCP 工具面，状态用语改为 `specified / implementation available / test executed / implemented`。
- **Activity 恢复**：v0.6.0 "不强制人工 re-arm"；v0.7.0 含非 `local_read` 类别的 Activity 恢复后缺省 `paused{reason: rearm_required}`，须 `activity.resume` 或 Policy `auto_resume`，武装事实入账本。
- **边缘离线一致性**：v0.6.0 只有一条保守规则、合并冲突规则未定义；v0.7.0 纳入正式设计（11 §3、ADR-0010）：节点归属、`node_id` / `origin_seq`、escrow、合并与封闭冲突枚举。
- **Effect 类别与只读判定**：v0.6.0 反复引用但无定义、只读由 Policy 逐条判定；v0.7.0 定义六值 `effect_category` 与三源合一，快路径只对 `local_read`，`external_read` 走完整协议。
- **来源标签**：v0.6.0 有三处工具结果标签例外；v0.7.0 一律 `untrusted_external`，唯一升级路径为 Verification / Memory 准入，并增 `flags`。
- **Policy 输出**：v0.6.0 四值 `outcome` 与六控制动作并存；v0.7.0 只保留四值为唯一输出，六控制动作降为 Activity 子系统内部词汇。
- **Cedar 子集**：v0.6.0 范围待定（Q7）；v0.7.0 由 ADR-0009 落定实体模型、语法子集与 `outcome` 注解约定。

### 实施中裁定的蓝图歧义

以下为 WP1–WP6 在蓝图未明确或内部表述不一处作出的裁定，均以 00 术语表为最终落点：

- **旧术语转述不写字面**：改名对照与修订历史中引用 v0.6.0 旧事件名 / 旧状态名时只作描述性转述（如"Approval 的五个旧事件"），不写旧字面，以使术语 grep 为零。
- **Level 速记不与 code span 同行**：叙述中的 Level-1 / Level-2 / Level-1b 不出现在反引号内，也不与含标识符的 code span 组合成"Level-1 Runtime"类复合标识。
- **`conflict.idempotency` 前提**：只在 Runtime **显式携带**幂等键且同键不同 canonical 参数时触发；未显式携带时幂等键即 canonical 参数 digest，同键必同参数，只可能重放。
- **`approval/asked` 带 `policy_decision`，`approval/decided` 与 `audit/observed` 不带**：准入类事件清单以此为准。
- **`subject_ref` 形状**：`{action_intent_id?, params_digest?, object_ref?}`——`effect_candidate` 填前两项，其他 subject 填 `object_ref`。
- **`escrow?: {node_id}`**：蓝图 §B-11 写作 `escrow: true, node_id`，00 §4 / §5 定为 `escrow?: {node_id}`，02 / 11 / ADR-0010 以 00 为准。
- **hooks 事件写法**：deny → `action-intent/rejected{reason: denied.policy, path: hook}`（准入类）；abstain → `audit/observed{path: hook, verdict: abstain}`（非准入类）；Policy 产出 `approve` / `modify` 时 hook 只能 abstain 并附 `outcome` 留痕，不按 `deny` 处理。
- **`audit/observed.path` 三值**：蓝图只列 `guard | hook`；WP3 为 `fysun-dsh` 拦截路径下的 DSH 原生只读工具增第三值 `adapter`，00 / 02 / 03 / 06 / ADR-0008 同步。
- **`os.report_receipt(execution, effect_id, receipt)`**：蓝图未列；H-3 选项 B 需要一条 Runtime → OS 的 receipt 回填调用，登记于 00 §7 / 05 §4 / 06 §5 / 08 §2 / 13 §6 / ADR-0002 §D1；配套 `fysund` 返回 `admitted(effect_id)` 即写 `effect/dispatch-started`，`receipt_timeout` 内无 receipt → `effect/outcome-unknown`。守卫路径 receipt 由 post 阶段自行回填，不经该调用。
- **`runtime.qualify` 只接受 `allow` / `deny`**：`Approval.subject` 封闭集无"Runtime 资格化"项，Policy 产出 `approve` / `modify` 一律按 `deny`（fail-closed），07 §1.2 / 04 §1.6 / ADR-0007 §D2 一致。
- **`approval_result_ttl` 超期语义**：07 §3.4 采用 `on_expiry = deny` → `effect/reconciled` 关闭未分发 Effect；是否需要独立终态登记为 12 §7 Q21（待确认）。
- **Policy 部署参数登记**：`approval_wait_timeout`、`approval_result_ttl`、`receipt_timeout`、`auto_resume` 登记于 00 §5 与 03 §7，明确为部署设定而非 Cedar 评估 context 键（ADR-0009 §D1 固定键集不变）。
- **ADR 背景段中的旧编号**：ADR-0007 / 0008 / 0009 / 0010 / 0012 背景段描述 v0.6.0 文档时的篇章编号统一加"旧"标记并括注现编号；ADR-0006 §D2 的"06 迁移清单"改指 09。

## v0.6.0

- 日期：2026-09-07
- 依据：[前沿对照审查报告](research/20260907-frontier-review-and-recommendations.md)（§8.1 改动清单、§8.2 ADR 提案）；
  owner 确认的战略方向为 OS ↔ Runtime 边界采用协议优先。
- 新增 ADR（均 accepted）：
  [ADR-0002 协议优先的 Runtime 边界](adr/0002-protocol-first-runtime-boundary.md)（部分 supersede ADR-0001 §D2）、
  [ADR-0003 Policy 对象与策略语言](adr/0003-policy-object-and-language.md)、
  [ADR-0004 来源标签与权限单调收窄（F9）](adr/0004-provenance-labels-and-monotonic-privilege.md)、
  [ADR-0005 Approval 一级对象与证据门控演进](adr/0005-approval-and-evidence-gated-evolution.md)、
  [ADR-0006 术语 `Intent` → `ActionIntent`](adr/0006-terminology-action-intent.md)。ADR-0001 正文未改。
- 全仓库术语变更：`Intent` → `ActionIntent`、`intent/*` → `action-intent/*`；Harness ABI → Runtime ABI；
  Composition ABI 降为内部接口；DSH 从"默认参考 Runtime + `fysun` bundle"改为"第一个 Level-2 参考适配器 `fysun-dsh`"；
  Handoff 状态对齐 A2A v1.0（`submitted / working / input_required / auth_required / completed / verified / failed / rejected / cancelled`）；
  Activity 增 `input_required`。

### 按文档

| 文档 | 主要变更 |
|---|---|
| README | 定位句补可判定限定与对外坐标摘要；状态 v0.6.0；索引加入 00 术语表、09 威胁模型、审查报告、CHANGELOG、归档；命名表 `fysun-dsh` 改为 Level-2 参考适配器，新增 `fysun-mcp-guard`（agentgateway 复用待评估）；许可证补 Apache-2.0 依赖 |
| 00 术语与对象命名对照表（新建） | 全仓库对象名、状态机、事件族、字段取值、F1–F9、Runtime ABI、错误码、组件名的唯一基准；v0.5.0 → v0.6.0 改名对照 |
| 01 定义与设计原则 | §1.1 定义加可判定限定；新增 §1.4 对外坐标（AOS 两平面供给声明）；P7/P8 更新；§4 新增 F9、F6 范围含 Policy、来源标签作为 F2 配套机制；§5 收益指标增三项；§6 增"不定义实现包格式、不自造策略语言、不自建记忆引擎"；§7 Runtime 行改为 Runtime ABI |
| 02 对象模型 | 新增一级对象 Approval（§2.11）、Policy（§2.12）、Memory（§2.13）与横切来源/完整性标签（§2.15）；`Intent` → `ActionIntent` 并重写准入流程（含 `policy_decision`、可选 `assurance`）；Verification 扩展字段（`verifier_kind`、`independence`、`calibration_ref`、`verdict`）；Handoff 状态机对齐 A2A 并叠加 `verified`、增 `context_id`；Activity `target` 不可变、增 `input_required` 与审批容量；Adjustment 增 `predicted_effect`、`evaluated_against`、`evidence_gate`；§3 增 Plan 不入对象模型；新增 §4 与 AOS 的对象映射 |
| 03 系统架构 | 分层图改为 Runtime ABI + A2A；Authority 子系统增 Approval、Policy 版本管理、F9 机械化；Resource 子系统增来源标签赋予、Memory 准入、Skills/MCP 目录；进程拓扑改为 mermaid 图（保留 ASCII 版）；§5.1 事件族按术语表补齐并改名；§6 恢复模型增 Policy 版本校验、Approval 水位、幂等键重放负例；§7 改为版本化 Policy 对象 + Cedar 子集；新增 §8 可观测性（OTLP、`fysun.*` 命名空间、`gen_ai.*` 钉版） |
| 04 接口与 ABI | 接口从三组减为两组；§2 重写为 Runtime ABI 三块结构（MCP 守卫 / 最小生命周期 RPC / 可选控制点）+ A2A 外部 Handoff + Agent Skills 实现包；Capability API 增 `approval.*`、`policy.explain`、`memory.*`、`capability.health`，`policy.describe` 返回版本；错误语义增 `input_required`、`auth_required`、`inconclusive`；Composition ABI 改为附录 A"内部适配器生命周期" |
| 05 Runtime 接入与 DSH 适配器（改题） | DSH 定位为第一个 Level-2 参考适配器 `fysun-dsh`；`fysun-tool-gate` 取消，其余六个子组件按 MCP 守卫路径 / Level-2 控制点适配器 / 最小生命周期 RPC 归类；双账本关联更新；§4 增"Agent Skills 为准"；§6 其他 Runtime 改为 Level-1 直接接入 |
| 06 迁移清单 | 迁移原则增"有标准用标准"；ActionIntent 改名；来源标签、Approval、Policy、Memory 标注为新建项；DSH tools 流水线改为经 MCP 守卫；Cordis 吸收收窄为 `fysun-dsh` 内部实现；新增 §6 新建项清单 |
| 07 演进与边缘 | §1.3 增可证伪预测规则；§1.4 机制增预测记录、评估对照、可选证据门控、健康查询；新增 §1.5 质量目标表示法（Skill Contract + checklist 化 V + `verifier_kind`）与 §1.6 技能库生命周期对照；§2.3 安全包络改为绑定设备 Resource 的 Policy 对象；§2.4 增包络校准责任与离线保守规则 |
| 08 MVP 计划 | MVP 边界改为 MCP 守卫 + Level-1 Runtime 优先；首个闭环加入 Approval 步骤；M2 拆为 M2a / M2b；M1/M3/M4 验收补充；度量增人工介入率、回滚率、验证独立性比例；非声明补两条；开放问题更新为 10 项 |
| 09 威胁模型与 conformance 声明（新建） | 资产、信任边界（含 mermaid 图）、T1–T10 攻击向量与机制、明确不防、负例向量索引（`NV-T<n>-<序号>` 占位与里程碑归属）、MVP conformance 声明 |
| docs/adr/README.md | 索引加入 ADR-0002 ~ ADR-0006 |
| docs/research/sources.md | 追加审查报告附录 A 的论文、协议规范、工业实践来源；探讨记录加入审查报告 |
| docs/research/20260907-frontier-review-and-recommendations.md | 未修改（保留为历史记录） |

### 报告中"可选"级建议的处理

全部落地：01 §7 Runtime 行、02 `assurance` 字段（Effect 候选与 Verification）、03 §8 可观测性。报告 §5.2 标注"不在本次落地"的威胁模型提纲按 owner 要求扩展为正式文档 09。

### 与报告不一致或报告未明确之处的决定

- 报告 §3.4 与 §5.3 都使用 `assurance` 一词但形状不同：Verification 的汇总结论字段定名为 `verdict`（`pass / pass_with_warning / fail / inconclusive`），`assurance{level, basis[], dependence}` 保留为可选字段，可同时出现在 Effect 候选与 Verification 上。
- Handoff `accepted` 不再是顶层状态，作为 `working` 的 `phase` 属性保留。
- `cancelled` 全仓库统一英式拼写，与 A2A `TASK_STATE_CANCELED` 的对应在对照表中注明。
- Execution 状态集不变；等待 Approval 时 Execution 为 `waiting`、所属 Activity 为 `input_required`。
- `fork_execution` 从最小生命周期 RPC 移除，升为控制点 `fork`（扩展调用名保留 `runtime.fork_execution`）。
- 新增事件 `resource/quarantined` 承接原 Composition ABI 的 quarantine 语义；Runtime 不得调用列表新增 `decide_approval`。
- Level-1 下无法归因的 Effect 参数值记为 `untrusted_external`（fail-closed），以 ADR-0004 为准。

## v0.5.0

- 设计日期：2026-09-06；归档日期：2026-09-07
- 内容：FysunOS 首轮设计稿（文内标注"设计稿 v0.1"）——定义与八条原则、底线 F1–F8、对象模型、四子系统架构、
  Capability API / Harness ABI / Composition ABI、DSH 作为默认参考 Runtime 的 `fysun` bundle 接入、迁移清单、
  演进与边缘、MVP 计划，以及 ADR-0001 创立决策与来源登记。
- 归档位置：[docs/archive/v0.5.0/](archive/v0.5.0/ARCHIVE.md)
