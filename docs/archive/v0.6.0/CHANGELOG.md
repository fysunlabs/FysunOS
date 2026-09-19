# 变更日志

本仓库只有设计文档。版本号指设计文档集合的整体版本，与 git tag 一致。

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
