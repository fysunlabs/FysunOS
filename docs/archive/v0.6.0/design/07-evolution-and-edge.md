# 07 · 演进与边缘

- 状态：设计稿 v0.6.0（2026-09-07）
- 变更依据：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) 与 ADR-0002 ~ ADR-0006
- 前置：[02 对象模型](02-object-model.md)（Capability、Adjustment、Policy、Verification）、[03 架构](03-architecture.md)
- 决策依据：[ADR-0001](../adr/0001-founding-decisions.md)、[ADR-0003](../adr/0003-policy-object-and-language.md)、[ADR-0005](../adr/0005-approval-and-evidence-gated-evolution.md)

这两个主题是对象模型的两条**设计检验线**：如果对象模型只适合桌面数字员工，它们会最先暴露
问题。二者都不是 MVP 必须完整实现的功能。

## 1. 演进：轻量、可回退、只在风险边界加检查

### 1.1 目标

用户校准与运行反馈能够改善后续行为，同时保留变化记录与恢复手段；**不**要求每次调整都经过
候选 → 评估 → 灰度 → 晋升的重流程，也**不**因能力频繁变化而不断创建新身份。

### 1.2 三种变化的区分

| 变化 | 载体 | 生效范围 | 是否需检查 |
|---|---|---|---|
| **运行时适应** | `Adjustment{scope: this_execution}` | 仅当前 Execution | 否（自动留痕） |
| **活动级调整** | `Adjustment{scope: this_activity}` | 该 Activity 的后续 Execution | 否，除非触及风险边界 |
| **能力默认变化** | `Adjustment{scope: capability_default}` → Capability 新实现版本或新契约版本 | 所有后续使用 | 契约不兼容时默认派生新 Capability；`evidence_gate` 开启时晋升前要求回归评估 |
| 模型或策略训练 | 不在 FysunOS 范围 | — | 独立流程 |

### 1.3 规则

- **本次任务这样做 ≠ 以后都这样做。** 临时调整与持久调整分开，但晋升只是一次 `adjustment.promote`，
  不是流水线。
- **契约保持、内部变化** → 同一 Capability 的新实现版本（单 Agent 变多 Agent、远端模型换
  本地模型、固定 Prompt 换检索驱动，都属此类）。
- **契约改变、职责扩大** → 默认派生新 Capability，用 `derived_from / evolved_from / forked_from`
  保留谱系；不把"能力已经变了"藏在版本号里。
- **正在运行的 Execution 绑定明确的 Capability 快照。** 新版本用于后续 Execution，或经 Policy 允许
  在安全边界内热切换；"培训员工"的一次修改不能意外影响多个进行中的活动。
- **自校正与用户反馈不能直接改写风险边界。** 触及 Grant 扩权、新外部动作、预算上调、设备安全
  的 Adjustment 自动标为高风险类，执行部署 Policy；其余直接生效。扩权唯一路径是 Approval（F9）：
  `approve_for_capability` 派生的 Adjustment 是唯一能把权限边界写进 `capability_default` 的来源。
- **每次 Adjustment 带可证伪预测。** `predicted_effect` 声明预期改善什么（`{statement, metric?, direction?}`）；
  后续 Verification 到达时 daemon 自动填充 `evaluated_against`（`supported` / `refuted` / `inconclusive`），
  事件为 `adjustment/predicted` 与 `adjustment/evaluated`。单次成功不等于新配置更好，但"是否更好"的
  对比现在由系统回答而非用户。
- **证据与配置分离。** Adjustment 生效后，后续 Verification 结果自动记为该实现版本的 evidence；
  evidence 与 `evaluated_against` 都只引用 Verification，不复制结论。

### 1.4 系统提供的机制（仅此）

记录、应用、观察、回退、晋升、谱系、**预测记录、评估对照、可选证据门控、健康查询**。

| 机制 | 内容 | 事件 / 接口 |
|---|---|---|
| 预测记录 | `adjustment.apply` 与 `os.propose_adjustment` 接受 `predicted_effect`；缺省允许为空，但为空时 `evaluated_against` 恒为 `inconclusive` | `adjustment/predicted` |
| 评估对照 | 生效后 N 次 Execution 的 Verification 汇总与预测对照；窗口 `window` 由 Policy 或调用方给定，缺省为下一次 Execution | `adjustment/evaluated` |
| 证据门控 | `evidence_gate` 由 Policy 或用户开启；`promote` 到 `capability_default` 前要求对**回归集**跑一次评估，结果记入 `evidence_gate.result`；未跑或 `fail` 则 promote 被拒。回归集 = 该 Capability 历史 `verified` Execution 的输入与验收 checklist，自然形成，不需人工构造（`regression_set_ref`） | `adjustment/promoted` 或拒绝 |
| 健康查询 | `capability.health(id)` 从 evidence 与 Adjustment 历史折叠三类信号：**冗余**（与另一 Capability 契约近似）、**过期**（evidence 全部来自已退役 Runtime，`runtime/detached` 后无新 evidence）、**反复失败的 checklist 项**（同一 `items[]` 项在多次 Verification 中 `fail`） | 只读查询，不产生事件 |

