# 04 · Agent as a Service 与自进化

- 状态：**提案草案**
- 日期：2026-09-12
- 本文件不替代现行 [`docs/redefinition`](../redefinition/README.md) 直至被采纳。
- 决策：[00](00-decisions.md) D1/D3/D6/D7。治理规则继承 v0.7.0 与 redefinition 06 中未与本提案冲突的部分。

## A. 服务平面

### A1. 结论

对外，Agent 是 **Service**：某个 Lock + 一组 Endpoint 绑定 + 一组 Plan。调用经 `fysun-gateway` 变成租户 Scope 下的 Activity/Execution。基座做发布/发现/调用/计量事件/版本/回滚；**不做**出账、定价引擎、聊天 UI。

一级对象只有 Service。其余：

| 名称 | 层级 | 谁写 | 谁读 | 热路径 | MVP |
|---|---|---|---|---|---|
| Service | 服务一级 | 发布者经 `fysund` | 发现、gateway、Policy | 否 | R2 |
| versions[] | Service 字段 | 发布/rollout | gateway 路由 | 路由在入口 EP | R2 |
| endpoints[] | Service 绑定 | 发布者 | gateway | 是（入口） | R2 |
| Tenant | 治理 | 运营者/自助注册 | 全部 Scope 过滤 | 鉴权在入口 | R2（R1 可单租户） |
| Plan | 附着 | `plan.attach`（准入） | Grant 派生、配额 | 配额在入口本地快照 | R2 |
| meter 事件 | 账本事件 | EP / `fysund` 折叠 | `meter.query`、导出 | 写入可异步 | R2 |

角色仍是发布者 / 运营者 / 租户 / 审核者。单机 Personal：一个 Tenant，Plan 可为 unlimited；允许 CLI `activity.create` 绕过 gateway，但 Scope 仍存在。

### A2. 发布、发现、调用

```text
compose.resolve → Lock
service.publish(service, lock_digest, endpoints[], plans[])
  → Policy（谁可发布何种 effect_category、签名、证据门）
  → 可选 evidence_gate（回归集 Verification 全 pass 才允许标 stable）
  → 签名 Agent Card（skills[] ↔ capabilities[]）
  → service/published
service.rollout(staged | canary{percent} | stable | rolled_back)
```

发现：`service.search` 返回摘要 + Agent Card + 分层证据 + health。可见 ≠ 可调用。MCP facade：每个 Capability 一个工具，长任务走 MCP Tasks / A2A 异步。

调用路径（热路径不含 `fysund` 转发）：

```text
消费者 → gateway（鉴权、Plan 配额本地评估）
       → fysund：invocation/received；create_or_wake Activity；签发衰减 Grant；match Runtime
       → Runtime.start_execution(lock, 结构化绑定)
       → Runtime ↔ 工具 EP（本地快照）
       → 回报与进度回 fysund；gateway 映射 A2A TaskState
```

`session_mode: per_call | per_context`。A2A `contextId` ↔ Activity.id。`input_required` ↔ 用户输入；`auth_required` ↔ Approval。运行中 Execution 不随 rollback 迁移。

### A3. 身份、委托、长任务、HITL

- **身份**：Tenant 凭据终止于 gateway。Execution 是 Principal，可被授权。不把 SPIFFE 写入对象模型（D8）。
- **委托链**：Handoff / 外部 A2A 调用签发衰减 Grant；跨租户只能经 Service 调用，禁止直接引用对方 Execution。外部 Artifact 一律 `untrusted_external`（ASI07/ASI03）。
- **长任务**：Activity 可无终点。长时间 HITL 用 `park` 释放 Runtime（[03](03-agent-and-runtime.md) §5）。恢复必再武装。
- **HITL**：Approval 结果集仍为 `approved_once / rejected / cancelled / unavailable`；审批面只由 `fysund` 提供（T11）。gateway 只透传 `needs_human(approval_id)`。Harness 文本不构成审批。

审批路由（Plan 可配）：

| 触发 | 缺省 |
|---|---|
| 消费者数据上的外部 mutation | 租户 contacts |
| Grant 扩张 / 扩权 Adjustment | 发布者 + 租户双签 |
| Memory 准入 | 租户 |
| 供应链解除隔离 | 运营者 |

### A4. 计量与配额

Plan → 根 Grant（tokens / calls / effects{category} / runtime_seconds / concurrent_executions / approvals / sampling）。Execution Grant 从根衰减。超限 → `quota.exceeded`；不得再 `execution/started`。

计量来源：gateway（calls、并发）、工具 EP（effects）、`fysund`（时长、审批）、可选 provider_proxy（tokens）。Harness 自报 token 标 `untrusted_external`，Policy 决定是否计入。

导出 OTLP / 批量；价格只在 `model.pricing_ref` 作提示。

### A4.1 OWASP Agentic Top 10（2026）对照

