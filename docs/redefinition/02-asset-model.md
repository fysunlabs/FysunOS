# 02 · 能力资产模型

- 状态：提案草案 v0.8.0-draft（2026-09-11）
- 决策依据：[ADR-0013 草案](adr-0013-draft-repositioning.md) §D2、§D7
- 前置：[00 术语表](00-terminology.md)、[01 定位](01-positioning.md)

本文回答"什么算能力资产、怎么描述、怎么版本化、怎么声明效果与兼容、生命周期如何"。字段是设计意图，正式 schema 在实现阶段登记并配负例。

## 1. 什么是资产、什么不是

**资产** = 可复用、可分发、可被 Agent Definition 引用之物。判定标准三条同时满足：有稳定身份与版本；内容可 digest；可脱离某次 Execution 存在。

| 是资产（`kind`） | 不是资产 |
|---|---|
| `prompt` 提示词包（系统提示、角色片段、模板） | 一次 Execution 中装配出的具体上下文（context_view） |
| `skill` Agent Skills 目录（`SKILL.md` + 资源） | Skill 正文在模型输入中的呈现 |
| `tool` 单个工具描述符 + 实现引用（本地二进制 / 函数 / HTTP） | 工具的一次调用结果（Artifact，带标签） |
| `mcp_server` MCP server 的启动 / 连接描述 + 工具目录快照 | 与 server 的一条会话 |
| `harness` Agent 程序（loop 实现、prompt 装配、工具分发、会话日志） | Harness 进程、会话账本 |
| `loop` Harness 消费的策略配置（步数预算、继续 / 停止、重试、委派、compaction 触发） | Loop 运行状态（armed / step 计数） |
| `model` Provider + 模型标识 + 参数约束 | 模型的一次调用、KV cache |
| `memory_provider` 记忆引擎接入描述 | 记忆条目（Memory 对象，治理侧） |
| `knowledge` 只读知识包（文档集、索引快照） | 检索结果片段 |
| `runtime` Harness 的安装实例描述（Node、路径、`bundle_digest`） | Runtime 进程 |
| `agent` Agent Definition（资产组合） | Activity / Execution |

设备、账号、工作区路径、上下文源等仍是 **Resource**（授权与计量对象），不是资产；两者都可被 Grant 覆盖。

## 2. 统一 manifest

```yaml
apiVersion: fysun.io/v1alpha1
kind: skill                      # 00 §3 封闭枚举
metadata:
  name: acme/trend-research
  version: 1.2.0                 # SemVer；预发布与构建元数据允许
  digest: sha256:…               # 内容 canonical digest；由打包工具计算，fysund 登记时重算比对
  publisher: principal:acme      # 登记者 Principal
  source: { repo: https://…, revision: …, oci: registry.example/acme/trend-research@sha256:… }
  license: Apache-2.0
  signatures: [ { keyid: …, sig: … } ]   # 可选；Policy 可要求
  labels: { … }                  # 自由标签，供发现
spec:                            # 按 kind 的专有字段（§3）
  …
declares:                        # 全部 kind 共用的声明面
  effects: [ { tool: web.fetch, effect_category: external_read } ]   # 该资产会引入的工具效果（六值，继承）
  requires:                      # 依赖与运行条件
    assets: [ { kind: mcp_server, ref: "mcp_server:acme/search@^2" } ]
    control_points: [ ]          # 对 Harness 控制点的要求（如 context_injection）
    resources: [ { type: device, … } ]
    environment: { offline: false, latency_ms: …, … }
  provides:                      # 对外提供什么（供组合解析）
    tools: [ web.fetch ]
    control_points: [ ]          # harness 专用
  compatibility: { harness: ["harness:deepseek/dsh@>=0.7 <0.8"], model: [ … ] }
evidence: [ ]                    # 登记后由 fysund 追加，不由发布者填写
```

规则：

- `metadata.digest` 是身份的一部分；`fysund` 登记时重算，不一致 → `asset.digest_mismatch`。
- `declares.effects[]` 必填（可为空数组，缺字段 → `asset.invalid_manifest`）。它是 Effect 类别三源合一中来源 (a) 的**资产级**形态：
  Capability `bounds.tool_effects[]` 从组合中的全部资产 `declares.effects[]` 汇总生成，只能再收窄（03 §2）。
- `evidence[]` 只由 `fysund` 追加（Verification 引用、conformance 结果、消费反馈折叠）。
- manifest 不含任何 secret 值槽（`SecretRef` 三规则继承）。

