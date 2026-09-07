# 06 · 迁移清单

- 状态：设计稿 v0.6.0（2026-09-07）
- 变更依据：[前沿对照审查报告](../research/20260907-frontier-review-and-recommendations.md) 与 ADR-0002 ~ ADR-0006
- 基线：CognitiveOS `agent-kernel@6f562bf1bdb86582955de36eb93e51126573f89c`（分支
  `personal/P14-T04-member-join`）；DSH `cd5ef8148158c3a752a658978873241fdf8e2bbc`；
  Cordis 4.0.1（DSH vendored）。见 [来源登记](../research/sources.md)。

FysunOS 不是 CognitiveOS 的原地重构。迁移原则：

1. **迁语义与经验证的机制，不迁目录结构。** 迁入的代码要按 FysunOS 对象模型改名与重组。
2. **凡 DSH 已有且不涉及权威的，不迁。** Personal 自研的 Runtime 片段整体放弃。
3. **业务对象回到应用层。** Project / Member / Routine / 数字员工在 FysunOS 上作为应用实现。
4. **合同与负例迁入时不得削弱。** 变更需在 FysunOS 的正式流程中重新登记。
5. **有标准的部分用标准，不迁自定义等价物。** 工具面走 MCP，外部委派走 A2A，实现包走 Agent Skills。

## 1. 从 CognitiveOS core 迁入

| 迁入 | 参考位置（迁移时按实际代码审阅） | 落到 FysunOS |
|---|---|---|
| ActionIntent/Effect 协议：persist-before-dispatch、幂等键（canonical 参数 digest）、fencing epoch、`OUTCOME_UNKNOWN` 只能 reconcile 或 quarantine | `core/crates/cognitive-kernel/src/effects.rs` | Authority 子系统 |
| 确定性 replay 与投影 digest 稳定性 | `core/crates/cognitive-kernel/src/replay.rs` | 治理账本（改为 event-sourced 唯一真源） |
| 恢复固定顺序与 checkpoint epoch 校验 | `core/crates/cognitive-kernel/src/recovery.rs` | Activity 子系统恢复模型（补 Approval 水位） |
| 上下文 authorize-before-rank、预算过滤、显式损失、digest 绑定 | `core/crates/cognitive-kernel/src/context.rs` | Resource 子系统上下文装配服务（补来源标签赋予） |
| Capability Grant 衰减、Lease/CAS 围栏语义 | `cognitive-kernel` / `cognitive-domain` | Grant 对象 |
| `SecretRef` 与 Secret Store 边界；用户指向的凭据导入边界 | AXIOMS A5 / ADR-0055 | Resource 子系统 Secret Store |
| 负例向量与符合性 runner 的方法论（向量与实现分离、`not-run` 永不等于 pass） | `core/conformance/`、`core/tests/golden/` | FysunOS conformance（重新登记，不直接复制向量内容） |
| 公理 A1–A6、A8 | `docs/governance/AXIOMS.md` | 底线 F1–F6、F8 |

**新建而非迁移**：来源/完整性标签（`label`）、Approval、Policy 版本化（Cedar 子集）、Memory 记录
在 CognitiveOS 中**无对应机制**，不是迁移项，按 [02 对象模型](02-object-model.md) 新建（清单见 §6）。

**不迁**：双内核 / 七层 / 三平面白皮书结构；六族资源作为"必修课"；企业多租户与联邦默认路径；
现有 `core/specs` 的全部 schema 原样（按新对象模型重新建模）；AKP 作为唯一适配协议的地位
（被 Runtime ABI 取代：MCP 守卫 + 最小生命周期 RPC；可借鉴其 envelope 设计）。

## 2. 从 CognitiveOS Personal 迁入

| 迁入 | 参考位置 | 落到 FysunOS |
|---|---|---|
| Rust daemon 作为唯一权威写入者 + SQLite WAL 存储 | `personal/apps/kernel-server`、`personal/crates/cognitive-store` | `fysund` |
| 独立 Verification 执行器（不接受 Agent 或 executor 自报完成） | `personal/apps/kernel-server/src/personal/verification_executor.rs` | Authority 子系统 Verifier（补 `verifier_kind` / `independence` 记录） |
| Provider 代理（凭据只在 daemon 注入） | Personal Provider proxy | Resource 子系统 |
| DSH 进程外 stdio broker 的经验（含 stale bearer 等故障记录） | `personal/crates/cognitive-runtime/src/hosted_dsh_broker.rs`、`docs/bug/dsh-*.md` | `fysun-rpc` 与 Runtime 管理 |
| 动态工具生命周期（描述符、可用性、隔离） | `personal/crates/cognitive-runtime/src/dynamic_tool_ecosystem.rs` | Resource 目录 + `resource/*` 生命周期事件（`registered / bound / unavailable / quarantined`） |
| SQLite 编号迁移机制（只追加不改写） | handbook `store-and-migrations` | 治理账本 schema 管理 |
| Control Plane 中"活动 / 证据 / 审计 / Installed Agent"视图的产品经验 | ADR-0056 / ADR-0059 | Web UI（daemon 静态服务）；Approval 呈现面为第一优先级 |
| 安装、user-service、daemon status 的运维经验 | `personal/deploy`、handbook | `fysun` CLI 与部署 |