**改进策略本身**（如何归因、如何生成候选、如何评估）仍是普通 Capability 或应用层扩展，不进内核。
上表只是让"改进策略作为普通 Capability"可行的最小系统支持。

### 1.5 质量目标的表示法

回答 [08 开放问题 4](08-mvp-plan.md#7-开放问题待后续-adr)："怎样算做好"如何机器可判定。

`Capability.contract.quality` 采用 Skill Contract 骨架 `(P 前置条件, O 操作, A 产物, V 验证器, F 已知失败模式)`
（SkillOps，arXiv:2605.13716）：

| 分量 | 内容 | 谁写 |
|---|---|---|
| P 前置条件 | 输入必须满足什么；不满足时 Capability 应拒绝而非降质 | 用户与 Agent 对话生成 |
| O 操作 | 做什么（自然语言，P1） | 同上 |
| A 产物 | 产出的 Artifact 类型与 schema 引用 | 同上；schema 由系统结构化 |
| **V 验证器** | **checklist**：分解为可独立判定的是/否项，每项标注 `verifier_kind`（`deterministic` / `model_judge` / `human`） | 用户与 Agent 生成项目文字；系统只结构化项与 verifier 类型 |
| F 已知失败模式 | 已观察到的失败方式，来自 `capability.health` 反复失败项与用户反馈 | 系统折叠 + 用户补充 |

V 的规则：

- 每一项独立判定，不允许"整体感觉良好"式的单项；Verification 的 `items[]` 与 V 的项一一对应，
  `rubric_ref` 指向 V 的版本。
- `model_judge` 项何时可替代 `human`：该 judge 在校准集上的一致性（Krippendorff α）达到阈值——
  探索用途 ≥ 0.67、安全相关 ≥ 0.70、部署门控 ≥ 0.80（arXiv:2608.00794）；Verification 通过
  `calibration_ref` 引用校准集与 α，达不到或缺失记 `uncalibrated`，此时该项等同于未由 judge 判定。
- Policy 可为 Effect 类别或 Capability 设定最低 verifier 要求（例如"发布类动作至少 `deterministic` +
  `human`"）；达不到即 `verdict = inconclusive`，不是通过。
- `downstream_outcome` 不出现在 V 里；它是事后 evidence 的来源，不是验收项。

示例（"趋势调研 → 内容包"能力的 V）：

| # | checklist 项 | `verifier_kind` | 判定依据 |
|---|---|---|---|
| 1 | 内容包符合 `content_pack` schema | `deterministic` | schema 校验 |
| 2 | 每条趋势引用至少一个可访问来源 URL | `deterministic` | 来源引用解析 + HTTP 状态读取 |
| 3 | 来源 URL 全部来自 Policy 允许的域名集合 | `deterministic` | 与 Policy 对照 |
| 4 | 正文语气符合"克制、不使用营销词"要求 | `model_judge` | `calibration_ref` 指向语气校准集，α ≥ 0.70 |
| 5 | 内容可以对外发布 | `human` | 用户在 Approval 中的决定即为此项的 Verification |

### 1.6 与技能库生命周期研究的对照

对照 *Dynamic Agent Skills* 八阶段生命周期（arXiv:2607.10113）：

| 阶段 | FysunOS 承载 |
|---|---|
| 证据获取 | Verification → `Capability.evidence` |
| 提案 | Adjustment / `os.propose_adjustment` |
| 验证/准入 | 风险边界 Policy 检查 + 可选 `evidence_gate` |
| 存储 | Capability 版本 + 实现包 digest（Agent Skills 目录） |
| 检索/组合 | `capability.find`、Handoff；OPC 场景库小，平面检索退化暂不成问题 |
| 维护 | `capability.health(id)` |
| 蒸馏/可移植 | 契约与实现分离、Skills 包、Runtime ABI |
| 治理 | 谱系、回退、`risk_class`、Policy 版本 |

## 2. 边缘：同一对象模型，不同 Runtime 与策略

### 2.1 边缘场景对模型的要求

自动灌溉、自动采摘、设备异常诊断这类场景不是"把云端 Agent 部署到边缘"，而是**能力在不同
节点上的部署与协同**。它们要求：

- 长期活动（持续感知、事件响应、维持状态）而非一次性任务 → **Activity**；
- 能力声明延迟、吞吐、能耗、存储、离线行为、可接受不确定性 → **Capability.requirements**；
- 目标环境是否满足、能否用本地模型/规则模块/远端调用替代、不满足时明确拒绝 → **环境匹配**；
- 执行器动作是不可逆外部效果 → **ActionIntent / Effect + 物理扩展字段**；
- 动作边界随设备而不同 → **绑定到设备 Resource 的包络 Policy**；
- 云端与边缘之间的分工 → **Handoff**（而不是进程迁移）。

### 2.2 边缘部署形态

```text
中心节点  fysund  ──── Handoff / 产物引用 / 包络 Policy 版本下发 ────  边缘节点  fysund-edge
                                                                       ├── 确定性包络校验器
                                                                       └── 边缘 Runtime（Rust/Python）
                                                                             ├── 决策核心：本地小模型 / 规则 / 远端调用
                                                                             └── 适配器：传感器、执行器、网关（MCP server）
```

- `fysund-edge` 是同一账本模型的精简实现：本地 Activity/Execution、本地 Grant 与预算、本地
  Effect 协议、本地 Policy（含包络 Policy）；联网后与中心节点交换 Handoff 与证据。
- 边缘 Runtime 实现 Runtime ABI 最小面（MCP 客户端 + 最小生命周期 RPC）；决策核心可以完全不是 LLM。
- "策略下发" = 下发包络 Policy 的版本（`policy/loaded` 在边缘账本落地）；不下发模型或进程。
- 每个边缘能力独立资格化；软件模拟中的成功不等于硬件动作已通过验证。

### 2.3 物理 Effect 扩展

**安全包络是绑定到设备 Resource 的 Policy 对象**，不是 Effect 字段：

| 层 | 字段 | 说明 |
|---|---|---|
| 包络 Policy（绑定设备 Resource） | `velocity_bounds`、`geofence`、`interface_allowlist`、`rate_limit`、`deadman`、`emergency_stop_ref` | 版本化、带 `source` 与校准记录；同一能力在不同设备上自然适用不同包络 |
| Effect（物理扩展字段） | `envelope_ref`、`envelope_check` | 引用所用包络 Policy 版本与校验结果（通过 / 拒绝项） |
| Effect（物理扩展字段） | `deadline`、`sensor_timestamp`、`uncertainty`、`reversibility_class`、`actuator_receipt`、`physical_post_state_verifier` | 保留在 Effect 上 |

- 校验器在 `fysund-edge` 内以**确定性代码**运行，不经模型；候选动作不满足包络即 `denied.policy`，
  `envelope_check` 记录拒绝项。
- Policy 可要求高 `reversibility_class` 动作必须人工确认（Approval）或双源传感器确认。
- `emergency_stop_ref` 指向宿主控制器的急停接口；FysunOS 只记录引用与触发事件，不实现急停。

### 2.4 明确边界

- **硬实时不在 FysunOS 内。** 运动控制、急停、时序安全由宿主控制系统（PLC、RTOS、ROS 2
  实时控制器）承担；FysunOS 管理感知解释、决策与动作请求，并记录 receipt。
- **包络声明的校准是部署者责任。** 校验器只能保证动作符合**声明的**包络，不能保证声明与世界
  一致（"通过校验 ≠ 世界安全"）；Policy 模型要求包络声明带来源与校准记录，缺失时 Policy 载入
  仍成功但 `capability.match` 对该设备返回 `not_runnable(missing[])`。
- **离线保守规则。** `fysund-edge` 离线期间只允许本地 Policy 已预授权的 Effect 类别；需要
  Approval 的一律阻塞（Execution 进入 `waiting`，Activity 进入 `input_required`）；联网后本地
  事件按 `seq` 合并到中心账本，冲突规则待 ADR（[08 开放问题 9](08-mvp-plan.md#7-开放问题待后续-adr)）。
- **不承诺透明迁移。** 模型内部状态、任意进程、凭据与设备权限不随能力包迁移；跨环境复用
  的单位是 Capability 契约 + 目标环境可用的实现 + 重新签发的 Grant。
- **算力下沉不自动提供实时保证。** 边缘只是减少网络依赖。

### 2.5 可移植性的三个层次（分别承诺）

| 层次 | 含义 | FysunOS 承诺 |
|---|---|---|
| 能力可移植 | 同一契约与实现包在兼容环境部署 | 是：任何 Level-1 Runtime + 满足 `requirements` 的工具面（经环境匹配） |
| 执行可恢复 | 活动进行到一半，携带必要状态在另一环境重新绑定资源并继续 | 部分：以 Activity.continuity 与产物引用为单位，不搬会话与进程 |
| 行为适配 | 换模型、工具、设备后仍满足契约 | 是（换实现，重新验证） |

三者难度不同，不合并为"透明迁移"。