## 3. 各 kind 的专有字段（`spec`）

### 3.1 `prompt`

`role`（system / developer / user-template）、`content_ref`（文件）、`variables[]`（名、类型、必填、来源约束：可要求 `integrity >= trusted_user`）、
`composition_hints`（前置 / 后置、可被 compaction 折叠与否）。**基座不装配 prompt**，只登记与版本化；装配是 Harness 的事。

### 3.2 `skill`

采用 Agent Skills 规范：`skill_dir`（含 `SKILL.md`）、`allowed-tools[]`、`compatibility`。`declares.effects[]` 必须覆盖 `allowed-tools` 中每个工具（未覆盖者按 `external_mutation`，
并在 `asset.health` 标"未分类"，继承 v0.7.0 02 §2.1）。

### 3.3 `tool`

`descriptor`（JSON Schema 输入 / 输出，与 MCP tool 描述符同形）、`impl`（`mcp` 引用 / 本地命令 / HTTP）、`default_effect_category`（来源 (b)）、
`annotations`（MCP tool annotations，只作登记建议输入，不作类别依据，继承 ADR-0008）、`reversibility`、`idempotency`（是否支持幂等键）。

### 3.4 `mcp_server`

`transport`（stdio / streamable_http）、`launch`（命令、参数、环境变量名——**只列名**，值经 `SecretRef`）或 `url`、`credentials[]: SecretRef`、
`tools_snapshot`（登记时 `tools/list` 快照 + digest，用于 T2 供应链漂移检测）、`resources_snapshot`、`prompts_snapshot`、每工具 `default_effect_category`。

### 3.5 `harness`

- `runtime_abi`：实现的最小面版本；`control_points[]`（`context_injection` / `checkpoint` / `fork` / `compaction_hook` / `session_link` / `subagent_delegation`，继承）；
- `native_effect_tools[]`：`{tool, effect_category}`，**如实声明，宁多勿少**（继承 ADR-0007）；
- `loop_schema`：该 Harness 接受的 Loop 策略参数 schema（`loop` 资产按此校验）；
- `prompt_slots[]`：可注入 prompt 资产的位置（system / developer / per-turn）；
- `skill_loading`：支持的 skill 目录布局与加载时机；
- `tool_surface`：MCP client 能力（tasks / elicitation / resources / prompts 支持位）、hooks 支持（PreToolUse 等）、可否把工具面限制为指定 server；
- `session_ledger`：会话账本格式与可访问性（`session_link` 控制点的前提）；
- `install`：安装方式（OCI image / npm / 二进制 / 源码 revision）与 `bundle_digest` 计算规则；
- `adapter?`：若需适配器（如 `fysun-dsh`），引用适配器资产与其前置条件（如"禁止 approval policy `never`"）。

### 3.6 `loop`

`harness_ref`（目标 Harness 与版本范围）、`params`（按该 Harness 的 `loop_schema` 校验）：`max_steps`、`stop_conditions[]`、`retry`、`delegation`（是否允许 subagent、
最大并发、每个子任务预算比例）、`compaction`（触发阈值）、`human_in_loop`（何时向 Harness 请求输入——注意这不是审批面，审批只由 `fysund` 提供）。
Loop 是**配置**，不是代码；Harness 是代码。基座不解释 Loop 语义，只校验与绑定。

### 3.7 `model`

`provider`、`model_id`、`params_bounds`（温度、上下文窗口、输出上限）、`pricing_ref`（供计量换算，非权威）、`credentials: SecretRef`、
`access`（`direct` = Harness 直连，F5 范围外；`via_ep` = 经 `provider_proxy` 执行点，F5 范围内）。

### 3.8 `memory_provider`

`engine`（MemOS / Mem0 / Letta / 自建）、`endpoint` / `launch`、`credentials: SecretRef`、`scoping`（是否支持 per-tenant / per-activity 命名空间——多租户必需）、
`admission_hook`（准入事件如何回报 `fysund`，继承 Memory 准入模型）。

### 3.9 `knowledge`

`content_ref`（文档集 / 索引快照，内容寻址）、`integrity_default`（缺省 `verified_data` 需有 Verification 引用，否则 `untrusted_external`）、`freshness`、`retrieval`（检索接口约定）。

### 3.10 `runtime`

