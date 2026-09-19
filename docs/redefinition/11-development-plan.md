# 11 · 开发计划（重定义提案）

- 状态：提案草案 v0.8.0-draft（2026-09-12）；**本文件为可执行计划**，不是实现声明
- 决策依据：[ADR-0013 草案](adr-0013-draft-repositioning.md)；里程碑骨架见 [10 路线图](10-roadmap.md) §3
- 术语以 [00 术语表](00-terminology.md) 为本文件夹唯一基准
- 复用基线登记于 [docs/research/sources.md](../research/sources.md)；本文件引用时不臆造未核验 revision
- 本仓库当前**只有设计文档，没有可运行代码**（根 [README.md](../../README.md)）。本计划从空实现起步，选择性迁入 CognitiveOS / Personal 的语义与机制，并把 DeepSeek Harness 作为首个生态 Harness，而非内核。

> **正式定义句**（不得改写）：FysunOS 是 Agent 能力管理（提示词、Skills、工具、MCP、Harness、Loop）和 Runtime 控制的运行基座。

四类状态用语（继承 CognitiveOS / v0.7.0 12 §5）：本计划全部条目为 **specified**。规范存在 ≠ 实现存在 ≠ 测试已执行 ≠ Profile 已符合。

---

## 0. 文档地图与完成状态

本文件按「先骨架、后逐章补全」写成。下表是阅读入口：每章回答一个可判定问题，并标明与 redefinition 文档的追溯。

| 章 | 回答的问题 | 状态 |
|---|---|---|
| 1 | 计划覆盖什么、不覆盖什么、怎样算成功 | 已补全 |
| 2 | 实现期不可违背的原则、底线、禁令 | 已补全 |
| 3 | 与 ADR-0013、10 路线图、v0.7.0 设计的关系 | 已补全 |
| 4 | 复用谁、怎么复用、明确不复用什么 | 已补全 |
| 5 | 仓库形态、进程边界、crate / 包布局 | 已补全 |
| 6 | 证据纪律、CI、钉版、文档同步 | 已补全 |
| 7 | R0–R5 总览与依赖 | 已补全 |
| 8–13 | 各阶段工作包、验收、负例、复用动作 | 已补全 |
| 14 | 开工前必须跑完的 spike | 已补全 |
| 15 | 测试分层与 conformance 分列 | 已补全 |
| 16 | 度量：先测基线、后定阈值 | 已补全 |
| 17 | 风险、许可证、供应链 | 已补全 |
| 18–19 | 开放问题与待所有者决策 | 已补全 |
| 20 | 设计条款 → 工作包追溯 | 已补全 |

阅读顺序：本章地图 → §2 约束 → §4 复用 → §7 总览 → 当前阶段专章 → §14 spike。实现代理以当前阶段专章为任务单，不以根 README 的 v0.7.0 定义句为准。

---

## 1. 目的、范围与成功判据

### 1.1 目的

把 [docs/redefinition](README.md) 从提案变成可验证实现：先建立资产登记与 Runtime 控制的最小闭环，再叠服务平面与执行点治理，最后把演进、边缘与 Personal 三合一应用接回去。过程中**优先依赖、抽取或借鉴已有开源实现**，只在 FysunOS 特有合同（账本权威、资格化、Lock、EP 契约、Tenant）上自研。

### 1.2 范围内

- 五个进程 / 入口：`fysund`、`fysun` CLI、`fysun-gateway`、`fysun-mcp-guard`（独立进程，可选）、`fysun-dsh`（Harness 适配器）。边缘 `fysund-edge` 在 R5。
- 十一类资产的 manifest schema、OCI 打包、登记、解析锁定、资格化。
- Runtime ABI 最小面与 EP 契约。
- 服务平面对象：Service / Tenant / Plan / Meter / Endpoint / Agent Card。
- 治理横切：Grant / Policy（Cedar 子集）/ Approval / 来源标签 / Verification / 供应链事件。
- 符合性向量、金样、负例；文档回填 `docs/design`（R0 启动，实现中持续）。

### 1.3 范围外（明确不做）

与 [01 §8](01-positioning.md) 一致，实现计划额外钉死：

- 不实现 loop、prompt 装配、工具选择、compaction、沙箱、终端、LSP、聊天 UI。
- `fysund` 不做数据面转发（不作为 harness 的 MCP server、不代理模型请求、不渲染 context_view 文本）。
- 不计费结算、不自造包格式 / 策略语言 / 传输协议。
- 不把 CognitiveOS 目录结构、AKP 作为唯一适配协议、`LoopDriver`、Cordis 内核、agentgateway 嵌入 `fysund`。
- 不在 ADR-0013 采纳前把本文件夹条款写进现行 `docs/design` 正文（R0 的文档工作以 ADR 采纳为闸门）。

### 1.4 成功判据（计划级）

1. **R2 首个闭环可演示且负例全过**：登记 prompt + skill + mcp_server + harness + loop + model → 定义 Agent → 解析 Lock → 匹配 Runtime → 运行 → 独立验证 → 崩溃恢复。`fysund` 不出现在工具 / 模型转发链上（ADR-0013 负例 2）。
2. **复用可审计**：每个自研模块都能指出「为何不复用」或「复用了哪一仓库的哪一层」；许可证与 NOTICE 完整。
3. **声明诚实**：任何 milestone 完成声明必须链接 campaign 证据；局部测试通过不构成 Profile 符合。

---

## 2. 不可违背约束

实现、评审、CI 都按本节检查。违反即失败，不开豁免工单。

### 2.1 原则 P1–P11（[01 §6](01-positioning.md)）

尤其：

- **P8**：不规定决策核心，但登记并承载它。Harness / Loop 是资产，不是 `fysund` 内部模块。
- **P9**：无 manifest、无 digest 的东西不能进入 Agent Definition。
- **P10**：对外形态是服务（R3 起强制；Personal 可绕过 gateway，但 Tenant / Scope 仍存在，见 [10 §7](10-roadmap.md) 决策 5 的本文默认）。
- **P11**：基座不在热路径。

### 2.2 底线 F1–F9（[01 §7](01-positioning.md)）

| ID | 实现落点 | 热路径禁令 |
|---|---|---|
| F1 | 只有 `fysund` 写权威账本、签发 Grant / Approval、推进 Activity / Execution / Effect / Verification | 执行点不得本地写出 `approval/decided` / `grant/issued` |
| F2 | 模型与外部 Agent 输出皆候选 | 执行点赋予 `untrusted_external` |
| F3 | persist-before-dispatch 在 **EP 本地**成立，回报后 `fysund` 重建投影 | 无回报 Effect 不得 `verified` |
| F4 | 独立 Verifier | Agent / Harness 自报完成无效 |
| F5 | `SecretRef` 只向资格化 EP 按 Grant 解析 | 描述面无值槽；Harness 自持凭据不在范围 |
| F6 | 公共契约、负例不为某个实现让路 | 删负例 = 计划失败 |
| F7 | Policy 版本化下发；能力不自扩权 | EP fail-closed |
| F8 | 不覆盖非本任务并发改动 | Harness + EP |
| F9 | 有效权限 = Grant ∩ Policy ∩ 工具面；收窄自动、扩张只经 `approved_once` | 工具面声明来源含 `tools_allow` / `enabled: false` |

### 2.3 架构禁令

