# 归档 · v0.6.0

- 快照日期：2026-09-07（文档正文标注为"设计稿 v0.6.0"）
- 归档日期：2026-09-08
- 对外版本号：**v0.6.0**（git tag `v0.6.0`，commit `3e65b64`）

## 内容

本目录是 FysunOS 设计文档 v0.6.0 的原样快照，保持原相对目录结构：

| 归档路径 | 原路径 |
|---|---|
| `README.md` | `/README.md` |
| `CHANGELOG.md` | `/docs/CHANGELOG.md` |
| `design/00-terminology.md` … `design/09-threat-model.md`（10 篇） | `/docs/design/` |
| `adr/README.md`、`adr/0001-founding-decisions.md` … `adr/0006-terminology-action-intent.md`（7 个） | `/docs/adr/` |
| `research/sources.md` | `/docs/research/sources.md` |

审查报告 `docs/research/20260907-frontier-review-and-recommendations.md` 本身是历史记录、不随版本修订，不复制。

归档副本**不再修改**。副本中的相对链接指向的是归档时的相对位置，在本目录内可能无法解析；请以路径对照表回到当前版本查看。

## 归档原因

2026-09-08 依据 [v0.7.0 全量重构蓝图](../../research/20260908-v0.7.0-refactor-blueprint.md)（前置分析 + 外部调研 + 所有者决策）对全部设计文档做全量重构：`docs/design` 从 10 篇重组为 14 篇（00–13），ADR-0001 ~ ADR-0006 正文直接修订并新增 ADR-0007 ~ ADR-0012。为保留重构前的完整状态以便对照，先将 v0.6.0 原样归档（见 [CHANGELOG](../../CHANGELOG.md)）。

## 与 v0.7.0 的主要差别（摘要）

- 文档地图：v0.6.0 为 00–09 十篇；v0.7.0 拆分为 00–13 十四篇（04 拆为 Capability API / Runtime ABI / MCP 守卫规范三篇；05 拆为 Runtime 接入与资格化 / DSH 参考适配器两篇；07 拆为演进 / 边缘与离线账本一致性两篇）。
- ADR 规则：v0.6.0 "ADR 只追加不改写"；v0.7.0 起 accepted 的 ADR 可在版本重构中直接修订正文，但必须带 `## 修订历史` 段。
- Runtime 原生工具：v0.6.0 对 Level-1 Runtime 的 conformance 声明一律"满足"；v0.7.0 要求 `describe()` 如实声明 `native_effect_tools[]`，conformance 表按 Runtime 类型分列，hooks.json 阻断路径作为 Level-1b 补充（ADR-0007）。
- Effect 类别：v0.6.0 反复引用但无定义；v0.7.0 定义六值 `effect_category` 与三源合一分类（ADR-0008）。
- Approval：v0.6.0 状态集 `pending / approved / denied / modified / expired`，事件五个；v0.7.0 改为封闭结果集 `approved_once / rejected / cancelled / unavailable`，事件 `approval/asked` / `approval/decided` 配对，审批面只由 `fysund` 提供（ADR-0011）。
- Cedar 子集范围与实体模型（ADR-0009）、边缘离线账本一致性（ADR-0010）由开放问题转为正式决策；agentgateway 复用评估关闭为自实现（ADR-0012）。
- 来源标签：删除 v0.6.0 中工具结果标签的三处例外，工具结果一律 `untrusted_external`。
- 完整差别以 CHANGELOG v0.7.0 段、ADR-0001 ~ ADR-0006 的修订历史与 ADR-0007 ~ ADR-0012 为准。