来源：[OWASP GenAI 下载页](https://genai.owasp.org/download/52117)（检索 2026-09-12）。本表是覆盖声明，不是另造分类学。

| ID | 风险 | 本提案控制 |
|---|---|---|
| ASI01 目标劫持 | 注入改写目标 | 来源标签；prompt 变量完整性约束；Knowledge 缺省完整性；F2 候选≠事实 |
| ASI02 工具滥用 | 合法工具被链式误用 | 绑定期工具面；F9；未分类 = mutation；sampling 预算 |
| ASI03 身份与权限滥用 | 委托链过宽、旧凭据 | Execution Principal；Grant 只衰减；Secret 每次重解析；跨租户禁直连 Execution |
| ASI04 供应链 | 第三方 skill/MCP/harness | digest、签名 Policy、snapshot 漂移、quarantine/revoke 级联 |
| ASI05 意外代码执行 | Agent 生成代码 | 原生工具须声明；无 EP 则 `not_covered`；沙箱不在内核 |
| ASI06 记忆/上下文投毒 | 脏记忆污染后续 | Memory 准入；工具结果 `untrusted_external`；租户命名空间 |
| ASI07 不安全 A2A | 伪造/篡改委派 | 签名 Agent Card；外部产物 untrusted；内部 Handoff 走账本 |
| ASI08 级联失败 | 单点扩成全局 | fencing；预算；审批限流；canary；revoke 停新流量 |
| ASI09 HITL 利用 | 骗过审批人 | 审批面只在 `fysund`；不复制参数进卡片；Harness 文本无效 |
| ASI10 Rogue | 漂移超范围 | 资格化纪元；F9；优化器包络；quarantine |

### A5. 服务负例

1. 未 published 的 Lock 可被 Endpoint 调用。
2. 租户 A 读到租户 B 的 Memory / Artifact / Secret。
3. Plan 禁止的 `effect_category` 被 EP 放行。
4. `quota.exceeded` 后仍 `execution/started`。
5. Agent Card `skills[]` 与 Lock `capabilities[]` 不一致。
6. 资产 `revoked` 后 version 仍 `stable`。
7. gateway 本地 `approval/decided`。
8. canary 比例与 `service/rollout` 事件对不上。

## B. 自进化最小闭环

### B1. 结论

内核提供 **证据 → 归属 → Adjustment → 门控发布 → 回滚**。不内置 ACE/GEPA/DGM。优化器是可插拔生产者：读轨迹与 Verification，写 **profile 层** 候选 Adjustment。

这与 2026 文献同构：ACE 改上下文 playbook（[arXiv:2510.04618](https://arxiv.org/abs/2510.04618)）；GEPA 改 prompt 并维持 Pareto 前沿；DGM 改脚手架但用基准门（[arXiv:2505.22954](https://arxiv.org/abs/2505.22954)）。OS 的工作是让这些改动可追溯、可限制、可回滚，而不是实现搜索算法。

### B2. 证据模型

Verification 仍是唯一「好不好」对象（不设点赞）。归属四层：

| 层 | 何时失效 |
|---|---|
| Capability | 契约变更 |
| Agent Definition | 组合结构变更（不只是 profile） |
| Lock.contract / Lock.substrate | 对应 digest 变 |
| Lock.profile | 微调；**旧证据可 `inherited`** |
| 资产版本 | 按 `attributable_to` |

`downstream_outcome` 归 Service。conformance 归 harness / EP / Runtime 实例。配置声明与 `compatibility` 不是证据。

质量表示仍用 Skill Contract `(P,O,A,V,F)`；V 的 checklist 项标 `verifier_kind`。`inconclusive ≠ pass`。judge 噪声已知（RuVerBench）；部署门需要 `human` 或已校准的 `model_judge`，阈值不在本文件写死。

### B3. 闭环步骤

```text
Execution 结束 / 失败 / 低分 Verification
  → 轨迹（会话账本引用 + 治理事件 + checklist 结果）入 Artifact
  → attribution[]：prompt | skill | loop | tool_surface | harness | model | 环境
     （可由人填写，或由优化器提议、人/Policy 确认）
  → Adjustment 候选（默认 profile 层）
  → 安全包络检查（下表）
  → 证据门：回归集 + 可选影子 Execution（同一 Lock 契约/基座，仅换 profile）
  → service.rollout canary → 指标（通过率、成本、人工介入率）→ stable 或 rollback
```

| 可变（Mutable） | 不可变（除非 Approval + 新 substrate/contract） |
|---|---|
| Prompt 正文与 few-shot | `bounds.tool_effects[]` 升级 |
| Loop params（步数、重试、compaction 阈值） | 新增工具 / MCP server / 放宽 `reject_native_tools` |
| 检索 top-k、记忆抽取阈值 | Grant、配额、Plan `allowed_effect_categories` |
| 在已允许槽位内换 prompt 版本 | 设备安全、secret 范围 |

影子评估：用已 `verified` 的脱敏输入重放。未达 Policy 阈值不得 `stable`。A/B 即 canary 按租户或调用比例，路由必须可审计（A5.8）。

回滚 = `service.rollout(prev, stable)`。优化器失败不得留下半发布 Lock。租户 A 的 Adjustment 不传播到租户 B（除非发布者显式 promote 到 Definition 默认）。

### B4. 优化器插件边界

```text
optimizer.describe() → {targets: [prompt, loop_profile], needs: [trajectory, verification]}
optimizer.propose(lock, evidence) → Adjustment[]     # 只许 profile
fysund：包络检查 → 可选自动 apply(this_activity) 或排队 Approval
```

优化器：不是 PDP；不能 `grant.issue`；不能标 Verification `pass`；不能读 Secret 明文。登记为普通进程或 harness 资产。内核提供 `trajectory.export`（脱敏）与 `adjustment.apply`。

### B5. 演进负例

1. 契约/基座未变，系统却把旧 Verification 从发现界面抹掉（应保留 `inherited`）。
2. `inherited` 证据被当作新 substrate 的 `evidence_gate` pass。
3. 优化器把 `external_read` 改成更低类别，或加入新工具，未经 Approval。
4. 影子集未跑过的 `stable` rollout（当 Policy 开启证据门时）。
5. 独立于 Verification 的「评分」对象入账本。
6. 租户 A 的反馈自动生效于租户 B。