- `fysund` 进程出现在任一 `tools/call` 或模型请求转发链 → 失败（[ADR-0013](adr-0013-draft-repositioning.md) 负例 2，[04 §4.4](04-runtime-substrate.md)）。
- `start_execution` 不得携带渲染好的上下文文本（[04 §2](04-runtime-substrate.md)、[04 §8](04-runtime-substrate.md) 负例 2）。
- 控制通道是 Runtime ABI，不是 MCP（[07 §7](07-ecosystem-and-interop.md) 负例 2）。
- 资产分发只用 OCI artifact（[07 §7](07-ecosystem-and-interop.md) 负例 1）。
- 公开 API 不得出现 `grant.issue`（[08 §9](08-api-surface.md) 负例 2）。
- 定义句不得把基座写成「实现 loop / prompt 装配 / 沙箱」或「运行时（runtime）」（[00 §8](00-terminology.md)）。

### 2.4 多主体缺省

MVP 起即多主体（[ADR-0013](adr-0013-draft-repositioning.md) §D6）。单机部署可以只有一个 Tenant，但对象模型、事件字段 `tenant_id`、Scope 前缀从 R1 就存在。跨租户不可见是缺省。

---

## 3. 与现行文档 / ADR 的关系

| 来源 | 本计划如何对待 |
|---|---|
| 本文件夹 00–10、ADR-0013 草案 | **现行实现规格**。冲突时以本文件夹为准。 |
| [10 §3](10-roadmap.md) R0–R5 | 阶段骨架；本文件把每阶段拆成工作包、依赖、复用动作与验收。 |
| [v0.7.0 12 MVP](../design/12-mvp-plan.md) | **被重写**。M0–M5 的可判定负例与度量**继承**，阶段编号改为 R*；M2a「`fysund` 作为 MCP 中间层」**废止**，Effect 协议改落 EP。 |
| [v0.7.0 09 迁移清单](../design/09-migration-inventory.md) | 复用决策的底稿；本文件按重定义平面（资产 / 运行 / 服务 / EP）重排，并更新「不迁」项（Provider 代理不再是 F5 前提）。 |
| ADR-0001 ~ 0012 | 保留；R0 按 [10 §2](10-roadmap.md) 修订 ADR-0002 / 0007 / 0012。ADR-0012「不嵌入 agentgateway」保留，「不前置」按执行点框架重开。 |
| 根 README v0.7.0 定义句 | 在 ADR-0013 采纳前仍是对外现行定义；实现分支以本文件夹为准，R0 同步根 README。 |

文档闸门：**R0 完成前**，实现可以建空仓库骨架与 spike，但不得把 v0.7.0 的「`fysund` 内联守卫」写进代码注释或模块名。

---

## 4. 复用策略与供应链

原则（继承 [09 迁移清单](../design/09-migration-inventory.md) 五条，按重定义收口）：

1. 迁语义与经验证机制，不迁目录结构。
2. DSH 已有且不涉及权威的，不迁进 `fysund`。
3. 业务对象回到应用层（Personal 是 R5 应用，不是 OS 对象）。
4. 合同与负例迁入时不得削弱。
5. 有标准的部分用标准：MCP / A2A / Agent Skills / OCI / Cedar / OTel。

复用层级（每个依赖必须归入一层，禁止含糊「参考一下」）：

| 层级 | 含义 | 例子 |
|---|---|---|
| **A 直接依赖** | crate / npm 进入 lockfile | `cedar-policy`、`rusqlite`、OCI client、MCP SDK、OTel |
| **B 适配 / 进程外使用** | 独立进程或 bundle，不链进 `fysund` | DeepSeek Harness、可选 agentgateway 作 `external_gateway` |
| **C 抽取移植** | 阅读上游代码，按新对象模型改名重写进本仓 | CognitiveOS Effect / replay / recovery |
| **D 借鉴模式** | 只吸收不变量与流程，不复制代码 | DSH scope 不可区分性、SecretRef 三规则、Cordis dispose 语义 |
| **E 明确不复用** | 文档写明原因；CI 可 grep 禁词 | Cordis 作内核、`LoopDriver`、自造包格式 |

核验日期：2026-09-12。未能确定的 revision 写「待开工钉版」，不编造。

### 4.1 CognitiveOS（`agentkernel/cognitive-os`）

| 项 | 事实 |
|---|---|
| URL | https://github.com/agentkernel/cognitive-os |
| 许可证 | Apache-2.0 |
| 语言 | Rust + TypeScript（core 合同 / personal 实现） |
| 成熟度 | README 声明 core / personal **1.0.0 已定稿**（tag `core-v1.0.0` / `personal-v1.0.0`）；enterprise **未开始**；四类状态用语强制 |
| 本仓已登记基线 | 本地 `agent-kernel@6f562bf1bdb86582955de36eb93e51126573f89c`（[sources.md](../research/sources.md)）；公开仓 HEAD 待开工对照钉版 |
| FysunOS 关系 | **前身调研对象，不是原地重构**（根 README）。Personal 是 R5 第一个三合一应用（[07 §6](07-ecosystem-and-interop.md)）。 |

**C 抽取移植（优先顺序）**

| 上游（迁移时按实际路径审阅） | 落到 FysunOS | 重定义注意 |
|---|---|---|
| `cognitive-kernel/src/effects.rs` Intent / Effect、幂等冲突硬拒绝、补偿需新 Grant | EP 本地 persist + `fysund` 投影核对；错误码 `conflict.idempotency` | 执行位置从内核挪到 EP（[04 §4.2](04-runtime-substrate.md)）；协议语义不削弱 |
| `replay.rs` 确定性 replay、投影 digest | `fysund` 治理账本 | 事件族按 [00 §5](00-terminology.md) 扩展 |
| `recovery.rs` 固定恢复顺序 | Activity / Execution 恢复 + 重新武装 | 增 Approval 水位、EP 未回报对账（Q25） |
| Grant 衰减、fencing | Grant 对象；Plan 派生根 Grant | 增 `root_of: plan_ref`、`ep_scope` |
| SecretRef / Secret Store | `fysund` 解析 + EP 注入 | F5 改为只向资格化 EP 解析，**不再**以 Provider 代理为前提 |
| `context.rs` 信任等级 | 来源标签的上下文层先例 | 不直接迁取值集；沿参数流扩展到 Effect（ADR-0004） |
| conformance runner 方法论、四类状态用语、golden digest | `fysun-conformance` | 向量按 EP 组合重登记，不原样复制 |
| `cognitive-akp/.../deepseek_harness.rs` 的 `authority_claim` / `secret_shaped` | `label.flags` | **AKP 协议本身不迁**，Runtime ABI 取代 |
| Personal `kernel-server` + `cognitive-store`（SQLite WAL、编号迁移） | `fysund` 单机存储 | daemon **退出数据面**；不要把 MCP server 一并迁入 |
| Personal verification executor | Verifier | 补 `verifier_kind` / `independence` |
| Personal hosted DSH broker 故障经验 | `fysun-dsh` 进程管理 | 适配器在 `fysund` 外 |
| Personal deploy / CLI 经验 | `fysun` CLI、安装面 | 子命令按新 API 面重做 |

**E 不迁**

- `LoopDriver` / 内核内 loop 边界。
- 双内核白皮书结构、六族资源必修课、现有 `core/specs` 原样 schema。
- AKP 作为唯一适配协议。
- Pi adapter、conversation projection、Project / Member / Routine / 数字员工 OS 对象。
- Personal 自研 Runtime 片段（凡 DSH 已有且不涉及权威的）。
- Provider 代理作为 F5 前提（降为可选 `provider_proxy` EP）。

**建议 spike**：对照公开仓 `personal-v1.0.0` 与本地 `6f562bf` 做一次 diff 清单，列出仍值得抽取的文件与已过时的内联守卫代码，产出 `docs/research/YYYYMMDD-cognitiveos-extract-map.md`。

