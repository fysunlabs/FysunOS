# 03 · 系统架构

- 状态：设计稿 v0.1（2026-09-06）
- 前置：[01 定义](01-definition-and-principles.md)、[02 对象模型](02-object-model.md)
- 决策依据：[ADR-0001](../adr/0001-founding-decisions.md)（Rust daemon、event-sourced 账本、DSH 为默认 Runtime）

## 1. 分层

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ 应用层   Personal / OPC 数字员工 · IoT 应用 · Agent Shell · 其他产品        │
├──────────────────────── Capability API ─────────────────────────────────┤
│ FysunOS                                                                 │
│   Capability 子系统   契约注册、谱系与证据、环境匹配                        │
│   Activity   子系统   Activity/Execution 生命周期、调度与触发、检查点、Handoff │
│   Authority  子系统   Principal/Scope、Grant/预算、Intent/Effect、Verifier、账本 │
│   Resource   子系统   模型代理、Secret Store、上下文/记忆/知识、工具与设备目录、计量 │
├──────────── Harness ABI（Runtime ↔ OS）· Composition ABI（组件图 ↔ OS）───┤
│ Runtime 层  DSH（默认参考 Runtime，Cordis 组合）· 边缘 Runtime · 经 DSH      │
│            subagent provider 接入的 Codex / Claude Code / ACP 等             │
└─────────────────────────────────────────────────────────────────────────┘
```

应用通过 Capability API 使用系统；对话助手、GUI、CLI、无人值守服务都只是入口，不决定架构。
Runtime 通过 Harness ABI 接入；OS 不知道也不关心 Runtime 内部的 loop 与 prompt。

## 2. 四个子系统

### 2.1 Capability 子系统

- 注册与解析 Capability 契约、实现引用、运行要求；维护谱系与证据。
- **环境匹配**：把 `requirements` 对照目标环境的实际资源与 Runtime 控制点——满足则运行，
  可替代则重绑定（换模型/工具/Runtime），缺少必要条件则**明确报告**不可运行或部分满足。
  不设兼容等级制度，不静默降质。
- 提供发现接口：应用或 Agent Shell 可以按行为描述查找可用能力。

### 2.2 Activity 子系统

- Activity / Execution 生命周期状态机；触发（人工、事件、周期、webhook）与调度。
- 检查点与恢复：恢复顺序固定——先读账本投影，再校验 `fencing_epoch`，再对账 outcome-unknown
  的 Effect，最后才通知 Runtime `resume_execution`。
- Handoff 的签发、接受、交付、验证关联。
- 等待、取消、超时、预算耗尽的统一处理；daemon 拥有 wait/switch/escalate 策略，不允许
  Runtime 静默无限重试。

### 2.3 Authority 子系统

- Principal/Scope 管理；Grant 签发、衰减、撤销、预算记账。
- Intent/Effect/Receipt 协议执行；unknown-outcome 的 reconcile / quarantine。
- 独立 Verifier 调度（确定性检查优先）。
- 事件账本的唯一写入者；所有投影由它折叠。
- 部署策略引擎：读取环境声明的策略（什么可自动执行、何时需人工），对候选做准入。

### 2.4 Resource 子系统

- Provider 代理：Runtime 的模型请求经 daemon 转发，daemon 注入凭据、计量 Token 与费用。
- Secret Store：批准的后端（OS keyring / 文件加密 / 硬件），只对外暴露 `SecretRef`。
- 上下文装配服务：按 Grant 与预算过滤候选片段（authorize-before-rank），产出带 digest 与
  显式损失说明的受控上下文视图，交 Runtime 渲染。
- 记忆/知识准入：Runtime 提交的记忆候选只有经准入才成为 durable 知识。
- 工具/适配器/设备目录：描述符、可用性、隔离状态、动态生命周期事件。

## 3. 进程拓扑

```text
                 ┌──────────────┐        ┌──────────────────────────────┐
  fysun CLI ────▶│              │◀──────▶│ DSH Runtime（子进程）           │
  Web UI    ────▶│   fysund     │  sdk/  │  dsh --profile fysun          │
  Agent Shell ──▶│  (Rust)      │  acp   │  = dsh-base + fysun bundle    │
  应用 SDK  ────▶│              │        └──────────────────────────────┘
                 │  SQLite WAL  │◀──────▶ 其他 Runtime（边缘 / 自研 / 第三方）
                 │  事件账本    │
                 └──────┬───────┘
                        │ Provider 代理（凭据只在此注入）
                        ▼
                  模型服务 / 外部 API