**不迁**：Pi adapter；conversation projection（ADR-0058 的 Path B transcript）；自研 loop / 工具 /
会话片段；Project / Member / Routine / Conversation / Vault 作为 OS 对象（回到应用层）；
调度服务中与 Routine 耦合的部分（Activity triggers 重新实现）。

## 3. 从 DeepSeek Harness 复用（不迁入 OS，作为 Runtime 直接使用）

| 直接使用 | 说明 |
|---|---|
| agent loop、system-prompt 装配、tools 执行流水线（工具面经 MCP 守卫的 `tools/list` / `tools/call` 提供） | 决策核心 |
| session log、fork / resume / compaction、crash repair | Runtime 会话账本 |
| subagent 六种 provider（in-process spawn/fork、ACP、Codex、Claude Code、dsh-sdk） | 委派执行载体；其中 Codex / Claude Code 等亦可作为 Level-1 Runtime 直接接入（[05 §6](05-runtime-and-dsh-integration.md)） |
| sandbox / subprocess / terminal / e2b / fs seam | 执行环境 |
| skills（以 Agent Skills 规范为准）、plan mode、goals（同会话）、jobs、workflow 脚本 | 决策核心辅助能力 |
| Web 对话 UI、sdk / acp / headless profile | 入口与驱动通道 |

| 作为设计参考吸收（不复制代码） | 落到 FysunOS |
|---|---|
| "模型可见即已记录"不变量 | 写入 Runtime ABI 的 `session_link` 控制点义务（[04 §2.6](04-abi.md)） |
| seam 三角（Definition / Provider / Consumer） | Resource 与适配器的可替换性表达 |
| profile / bundle / patch 逐层叠加 | `fysun-dsh` 适配器的组织方式 |
| Agent Teams 的 roster / mailbox（queued-minus-delivered）/ task DAG（revision CAS、`blockedBy` 无环、advisory writeScopes） | Handoff 对象与协作事实的持久化 |
| Ralph 交接（有界结构化报告：状态、摘要、证据、后续步骤、阻塞） | Handoff.deliverables 与 Activity.continuity 的形态 |
| Session 投影缓存的"冷读取阶梯"（缓存行 + 尾部回放） | 治理账本投影缓存策略 |

**不吸收为 OS 语义**：见 [05 §4](05-runtime-and-dsh-integration.md#4-明确不从-dsh-吸收为-os-语义的东西)。

## 4. 从 Cordis 吸收（收窄为 `fysun-dsh` 适配器内部实现）

| 吸收 | 落到 FysunOS |
|---|---|
| 可撤销效果（每个上下文变换携带逆操作，卸载时 LIFO 撤销） | 适配器内 dispose 语义；账本只记 `resource/unavailable \| quarantined` 与由此触发的 Grant 撤销、Effect 对账 |
| 反应式依赖（coeffect 规格驱动激活/停用/重载） | 适配器内 mount 排序；不进入账本 |
| loader 的配置 reconciliation、HMR | 适配器内部；OS 侧不实现 HMR |

内部 trait 参考见 [04 附录 A](04-abi.md)。

**不做**：以 Cordis 为内核基底；把插件图当信任/委派图；fork 或 vendor Cordis；把上述任何一项
作为 OS 公共接口暴露。

## 5. 迁移顺序（与 MVP 计划对齐）

1. 先落对象模型与账本 schema（新建，不复制 `core/specs`）；
2. 迁 ActionIntent/Effect、replay、recovery、context 四个内核模块并按新对象改名；
3. 迁 Verifier 与 Provider 代理；
4. 以 DSH stdio broker 经验实现 `fysun-rpc`；MCP 守卫（`fysun-mcp-guard`）先于 DSH 适配器落地
   （对应 08 M2a → M2b）；
5. 其余按 [08 MVP 计划](08-mvp-plan.md) 的闭环需要逐项拉取。

每次迁入都在 [来源登记](../research/sources.md) 记录来源 revision 与路径，保证可追溯。

## 6. 新建项清单（不迁移、无来源）

以下项在 CognitiveOS、DSH、Cordis 中均无对应机制，按 v0.6.0 设计稿新建：

| 新建项 | 定义位置 | 依据 |
|---|---|---|
| 来源/完整性标签 `label` | 02 §2.15 | ADR-0004 |
| Approval 一级对象 | 02 §2.11 | ADR-0005 |
| Policy 对象（Cedar 子集） | 02 §2.12、03 §7 | ADR-0003 |
| Memory 记录 | 02 §2.13 | 02 |
| MCP 守卫（`fysun-mcp-guard`） | 04 §2.1 | ADR-0002 |
| A2A client（外部 Handoff） | 04 §2.7 | ADR-0002 |
| Agent Skills 登记（来源 + digest + `allowed-tools` 上界） | 04 §2.8、05 §5 | ADR-0002 |
| OTLP 导出 | 03 §8 | 03 |