### 4.2 DeepSeek Harness（`deepseek-ai/deepseek-harness`）

| 项 | 事实 |
|---|---|
| URL | https://github.com/deepseek-ai/deepseek-harness |
| 许可证 | MIT |
| 语言 | TypeScript（Cordis 插件树）；Python / TS SDK 经 JSON-RPC stdio 驱动 |
| 成熟度 | **developer preview**，README 明示 **THERE WILL BE COMPATIBILITY-BREAKING CHANGES** |
| 文档 | https://deepseek-harness.github.io/deepseek-harness/ |
| 本仓已登记基线 | `cd5ef8148158c3a752a658978873241fdf8e2bbc`（[sources.md](../research/sources.md)） |
| FysunOS 关系 | 首个 Harness 资产 + `fysun-dsh` 适配器上游（[07 §2](07-ecosystem-and-interop.md)）；headless 为 R2 薄包装候选 |

**B 进程外使用（OS 不吸收决策核心）**

直接使用：agent loop、system-prompt 装配、tools 流水线、session log、fork/resume/compaction、sandbox/fs/shell、skills、headless / sdk / web profiles。这些全部留在 Harness 进程。

**D 借鉴模式（写入合同，不复制代码）**

- 「模型可见即已记录」→ `session_link` 控制点。
- scope 过滤不可区分性 → EP `tools/list`（[06](../design/06-mcp-guard.md) 降级后仍适用）。
- Secret 三规则：每操作重解析、描述面无值槽、空值即未配置。
- `start()` 前能力位校验、缺失则大声失败。
- `armed/disarmed` 不持久化 → 恢复后重新武装，武装事实改由账本承载。
- `tools/post-execute` → `fysun-dsh` 原生工具 receipt。
- `approval/request` waterfall → 拦截后全部转 `fysund`，禁止 DSH approval policy `never`。
- hooks.json PreToolUse 退出码 2 → `hooks` 类 EP。
- profile / bundle / patch 叠加 → `fysun-dsh` 组织方式。
- `SubagentResult.diagnostic` ≤ 4096 B 脱敏 → Handoff / error_report。

**E 不照搬**

- 以会话日志最后一条事件为 approval / sandbox / permission 真源。
- DSH 自绘审批卡片作为审批面（T11）。
- Agent Teams 全量入治理账本（只迁 Handoff 状态 + deliverables）。
- DSH `packages/guard/`（loop 卫生插件）当作安全守卫。
- 把 DSH 源码 vendor 进 `fysund`。

**钉版策略**：`harness:deepseek/dsh@<semver>+<gitsha>`；bundle 校验 `dsh_version` 与 `schema_digest`；CI 对上游最新 tag 跑非阻塞漂移预警（继承 v0.7.0 08 §6）。preview 期间默认**拒绝自动升小版本**。

**R2 选型（待所有者决策 2）**：本计划默认路径是 **R2 用 DSH headless 薄包装跑通 Runtime ABI 最小面**，并行启动 `fysun-dsh` spike；R5 才要求全控制点。理由：preview 接口不稳定，先把 OS 合同测绿，再把适配器做厚。

### 4.3 Cordis（`cordiverse/cordis`）

| 项 | 事实 |
|---|---|
| URL | https://github.com/cordiverse/cordis （npm 亦见 `cordis` / `@deepseek-ai/cordis`） |
| 许可证 | MIT |
| 语言 | TypeScript / JavaScript |
| 成熟度 | README：**API is not yet stable**；论文 arXiv:2608.25512 |
| 与 DSH 关系 | DSH「everything is a plugin」的内核；本仓登记 vendored 上游 **4.0.0-rc.7**，DSH 清单 **4.0.1**，上游 tag **v4.0.0-rc.9**（commit 待查证） |
| FysunOS 关系 | **仅** `fysun-dsh` 适配器内部生命周期（[07 §6](07-ecosystem-and-interop.md)、[ADR-0001](../adr/0001-founding-decisions.md) §D2） |

**D 借鉴（适配器内部）**：可撤销效果（卸载 LIFO dispose）、反应式依赖、loader reconciliation / HMR。账本只记资源不可用与 Grant 撤销，不记插件图。

**E 禁止**

- 以 Cordis 为 `fysund` 内核或公共接口。
- fork / vendor Cordis 进 OS 仓库。
- 把插件图当信任 / 委派图。
- 把 `isolate(name)` 当作安全隔离——它只是服务名命名空间（[09 §4](../design/09-migration-inventory.md) 一手资料确认）。

实现含义：`fysund` 的 Cargo.toml **不得**出现 cordis。适配器若需 Cordis，通过 DSH 自己的 vendor / 其声明的 `@deepseek-ai/cordis` 版本，由 `fysun-dsh` lockfile 钉死。

### 4.4 标准与其它开源组件（按平面）

只列计划真正会碰到的；每项必须有落点。许可证以各项目 LICENSE 为准，开工时写入 NOTICE。

