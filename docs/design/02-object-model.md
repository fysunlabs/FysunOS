# 02 · 对象模型

- 状态：设计稿 v0.1（2026-09-06）
- 前置：[01 定义与设计原则](01-definition-and-principles.md)

本文定义 FysunOS 权威账本中的一级对象。字段列表是**设计意图**，不是 schema；正式 schema 在
实现阶段以机器可读形式登记，并配负例向量。

## 1. 对象总览

```text
Principal ──owns──▶ Scope
                      │
   Capability ◀───── declares/uses ───── Application
      │ contract · implementation · requirements · evidence · lineage
      │
   Activity ──runs as──▶ Execution ──binds──▶ { Capability snapshot, Runtime, Resource bindings, Agent Core config }
      │                     │
      │                     ├── Intent ──▶ Effect ──▶ Receipt ──▶ Verification
      │                     ├── Handoff ──▶ (Capability | Execution)
      │                     └── Checkpoint
      │
   Adjustment ──scoped to──▶ { Execution | Activity | Capability default }
      
   Grant ──attenuates──▶ Resource / Capability use, per Principal+Scope, with Budget & epoch
   Event ──folds into──▶ Projection   （账本唯一真源；状态是投影）
   Artifact  内容寻址产物，被 Execution / Handoff / Verification 引用
```

## 2. 一级对象

### 2.1 Capability

| 字段组 | 内容 |
|---|---|
| `id`, `contract_version` | 稳定身份；契约兼容变化递增版本，契约不兼容变化默认产生新 Capability（见谱系） |
| **contract** | `behavior`（输入/输出 schema 或持续行为描述）、`quality`（质量目标与判定方式）、`bounds`（成本、时间、权限、风险边界）、`verification`（Verifier 类型与参数） |
| **implementation** | 当前实现引用：Prompt/Skill/代码包/工具组合/决策核心配置，带 `digest`；一项 Capability 可有多个实现版本 |
| **requirements** | 需要的 Resource 类型与最低条件、Runtime 控制点（Harness ABI 能力位）、环境要求（延迟、离线、能耗等） |
| **evidence** | 运行证据引用：在什么环境、什么输入下的验证结果；配置声明不算证据 |
| **lineage** | `derived_from` / `forked_from` / `evolved_from` 关系与触发原因 |
| `status` | `draft` / `active` / `withdrawn`；不设 candidate→proven 的强制阶梯 |

### 2.2 Activity

| 字段 | 内容 |
|---|---|
| `id`, `principal`, `scope` | 归属 |
| `target` | 有终点的目标（含验收条件）**或**持续服务目标（响应要求、停止条件、观察指标） |
| `constraints` | 预算、时间窗口、允许的资源与外部动作范围（映射为 Grant 请求） |
| `capability_ref` | 主 Capability 及固定的实现快照 |
| `triggers` | 人工、事件、周期、外部 webhook；触发产生新 Execution 或唤醒现有 Execution |
| `policy_binding` | 部署环境策略引用（哪些动作可自动执行、何时需人工） |
| `state` | `created` / `active` / `waiting` / `paused` / `blocked` / `completed` / `cancelled`；`blocked` 携带原因与恢复动作 |
| `continuity` | 跨 Execution 必须保留的产物引用与摘要（不是聊天记录） |

Activity 是**状态，不是调度器**（吸收 DSH goal 的定位），但与 DSH goal 不同：它由 `fysund`
持久拥有，进程重启后按 `policy_binding` 决定是否自动恢复，不强制人工 re-arm。

### 2.3 Execution

| 字段 | 内容 |
|---|---|
| `id`, `activity_id`, `parent_execution_id?` | 归属与 fork/委派谱系 |
| `capability_snapshot` | 本次生效的契约版本与实现 digest |
| `runtime_binding` | Runtime 类型与实例（如 `dsh@<version>#<pid>`）、Runtime 侧会话标识（`session_id`） |
| `resource_bindings` | 模型、SecretRef、上下文源、工具/适配器、设备的实际绑定与 Grant 引用 |
| `agent_core_config` | 决策核心的有效配置（Prompt 装配策略、Loop 策略参数、Skill 集）——以摘要与 digest 记录，正文由 Runtime 持有 |
| `fencing_epoch` | 写入围栏；旧 Execution 恢复后不能覆盖新 epoch 的 Effect |
| `checkpoints[]` | 可恢复点：Runtime 侧检查点引用 + OS 侧已对账 Effect 水位 |
| `state` | `starting` / `running` / `waiting` / `checkpointed` / `finished` / `failed` / `cancelled` |

