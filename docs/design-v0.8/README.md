# FysunOS v0.8 · 重新设计完整设计说明（综合定稿稿）

- 状态：**综合定稿稿（待所有者采纳）**。本文件夹是把 `docs/redefinition`（v0.8.0-draft，2026-09-11）、
  `docs/proposal-v0.8`（2026-09-12）、两份 2026-09-12 独立评审报告与现行 v0.7.0 设计
  （`docs/design` + `docs/adr`，2026-09-08）**综合为一份可独立阅读的完整设计说明**的结果。
- 主文档：[fysunos-design-spec.md](fysunos-design-spec.md)——重新设计后的完整设计说明。
- 在所有者采纳之前，现行设计仍为 [`docs/design`](../design/00-terminology.md)（v0.7.0）与已采纳 ADR-0001–0012；
  本文件夹**不修改任何既有文件**。

## 文件

| 文件 | 内容 |
|---|---|
| [fysunos-design-spec.md](fysunos-design-spec.md) | 完整设计说明：定位与定义、原则 / 底线 / 不变量、四层对象模型、资产模型、组合与分层 Lock、Runtime 基座与执行点、治理、AaaS、演进与自进化、架构与账本、API 面、威胁模型与 conformance、路线图、开放问题与待决策项 |
| README.md | 本索引 |

## 综合来源与裁定关系

| 来源 | 本说明对它做什么 |
|---|---|
| [`docs/redefinition`](../redefinition/README.md)（v0.8.0-draft） | **吸收**：双职责定位、`fysund` 退出数据面热路径、执行点（EP）模型、Agent Definition、服务平面、多主体进核心、`os.observe`。**改写**：11 种资产 → 8 种 kind（`loop` / `tool` / `runtime` 降级或删除）、服务对象收敛为 Service 一级、六组 API → 四组 + 两契约、扁平 Lock → 分层 Lock。 |
| [`docs/research/20260912-redefinition-review-and-refactor-report.md`](../research/20260912-redefinition-review-and-refactor-report.md) | **采纳**其 P0 全部与多数 P1：四层实体、分层 Lock + 差分证据继承、04/EP 契约继承 ADR-0008 D5、`published` 以 qualified EP 覆盖为门（I6）、MCP 钉 2026-07-28（Sampling 缺省拒绝）、自进化带 Immutable Zone 的离线插件、不变量 I1–I12、负例 N-01–N-12。 |
| [`docs/research/20260912-redefinition-audit-and-improvement-plan.md`](../research/20260912-redefinition-audit-and-improvement-plan.md) | **采纳**：P0-1–P0-4、P1-1–P1-8（EP 一致性、MCP 授权、Cedar 实体扩展须先修 ADR-0009、A2A 1.0 Card 形状、shared 池化延后）、冲突表 C1–C12 的处置、R0–R5 门禁（压缩为 R0–R3 + 后置项）、Q30–Q33。 |
| [`docs/proposal-v0.8`](../proposal-v0.8/README.md) | **主体骨架**：D1–D8 拍板决策、八种资产、三层 Lock、F3 分级、Service 单一一级服务对象、R0–R3、OWASP 对照、2026 锚点。其未显式编码的评审共识（如 I6 发布门）在本说明中补齐。**拒绝**项照录（前缀槽硬资格化、SPIFFE 进内核、Arrow/零拷贝、数值置信度折损、内生 Evolution Engine、3 层 9 类改名）。 |
| v0.7.0（`docs/design` + ADR-0001–0012） | **继承**：F1–F9、event-sourced 治理账本与唯一权威写入者、ActionIntent → Effect → Receipt → Verification、六值 `effect_category` 三源合一（ADR-0008）、Cedar 子集（ADR-0003/0009）、Approval 封闭结果集与审批面（ADR-0011）、Activity/Execution 状态机与恢复重新武装、Runtime 资格化纪元与 `native_effect_tools[]`（ADR-0007）、边缘离线一致性（ADR-0010）、威胁模型 T1–T11。 |

## 阅读顺序

主文档 §0（由来）→ §1–§2（定位 / 原则 / 不变量）→ §3–§5（对象 / 资产 / 组合）→ §6（Runtime 与执行点，核心）→ §7–§9（治理 / AaaS / 演进）→ §10–§13（架构 / API / 威胁模型 / 路线图）→ §14–§15（开放问题 / 待决策项）。