| 组件 | 许可证（已知） | 层级 | 落点 | 不用于 |
|---|---|---|---|---|
| [MCP 规范](https://modelcontextprotocol.io) + 官方 SDK（Rust 优先评估 `rmcp` / 等价成熟 crate） | 规范开放；SDK 各自 | A 或 B | `mcp_server` 资产、`mcp_guard` EP、Service MCP facade | `fysund`↔Runtime 控制通道 |
| [A2A](https://a2a-protocol.org) v1.0 | 规范开放 | A/B | `fysun-gateway` Endpoint、Agent Card、跨租户 Handoff | 基座内部 Handoff 传输 |
| [Agent Skills](https://agentskills.io) | Apache-2.0 | D + 格式 | `skill` 资产 | 其它 kind 的包格式 |
| OCI artifact（`oci-client` / ORAS / `oras-go` 的 Rust 等价） | Apache-2.0 常见 | A | `fysun asset pack/push`；Lock digest | 容器编排 |
| [Cedar](https://github.com/cedar-policy/cedar) `cedar-policy` crate | Apache-2.0 | A | Policy 对象、EP 快照评估 | Plan / 计量表达 |
| [OpenTelemetry](https://opentelemetry.io) Rust SDK | Apache-2.0 | A | `fysun.*` 导出；钉 `gen_ai.*` commit（Q15） | 账本存储 |
| [agentgateway](https://github.com/agentgateway/agentgateway) | Apache-2.0 | B（R4 评估） | 候选 `external_gateway` EP；借用 Pass/Reject/Mutated | **不嵌入** `fysund`（ADR-0012 §D1 保留） |
| jsonschema / schemars | 各异 | A | manifest schema、负例向量 | — |
| rusqlite + 编号迁移 | MIT | A / C | 单机账本；迁移经验来自 Personal | 多节点共识（R5 前不做） |
| Sigstore / cosign（Q22） | Apache-2.0 | A 候选 | 资产签名 | 未决策前 Policy 可关签名要求 |
| MemOS / Mem0 / Letta | 各异 | B | `memory_provider` 资产 | 不自建记忆引擎 |
| Claude Code / Codex hooks.json | 文档 | D | `hooks` EP | 安全边界宣称 |
| zizmor / cargo-deny / cargo-audit | 各异 | A（CI） | 供应链扫描 | — |

名称消歧（避免复用错项目）：

- **CORDIS**：本计划仅指 [cordiverse/cordis](https://github.com/cordiverse/cordis)（DSH 插件内核）。欧盟 CORDIS 科研门户、其它同名仓库不在范围内。
- **Cognitive OS**：仅 [agentkernel/cognitive-os](https://github.com/agentkernel/cognitive-os)。`aegntic/cognitive-os`、`eugenexonr/cognitive-os` 是不同项目，不引用。
- **AgentKernel**：`cih1996/AgentKernel`、`thrashr888/agentkernel` 与本前身无关，不引入。

### 4.5 复用决策总表（实现评审用）

| FysunOS 模块 | 复用 | 自研原因 |
|---|---|---|
| 治理账本 / replay / 恢复 | C：CognitiveOS | 事件模型已变，必须改名重写，但算法不重造 |
| Cedar PDP | A：cedar-policy | 不自造策略语言 |
| manifest / Lock schema | 自研 + A：jsonschema | 十一 kind 是本产品合同 |
| OCI pack/push | A：OCI client | 不自造包格式 |
| Runtime ABI 传输 | 自研 JSON-RPC；可借鉴 DSH jsonrpc-agent 形状 | 控制通道不是 MCP |
| `fysun-dsh` | B：DSH + D：Cordis 生命周期 | 适配器，不吸收 loop |
| DSH headless 薄包装 | B：`dsh --profile headless` | R2 最小面 |
| `fysun-mcp-guard` | 自研进程；C：Effect 协议；D：agentgateway 三值 | ADR-0012 不嵌入；重定义改为独立进程 |
| `fysun-gateway` | A：A2A/MCP SDK；B：后期可对照 agentgateway | 入口 EP + AaaS，不是通用 API 网关替代 |
| Secret Store | C：Personal 边界；后端 Q2 | 解析权在 `fysund` |
| Verifier | C：Personal executor | 独立性记录是本仓合同 |
| 沙箱 / 终端 / LSP | E：留给 DSH | P8 |
| 记忆引擎 | B：第三方 provider | 01 §8 |
| UI | 最小 Approval 面自研；聊天 UI 不做 | 审批面只由 `fysund` 提供 |

---

## 5. 目标仓库形态与组件边界

### 5.1 进程与信任边界

```text
[消费者]  A2A / MCP / HTTP / CLI
              │
              ▼
     fysun-gateway          ← EP kind=gateway（数据面入口）
              │  invocation / policy_pull / observe
              ▼
           fysund           ← 唯一权威写入者、PDP、Approval；不在转发链
              │  Runtime ABI (stdio JSON-RPC / 后期 gRPC)
              ▼
     Runtime = Harness 资格化实例（DSH headless 或 fysun-dsh）
              │
              ├── hooks EP（进程内 PreToolUse）
              ├── fysun-mcp-guard（可选独立进程）
              └── harness_adapter（fysun-dsh 内部拦截）
                         │
                         ▼
                  工具 / MCP server / Provider
```

`fysun` CLI 只打管理 API 与 hooks 命令，不持有权威。

### 5.2 建议目录（R0 建骨架，允许在 R1 微调一次）

本仓从零开始，**不要**复制 CognitiveOS 的 `core/personal/enterprise/clients` 四段式——那是另一个产品的身份（CognitiveOS ADR-0054）。建议：

```text
fysun-os/
  docs/                    # 已有；redefinition 在 docs/redefinition
  schemas/                 # JSON Schema draft 2020-12：manifest、事件、Lock、错误
  conformance/             # 声明式向量（按 EP 组合分列）
  crates/
    fysun-types            # 对象、事件、错误码；无 IO
    fysun-schema           # schema 绑定生成
    fysun-ledger           # event-sourced 存储、replay、投影
    fysun-policy           # Cedar 子集封装、simulate
    fysun-assets           # digest、OCI pack、manifest 校验
    fysun-compose          # resolve / match / lock
    fysun-runtime-abi      # JSON-RPC 类型与 server/client
    fysun-ep-contract      # EP 契约类型
    fysun-conformance      # runner
  apps/
    fysund                 # 组合根：Registry/Composition/Runtime/Service/Governance
    fysun                  # CLI
    fysun-gateway          # R3
    fysun-mcp-guard        # R4（R2 可用同 crate 的最小 harness_adapter 替身）
  adapters/
    fysun-dsh              # TS bundle + profile；独立 pnpm workspace
  deploy/                  # systemd / windows 用户服务
  NOTICE                   # CognitiveOS / DSH / Cedar 等归属
```

语言默认：**Rust 2021 + 钉 rust-toolchain**（继承 CognitiveOS / 根 README 方向）；`fysun-dsh` 为 TypeScript，与 DSH 同工具链（Node ≥ 当前 DSH 要求，pnpm）。禁止第三个运行时进入 TCB，除非 spike 证明某 SDK 只存在于该语言且仅用于非权威适配器。

### 5.3 API 分组与实现归属（[08](08-api-surface.md)）

| 组 | 宿主 | R 阶段 |
|---|---|---|
| Registry | `fysund` | R1 |
| Composition | `fysund` | R2 |
| Runtime API + Runtime ABI | `fysund` ↔ Runtime | R1 资格化 / R2 运行 |
| Service | `fysund` 目录 + `fysun-gateway` 调用面 | R3 |
| Governance | `fysund` | R1 起最小面，R4 闭合 |
| EP 契约 | `fysund` ↔ EP | R2 最小、R4 完整 |

传输：本机 stdio / Unix socket（及 Windows named pipe）JSON-RPC；跨机 gRPC 或 HTTP JSON-RPC 在 R3 定（[08 §6](08-api-surface.md)）。

---

## 6. 工程操作系统

### 6.1 证据纪律

任何 PR / release 说明必须能指出属于四类状态的哪一类。禁止「已支持 F3」这类句子，除非附：vectors 列表 + runner 报告 digest + EP 组合名。

安全负例不可 `not-applicable` 降级豁免（继承 CognitiveOS conformance README）。

### 6.2 CI 门（从 R1 起强制，R0 先打通构建）

| 门 | 命令（名称可调） | 失败含义 |
|---|---|---|
| 构建 | `cargo build --workspace`；适配器 `pnpm -r build` | 实现不可用 |
| 单测 | `cargo test --workspace` | — |
| schema 一致性 | registry ↔ schema ↔ 文档枚举 grep | 合同漂移 |
| 定义句 / 热路径禁词 | grep：`fysund` 不得描述为 MCP server / 代理 / 渲染方；文档不得声称实现 loop | ADR-0013 负例 6 |
| replay 金样 | 两次 replay 投影 digest 相同 | F1 |
| cargo-deny / 许可证 | 新依赖必须已分类 A–E | 供应链 |
| conformance | 当前里程碑适用向量 | 局部绿 ≠ Profile |

### 6.3 上游钉版

| 上游 | 钉在 | 漂移处理 |
|---|---|---|
| DSH | git sha + 声明 semver；写入 harness 资产 `bundle_digest` 规则 | 每日非阻塞 job 对最新 tag 跑 R2 最小向量 |
| Cordis | 随 DSH vendor，不单独升级 | 只在 `fysun-dsh` changelog 记录 |
| CognitiveOS 抽取 | 抽取时记录 source path + revision + 改名对照 | 不跟踪上游功能列车 |
| Cedar / MCP SDK / A2A | Cargo/pnpm lock | 主版本升级走 ADR |

### 6.4 文档同步

- 实现与 `docs/redefinition` 冲突：改实现，或提 ADR，不静默改负例。
- ADR-0013 采纳后按 [10 §2](10-roadmap.md) 回填 `docs/design`；本计划文件在采纳后可升为 `docs/design/12` 的继任者（编号以当时文档地图为准）。

---

## 7. 阶段总览（R0–R5）

依赖：R0 → R1 → R2 是硬序列。R3 与 R4 按 [10 §7](10-roadmap.md) 决策 3 的**本文默认：先 R3 服务平面**；R2 必须已有一个最小 EP（headless 薄包装作 `harness_adapter` 或最小 `mcp_guard`）覆盖首个闭环的 F3。R5 依赖 R3+R4。

```text
R0 定位闸门 ──▶ R1 资产/账本/资格化 ──▶ R2 组合+首次运行
                                            │
                                            ├─▶ R3 AaaS（gateway/租户/计量）
                                            └─▶ R4 EP 治理闭合（可部分并行，合并点在 R4 结束）
                                                      │
                                                      ▼
                                                 R5 演进/生态/边缘/Personal
```

| 阶段 | 用户可感知交付 | 退出条件（摘要） |
|---|---|---|
| R0 | 术语与 ADR 一致；空仓骨架 | 一致性 grep 通过；ADR-0013 负例 6 |
| R1 | `fysun asset *`；Runtime 能 qualify | 02 §9 负例、replay 稳定、`runtime/qualified` 含 `harness_asset_id` |
| R2 | 一个 Agent 跑完并恢复 | 03 §8、04 §8 负例 2/8；热路径禁令 |
| R3 | 两租户调用同一 Service | 05 §12 负例 |
| R4 | F3/F7/F9 按 EP 组合宣称 | 04 §8 负例 3–6；06 §10；`fysund` 不在转发链 |
| R5 | 换 Harness 证据不串；Personal 单路径 | 09 §9；两 Harness 跑同一 Lock 变体 |

并行规则：文档回填、spike、schema 草稿可与上一阶段尾部重叠；**资格化、Lock、Effect 协议**不可跳级。

---

## 8. R0 · 定位与术语闸门

**目标**：让后续代码只有一套名字、一套禁令、一套阶段语言。

### 8.1 工作包

| ID | 工作 | 产出 | 复用 |
|---|---|---|---|
| R0-D1 | 所有者采纳或修订 ADR-0013 | `docs/adr/0013-*.md` 从 draft → accepted，或书面延期范围 | — |
| R0-D2 | 按 [10 §2](10-roadmap.md) 给 `docs/design` 打继承 / 降级条；重写 01 / 03 / 12 或先加 banner 指向本文件夹 | 无双定义句 | — |
| R0-D3 | 根 README 定义句与组件表与 00 §8 / README 命名对齐 | 对外文字一致 | — |
| R0-D4 | 仓库骨架：workspaces、rust-toolchain、schemas/ 空目录、NOTICE 初稿 | 可 `cargo new` 风格构建 | CognitiveOS 的 toolchain / CI 门经验（D） |
| R0-D5 | 一致性检查脚本：禁词、术语表枚举 ↔ schema 占位 | CI job | CognitiveOS `check:consistency` 模式（D） |
| R0-D6 | 抽取地图：CognitiveOS personal-v1.0.0 vs 本地 6f562bf；DSH 当前 HEAD vs cd5ef81 | `docs/research/` 一篇钉版笔记 | C 的输入 |

### 8.2 验收

- grep：`fysund` 不再被描述为 MCP server / 代理 / 渲染方。
- ADR-0013 负例 6：设计文档不把基座写成实现 loop / prompt 装配 / 沙箱。
- 「运行基座」不被写成「运行时（runtime）」。
- 本计划与 00 术语种类枚举一致（11 个 kind）。

### 8.3 退出后才能做的事

R1 的 schema 冻结、公开 crate 命名、许可证选择（Q6）。R0 未完成不得宣称 v0.8.0。

---

## 9. R1 · 资产与 Runtime 登记

**目标**：能力管理的「登记」一半可运行；治理账本成为唯一真源；Runtime 能被资格化，但还不必跑 Agent。

### 9.1 工作包

| ID | 工作 | 细节 | 复用 |
|---|---|---|---|
| R1-A1 | `fysun-types` + 事件 schema | 00 §4–§5 事件族；通用字段含 `node_id` / `origin_seq` / `tenant_id?` | C：CognitiveOS 事件信封经验 |
| R1-A2 | `fysun-ledger` | append-only、SQLite WAL、编号迁移、投影、两次 replay digest | C：store + replay |
| R1-A3 | manifest JSON Schema（11 kind） | 02 §2–§3；`declares.effects[]` 必填可空；无 secret 值槽 | A：schemars/jsonschema |
| R1-A4 | canonical digest | 目录 canonical + OCI config=manifest | A：OCI；算法钉 SHA-256，media type 走 Q22 临时值并标注 |
| R1-A5 | `fysun asset pack/push/register/publish` | 07 §3 流程；`fysund` 只存登记与 digest | A：OCI client |
| R1-A6 | Registry API | 08 §1；描述面无本体内容 | — |
| R1-A7 | Policy 载入 + `policy.simulate` | Cedar 子集、schema 校验；实体模型按 06 §3 扩展（Tenant/Asset/Service 可先占位） | A：cedar-policy |
| R1-A8 | Runtime 登记 / qualify | `describe()` 五字段与 harness manifest 互证；`runtime/qualified` 必含 `harness_asset_id` | D：DSH describe 形状；C：Personal 资格化经验 |
| R1-A9 | SecretRef 登记 API | 只返回引用；rotate/revoke | C：Secret 边界 |
| R1-A10 | 负例向量 02 §9 1–7、04 §8 负例 1 | 声明式 YAML + runner 雏形 | D：conformance 方法论 |

### 9.2 首个可登记资产包（fixtures）

用仓库内 fixtures，不依赖真实网络账号：

- `prompt:fysun/demo-persona@0.1.0`
- `skill:fysun/trend-research@0.1.0`（Agent Skills 目录）
- `mcp_server:fysun/mock-search@0.1.0`（stdio mock）
- `harness:deepseek/dsh@<钉版>`（先登记 manifest，即使二进制稍后在 R2 才装）
- `loop:fysun/careful@0.1.0`
- `model:fysun/mock@0.1.0`（fake provider，避免 R1 绑定真实模型）

### 9.3 验收（可判定）

- 02 §9 负例 1–7（实现阶段把草案负例登记为 `NV-R1-*`）。
- replay 两次投影 digest 稳定。
- `asset/published` 带 `policy_decision`。
- `runtime/qualified` 带 `harness_asset_id`、`control_points[]`、`native_effect_tools[]`、`describe_digest`、`bundle_digest`。
- `describe()` 与 harness manifest 不一致 → 拒绝资格化（04 §8 负例 1）。
- 描述面 / `tools/list` 无明文值槽。
- 无 `grant.issue` 公开方法。

### 9.4 非目标

不在 R1 实现 `compose.resolve`、gateway、EP 完整协议。可做 in-memory fake Runtime 仅用于 qualify。

---

## 10. R2 · 组合与首次运行

**目标**：资产组合成 Agent，绑到资格化 Runtime，跑完一个真实闭环，崩溃可恢复。`fysund` 已不在数据面。

### 10.1 工作包

| ID | 工作 | 细节 | 复用 |
|---|---|---|---|
| R2-C1 | Capability 契约类型 | 删除 `implementation`；`tool_effects[]` 由组合汇总后确认 | 自研合同 |
| R2-C2 | `agent.define` / `compose.resolve` / `compose.match` / `compose.diff` | 03 §3–§4；Lock 只含 digest | — |
| R2-C3 | 效果汇总 `max_severity` | 02 §6；只能收窄 | ADR-0008 |
| R2-C4 | Runtime ABI server/client | describe/start/resume/cancel/health；`os.observe`；start **不带渲染文本** | D：DSH jsonrpc-agent 形状；**不**用 MCP |
| R2-C5 | **默认**：DSH headless 薄包装 | 实现最小 ABI；Lock 交给 harness 自行装配；查证 Q16 | B：`dsh --profile headless` |
| R2-C6 | 最小 EP | 薄包装内 `harness_adapter` **或** 最小 `fysun-mcp-guard` 进程：persist-before-dispatch、回报、幂等键 | C：effects.rs 语义 |
| R2-C7 | Activity / Execution 状态机 | 继承 v0.7.0；字段增 `lock_digest`、`harness_asset_id`、`ep_ids[]`、`tenant_id` | C：recovery |
| R2-C8 | 独立 Verifier | schema + 来源 checklist；`inconclusive ≠ pass` | C：verification_executor |
| R2-C9 | 恢复顺序 | 账本 → Policy 版本 → Runtime 资格化 → fencing → outcome-unknown → Approval 水位 → 重新武装 → resume | C：recovery.rs + DSH armed 语义 |
| R2-C10 | `fysun` CLI | `agent compose/resolve`、`activity *`、`hook pre-tool-use`（可先 stub） | D：Personal CLI |
| R2-C11 | 并行 spike：`fysun-dsh` | 不阻塞 R2 退出；为 R5 准备 | B/D：DSH + Cordis |

### 10.2 首个闭环剧本（把 v0.7.0 12 §2 按重定义改写）

```text
1. 登记 R1 fixtures 并 publish
2. agent.define（harness=DSH headless 包装，loop、prompts、skill、mock mcp、mock model）
3. compose.resolve → Lock（全 digest）
4. compose.match → 已 qualified 的 Runtime
5. activity.create(lock_digest, tenant=owner) + start
6. fysund → start_execution(lock, bindings, grants_summary, inputs)  // 无渲染文本
7. Harness 按 Lock 拉取资产、装配、调用 mock 工具
8. EP：ActionIntent 本地持久化 → 分发 → receipt → ep.report
9. Verifier 对照契约判定
10. 一次 Approval（Policy 对某 mutation 产出 approve）→ os.observe 查询，不走 fysun.await_approval 系统工具
11. kill fysund 与 Runtime；恢复后 paused{rearm_required} → activity.resume
12. 一次 Adjustment（this_activity 改 loop.max_steps）
13. 被过滤工具调用：EP 返回与未知工具不可区分错误；账本记 rejected
```

### 10.3 验收

- 03 §8 负例 1–5：Lock 无版本范围；缺控制点不得 runnable；等等。
- 04 §8 负例 2、8：start 无渲染文本；`fysund` 不在转发链（可用进程级网络 / 管道断言）。
- 闭环 1–13 自动化。
- `execution.observe` 不返回 Approval 的 `decided_by` / `reason` / `presentation`（08 §9 负例 3）。
- 未分类工具按 `external_mutation`；`conflict.idempotency` 硬拒绝。
- 补偿必须新 Grant + 新 ActionIntent。

### 10.4 R2 的 EP 最小能力位

`persist_intent` + `receipt` 必须有；`filter_tools` 必须有。`labels` / `secret_inject` / `meter` 可在 R4 补齐，但 Secret 若在闭环出现，必须走 `SecretRef`（否则 fixture 禁止含凭据）。

---

## 11. R3 · 服务平面

**目标**：Agent 作为服务被发现与调用；两租户隔离。

### 11.1 工作包

| ID | 工作 | 细节 | 复用 |
|---|---|---|---|
| R3-S1 | Service / ServiceVersion / Endpoint 对象 | 05 §2 | — |
| R3-S2 | `fysun-gateway` | A2A Endpoint + Agent Card + MCP facade；鉴权 / 配额 / 计量 | A：A2A/MCP SDK；D：agentgateway 协议意识（不嵌入） |
| R3-S3 | Tenant / Plan / Meter | Plan 派生根 Grant；`meter/recorded` | — |
| R3-S4 | `service.publish / rollout` | staged → canary → stable；canary 可审计 | — |
| R3-S5 | Agent Card 生成与签名 | skills ↔ capabilities；gateway `/.well-known/agent-card.json` | A2A 规范 |
| R3-S6 | Runtime 池化 | 缺省 `dedicated`；`shared` 要求 `session_isolation`（04 §8 负例 7） | Q24 |
| R3-S7 | 租户隔离测试 | Memory / Artifact / Secret / Activity | — |
| R3-S8 | 审批路由最小面 | 05 §10 缺省表；gateway 只透传 `needs_human` | ADR-0011 |
| R3-S9 | Personal 旁路 | 允许 `activity.create` 不经 gateway；仍有 Tenant=owner、Plan=unlimited | 10 §7 决策 5 |

### 11.2 验收

05 §12 负例 1–8 全部自动化。特别：

- 两租户调用同一 Service：用量分离、Memory 分离、审批路由正确。
- 未 publish 的 Lock 不能经 Endpoint 调用。
- Agent Card 与 Lock 不一致 → 失败。
- 配额超出后无新 `execution/started`。
- gateway 进程可以水平起两个实例，权威仍在 `fysund`。

### 11.3 非目标

Trust 对象完整设计（R4 / Q27）、计费、聊天 UI、跨基座联邦。

---

## 12. R4 · 执行点与治理闭合

**目标**：F3 / F7 / F9 按 EP 组合可宣称；Policy 下发与 stale fail-closed；Approval 不泄漏到数据面。

### 12.1 工作包

| ID | 工作 | 细节 | 复用 |
|---|---|---|---|
| R4-E1 | EP 契约完整实现 | describe / policy_pull / report / resolve_secret / health；mTLS 跨机可选 | — |
| R4-E2 | `fysun-mcp-guard` 独立进程 | 继承 v0.7.0 06 规则；**进程外**；不再链进 fysund | C：Effect；D：Pass/Reject/Mutated |
| R4-E3 | `hooks` EP | `fysun hook pre-tool-use`；只拦不记 | D：hooks.json |
| R4-E4 | Policy 下发 + `policy_stale_ttl` | 同一 `policy_version` 在 fysund 与 EP 同输入同 outcome | A：cedar-policy 两边同版本 |
| R4-E5 | Approval 闭合 | 配对事件；路由集合；重新武装 | ADR-0011 |
| R4-E6 | 来源标签沿参数流 | 工具结果 `untrusted_external`；flags | C+扩展 ADR-0004 |
| R4-E7 | Trust 对象 MVP | Q27 的最小字段：签名密钥、conformance digest、publisher id | 关闭 v0.7.0 Q11 的「远期」 |
| R4-E8 | conformance 按 EP 组合分列 | 04 §4.5 表 | D：四类状态用语 |
| R4-E9 | agentgateway 评估 | 作为 `external_gateway` 的适配 spike：能否吃 Cedar 快照并回报 EP 事件 | B；不嵌入 |
| R4-E10 | 可选 `provider_proxy` | 多租户 token 计量；Q26 | C：Personal proxy **可选** |

### 12.2 验收

- 04 §8 负例 3–6：未资格化回报拒绝；EP 不写 Approval/Grant；stale 只放行 `local_read`；同一调用不得双 Effect。
- 06 §10 负例 1–5（Policy 一致性、审批路由、供应链、跨租户、标签）。
- ADR-0013 负例 2、3、4。
- v0.7.0 M1/M2a Effect 协议负例在 **EP 上**重跑通过（幂等、快路径仅 local_read、不可区分性等）。
- `fysun-mcp-guard` 是否保留为第一方参考 EP：按决策 4；若降级，hooks + `fysun-dsh` 必须能单独构成一个已宣称的 EP 组合。

### 12.3 与 ADR-0012 的关系

「不嵌入 agentgateway」仍然有效。重定义把守卫从 `fysund` 内拆出，**自实现独立进程**是默认；外部网关是另一 EP kind，不是替代账本。

---

## 13. R5 · 演进与生态

**目标**：组合可演进且证据不串味；第二个 Harness；边缘；Personal 不再维护第二执行路径。

### 13.1 工作包

| ID | 工作 | 细节 | 复用 |
|---|---|---|---|
| R5-V1 | Adjustment 作用于组合 | 03 §7；扩权走 Policy | ADR-0005 |
| R5-V2 | 证据分层与发布证据门 | 09 §2、§5 | — |
| R5-V3 | 弃用 / 撤销级联 | Lock / ServiceVersion `not_runnable` / `suspended` | — |
| R5-V4 | `fysun-dsh` 全控制点 | 六个控制点；`approval/request` 拦截；`tools/post-execute`；禁止 policy `never`；bundle 钉版 | B/D：DSH、Cordis dispose |
| R5-V5 | 第二 Harness | Claude Code 或 Codex 薄包装 + hooks + mcp_guard | D：hooks.json |
| R5-V6 | `fysund-edge` | 继承 v0.7.0 11 §3；带本地账本的 EP 宿主 | C：账本；ADR-0010 |
| R5-V7 | Personal 三合一 | 发布者+运营者+租户；业务对象在应用层 | C：Personal 产品经验，不迁 OS 对象 |
| R5-V8 | 消费反馈只经 Verification | 09 §6；无评分对象 | — |

### 13.2 验收

- 09 §9 负例 1–4。
- 同一 Lock 的两个变体（不同 Harness）证据分别累积，不迁移。
- Personal 路径：无第二套工具执行 / 第二套账本。
- 边缘：离线 `approve` 阻塞；escrow 越界拒绝；合并冲突不自动补偿（继承 NV-T8-*）。

---

## 14. Spike / PoC（R0 末–R1 初必须完成）

每个 spike 有明确通过 / 放弃标准，避免「研究型无限期」。默认时间盒 3–5 个工作日。

| ID | 问题 | 方法 | 通过标准 | 失败则 |
|---|---|---|---|---|
| S1 | DSH headless 是否含独立于 bundle 的原生工具（Q16） | 钉版安装，跑空任务，抓 tools 列表与进程模块 | 清单写入 harness manifest `native_effect_tools[]`；若有原生工具，R2 EP 必须覆盖或 Policy 拒绝非 local_read | 改 R2 默认 EP 组合 |
| S2 | DSH JSON-RPC / sdk-minimal 能否承载 Runtime ABI 而不经 Web UI | 对照官方 Python/TS SDK 与 `dsh --profile headless` | 能 `start`/`cancel` 或可写 ≤200 行包装 | 自研薄包装进程，仍不把 loop 迁入 OS |
| S3 | OCI artifact 以 manifest 为 config 的 media type | 用 `oci-client` 推到本地 registry（zot / distribution） | pack/push/pull digest 稳定 | 暂用目录 canonical digest，Q22 跟踪 |
| S4 | cedar-policy crate 在 EP 进程评估同一 snapshot | 两边加载同一 policy+entities | 同输入同 outcome | 禁止 EP 本地「简化评估器」 |
| S5 | CognitiveOS effects.rs 与重定义 EP 本地持久化的接口差 | 读 `personal-v1.0.0` 实现，列必迁类型 | 抽取地图获 owner 签字 | 按 02 语义全新实现，仍保留负例 |
| S6 | Windows named pipe vs stdio 作为本机 ABI（开发机为 win32） | 原型往返 | CI 与开发机各一条路径 | 文档限定 R1 仅 stdio |
| S7 | agentgateway 作 external_gateway 的差距 | 对照 EP 契约 vs mcpGuardrails | 差距表；R4-E9 决策 | 保持自实现 mcp_guard |
| S8 | A2A Agent Card 签名与 gateway 暴露 | 最小 server | Card 字段能从 capabilities 派生 | R3 先 HTTP Endpoint，A2A 延后但对象模型保留 |

S1、S2、S4、S5 为 R2 开工阻断项。

---

## 15. 测试与 conformance

### 15.1 分层

| 层 | 内容 | 何时绿 |
|---|---|---|
| 单元 | 类型、digest、Cedar 封装、状态机 | 每个 PR |
| Schema 向量 | 非法 manifest、Lock 含范围、secret 值槽 | R1 |
| Replay 金样 | 固定事件序列 → 投影 digest | R1 |
| EP 契约向量 | 按 04 §4.5 组合分列 | R2 最小组；R4 全表 |
| 闭环 e2e | §10.2 剧本 | R2 |
| 多租户 e2e | 两 Tenant | R3 |
| 故障 | 杀进程、stale policy、双 EP 重复 Effect、幂等冲突 | 随对应 R |
| 安全负例 | T12–T15 + 继承 T1–T11 中仍适用者 | 不得豁免 |

### 15.2 conformance 组合（实现登记名）

| Profile 名 | EP 组合 | 最早宣称阶段 |
|---|---|---|
| `ep.none.lifecycle` | 无 EP | 永不宣称 F3/F7/F9 |
| `ep.headless.adapter-min` | harness_adapter 最小 | R2 |
| `ep.mcp-guard.gateway` | mcp_guard + gateway | R4 |
| `ep.mcp-guard.hooks` | mcp_guard + hooks | R4 |
| `ep.dsh.adapter` | fysun-dsh | R5 |
| `ep.dsh.adapter+proxy` | + provider_proxy | R5 可选 |

### 15.3 负例编号

实现阶段统一 `NV-T<威胁>-<序号>` 或 `NV-R<阶段>-<序号>`，并在 [13 威胁模型](../design/13-threat-model.md) 修订时交叉引用。草案文档中的「负例 1–n」在 R0-D2 录入登记表，禁止只存在于散文。

---

## 16. 度量

继承 v0.7.0 12 §4，并新增 [10 §4](10-roadmap.md) 项。R1 先打点，R2 末出基线报告，阈值在有 10 次以上闭环后再定。

| 度量 | 来源 |
|---|---|
| daemon 启动耗时、空闲内存 | fysund |
| 调度延迟 trigger → start_execution | 账本时间戳 |
| 资产登记到可解析的步骤数 | CLI 脚本 |
| 新 Harness 打包到 `runtime/qualified` 的步骤数 | 脚本 |
| `compose.resolve` 时延 | 直方图 |
| Service 发布到首个调用步骤数 | R3 |
| EP 回报延迟与拒绝率 | R4 |
| 每租户用量分离正确率（负例通过率） | R3 |
| 人工介入率、审批不可用率、回滚率 | 继承 |
| 验证独立性比例 | 继承 |
| `runtime/disqualified` 按 reason | 继承 |
| fysund CPU 在工具调用期间 ≈ 0（热路径禁令的性能表达） | R2 起抽样 |

---

## 17. 风险、许可证与供应链

| 风险 | 级别 | 缓解 |
|---|---|---|
| DSH preview 频繁破接口 | 高 | sha 钉版；薄包装隔离；漂移 job 非阻塞；R2 不依赖全控制点 |
| Cordis API 不稳 | 中 | 不进入 fysund；只经 DSH vendor |
| CognitiveOS 公开仓与本地基线分叉 | 中 | S5 对照；抽取时记录 path+sha |
| 误把同名项目当依赖 | 中 | §4.4 消歧表；CODEOWNERS 审依赖 PR |
| FysunOS 自身许可证未定（Q6） | 高 | R0 决策；在未定前不发布二进制；NOTICE 先列第三方 |
| Cedar 子集与 EP 评估分叉 | 高 | 同一 crate 版本 + S4 金样 |
| Windows 开发机链接 Rust 限制（CognitiveOS 曾登记） | 中 | 开工时记录本机工具链；CI 用 Linux runner 为权威 |
| 把 mcp_guard 再内联回 fysund「图省事」 | 高 | CI 架构测试：fysund 二进制不链接 MCP client 转发路径 |
| 资产签名方案拖延（Q22） | 中 | R1 Policy 默认不要求签名；namespace 允许列表兜底 |
| `shared` Runtime 隔离无法证明（Q24） | 中 | R3 缺省 dedicated |
| 计量信任（Q26） | 中 | Harness 自报 token 标 untrusted_external；多租户计费级计量需 proxy |

第三方许可证（已知，R0 写入 NOTICE）：

- CognitiveOS 抽取代码：Apache-2.0（保留版权与 NOTICE）。
- DSH / Cordis：MIT（适配器侧）。
- Cedar：Apache-2.0。
- Agent Skills 规范：Apache-2.0。

本仓 LICENSE 待 Q6。在此之前，抽取 CognitiveOS 代码的文件头必须可独立合规。

---

## 18. 开放问题在计划中的处置

继承 [10 §6](10-roadmap.md)。开发计划只分配**何时必须关闭**，不擅自替 owner 选值。

| ID | 最迟关闭点 | 本计划缺省（可被 owner 推翻） |
|---|---|---|
| Q1 传输 | R3 | 本机 JSON-RPC；跨机 gRPC |
| Q2 Secret Store 后端 | R1 可用文件/OS keyring；生产后端 R4 | R1：本机加密文件 |
| Q3 Web UI | R4 最小 Approval 页 | 不做聊天 UI |
| Q4 judge 校准 | R5 | human verifier 为主 |
| Q6 许可证 | **R0** | 无缺省；阻断发布 |
| Q12 多节点一致性 | R5 后 | R1–R4 单节点 |
| Q13 CaMeL | 不进 MVP | 标签不宣称值级 IFC |
| Q15 gen_ai.* | R4 | 钉 commit，自有 `fysun.*` |
| Q16 DSH headless 原生工具 | **S1，R2 前** | — |
| Q17–Q20 | R4 | 继承 v0.7.0 行为，不扩展 |
| Q22 manifest 签名 / media type | R1 临时；R4 可要求签名 | SHA-256；media type 临时 `application/vnd.fysun.manifest.v1+json` |
| Q23 loop_schema 通用子集 | R2 | 不强制通用子集；只校验目标 harness schema |
| Q24 shared 隔离证明 | R3 | 缺省 dedicated |
| Q25 EP 崩溃未回报 | R4 | 继承 outcome-unknown；TTL 显式配置 |
| Q26 计量可信度 | R3 声明；R4 proxy 可选 | 自报不计入计费级 |
| Q27 Trust 对象 | R4 | 最小字段见 R4-E7 |
| Q28 gateway A2A 能力位 | R3 | 先 request/response；streaming 能力位如实声明 |
| Q29 BYO 槽位 | R3 | 允许 prompt/skill/model 凭据；效果类别不可降级在 `fysund` 校验 |

已关闭且计划必须遵守：Q5 Tenant 核心；Q11 Trust 改 R4 必做；Q14 不做 Provider 路由；Q21 投递 TTL 废止，转 Q25。

---

## 19. 待所有者决策（开工清单）

[10 §7](10-roadmap.md) 五项保持；本计划给出**推荐值**（推荐 ≠ 已采纳）：

| # | 问题 | 本计划推荐 | 影响 |
|---|---|---|---|
| 1 | 是否采纳 ADR-0013 | 采纳后再写生产代码 | R0 闸门 |
| 2 | R2 首个 Harness | DSH headless 薄包装；`fysun-dsh` 并行 spike | §10、S1/S2 |
| 3 | R3 vs R4 先后 | 先 R3（对外）；R2 最小 EP 已覆盖首个 F3 | §7 |
| 4 | 是否保留第一方 `fysun-mcp-guard` | **保留为参考 EP**（独立进程）；hooks+dsh 为第二组合 | R4-E2 |
| 5 | Personal 是否强制经服务平面 | 允许绕过 gateway；Tenant 仍存在 | R3-S9 |

额外决策：

| # | 问题 | 推荐 |
|---|---|---|
| 6 | Q6 许可证 | 需 owner 选：Apache-2.0 与 CognitiveOS 抽取最省事 |
| 7 | 是否把 agentgateway 列入 R4 正式 EP | 仅 spike；默认不依赖 |
| 8 | 本仓 crate 是否开源同步于实现 | R2 闭环后再谈，避免 preview 期合同漂 |

---

## 20. 追溯矩阵（设计 → 工作包）

| 设计条款 | 工作包 |
|---|---|
| ADR-0013 D1 定义句 | R0-D2/D3、CI 禁词 |
| D2 十一 kind 资产 | R1-A3/A5 |
| D3 Runtime=资格化实例 | R1-A8、R2-C4 |
| D4 三平面 + 不在热路径 | 全阶段架构测试、R2-C6、R3-S2、R4-E2 |
| D5 Policy 下发 / approve 回中心 | R1-A7、R4-E4/E5 |
| D6 Tenant 核心 | R1 字段、R3-S3/S7 |
| D7 标准优先 | §4.4、R1-A4、R3-S2 |
| D8 F1–F9 执行者 | §2.2、R2/R4 负例 |
| 02 资产模型 | R1 |
| 03 组合 | R2-C1–C3 |
| 04 Runtime / EP | R2-C4–C9、R4 |
| 05 AaaS | R3 |
| 06 治理变化 | R1-A7、R4 |
| 07 生态 | §4、R5-V4/V5 |
| 08 API | 各 R 的 API 行 |
| 09 演进 | R5-V1–V3 |
| 10 R0–R5 | §8–§13 |
| v0.7.0 09 迁移清单 | §4.1–4.3 |
| ADR-0012 不嵌入 | R4-E2/E9 |

---

## 附录 A · 首 30 日建议节奏（假定 ADR-0013 已采纳）

仅作排期参考，不是承诺日历。

| 工作日 | 焦点 |
|---|---|
| 1–3 | R0 文档闸门、NOTICE、骨架、禁词 CI；发起 S1/S2/S5 |
| 4–8 | 完成 S1/S2/S4/S5；冻结 DSH sha；ledger 原型 |
| 9–15 | R1 schema + pack/register + replay 金样 |
| 16–22 | Runtime qualify + Policy simulate + 负例 NV-R1 |
| 23–30 | compose.resolve 原型 + headless 薄包装 hello；R2 剧本 1–6 手工打通 |

---

## 附录 B · 给实现代理的硬指令

1. 先读本文件夹 01、00、04、08，再写代码。
2. 新增依赖必须在本文件 §4 分类；未分类依赖禁止合并。
3. 不要在 `fysund` 里创建 `mcp` 转发模块。
4. 不要实现 loop。需要 loop 时登记 `loop` 资产并交给 Harness。
5. 不要复制 CognitiveOS 目录名冒充进度。
6. 每个里程碑 PR 必须列出：新增向量、关闭的指定负例、未宣称的 Profile。
7. 与本计划冲突时，开 ADR 或修订本文件，不要只改代码。

---

## 附录 C · 修订记录

| 日期 | 说明 |
|---|---|
| 2026-09-12 | 初稿：按 redefinition 00–10 与 ADR-0013 展开 R0–R5；核验 CognitiveOS / DSH / Cordis 及标准组件的复用边界。外部 revision 以 sources.md 已登记值为准，公开仓 HEAD 待开工钉版。 |