**Execution 与 Runtime Session 的关系**：一个 Execution 对应 Runtime 中一个（或 fork 出的多个）
Session；Session 日志是 Runtime 的会话账本，Execution 是 OS 的治理账本条目。两者通过
`execution_id / session_id / causation_id / fencing_epoch` 关联，**不合并**（见 [03 架构 §4](03-architecture.md#4-双账本)）。

### 2.4 Principal / Scope

- **Principal**：用户、Runtime 实例、某个 Execution、外部系统。Execution 是可被授权的主体，
  这样委派出去的工作拥有独立身份与预算。
- **Scope**：资源可见与可写的范围（工作区路径、设备组、账号、数据域）。Grant 总是绑定
  `(principal, scope)`。

### 2.5 Grant

| 字段 | 内容 |
|---|---|
| `principal`, `scope`, `resource_or_capability` | 授权对象 |
| `attenuation` | 相对上游 Grant 只能收窄：允许的操作、路径前缀、速率、单次上限 |
| `budget` | 金额/Token/调用次数/时长预算及消耗记账 |
| `expiry`, `revocation_epoch` | 到期与撤销；撤销是事件，立即使后续 Intent 拒绝 |
| `issued_by` | 总是 `fysund`；Runtime 只能 `request_resource`，不能签发 |

委派（Handoff）时子 Execution 的 Grant 由 daemon 从父 Grant 衰减签发，**不会**因转交而扩大。

### 2.6 Intent → Effect → Receipt → Verification

直接迁移 CognitiveOS 的外部效果协议：

```text
候选（Runtime 提交的 tool call / 动作请求）
  -> 策略准入（部署策略 + Grant 检查）
  -> 持久化 Intent（幂等键 = canonical 参数 digest）
  -> 持久化 Effect（状态 authorized）
  -> 在 fencing_epoch 下分发
  -> Receipt（成功/失败）或 outcome-unknown
  -> outcome-unknown 只能 reconcile 或 quarantine，不能假定成功
  -> 固定 post-state
  -> 独立 Verification（按 Capability.contract.verification）
  -> commit / compensate
```

- 只读、可逆的本地观察可走 `NoExternalCommitment` 快路径：仍需描述符、Scope 与审计事件，
  不必持久化 Intent/Effect。
- 文件写入、网络 mutation、凭据使用、设备动作必须走完整协议。
- 物理 Effect 额外字段：deadline、safety envelope、sensor timestamp、uncertainty、
  reversibility class、actuator receipt、physical post-state verifier（见 [07 边缘](07-evolution-and-edge.md)）。

### 2.7 Verification

- 由与执行者不同的 Verifier 执行；优先确定性检查（测试、digest、schema、外部状态读取）；
  model-as-judge 只在机械检查不存在时辅助，且必须在执行者可写面之外。
- 报告不可变；过期由 Verification 生命周期承载，不改写旧报告。
- Verification 结果驱动 Activity 阶段完成，也作为 Capability 的 evidence 记录。

### 2.8 Handoff

当一个 Execution 调用另一个 Capability 或委派给另一个 Agent 时，系统自动记录：

| 字段 | 内容 |
|---|---|
| `from_execution`, `to` | 目标是 Capability（由 OS 解析并启动子 Execution）或已有 Execution |
| `objective`, `inputs[]`, `expected_outputs` | 委派目标、输入产物引用、期望产出与验收关系 |
| `grant` | 衰减后的子 Grant |
| `state` | `issued` / `accepted` / `in_progress` / `delivered` / `verified` / `rejected` / `cancelled` |
| `deliverables[]` | 产物引用与子 Execution 的 Verification 引用 |

Handoff 不要求预先定义团队或工作流；简单调用只产生一次 Handoff 记录。参考 DSH Agent Teams
的 roster / mailbox / task DAG 设计，但真源在 OS 账本而非 Lead Session。参与者完成自己的部分，
不替代父 Execution 对整体产出的检查。

### 2.9 Resource

统一发现、描述、授权、绑定、计量；保留类型语义，不做成同一种对象：

| 类型 | 例子 | 说明 |
|---|---|---|
| 模型服务 | Provider + model | 经 daemon 代理，计量 Token/费用 |
| Secret | `SecretRef` | 只在 Secret Store；Runtime 不见明文 |
| 上下文源 | 文件、URL、会话摘要、检索索引 | 装配前授权（authorize-before-rank），显式损失，digest 绑定 |
| 记忆 / 知识 | 持久已准入的知识 | 与"上下文"严格区分：Memory 是 durable 知识，Context 是一次 Execution 的受控视图 |
| 工具 / 适配器 | 文件系统、浏览器、发布接口、灌溉阀门 | 工具是能力提供方的一种实现形态；工具调用 = Effect 候选 |
| 设备 | 传感器、执行器、边缘节点 | 带延迟、能耗、离线行为描述 |
| 其他 Agent | 可委派的执行体 | 通过 Handoff 使用 |

### 2.10 Adjustment

承载"轻量演进"：

| 字段 | 内容 |
|---|---|
| `target` | 变更对象：Capability 实现/契约、Activity 策略、Execution 的决策核心配置或资源绑定 |
| `scope` | `this_execution` / `this_activity` / `capability_default` |
| `source` | 用户校准、Agent 自校正、评估结果 |
| `effective_from` | 立即 / 下一 Execution / 指定检查点 |
| `rollback_target` | 回退到的前一版本或快照 |
| `risk_class` | 是否触及 Grant 扩权、新外部动作、预算上调、设备安全；决定是否走部署策略检查 |

### 2.11 Event / Projection / Artifact

- **Event**：append-only 治理事实，带 `seq`、`time`、`causation_id`、`correlation_id`、`fencing_epoch`。
- **Projection**：由事件折叠出的当前状态（Activity 视图、预算余额、Grant 有效集、Capability 目录…），
  可删除、可重建、带 source watermark 与 digest。
- **Artifact**：内容寻址（digest）产物引用，payload 存于对象存储或文件系统，账本只存引用。

## 3. 不进入 OS 对象模型的概念

角色、团队、数字员工、Project、Routine、业务流程、Agent Shell、聊天消息、Prompt 正文、
Skill 正文、工具 schema、模型流式 chunk。前者属应用层组合；后者属 Runtime 会话账本。
