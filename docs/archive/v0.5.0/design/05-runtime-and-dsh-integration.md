# 05 · Runtime 与 DSH 接入

- 状态：设计稿 v0.1（2026-09-06）
- 前置：[04 ABI](04-abi.md)
- 决策依据：[ADR-0001](../adr/0001-founding-decisions.md) §D2（DSH 为默认参考 Runtime，进程外接入，不 fork）
- 基线：DeepSeek Harness 本机只读副本 `cd5ef8148158c3a752a658978873241fdf8e2bbc`，vendored
  Cordis 4.0.1（见 [来源登记](../research/sources.md)）

## 1. 为什么 DSH 是默认 Runtime 而不是 OS

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
| 权威底线（Grant、Intent/Effect、独立验证、secret 隔离） | 无，且明确不做 | **OS 拥有** |

把权威层放进 DSH 进程，会让它与模型驱动的工具执行、可自我修改的动态 Cordis 包同居——
Cordis 论文把"自演化 harness 运行中替换自身组件"列为动机场景。这与 F1–F5 直接冲突。
DSH 又处于 developer preview、明言破坏性变更、单语言 Node；把 OS 身份绑在它上面等于放弃
边缘 Runtime 的可能。因此：**DSH = 默认参考 Runtime；FysunOS = 权威 + 活动 + 能力。**

## 2. 接入形态：`fysun` bundle

FysunOS 对 DSH 的接入是一个叠在 `dsh-base` 之上的 bundle，配一个 `fysun` profile：

```text
dsh --profile fysun
  = dsh-base
  + fysun bundle
      ├── fysun-credentials       替换 ctx.credentials provider
      ├── fysun-approval          替换 ctx.approval 应答方
      ├── fysun-session-link      旁挂 ctx.sessionPersistence provider（账本关联）
      ├── fysun-subagent          新增 ctx.subagents provider（OS 调度的委派）
      ├── fysun-tool-gate         订阅 tools/pre-execute → execute → post-execute
      ├── fysun-context           订阅 agent/pre-step，注入受控上下文视图
      └── fysun-rpc               经 sdk profile 的 JSON-RPC 暴露 Harness ABI
```

| DSH 扩展点 | FysunOS 接入 | 对应 ABI |
|---|---|---|
| `tools/pre-execute → tools/execute → tools/post-execute`（waterfall） | 每次 tool call 变成 Effect 候选：pre 阶段 `os.submit_candidate`，未准入则短路返回拒绝；execute 在 `effect/dispatch-started` 之后进行；post 回填 Receipt。只读工具走审计快路径 | `submit_candidate` / `observe_effect` |
| `approval/request`（waterfall） | 由 `fysun-approval` 按部署策略与 Grant 应答；策略要求人工时转 `needs_human`；无策略即 fail-closed | 准入结果 |
| `ctx.credentials` seam | provider 只返回 `SecretRef`；`ctx.llm` 适配器的 Provider 请求经 daemon 代理，明文只在 daemon 注入 | `request_resource` |
| `ctx.sessionPersistence` seam | 在 SQLite/JSONL 后端之外旁挂一个 provider，把 `session_id / execution_id / causation_id / fencing_epoch` 与会话水位提交给 OS | `report_checkpoint` |
| `ctx.subagents` provider | 新增"由 FysunOS 调度"的委派实现：委派产生 Handoff 与子 Execution（可仍由 DSH in-process/ACP/Codex provider 实际执行），协作事实落在 OS 账本 | `request_handoff` |
| `agent/pre-step`（waterfall） | 注入 OS 装配的受控上下文视图（authorize-before-rank、显式损失、digest 绑定）；改写已领取消息 | `request_context` |
| `ctx.sandbox` seam | 视为 containment，不视为安全边界；Effect 协议不因沙箱而放松 | — |
| `dsh --profile sdk` / `acp` | 进程外驱动通道；`fysund` 以子进程方式启动 DSH，通过 JSON-RPC 调用 `start/resume/fork_execution` 等 | OS → Runtime |

DSH 保持原样运行其 loop、prompt、tools 流水线、compaction、session log、Web UI。FysunOS
不改写 DSH 源码；升级 DSH 时维护的是这个 bundle 与一组适配测试。

## 3. 双账本关联

```text
DSH session log（会话账本）                fysund 治理账本
  session/start ─── session_id ─────────▶ execution/started {session_id}
  tool/call ──────── candidate ─────────▶ intent/recorded → effect/authorized
  tool/result ◀───── receipt ───────────  effect/receipt
  step/end ───────── watermark ─────────▶ execution/checkpointed {session_watermark}
  fork ───────────── parentSession ─────▶ execution/forked
  subagent spawn ─── delegation ────────▶ handoff/issued + execution/started(child)
```

- DSH 的 session log 仍是模型所见与所答的真源；OS 不复制消息正文。
- OS 的账本是治理事实的真源；DSH 不能推进 Execution 状态。
- 跨账本 replay 测试：从两侧任一账本出发都能定位另一侧，并且 Effect 与 tool/result 一一配对。

## 4. 明确不从 DSH 吸收为 OS 语义的东西

- 把 Session 当全局真源（OS 有自己的 Activity/Execution）；
- 一次性 turn 级审批当 Grant（OS 的 Grant 有 principal/scope/budget/expiry/revocation）；
- session-local schedule 当调度（OS 的 Activity triggers 跨进程持久）；
- 动态 Cordis 包（`cordis_define/run`）当持久扩展——它只能是 `ephemeral-experiment`，
  不能直接生成 durable Capability 实现；
- Agent Teams 的 "exact live Agent" 对象身份当授权凭据（OS 使用 Execution 作为 Principal）。

## 5. 版本与供应链

- DSH 以**精确版本 + digest** 钉住，作为 OS 管理的 Installed Runtime 登记（版本、来源、健康、
  digest 可见）。
- 破坏性变更时先在隔离环境跑 bundle 适配测试，再升级；bundle 与 DSH 版本一一对应。
- DSH 与 Cordis 均为 MIT；FysunOS 不 vendor、不 fork。

## 6. 其他 Runtime

- **边缘 Runtime**（Rust/Python）：实现同一 Harness ABI 的最小子集（`start/resume/cancel`、
  `submit_candidate`、`report_checkpoint`），决策核心可以是本地小模型、规则模块或远端调用。
- **Codex / Claude Code / ACP**：不直接接 OS，经 DSH 的 subagent provider 接入，被 OS 视为
  DSH Runtime 内的委派目标；其内部状态留在远端进程，OS 只记录 Handoff 与产物。
- 每个新 Runtime 独立资格化；DSH 的证据不可转移到其他 Runtime。

## 7. 代价与对策

| 代价 | 对策 |
|---|---|
| 双账本的关联与 replay 复杂度 | 关联字段固定；跨账本 replay 测试进入 MVP 验收 |
| 进程外 broker 延迟 | 只读操作走审计快路径；Effect 协议只在 mutation 上 |
| DSH 破坏性变更 | 精确钉版；bundle 适配测试；升级前隔离验证 |
| Node 依赖不适合部分边缘设备 | 这正是 OS ≠ DSH 的理由；边缘用另一 Runtime 实现同一 ABI |
