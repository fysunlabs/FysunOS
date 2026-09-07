# 归档 · v0.5.0

- 快照日期：2026-09-06（文档正文标注为"设计稿 v0.1"）
- 归档日期：2026-09-07
- 对外版本号：**v0.5.0**（git tag `v0.5.0`）

## 内容

本目录是 FysunOS 首轮设计稿的原样快照，保持原相对目录结构：

| 归档路径 | 原路径 |
|---|---|
| `README.md` | `/README.md` |
| `design/01-definition-and-principles.md` … `design/08-mvp-plan.md` | `/docs/design/` |
| `adr/README.md`、`adr/0001-founding-decisions.md` | `/docs/adr/` |
| `research/sources.md` | `/docs/research/sources.md` |

归档副本**不再修改**。副本中的相对链接指向的是归档时的相对位置，在本目录内可能无法解析；请以路径对照表回到当前版本查看。

## 归档原因

2026-09-07 完成的[前沿对照审查报告](../../research/20260907-frontier-review-and-recommendations.md)对本版设计稿做了逐条审查，
提出协议优先的 Runtime 边界、Approval / Policy / Memory 一级对象、来源标签与权限单调收窄（F9）、术语 `Intent` → `ActionIntent`
等改动。随后全部设计文档依据该报告修订为 v0.6.0（见 [CHANGELOG](../../CHANGELOG.md)）。为保留修订前的完整状态以便对照，
先将 v0.5.0 原样归档。

## 与 v0.6.0 的主要差别（摘要）

- Runtime 边界：v0.5.0 为 Harness ABI（DSH 私有 seam 接入）+ Composition ABI；v0.6.0 为 Runtime ABI（MCP 守卫 + 最小生命周期 RPC + 可选控制点）+ A2A。
- DSH 角色：v0.5.0 为"默认参考 Runtime + `fysun` bundle"；v0.6.0 为"第一个 Level-2 参考适配器"。
- 对象模型：v0.6.0 新增 Approval、Policy、Memory，扩展 Verification、Handoff、Adjustment，并引入来源/完整性标签。
- 完整差别以 CHANGELOG 与 ADR-0002 ~ ADR-0006 为准。