```

- `fysund` 是唯一权威写入者，本机 owner-local、单主体启动（MVP），预留多主体与多节点的
  Scope 模型但不预建 RBAC。
- Runtime 是 `fysund` 管理的**隔离子进程**，经 stdio/本地 socket 的 JSON-RPC（DSH 的 `sdk`
  profile）或 ACP 驱动；不进程内链接，不共享地址空间。
- Web UI 由 daemon 静态服务；Agent Shell 是运行在 DSH 上的一个普通 Capability 组合，没有
  特殊系统身份，其可操作范围由 Grant 决定。
- 边缘节点运行 `fysund-edge`（同一账本模型的精简实现）与边缘 Runtime；与中心节点之间通过
  Handoff 与产物引用协作，不承诺进程或模型状态迁移。

## 4. 双账本

| 账本 | 拥有者 | 内容 | 真源范围 |
|---|---|---|---|
| **治理账本** | `fysund` | Principal/Scope、Capability、Activity/Execution、Grant、Intent/Effect/Receipt、Verification、Handoff、Adjustment、Resource 生命周期 | 一切治理事实 |
| **会话账本** | Runtime（DSH session log） | 模型消息、工具调用与结果原文、chunk、compaction、fork 谱系 | 模型所见与所答 |

关联字段：`execution_id`、`session_id`、`task/handoff_id`、`causation_id`、`correlation_id`、
`fencing_epoch`。两账本**关联不合并**：合并会让 OS 真源依附于某个 harness；不关联会产生
双重状态与重复恢复逻辑。跨账本 replay 测试是必备验证。

## 5. Event-sourced 治理账本

采用**有边界的 event-sourced** 模型（ADR-0001）：

- 事件是唯一真源；状态是投影；缓存/索引是派生数据，可删可重建，带 watermark 与 digest。
- **边界**：模型流式 chunk、KV cache、高频传感器采样、Runtime 内部步骤不进入治理账本；
  它们属于会话账本或遥测。
- SQLite WAL 单文件存储；事件表 append-only；投影表可重建。schema 变更走编号迁移，只追加
  步骤不改写已合并步骤。
- 每次 replay 产生的投影 digest 必须稳定，作为回归测试的客观面。

### 5.1 事件族（最小集）

```text
principal/created | principal/revoked
scope/defined | scope/revised

capability/registered | capability/revised | capability/derived
capability/evidence-recorded | capability/withdrawn

activity/created | activity/activated | activity/waiting | activity/paused
activity/resumed | activity/blocked | activity/completed | activity/cancelled
activity/policy-bound | activity/triggered

execution/started | execution/checkpointed | execution/resumed | execution/forked
execution/finished | execution/failed | execution/cancelled

handoff/issued | handoff/accepted | handoff/delivered | handoff/verified
handoff/rejected | handoff/cancelled

grant/issued | grant/attenuated | grant/consumed | grant/revoked

intent/recorded | intent/rejected | intent/replayed
effect/authorized | effect/dispatch-started | effect/receipt
effect/outcome-unknown | effect/reconciled | effect/quarantined | effect/compensated

verification/requested | verification/reported

adjustment/applied | adjustment/reverted | adjustment/promoted

resource/registered | resource/bound | resource/metered | resource/unavailable
runtime/attached | runtime/health | runtime/detached
```

每个事件携带 `seq`、`time`、`principal`、`causation_id`、`correlation_id`，与 Execution 相关的
事件另带 `execution_id` 与 `fencing_epoch`。

## 6. 恢复模型

1. 进程启动：读取账本，重建投影，校验 digest。
2. 对每个非终态 Execution：读取 `fencing_epoch`，递增得到新 epoch；旧 epoch 的后续写入被拒。
3. 对所有 `dispatch-started` 但无 `receipt` 的 Effect：进入 outcome-unknown，按适配器提供的
   对账能力 reconcile；无法对账的进入 quarantine 并阻塞相关 Activity（`blocked` 带恢复动作）。
4. 只有对账完成后，才向 Runtime 发 `resume_execution`（携带 OS 侧 Effect 水位）。
5. Activity 是否自动恢复由 `policy_binding` 决定；缺省策略是"恢复等待态、不自动发起新外部
   动作、等待触发"。

崩溃后**不是一律重跑**；外部结果未知时先对账；不承诺无条件 exactly-once。

## 7. 部署策略模型

部署环境提供一份策略声明（MVP 用本地文件），至少回答：

- 哪些 Effect 类别可自动执行、哪些需人工确认、哪些禁止；
- 预算与速率上限；
- 允许的 Runtime 与 Resource 类型；
- Activity 崩溃后是否自动恢复。

OS 只负责执行策略边界；不内置统一风险等级。产品（如 Personal）可以提供自己的默认策略
（例如"低风险自动、高风险确认"），但那是应用层选择。