`harness_ref`（哪个 Harness 资产版本的实例）、`node_id`、`install_path`、`bundle_digest`、`describe_snapshot`、`qualification`（投影）、`attachment`、`health`
（继承 v0.7.0 07 §1.1，`type` / `version` 由 `harness_ref` 派生）。Runtime 资产由 `runtime.register` 产生，通常不经资产仓库分发。

### 3.11 `agent`

Agent Definition，见 [03](03-composition.md)。

## 4. 版本、digest、来源与供应链

- **版本**：SemVer；`deprecated` 可指 `successor`；契约不兼容的变化产生新 `name` 或主版本并保留 `lineage`（继承 Capability 谱系规则）。
- **digest**：OCI artifact digest 或目录 canonical digest；Lock 只引用 digest，不引用浮动版本范围。
- **来源**：`source.repo + revision` 或 `source.oci`；登记者 Principal；Policy 可要求签名（`asset.unsigned`）、可限制允许的 namespace / registry。
- **供应链事件**：`asset/quarantined`（Policy 或运营者隔离；Lock 中含它的 Agent Definition `not_runnable.environment`，Service 版本 `suspended`）；
  `asset/revoked`（撤销，级联同上且不可恢复该版本）；`mcp_server.tools_snapshot` 与实际 `tools/list` 漂移 → 执行点回报 → `fysund` 标记待重新分类（继承 06 §2.3）。
- **打包与分发**：`fysun asset pack <dir>` 生成 OCI artifact（manifest 作为 config，内容为 layer），`fysun asset push` 到任意 OCI 仓库，`fysun asset register <ref>` 登记到 `fysund`；
  `fysund` 只存登记与 digest，不存内容（[07 §3](07-ecosystem-and-interop.md)）。

## 5. 依赖、兼容与解析输入

- `declares.requires.assets[]` 用版本范围表达；解析（03 §4）落为精确 digest 写入 Lock。
- `compatibility` 是**声明**，不是证据；证据来自 `evidence[]` 中该组合的 Verification / conformance 结果。发现接口同时返回两者。
- `harness` 的 `provides.control_points[]` 与其他资产的 `requires.control_points[]` 在解析时匹配，缺失 → `compose.requirements_unmet{missing[]}`，
  不降级（继承"大声失败"）。
- `loop.harness_ref` 与所选 `harness` 不匹配 → `asset.kind_mismatch`。

## 6. 效果声明的汇总

Agent Definition 的 `bounds.tool_effects[]` = 组合内全部资产 `declares.effects[]` ∪ 所选 Harness `native_effect_tools[]` 的并集，按 `max_severity` 合并同名工具；
Definition 只能收窄（剔除工具）不能降级类别；Policy 覆盖为来源 (c)。未出现在任何声明中的工具在执行点按 `external_mutation` 处理并回报"未分类"（继承 ADR-0008 fail-closed）。

## 7. 资产生命周期

```text
draft ──publish（Policy 评估；可要求证据门）──▶ published ──deprecate──▶ deprecated ──withdraw──▶ withdrawn
  │                                               │
  └──────────────── quarantine / revoke ──────────┴──▶ quarantined / revoked（级联 Lock 与 Service）
```

- `asset/published` 是准入类事件（携带 `policy_decision`）。
- `withdrawn` 的版本不能进入新 Lock，已存在的 Lock 与运行中的 Execution 不受影响（区别于 `revoked`）。
- 每个版本不可变；修正即新版本。

## 8. 发现

`asset.search{kind?, labels?, capability_text?, provides.tools?, compatibility?}` 返回 manifest 摘要 + `evidence` 摘要 + `health`（未分类工具、过期证据、被隔离依赖）。
发现不隐含授权：可见 ≠ 可用，可用由 Tenant Scope 与 Grant 决定。

## 9. 负例（R1）

1. Agent Definition 引用了无 `digest` 或 `digest` 重算不一致的资产 → 失败。
2. `skill` 资产 `allowed-tools` 中的工具未出现在 `declares.effects[]` 且执行点未按 `external_mutation` 处理 → 失败。
3. `harness` manifest 缺 `native_effect_tools` 字段（空数组合法）却被 `runtime.qualify` 通过 → 失败。
4. `loop` 资产参数不符合目标 Harness `loop_schema` 却进入 Lock → 失败。
5. manifest 或 `tools_snapshot` 中出现可放明文的凭据值槽 → 失败。
6. `asset/revoked` 后该 digest 仍能进入新 Lock，或含它的 ServiceVersion 仍为 `stable` → 失败。
7. `fysund` 存储了资产内容本体而非登记与 digest → 失败（边界越界）。
