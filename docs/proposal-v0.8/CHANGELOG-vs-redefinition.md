# 相对 `docs/redefinition` 的差异说明

- 状态：**提案草案**（审阅用，不是设计正文）
- 日期：2026-09-12
- 本文件不替代现行 [`docs/redefinition`](../redefinition/README.md) 直至被采纳。
- 对照对象：redefinition 全套（README、00–10、ADR-0013 草案，2026-09-11）。也吸收了 [`docs/research/20260912-redefinition-review-and-refactor-report.md`](../research/20260912-redefinition-review-and-refactor-report.md) 中成立的诊断，**拒绝**其中把 SPIFFE、Arrow、内生 Evolution Engine、前缀槽硬资格化写进内核的建议。

每篇：处置（吸收 / 改写 / 抛弃）+ 为什么。

| redefinition | 处置 | 为什么 |
|---|---|---|
| [README](../redefinition/README.md) | **改写** | 定义句、十一类资产、六组 API、R0–R5、组件表膨胀。一句话定义与双职责、`fysund` 不在热路径、命名（gateway / mcp-guard / dsh / edge）**吸收**。 |
| [00 术语](../redefinition/00-terminology.md) | **改写** | 一级对象表把 Endpoint/Plan/Meter/runtime kind 抬平。改名对照中「Harness/Loop 升一级 kind」只吸收 Harness。事件族与错误码前缀**吸收**后按新对象裁剪。 |
| [01 定位](../redefinition/01-positioning.md) | **吸收主干，改写对象图** | P1–P11、F1–F9 落点、不做清单、AOS「供给控制面」**吸收**。平面图里的十一类 Registry 与五服务对象**改写**。P8「登记 Loop」改为登记 Harness 及其 Profile。 |
| [02 资产](../redefinition/02-asset-model.md) | **改写** | 统一 manifest、digest、效果汇总、生命周期、负例 1–7 **吸收**。§3.2 skill / §3.4 mcp / §3.5 harness / §3.7–3.9 **吸收字段意图**。§3.3 `tool`、§3.6 `loop` 作为 kind、§3.10 `runtime` 作为 kind **抛弃**。 |
| [03 组合](../redefinition/03-composition.md) | **改写** | Agent Definition YAML、解析约束、变体、subagents、Adjustment 作用点 **吸收**。单一扁平 Lock 与「新 Lock 证据从零」**抛弃**，换分层 digest + `inherited`。 |
| [04 Runtime](../redefinition/04-runtime-substrate.md) | **吸收并收紧热路径** | EP 种类与契约、Policy 下发、`os.observe`、conformance 按 EP 组合分列、多租户池化、热路径禁令 **吸收**。所有 tool call 同步 WAL **抛弃**（回到 ADR-0008 读快路径并扩展 `external_read` 微批）。ABI 增加可选 `park`/`thaw`。Runtime 不再走 `asset.register`。 |
| [05 AaaS](../redefinition/05-agent-as-a-service.md) | **改写层级，吸收流程** | 角色、发布/rollout、Agent Card、调用序、审批路由、联邦「账本不共享」**吸收**。ServiceVersion/Endpoint/Tenant/Plan/Meter 五级平级 **抛弃**。Trust 对象从「R4 必做」降为开放问题（不进 R0–R3 退出条件）。 |
| [06 治理](../redefinition/06-governance.md) | **吸收** | Cedar 实体扩展、Grant `root_of: plan`、标签、T12–T15、审批面规则几乎原样进入本提案治理默认。OTel「与 gen_ai 关系开放」**吸收**并写明 Development 状态。未单列成篇，以免与 01/04 重复。 |
| [07 生态](../redefinition/07-ecosystem-and-interop.md) | **吸收标准表，不单列** | MCP/A2A/Skills/OCI/Cedar/OTel/hooks 用法与「不用于」列 **吸收**进 README 锚点与 00 D8。Harness 生态表仍有效，实现期回填。自造包格式负例保留。MCP 规范日期从 2025-11-25 **更新**为须跟踪 2026-07-28（拍板项 7）。 |
| [08 API](../redefinition/08-api-surface.md) | **改写** | 方法名大部分可复用。六组 → 四组；`runtime.*` 迁出 Registry；删除 `endpoint.*` 与 Meter 资源。EP/ABI 保持「契约不是资源」。 |
| [09 演进](../redefinition/09-evolution-and-evidence.md) | **改写** | 三种变化、checklist、发布证据门、消费反馈只走 Verification、跨租户不自动传播 **吸收**。「换资产证据清零」**抛弃**。补上优化器边界与安全包络（redefinition 只有人工 Adjustment，名为演进实为 SCM）。 |
| [10 路线图](../redefinition/10-roadmap.md) | **改写** | §1 为何重定义（v0.7.0 内联极限）**吸收**，这是本提案仍要重定位的原因。§2 对 v0.7.0 各篇继承/废止方向 **吸收**（守卫降级、投递废止、Q5 关闭）。R0–R5 **抛弃**，换更薄的 R0–R3。开放问题压缩。 |
| [ADR-0013 草案](../redefinition/adr-0013-draft-repositioning.md) | **改写后才能采纳** | D1 职责、D4 三平面+EP、D5 下发、D6 多主体、D7 标准、D8 F 线、否决的四个替代、负例 2–6 **吸收**。D2 十一类枚举、D3「Runtime 仍是资产 kind」、负例 1 把 loop/tool 当资产 **必须改**。 |

## 复检报告里采纳与拒绝

| 报告主张 | 本提案 |
|---|---|
| Runtime 不是资产 | **采纳** |
| Loop 不要 OCI 一级 | **采纳**（保留可追溯 Profile） |
| 取消独立 `tool` kind | **采纳** |
| 分层 Lock + 微调可继承证据 | **采纳**（不要数值 discount 内核化） |
| 只读快路径 / 并行只读 | **采纳**（对齐 ADR-0008 而非新发明） |
| MCP sampling 防御 | **采纳**，R2 |
| Freeze & thaw | **采纳为可选控制点**，不承诺 KV 迁移 |
| 优化器不进内核 | **采纳**；拒绝「内生 Evolution Engine 第四阶段必做」 |
| 确定性前缀槽作为 qualify 硬失败 | **拒绝** — 改为能力位；OS 不渲染 |
| SPIFFE 替换 Tenant token | **拒绝**进 R0–R3 |
| Arrow Flight / 环形缓冲为规范 | **拒绝** — 传输是 Q1 |
| 3 层 9 类新本体论 | **拒绝** — 压缩到 8 kind + Profile，不另造 L1/L2/L3 产品名 |

## 刻意未迁入本提案的 redefinition 内容

- 各 kind 的完整 YAML 字段清单：实现期出 schema 与负例向量即可，提案只保留必检项。
- conformance 大表的每一格：规则已在 01 §3，表在 R2 测试包里填。
- 对 v0.7.0 十四篇的逐文件「改写/降级」操作清单：那是采纳之后的文档工程，不是本方案要先做的事。
