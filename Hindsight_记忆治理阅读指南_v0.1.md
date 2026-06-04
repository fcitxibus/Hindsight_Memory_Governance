---
title: "Hindsight 记忆治理阅读指南"
subtitle: "v0.1｜基于 hindsight_final_production_kb_v2.zip 官方文档整理"
author: "根据 Hindsight 官方文档汇编"
date: "2026-06-04"
lang: zh-CN
---

# Hindsight 记忆治理阅读指南 v0.1

## 版本声明

本文是一份面向生产知识库、Agent 长期记忆与 Hindsight 集成实施的阅读指南。本文只依据 `hindsight_final_production_kb_v2.zip` 中保留的 Hindsight 官方文档编写，不引入社区经验、个人推测或未在文档中出现的配置结论。

本文中的“记忆治理”不是 Hindsight 官方单独命名的产品模块，而是基于官方文档中已经存在的能力，对记忆系统的隔离、写入、检索、推理、审计、运维和维护方式进行体系化整理。换言之，本文的治理框架来自官方能力的组合，而不是新增概念。

主要依据文件包括：

- `kb/00_core/best-practices.md`
- `kb/00_core/faq.md`
- `kb/01_deployment_config/configuration.md`
- `kb/01_deployment_config/storage.md`
- `kb/01_deployment_config/admin-cli.md`
- `kb/02_memory_architecture/retain.md`
- `kb/02_memory_architecture/retrieval.md`
- `kb/02_memory_architecture/reflect.md`
- `kb/02_memory_architecture/observations.md`
- `kb/02_memory_architecture/rag-vs-hindsight.md`
- `kb/03_api/main-methods.md`
- `kb/03_api/memory-banks.md`
- `kb/03_api/retain.md`
- `kb/03_api/recall.md`
- `kb/03_api/reflect.md`
- `kb/03_api/documents.md`
- `kb/03_api/mental-models.md`
- `kb/03_api/operations.md`
- `kb/03_api/bank-templates.md`
- `kb/06_operations/mcp-server.md`
- `kb/06_operations/monitoring.md`
- `kb/06_operations/performance.md`

---

# 1. 白皮书摘要

Hindsight 官方文档把系统定位为 Agent memory system，即为 AI Agent 提供长期记忆的系统。与传统 RAG 以文档块检索为中心不同，Hindsight 的官方文档强调：它存储结构化事实，构建实体与概念之间的关系，支持时间感知检索，并通过 `reflect()` 提供带有 disposition 的记忆推理能力。

在生产环境中，Hindsight 的记忆治理可以被归纳为七个控制面：

| 控制面 | 官方能力 | 治理目标 |
|---|---|---|
| 隔离控制 | Memory Bank、Tags、Tenant Extension | 防止用户、Agent、场景之间的记忆泄漏 |
| 写入控制 | Retain、Document ID、Context、Timestamp、Metadata、Observation Scopes | 让进入记忆系统的内容可追踪、可更新、可去重 |
| 抽取控制 | Retain Mission、Extraction Mode、Entity Labels | 控制系统记什么、不记什么、如何分类 |
| 巩固控制 | Observations、Observations Mission、Consolidation | 将多条事实转化为有证据支撑的稳定知识 |
| 召回控制 | Recall、Tags、Types、Budget、Max Tokens、Query Timestamp | 准确召回相关记忆并控制成本和延迟 |
| 推理控制 | Reflect、Disposition、Directives、Mental Models、Response Schema | 让记忆推理可约束、可复用、可审计 |
| 运维控制 | Operations、Monitoring、Performance、Storage、MCP Tool Allowlist | 保障异步任务、性能、可观测性和集成边界 |

本文的核心结论是：生产级 Hindsight 不是“把所有内容写进记忆”，而是围绕 Memory Bank、Retain、Recall、Reflect、Observations、Mental Models 和 Directives 建立可隔离、可追踪、可过滤、可审计的记忆系统。

依据：`kb/00_core/faq.md`、`kb/00_core/best-practices.md`、`kb/02_memory_architecture/retain.md`、`kb/02_memory_architecture/retrieval.md`、`kb/02_memory_architecture/reflect.md`。

---

# 2. 核心术语定义

## 2.1 Memory Bank

Memory Bank 是 Hindsight 的隔离单元。官方文档将其描述为独立的记忆存储，所有 `retain`、`recall`、`reflect` 操作都指向单个 bank。Bank 之间不共享数据。

治理含义：

- 一个用户一个 bank，是多数多用户应用的常见模式。
- 一个 Agent 一个 bank，适合 Agent 专属长期记忆。
- 单个共享 bank 加 tags，可以服务于需要跨用户聚合分析的场景，但必须通过严格标签过滤控制可见性。
- Bank 应在写入数据前配置，因为 bank 配置会影响抽取、观察、推理和工具边界。

依据：`kb/00_core/best-practices.md`、`kb/00_core/faq.md`、`kb/03_api/memory-banks.md`。

## 2.2 Retain

`retain()` 是写入操作。官方文档说明，调用 retain 时，Hindsight 会把对话和文档转化为结构化、可搜索的记忆，保留意义与上下文。Retain 不是简单存文本，而是抽取事实、实体、关系、时间信息和因果关系。

Retain 完成后，系统得到：

- 结构化事实；
- 统一后的实体；
- 由实体、时间、语义和因果连接组成的知识图；
- 历史查询和近期排序都可用的时间基础。

治理含义：Retain 是记忆治理的入口。写入质量决定后续召回和推理质量。

依据：`kb/02_memory_architecture/retain.md`、`kb/03_api/retain.md`、`kb/03_api/main-methods.md`。

## 2.3 Recall

`recall()` 是检索操作。官方文档说明，Recall 会并行运行四类检索策略：semantic similarity、keyword/BM25、graph traversal、temporal retrieval，然后通过融合和 reranking 返回单一排序结果列表。Recall 返回结构化事实，而不是直接生成答案。

治理含义：Recall 适合把记忆作为上下文交给外部 Agent 或 RAG 管线，由外部系统继续推理。它强调控制、速度和原始证据。

依据：`kb/02_memory_architecture/retrieval.md`、`kb/03_api/recall.md`、`kb/00_core/best-practices.md`。

## 2.4 Reflect

`reflect()` 是推理操作。官方文档说明，Reflect 会运行 agentic loop，自动搜索 memory bank，使用 mental models、observations、raw facts 形成分层证据，并按 bank 的 disposition traits 进行推理，最终返回合成答案。

Reflect 与 Recall 的边界：

- Recall 返回事实；
- Reflect 返回答案；
- Recall 适合外部系统自己推理；
- Reflect 适合让 Hindsight 基于记忆完成多步搜索、推理和回答。

依据：`kb/02_memory_architecture/reflect.md`、`kb/03_api/reflect.md`、`kb/00_core/faq.md`。

## 2.5 Observations

Observations 是由多条事实巩固出来的知识。官方文档说明，它们不是 LLM 临时编造的摘要，而是有具体 source memories 支撑、包含 proof count、并会随新证据支持、矛盾或扩展而更新的 deduplicated beliefs、preferences 和 learnings。

Observations 提供：

- 去重：多条重复事实转化为一个持久观察；
- 证据：每条 observation 关联具体支持记忆和引用；
- 演化：随新证据被增强、削弱、矛盾或标记为 stale；
- 新鲜度信号：stable、strengthening、weakening、new、stale 等趋势；
- 效率：以更紧凑知识支持检索。

依据：`kb/02_memory_architecture/observations.md`、`kb/03_api/recall.md`、`kb/00_core/best-practices.md`。

## 2.6 Mental Models

Mental Models 是保存下来的 `reflect` 响应。创建 mental model 时，Hindsight 会根据 `source_query` 执行 reflect，并存储结果。后续 reflect 会优先检查这些预计算内容，从而获得更快、更一致的答案。

官方文档给出的优先级为：

1. Mental Models：用户策划过的摘要；
2. Observations：已巩固的知识；
3. Raw Facts：原始事实，用于验证。

治理含义：Mental Models 适合反复读取、变化较慢、需要稳定输出的知识，例如用户画像、当前项目、技术栈、沟通风格、常见问题答案。

依据：`kb/03_api/mental-models.md`、`kb/02_memory_architecture/reflect.md`、`kb/00_core/best-practices.md`。

## 2.7 Directives

Directives 是 reflect 中必须遵守的硬规则。官方文档将 directives 与 disposition 区分开：disposition 是软性影响，directives 是显式约束，会注入 prompt，并要求在所有 reflect 响应中遵守。

适用场景包括：

- 合规规则；
- 隐私限制；
- 风格要求；
- 领域护栏；
- 必须引用来源等要求。

治理含义：Disposition 用于人格和风格，Directives 用于不可违反的边界。

依据：`kb/02_memory_architecture/reflect.md`、`kb/03_api/memory-banks.md`、`kb/03_api/bank-templates.md`。

## 2.8 Documents 与 Chunks

Documents 是 retained content 的容器，用来追踪来源、更新内容、批量删除相关记忆、按来源组织事实。Hindsight 在 retain 内容时会先切分成 chunks，再抽取事实。Chunks 与抽取出的 memories 一起存储，保留原始文本片段；当召回需要精确措辞或更丰富上下文时，可以通过 include chunks 返回。

治理含义：Document ID 是更新、替换、溯源和批量删除的关键字段；Chunks 是精确引用和上下文回溯的证据层。

依据：`kb/03_api/documents.md`、`kb/03_api/retain.md`、`kb/02_memory_architecture/retrieval.md`。

---

# 3. Hindsight 记忆治理的总体模型

## 3.1 治理对象

Hindsight 中需要治理的不是单一“记忆条目”，而是一组相关对象：

| 对象 | 作用 | 治理重点 |
|---|---|---|
| Bank | 隔离记忆、配置行为 | 用户/Agent/场景边界 |
| Document | 组织来源内容 | 更新、替换、删除、溯源 |
| Chunk | 保存原始文本片段 | 精确引用、上下文回查 |
| Memory Fact | 结构化事实 | 类型、标签、实体、时间、来源 |
| Entity | 人、组织、地点、概念等 | 解析、消歧、关系连接 |
| Observation | 巩固知识 | 去重、证据、演化、新鲜度 |
| Mental Model | 预计算 reflect 结果 | 稳定、高频、人工审阅知识 |
| Directive | Reflect 硬规则 | 合规、隐私、风格和护栏 |
| Operation | 异步任务 | 状态、失败、重试、取消、恢复 |

依据：`kb/03_api/memory-banks.md`、`kb/03_api/documents.md`、`kb/03_api/operations.md`、`kb/03_api/mental-models.md`。

## 3.2 治理生命周期

Hindsight 记忆治理生命周期可以按官方操作链整理为：

```text
配置 Bank
  ↓
Retain 写入内容
  ↓
抽取 facts / entities / relationships / time / causality
  ↓
Documents 与 chunks 保留来源线索
  ↓
Observations 自动巩固 durable knowledge
  ↓
Recall 按 query/tags/types/time/budget 检索
  ↓
Reflect 按 mental models → observations → raw facts 推理
  ↓
Operations / Monitoring / Admin CLI 进行运维治理
```

这个生命周期中，治理重点不在单次回答，而在“哪些内容进入系统、进入哪个 bank、带什么标签、可被谁召回、如何被巩固、如何被审计、如何被更新”。

依据：`kb/00_core/best-practices.md`、`kb/02_memory_architecture/retain.md`、`kb/02_memory_architecture/observations.md`、`kb/02_memory_architecture/retrieval.md`、`kb/02_memory_architecture/reflect.md`。

---

# 4. 官方阅读路径

## 4.1 第一层：先读核心概念

必须先读：

1. `kb/00_core/faq.md`
2. `kb/00_core/best-practices.md`
3. `kb/03_api/main-methods.md`

阅读目标：明确 Hindsight 与 RAG 的区别，掌握 Retain、Recall、Reflect 的边界，理解 Bank、Tags、Observations、Mental Models、Directives 的治理角色。

## 4.2 第二层：读记忆架构

继续读：

1. `kb/02_memory_architecture/retain.md`
2. `kb/02_memory_architecture/retrieval.md`
3. `kb/02_memory_architecture/reflect.md`
4. `kb/02_memory_architecture/observations.md`
5. `kb/02_memory_architecture/rag-vs-hindsight.md`

阅读目标：理解 Hindsight 为什么不是简单向量库；Retain 如何抽取事实，Recall 如何多策略检索，Reflect 如何使用 agentic loop 推理，Observations 如何把事实巩固成 durable knowledge。

## 4.3 第三层：读 API 与治理参数

继续读：

1. `kb/03_api/memory-banks.md`
2. `kb/03_api/retain.md`
3. `kb/03_api/recall.md`
4. `kb/03_api/reflect.md`
5. `kb/03_api/documents.md`
6. `kb/03_api/mental-models.md`
7. `kb/03_api/operations.md`
8. `kb/03_api/bank-templates.md`

阅读目标：把概念落到具体参数：`retain_mission`、`observations_mission`、`reflect_mission`、`entity_labels`、`document_id`、`timestamp`、`tags`、`tags_match`、`types`、`budget`、`max_tokens`、`response_schema`、`include.facts`、`include.tool_calls` 等。

## 4.4 第四层：读生产运维

继续读：

1. `kb/01_deployment_config/configuration.md`
2. `kb/01_deployment_config/storage.md`
3. `kb/01_deployment_config/admin-cli.md`
4. `kb/06_operations/performance.md`
5. `kb/06_operations/monitoring.md`
6. `kb/06_operations/mcp-server.md`

阅读目标：掌握存储、认证、MCP 暴露面、性能预算、监控指标、异步任务、zombie operation 恢复等生产运行边界。

---

# 5. Bank 隔离治理

## 5.1 Bank 是第一隔离边界

官方文档明确指出，Memory Bank 是隔离的记忆存储。Bank 之间不共享数据。所有 retain、recall、reflect 操作都指定单个 bank。

生产治理规则：

| 场景 | 官方文档中的可用模式 | 治理含义 |
|---|---|---|
| 多用户应用 | 每用户一个 bank | 简单、强隔离、适合个性化 |
| Agent 专属记忆 | 每 Agent 一个 bank | 防止不同 Agent 记忆互相污染 |
| 需要跨用户聚合分析 | 单 bank + tags | 必须严格控制 tags 和 tags_match |
| 团队共享知识 | 共享 bank + team/topic/scope tags | 需要显式区分共享与私有记忆 |

依据：`kb/00_core/best-practices.md`、`kb/00_core/faq.md`。

## 5.2 Tags 是共享 bank 的可见性控制

官方最佳实践中，tags 被定义为 visibility scope。带有 `user:alice` 的 memory，只会在 recall/reflect 调用包含对应 tags 且匹配模式允许时返回。

官方建议的命名模式包括：

| Tag 模式 | 用途 |
|---|---|
| `user:<id>` | 用户隔离 |
| `session:<id>` | 会话范围 |
| `team:<name>` | 团队共享知识 |
| `topic:<name>` | 领域过滤 |
| `scope:<name>` | 可见性层级 |

多租户最低要求：用户数据 retain 时必须至少包含 `user:<id>`。官方文档明确指出，省略该 tag 会使 memory 变为全局可见。

依据：`kb/00_core/best-practices.md`、`kb/03_api/recall.md`、`kb/03_api/reflect.md`。

## 5.3 tags_match 是泄漏控制开关

官方文档给出四种 tag 匹配模式：

| 模式 | 是否包含未标记记忆 | 条件 |
|---|---|---|
| `any` | 是 | 至少一个 tag 匹配，或未打 tag |
| `all` | 是 | 所有指定 tag 都存在，或未打 tag |
| `any_strict` | 否 | 至少一个 tag 匹配 |
| `all_strict` | 否 | 所有指定 tag 都存在 |

生产治理规则：

- 如果 bank 中既有全局知识又有用户私有知识，可以用 `any` 返回用户记忆和未标记的全局记忆。
- 如果需要完全分区，不允许用户间泄漏，使用 `any_strict` 或 `all_strict`。
- 对用户 + 主题的组合过滤，使用 `all_strict`。
- 对复杂条件，使用 `tag_groups` 的 `and`、`or`、`not` 树形结构。

依据：`kb/00_core/best-practices.md`、`kb/03_api/recall.md`。

## 5.4 Entity Labels 可以把分类变成可过滤标签

官方文档说明，`entity_labels` 可定义受控词表，在 retain 时由 LLM 给事实打上 `key:value` 分类标签。如果 `tag: true`，这些标签也会写入 memory unit 的 tags，从而可在 recall/reflect 中使用标准 tags 过滤。

治理含义：当一个 bank 中存在语义相近但用途不同的记忆，例如 rule 与 procedure，仅靠排序不能可靠区分。把 `entity_labels` 写入 tags 后，过滤会在数据库层执行，非目标记忆不会进入 ranking pipeline。

依据：`kb/00_core/best-practices.md`、`kb/02_memory_architecture/retain.md`、`kb/03_api/memory-banks.md`。

---

# 6. Retain 写入治理

## 6.1 Retain 不是“越多越好”

官方最佳实践中明确列出多个反模式：预先摘要再 retain、随机 document_id、缺少 context、用 metadata 做过滤、泛化 mission、同一请求中 retain 后立即 recall、缺少 timestamp 等。

生产治理规则：写入记忆前必须明确：内容来源、内容类型、所属用户/会话/主题、时间、更新方式、是否需要巩固 observations。

依据：`kb/00_core/best-practices.md`、`kb/03_api/retain.md`。

## 6.2 content 应保留原始上下文，不应先摘要

官方反模式指出，pre-summarizing before retain 会丢失实体关系、时间标记和结构上下文。官方建议是 retain raw content，让 Hindsight 负责抽取事实。

治理含义：不要为了“节省记忆”先把对话压缩成摘要再写入。应把有上下文的原始对话、文档段落、ticket 内容或会议内容交给 retain，并通过 context、tags、document_id 控制质量。

依据：`kb/00_core/best-practices.md`、`kb/02_memory_architecture/retain.md`。

## 6.3 context 字段必须描述内容性质和来源

官方最佳实践指出，`context` 对抽取质量有高影响，应始终设置。它描述内容的 nature and source。

治理含义：

- 对话应标明是用户支持对话、产品反馈、项目会议还是私人助手对话。
- 文档应标明是技术设计文档、FAQ、ticket、合同摘录还是研究笔记。
- `context` 不用于过滤；过滤使用 tags。

依据：`kb/00_core/best-practices.md`、`kb/03_api/retain.md`。

## 6.4 document_id 是去重、更新和删除的核心

官方文档说明，同一个 `document_id` 表示 upsert：重新 retain 同一 document_id 会删除旧版本并重新处理。`update_mode="replace"` 是默认行为；`update_mode="append"` 会把新内容追加到已有文档文本中并重新处理组合内容，delta retain 会跳过未变化 chunks。

生产治理规则：

- 会话、ticket、文档、文件必须使用稳定 document_id。
- 不得每次 retain 使用随机 UUID；官方文档指出这会创建重复文档。
- 对持续增长的 conversation，应始终用同一 ID retain 完整对话。
- 需要删除来源内容时，用 documents 能批量删除该来源产生的 memories。

依据：`kb/00_core/best-practices.md`、`kb/03_api/retain.md`、`kb/03_api/documents.md`。

## 6.5 timestamp 影响时间检索

官方文档说明，`timestamp` 应在有时间上下文时设置。它启用 temporal retrieval strategies。对于 conversations，可设置为会话开始时间。省略 timestamp 会禁用 temporal ranking。

Retain API 中还说明：

- 省略或 `null`：使用 ingestion 当前时间；
- ISO 8601 字符串：使用给定时间；
- `"unset"`：无 timestamp，适合 timeless material，如参考文档、书籍或虚构内容。

治理含义：时间敏感内容必须带真实事件时间；无真实事件时间的材料用 `"unset"`，避免错误时间污染 temporal retrieval。

依据：`kb/00_core/best-practices.md`、`kb/03_api/retain.md`、`kb/02_memory_architecture/retain.md`。

## 6.6 metadata 用于溯源，不用于过滤

官方最佳实践明确写明：metadata is not filterable，过滤应使用 tags。Metadata 会随 recalled memory 返回，适合链接到源系统、UI deep link 和 audit trails。

治理含义：

- `source`、`channel`、`thread_id`、`ticket_id`、`url`、`version` 等适合放 metadata。
- `user:<id>`、`topic:<name>`、`scope:<name>` 等检索可见性条件必须放 tags。

依据：`kb/00_core/best-practices.md`、`kb/03_api/recall.md`。

## 6.7 observation_scopes 控制巩固粒度

官方文档给出四种 observation scopes：

| 值 | 行为 | 使用场景 |
|---|---|---|
| `combined` | 所有 tags 一起做一次 observation pass | 默认，单用户 bank、一般用途 |
| `per_tag` | 每个 tag 独立做 observation pass | 用户行为观察需要隔离 |
| `all_combinations` | 所有 tag 子集组合 | 复杂多维分析，成本高 |
| 自定义列表 | 显式指定 scope list | 精确多租户控制 |

生产治理规则：多租户场景不应默认把所有标签混合巩固。需要用户级、团队级、组合级 observations 时，应显式定义 scopes。

依据：`kb/00_core/best-practices.md`、`kb/03_api/retain.md`、`kb/02_memory_architecture/observations.md`。

## 6.8 Sync 与 Async 边界

官方最佳实践说明：

- `async_=False` 默认用于需要继续前确认的场景；
- `async_=True` 用于 end-of-turn、end-of-session retain 以及用户面对的低延迟流程；
- 不应在同一轮中 retain 后立即 recall，因为 retain 是写操作，抽取出的 memories 不会立即可用。

治理含义：Agent 回合开始时 recall，回合结束时 retain。不要把 retain 当成即时上下文写入。

依据：`kb/00_core/best-practices.md`、`kb/03_api/retain.md`、`kb/06_operations/performance.md`。

---

# 7. 抽取治理：Missions 与 Entity Labels

## 7.1 三类 Mission 的边界

官方最佳实践强调，所有 mission 都接受自然语言，但必须具体。模糊 mission 会产生模糊结果。

| Mission | 影响范围 | 作用 |
|---|---|---|
| `retain_mission` | Retain | 告诉 LLM 抽取什么、忽略什么 |
| `observations_mission` | Observations consolidation | 控制巩固成什么类型的 durable patterns |
| `reflect_mission` | Reflect | 设置 bank 的身份、推理框架和响应取向 |

治理规则：不要用一个泛化描述覆盖所有阶段。写入、巩固、推理是不同控制面，必须分别配置。

依据：`kb/00_core/best-practices.md`、`kb/02_memory_architecture/retain.md`、`kb/02_memory_architecture/observations.md`、`kb/02_memory_architecture/reflect.md`。

## 7.2 retain_mission 要同时定义“记什么”和“不记什么”

官方文档指出，`retain_mission` 注入 fact extraction prompt，用来告诉 LLM 抽取什么、忽略什么。有效 mission 应列出需要的 fact types，例如 preferences、decisions、errors、commitments，并列出要忽略的内容，例如 greetings、small talk、scheduling logistics。

治理含义：高质量 retain_mission 不是“extract all information”，而是领域化抽取规则。

依据：`kb/00_core/best-practices.md`、`kb/02_memory_architecture/retain.md`、`kb/03_api/memory-banks.md`。

## 7.3 observations_mission 要强调 durable patterns

官方文档说明，`observations_mission` 可替换默认 durable-knowledge 规则，用来控制 observations 的形状。官方建议强调 durable patterns，避免 ephemeral observation noise；如果需要历史跟踪，要明确 contradiction detection。

治理含义：observations 不应成为临时状态的堆积层，而应承载稳定偏好、技能、关系、反复出现的模式和矛盾变化。

依据：`kb/00_core/best-practices.md`、`kb/02_memory_architecture/observations.md`。

## 7.4 reflect_mission 与 disposition 共同控制推理风格

官方文档说明，`reflect_mission` 提供身份上下文，说明 agent 是谁、关心什么，并让推理在对话间保持一致。Disposition traits 包括 skepticism、literalism、empathy，范围为 1 到 5。

治理含义：

- `reflect_mission` 不是抽取规则；它只影响 reflect。
- `disposition_skepticism` 控制信任/质疑倾向。
- `disposition_literalism` 控制灵活解释/字面解释倾向。
- `disposition_empathy` 控制事实导向/情绪语境导向倾向。

依据：`kb/02_memory_architecture/reflect.md`、`kb/03_api/memory-banks.md`。

## 7.5 entity_labels 是结构化分类能力

官方 Memory Banks API 将 `entity_labels` 定义为 label group 配置，字段包括 `key`、`description`、`type`、`values`、`fields`、`optional`、`tag`。其中 `tag: true` 时，抽取出的 label 会写成 tags，支持 recall/reflect 过滤。

治理含义：Entity Labels 适合把业务分类、知识形态、优先级、状态、角色等变成系统可识别的结构化记忆标签。

依据：`kb/03_api/memory-banks.md`、`kb/00_core/best-practices.md`。

---

# 8. Observations 巩固治理

## 8.1 Observations 是证据驱动的巩固层

Observations 官方定义为 consolidated knowledge，由多条 facts 构建。它们不同于单条 raw facts，也不是实时临时摘要。每条 observation 都有支持记忆、proof count 和 freshness trend。

治理含义：在生产使用中，Observations 是减少重复事实、提升长期偏好识别和模式归纳的重要层，但它必须由事实证据支撑。

依据：`kb/02_memory_architecture/observations.md`、`kb/03_api/recall.md`。

## 8.2 Observations 自动在 retain 后维护

官方文档说明，Observations 会在 retain 操作完成后由后台 consolidation 维护，而不是 retain 调用本身的一部分。

治理含义：系统设计上，write path 与 consolidation path 是分开的；生产监控必须关注 retain operations 与 consolidation operations 的状态。

依据：`kb/02_memory_architecture/retain.md`、`kb/02_memory_architecture/observations.md`、`kb/03_api/operations.md`。

## 8.3 Observations 可被 recall types 过滤

Recall 的 `types` 参数支持 `world`、`experience`、`observation`。官方文档说明，只查 `observation` 可返回 consolidated patterns；查 `world`、`experience` 可返回 raw facts。

治理含义：

- 高层用户画像、长期偏好、行为模式：优先查 `observation`。
- 需要原始证据和引用敏感场景：查 `world`、`experience`。
- 不设 `types`：三类都查。

依据：`kb/00_core/best-practices.md`、`kb/03_api/recall.md`。

---

# 9. Recall 召回治理

## 9.1 Recall 使用四类并行检索策略

官方 Retrieval 文档将 Recall 的检索策略称为 TEMPR，包含：

| 策略 | 解决的问题 |
|---|---|
| Semantic Search | 语义相近、同义表达、自然语言问题 |
| Keyword Search / BM25 | 专有名词、技术词、唯一标识符、精确短语 |
| Graph Traversal | 实体之间的间接关系和多跳连接 |
| Temporal Search | 历史时间、时间范围、相对时间、前后关系 |

检索后，结果会融合；出现在多个策略中的 memories 排名更高，rank 比 raw score 更重要，最后使用 neural model rerank。

依据：`kb/02_memory_architecture/retrieval.md`、`kb/03_api/recall.md`。

## 9.2 Recall query 是唯一必填字段

Recall API 中，`query` 是唯一必填字段。它同时驱动四种检索策略：用于 semantic embedding、BM25 tokenization、graph traversal seed、temporal parsing；检索后还会被传给 cross-encoder reranker。官方配置还说明超过 500 tokens 的 recall query 会被拒绝。

治理含义：query 不应塞入过长上下文；长上下文应通过 tags、types、metadata 溯源、document_id 和外部提示组织，而不是把全部内容塞入 recall query。

依据：`kb/03_api/recall.md`、`kb/01_deployment_config/configuration.md`。

## 9.3 Budget 控制检索深度，不等于 max_tokens

官方文档明确区分：

- `budget` 控制搜索深度、候选池、图遍历和 reranking 工作量；
- `max_tokens` 控制返回的 memory text 总 token 量。

Recall 的 budget 可为 `low`、`mid`、`high`。Best Practices 中给出的延迟范围为：

| Budget | 延迟 | 使用场景 |
|---|---|---|
| `low` | 50–100ms | 简单事实查询、单跳问题 |
| `mid` | 100–300ms | 多跳推理、关系查询，默认 |
| `high` | 300–500ms | 深度探索、复杂跨领域模式 |

生产治理规则：默认用 `mid`；高频 agent loop 用 `low`；只有明确 deep recall 时用 `high`。

依据：`kb/00_core/best-practices.md`、`kb/02_memory_architecture/retrieval.md`、`kb/03_api/recall.md`。

## 9.4 max_tokens 按 Agent 上下文预算治理

官方文档说明，Hindsight 为 Agent 设计，不以 top-k 为主要模型，而是以 token budget 返回结果。`max_tokens` 默认 4096；只统计每条 fact 的 `text` 字段，metadata、tags、entities 等不计入该预算。reranking 后，facts 按相关性顺序填入，直到预算耗尽。

治理含义：

- 高质量检索不是固定 top-k，而是给 Agent 分配多少 memory context。
- 精简问答可用较小 token 预算。
- 综合总结可扩大 token 预算。

依据：`kb/02_memory_architecture/retrieval.md`、`kb/03_api/recall.md`。

## 9.5 include 只在需要证据层时打开

官方最佳实践给出的 include 选项：

| 选项 | 默认 | 何时启用 |
|---|---|---|
| `include.entities` | Enabled | 保持开启，提供图遍历实体上下文 |
| `include.chunks` | Disabled | 需要原文措辞或来源引用时 |
| `include.source_facts` | Disabled | 审计 observation provenance 时 |

治理含义：默认返回结构化事实；只有需要原文、精确引用或审计来源时才扩大返回内容。

依据：`kb/00_core/best-practices.md`、`kb/02_memory_architecture/retrieval.md`、`kb/03_api/recall.md`。

## 9.6 query_timestamp 用于时间敏感查询

官方文档说明，`query_timestamp` 用来锚定相对时间表达与 recency scoring。例如“今年一月团队在做什么”需要把查询时间固定到具体时间点。

治理含义：凡是含“最近、上周、今年一月、之前、之后”等相对时间的 recall，应显式设置 `query_timestamp`，避免时间解释不稳定。

依据：`kb/00_core/best-practices.md`、`kb/03_api/recall.md`。

---

# 10. Reflect 推理治理

## 10.1 Reflect 是带证据搜索的 agentic loop

官方文档说明，Reflect agent 具有以下工具：

| 工具 | 作用 | 优先级 |
|---|---|---|
| `search_mental_models` | 用户策划过的摘要 | 最高 |
| `search_observations` | 巩固知识 | 高 |
| `recall` | 原始事实 | 后备 |
| `expand` | 获取更多记忆上下文 | 按需 |
| `done` | 完成最终答案 | 准备好时 |

Reflect agent 必须先收集证据才回答，最多运行 10 次迭代，并且只允许引用实际检索到的 ID。

依据：`kb/02_memory_architecture/reflect.md`。

## 10.2 Reflect 适合合成答案，不适合替代所有 recall

官方 FAQ 与最佳实践均说明：当需要 raw facts、最大控制、简单事实查询、低延迟或自行构建答案合成层时使用 recall；当需要 ready-to-use answer、disposition-aware response、多步推理、structured output 或 citations 时使用 reflect。

官方 FAQ 还给出延迟对比：Recall 约 50–500ms；Reflect 约 1–10s。

治理含义：生产系统中不能把所有记忆查询都变成 reflect。高频、低延迟、可控上下文注入优先 recall；复杂判断、建议、画像、综合回答用 reflect。

依据：`kb/00_core/faq.md`、`kb/00_core/best-practices.md`、`kb/03_api/reflect.md`。

## 10.3 Disposition 是软约束

官方文档将 disposition traits 定义为影响 reflect 解释和推理的三个特征：

| Trait | 范围 | 低值含义 | 高值含义 |
|---|---|---|---|
| Skepticism | 1–5 | 信任、接受表面信息 | 怀疑、质疑主张 |
| Literalism | 1–5 | 灵活解释、读出潜台词 | 字面解释、按事实理解 |
| Empathy | 1–5 | 抽离、聚焦事实逻辑 | 考虑情绪语境 |

治理含义：Disposition 用于形成一致角色，而不是强制合规规则。强制规则必须使用 Directives。

依据：`kb/02_memory_architecture/reflect.md`、`kb/03_api/memory-banks.md`。

## 10.4 Directives 是硬约束

官方文档明确区分 Directives 与 Disposition：Disposition 是 soft influence，Directives 是 hard rules。Directives 只影响 reflect，会被注入 prompt，并要求 agent 在响应中遵守。

生产治理规则：

- 合规、隐私、引用、风格等不可违反要求放 Directives。
- 个性、谨慎程度、表达取向放 Disposition。
- Directives 可通过 API 创建、列出、更新和删除，也可通过 bank templates 复制。

依据：`kb/02_memory_architecture/reflect.md`、`kb/03_api/memory-banks.md`、`kb/03_api/bank-templates.md`。

## 10.5 response_schema 用于程序化输出

Reflect API 说明，`response_schema` 是可选 JSON Schema。提供后，LLM 会生成符合 schema 的响应，并通过 `structured_output` 返回解析结果；这种模式适合程序消费，而不是展示自然语言段落。

治理含义：当 reflect 结果要进入工作流、自动审批、路由、评分或 UI 结构化展示时，应使用 response_schema，而不是要求模型自由输出固定格式。

依据：`kb/03_api/reflect.md`、`kb/00_core/best-practices.md`。

## 10.6 include.facts 与 include.tool_calls 是审计开关

官方最佳实践说明：

| 选项 | 用途 |
|---|---|
| `include.facts=True` | 暴露哪些 memories 和 mental models 被使用，支持透明度和审计 |
| `include.tool_calls=True` | 暴露内部 search loop 的完整执行轨迹，适合调试 |

官方建议：生产中启用 `include.facts` 做审计；`include.tool_calls` 只在开发阶段启用。

依据：`kb/00_core/best-practices.md`、`kb/03_api/reflect.md`。

---

# 11. Mental Models 治理

## 11.1 Mental Models 用于高频、稳定、需一致的知识

官方最佳实践建议在以下情况创建 mental model：

- 常见重复查询，需要一致答案；
- 高频 Agent 需要 sub-100ms 响应；
- 每次请求都读取的用户画像或 persona；
- 已由人工审阅或批准的知识摘要；
- 跨会话且变化较慢的状态，如偏好、技能、背景。

治理含义：Mental Model 是生产系统中的“策划过的记忆摘要”，不是所有知识的大杂烩。

依据：`kb/00_core/best-practices.md`、`kb/03_api/mental-models.md`。

## 11.2 Tags 同时影响 Mental Model 的构建和可见性

官方最佳实践说明，Mental Model 上的 tags 同时过滤“用于构建它的 memories”和“哪些 recall/reflect 调用能看到它”。Mental Model API 也说明，tags 默认用 `all_strict` matching，所以只有携带所有指定 tags 的 memories 会被读取。

治理含义：用户级 mental model 必须带用户 tag；团队级 mental model 必须带团队或范围 tag；全局 mental model 不带 tags 时对全局可见。

依据：`kb/00_core/best-practices.md`、`kb/03_api/mental-models.md`。

## 11.3 Refresh 策略必须按知识变化速度选择

官方 Mental Models API 支持 `trigger` 设置，其中 `refresh_after_consolidation` 可在 observations consolidation 后自动刷新。Refresh mode 包括：

- `full`：每次从头生成完整内容，简单、可预测；
- `delta`：输出 typed operations，对已有文档结构做增删改，适合长期存在的技能、playbook 或 onboarding summary。

官方给出的使用边界：实时 dashboards 与用户 preferences 可启用自动刷新；policy summaries 与 FAQ answers 更适合手动刷新，因为它们变化不频繁或需要审阅。

依据：`kb/03_api/mental-models.md`、`kb/00_core/best-practices.md`。

## 11.4 Mental Model 粒度必须窄

官方最佳实践明确指出，应创建 narrow、scoped models，每个 knowledge dimension 一个 model；“Everything about the user” 类型的模型低准确、刷新慢、难以 scoped。

可用粒度示例：

- User Profile；
- Current Projects；
- Technical Stack；
- Communication Style。

依据：`kb/00_core/best-practices.md`、`kb/03_api/mental-models.md`。

---

# 12. Operations 与异步任务治理

## 12.1 Operations 有明确生命周期

官方 Operations API 定义了如下状态：

| 状态 | 含义 |
|---|---|
| `pending` | 任务已排队，未被 worker 领取，或因 backpressure 被延后 |
| `processing` | worker 已领取并正在执行 |
| `completed` | handler 成功返回 |
| `failed` | handler 抛错，`error_message` 包含原因，可 retry |
| `cancelled` | 在 worker 领取前被取消；processing 中的操作不支持取消 |

治理含义：生产系统必须把 retain、file convert、consolidation、mental model refresh、graph maintenance、webhook delivery 当成可观测的后台任务，而不是同步函数调用。

依据：`kb/03_api/operations.md`。

## 12.2 Operation types 对应不同维护任务

官方文档列出的 operation types 包括：

- `retain`；
- `file_convert_retain`；
- `consolidation`；
- `refresh_mental_model`；
- `graph_maintenance`；
- `webhook_delivery`。

治理含义：不同任务失败的影响不同。Retain 失败影响写入，consolidation 失败影响 observations，refresh_mental_model 失败影响预计算答案，graph_maintenance 失败影响图结构维护，webhook_delivery 失败影响外部系统通知。

依据：`kb/03_api/operations.md`。

## 12.3 Zombie operations 需要稳定 worker_id

官方 FAQ 定义 zombie operation：后台任务卡在 `processing`，因为领取任务的 worker 已消失，通常发生在 Docker 容器重启后。根因通常是 `HINDSIGHT_API_WORKER_ID` 不稳定；默认 worker 使用容器 hostname，而 Docker 每次重启可能变化。

恢复方式：

```bash
hindsight-admin decommission-worker <old-worker-id>
hindsight-admin decommission-workers
```

预防方式：为 worker 设置稳定的 `HINDSIGHT_API_WORKER_ID`。官方文档说明 Helm chart 通过 StatefulSet 绑定 pod name 已处理该问题。

依据：`kb/00_core/faq.md`、`kb/01_deployment_config/admin-cli.md`。

## 12.4 Webhooks 是 operation 的外部通知机制

官方 Operations 文档说明，在某些操作完成后，例如 consolidation 完成且 bank 配置了 webhook，Hindsight 会入队 `webhook_delivery` task。handler 会 POST payload 到配置 URL，并在 transient failures 上重试。

配置项包括：

- `HINDSIGHT_API_WEBHOOK_URL`；
- `HINDSIGHT_API_WEBHOOK_SECRET`；
- `HINDSIGHT_API_WEBHOOK_EVENT_TYPES`；
- webhook delivery poll interval、batch size、max attempts 等。

依据：`kb/03_api/operations.md`、`kb/01_deployment_config/configuration.md`、`kb/03_api/webhooks.md`。

---

# 13. MCP 与工具暴露治理

## 13.1 MCP server 默认启用

官方 MCP Server 文档说明，Hindsight 内置 Model Context Protocol server，默认启用，挂载在 API server 的 `/mcp`。每个 memory bank 有自己的 MCP endpoint：

```text
http://localhost:8888/mcp/{bank_id}/
```

可以通过环境变量禁用：

```bash
export HINDSIGHT_API_MCP_ENABLED=false
```

依据：`kb/06_operations/mcp-server.md`、`kb/01_deployment_config/configuration.md`。

## 13.2 MCP 工具必须最小暴露

官方配置文档提供 `HINDSIGHT_API_MCP_ENABLED_TOOLS`，用于限制 server level 注册哪些 MCP tools。Memory Banks API 还提供 bank 级 `mcp_enabled_tools` allowlist；未在列表中的工具调用会返回错误。

治理含义：

- 只读部署可只暴露 `recall`。
- 只允许检索和推理时，可暴露 `recall` 与 `reflect`。
- 允许写入时才暴露 `retain`。
- 高权限工具如 `clear_memories`、`delete_bank`、`update_bank` 不应默认暴露给不可信 Agent。

最后一条是由官方 allowlist 能力推出的治理用法；官方文档没有给出“默认禁止删除工具”的原文要求，因此本文将其作为基于 allowlist 的安全使用准则，而不是 Hindsight 的强制默认行为。

依据：`kb/01_deployment_config/configuration.md`、`kb/03_api/memory-banks.md`、`kb/06_operations/mcp-server.md`。

## 13.3 MCP 认证与传输配置

官方配置中包含：

- `HINDSIGHT_API_MCP_AUTH_TOKEN`：MCP Bearer token；
- `HINDSIGHT_API_MCP_STATELESS`：是否使用 stateless HTTP transport；
- `HINDSIGHT_API_MCP_LOCAL_BANK_ID`：local MCP 使用的 bank ID；
- `HINDSIGHT_API_MCP_INSTRUCTIONS`：追加到 retain/recall tool descriptions 的额外说明。

治理含义：公开或跨工具使用 MCP 时，应明确认证 token、传输模式和可用工具集合。

依据：`kb/01_deployment_config/configuration.md`、`kb/06_operations/mcp-server.md`。

---

# 14. 部署、存储与认证治理

## 14.1 存储后端

官方 Storage 文档说明，Hindsight 以 PostgreSQL 作为主要存储后端，并提供 Oracle AI Database 作为企业部署替代方案。PostgreSQL 提供：

- pgvector 向量搜索；
- full-text search；
- relational data；
- JSONB；
- recursive CTEs 支持图查询。

生产环境使用 PostgreSQL 15+，并要求 pgvector 0.5.0+。本地开发在未配置 `HINDSIGHT_API_DATABASE_URL` 时可使用 pg0，数据存于 `~/.hindsight/pg0/`。

依据：`kb/01_deployment_config/storage.md`。

## 14.2 认证默认关闭，生产应启用 API key 认证

官方 Configuration 文档说明，Hindsight 默认无认证。生产部署可启用内置 API key tenant extension：

```bash
export HINDSIGHT_API_TENANT_EXTENSION=hindsight_api.extensions.builtin.tenant:ApiKeyTenantExtension
export HINDSIGHT_API_TENANT_API_KEY=your-secret-api-key
```

启用后，请求必须包含：

```text
Authorization: Bearer your-secret-api-key
```

没有有效 API key 的请求会返回 `401 Unauthorized`。高级认证如 JWT、OAuth、多租户 schemas，需要自定义 `TenantExtension`。

依据：`kb/01_deployment_config/configuration.md`、`kb/06_operations/extensions.md`。

## 14.3 本文不声明未被官方文档确认的安全能力

在当前依据文档中，本文不对以下内容做事实声明：

- 是否内置 RBAC；
- 是否内置端到端加密；
- 是否通过特定合规认证；
- 是否提供固定数据保留周期；
- 是否默认加密所有存储字段。

本文只确认官方文档明确写出的内容：默认无认证，生产可启用 API key tenant extension；MCP 可配置 auth token 和工具 allowlist；Bank 和 tags 可用于记忆隔离与可见性控制；Documents 与 metadata 支持来源追踪。

依据：`kb/01_deployment_config/configuration.md`、`kb/03_api/memory-banks.md`、`kb/06_operations/mcp-server.md`。

---

# 15. 性能治理

## 15.1 Hindsight 优先优化读性能

官方 Performance 文档说明，Hindsight 的三类性能重点是：

- Retain：大规模 memory storage 的 batch processing 与 async operations；
- Recall：sub-second semantic search 与 configurable thinking budgets；
- Reflect：disposition-aware answer generation 与 controllable compute。

官方文档还说明系统架构优先读性能，因为 memory system 的典型使用模式是写一次、读多次。

依据：`kb/06_operations/performance.md`。

## 15.2 Retain 的性能瓶颈主要在 LLM

官方 Models 与 Performance 文档指出，LLM 是 retain operations 的主要瓶颈。Performance 文档还说明，retain 不需要最聪明的模型；事实抽取通常可由较小、更快、更便宜的模型完成。

治理含义：生产中应把 retain LLM、reflect LLM、consolidation LLM 作为不同控制面看待。配置文档提供 retain、reflect、consolidation 各自的 LLM provider、model、api key 等变量。

依据：`kb/01_deployment_config/models.md`、`kb/01_deployment_config/configuration.md`、`kb/06_operations/performance.md`。

## 15.3 Recall 通过 budget 与 max_tokens 控制成本和延迟

Recall 的 `budget` 影响检索深度；`max_tokens` 影响返回上下文大小。官方文档强调二者互相独立。

治理规则：

- 高频聊天回复：低 budget、小 max_tokens；
- 普通文档问答：mid budget、默认级上下文；
- 研究型查询：high budget、更大 max_tokens；
- 不得把 high budget 设为所有查询默认值，官方最佳实践将其列为反模式。

依据：`kb/02_memory_architecture/retrieval.md`、`kb/00_core/best-practices.md`、`kb/06_operations/performance.md`。

## 15.4 Monitoring 是生产必需控制面

官方 Monitoring 文档说明，Hindsight 提供 Prometheus metrics、OpenTelemetry distributed tracing 和 Grafana dashboards。可用指标包括：

| 指标类别 | 例子 |
|---|---|
| Operation Metrics | `hindsight.operation.duration`、`hindsight.operation.total` |
| LLM Metrics | `hindsight.llm.duration`、`hindsight.llm.calls.total`、input/output tokens |
| HTTP Metrics | `hindsight.http.duration`、`hindsight.http.requests.total` |
| Database Pool Metrics | 数据库连接池相关指标 |
| Process Metrics | 进程资源相关指标 |

治理含义：生产环境应至少监控 retain/recall/reflect 延迟、operation 成功率、LLM 调用延迟、token 消耗、HTTP 5xx、数据库池利用率。

依据：`kb/06_operations/monitoring.md`、`kb/06_operations/performance.md`。

---

# 16. Bank Templates 与配置复用治理

## 16.1 Bank Template 是 declarative JSON manifest

官方 Bank Templates API 定义：bank template 是描述 bank 完整设置的 JSON manifest，包括配置覆盖、mental models、directives 等。导入一个 manifest，可一次性配置 bank。

模板适合：

- 复制一致配置到多个用户或 Agent；
- 新用户 onboarding；
- 分享推荐配置；
- 框架集成随包提供推荐模板。

依据：`kb/03_api/bank-templates.md`、`kb/00_core/templates.md`。

## 16.2 Template Schema 是治理基线

官方 schema 包括：

- `version`，当前为 `"1"`；
- `bank`：reflect_mission、retain_mission、retain_extraction_mode、retain_chunk_size、disposition、enable_observations、observations_mission、entity_labels 等；
- `mental_models`：id、name、source_query、tags、max_tokens、trigger；
- `directives`：name、content、priority、is_active、tags。

治理含义：生产系统中不应手工散落配置。对不同 Agent 类型、业务域、租户类型，应形成 template manifest，作为可审计、可复制、可 dry-run 的配置基线。

依据：`kb/03_api/bank-templates.md`。

## 16.3 Import 行为

官方文档说明：导入 template 时，如果 bank 不存在会自动创建；config 字段作为 per-bank overrides 应用；mental models 按 id 匹配，存在则更新，不存在则创建；directives 按 name 匹配；mental model 内容异步生成，响应包含 operation_ids。

依据：`kb/03_api/bank-templates.md`、`kb/03_api/operations.md`。

---

# 17. 反模式清单

以下反模式均来自官方 Best Practices，并按治理层重新整理：

| 反模式 | 后果 | 正确做法 |
|---|---|---|
| retain 前先摘要 | 丢失实体关系、时间标记、结构上下文 | retain 原始内容，让 Hindsight 抽取 facts |
| 每次 retain 使用随机 document_id | 每次生成新文档，造成重复 | 使用稳定 session/ticket/document ID |
| 省略 context | 显著降低抽取质量 | 始终描述数据类型和来源 |
| 用 metadata 做过滤 | metadata 不可过滤 | 过滤条件必须用 tags |
| mission 过于泛化 | 抽取噪声、低价值记忆 | 明确领域、数据类型、忽略项 |
| 多租户 bank 中用 `tags_match="any"` | 可能跨用户泄漏 | 用户分区用 `any_strict` 或 `all_strict` |
| 同一请求 retain 后立即 recall | 新写入 memory 未完成索引 | 回合开始 recall，回合结束 retain |
| 一个 mental model 覆盖所有内容 | 准确率低、刷新慢、难以 scoped | 每个知识维度一个 model |
| 所有 recall 都用 high budget | 慢且成本高 | 简单查询 low，默认 mid，深度查询 high |
| retain 缺少 timestamp | 禁用 temporal retrieval strategies | 有真实时间时设置 timestamp |

依据：`kb/00_core/best-practices.md`。

---

# 18. 生产实施检查表

## 18.1 Bank 设计检查

- 是否明确每个用户、Agent、团队或场景对应哪个 bank？
- 如果使用共享 bank，是否定义了 `user:<id>`、`team:<name>`、`topic:<name>`、`scope:<name>` 标签规范？
- 是否明确 per-user bank 与 shared bank 的取舍？
- 是否将 bank 配置做成 template，而不是手工配置？

依据：`kb/00_core/best-practices.md`、`kb/03_api/bank-templates.md`。

## 18.2 Retain 检查

- 每条用户数据是否带 `user:<id>` tag？
- 是否设置稳定 `document_id`？
- 是否设置 `context`？
- 是否设置正确 `timestamp` 或显式 `"unset"`？
- 是否区分 metadata 与 tags？
- 是否明确 `observation_scopes`？
- 是否避免同一轮 retain 后立即 recall？

依据：`kb/00_core/best-practices.md`、`kb/03_api/retain.md`。

## 18.3 Recall 检查

- 是否选择正确 `tags_match`？
- 是否需要 `types` 过滤？
- 是否将 `budget` 与 `max_tokens` 分开配置？
- 时间敏感查询是否设置 `query_timestamp`？
- 是否只在需要时打开 `include.chunks` 或 `include.source_facts`？
- 是否将复杂条件用 `tag_groups` 表达？

依据：`kb/03_api/recall.md`、`kb/00_core/best-practices.md`。

## 18.4 Reflect 检查

- 是否只有需要合成答案时使用 reflect？
- 是否配置 `reflect_mission`？
- 是否设置合适的 skepticism、literalism、empathy？
- 合规、隐私、引用、风格要求是否写成 directives？
- 是否为程序化输出设置 `response_schema`？
- 生产审计是否启用 `include.facts`？
- `include.tool_calls` 是否仅在开发调试开启？

依据：`kb/02_memory_architecture/reflect.md`、`kb/03_api/reflect.md`、`kb/00_core/best-practices.md`。

## 18.5 运维检查

- 是否配置 PostgreSQL 15+ 与 pgvector 0.5.0+？
- 生产是否启用 API key authentication 或自定义 TenantExtension？
- MCP 是否启用最小工具 allowlist？
- 是否监控 operation、LLM、HTTP、DB pool、process metrics？
- 是否为 worker 设置稳定 `HINDSIGHT_API_WORKER_ID`？
- 是否准备 zombie operations 的 admin CLI 恢复流程？
- 是否把 mental model refresh、consolidation、webhook delivery 当作 operations 监控？

依据：`kb/01_deployment_config/storage.md`、`kb/01_deployment_config/configuration.md`、`kb/01_deployment_config/admin-cli.md`、`kb/06_operations/monitoring.md`、`kb/03_api/operations.md`。

---

# 19. 最小可行阅读顺序

如果只读 10 份文档，按以下顺序：

1. `kb/00_core/faq.md`
2. `kb/00_core/best-practices.md`
3. `kb/02_memory_architecture/retain.md`
4. `kb/02_memory_architecture/retrieval.md`
5. `kb/02_memory_architecture/reflect.md`
6. `kb/02_memory_architecture/observations.md`
7. `kb/03_api/memory-banks.md`
8. `kb/03_api/retain.md`
9. `kb/03_api/recall.md`
10. `kb/03_api/reflect.md`

如果要进入生产部署，再补读：

11. `kb/03_api/mental-models.md`
12. `kb/03_api/operations.md`
13. `kb/03_api/bank-templates.md`
14. `kb/01_deployment_config/configuration.md`
15. `kb/01_deployment_config/storage.md`
16. `kb/06_operations/mcp-server.md`
17. `kb/06_operations/monitoring.md`
18. `kb/06_operations/performance.md`

---

# 20. 一句话总结

基于官方文档，Hindsight 的生产级记忆治理可以定义为：

> 以 Memory Bank 为隔离边界，以 Retain 控制写入质量，以 Tags、Types、Budget、Timestamp 控制召回，以 Observations 巩固长期知识，以 Mental Models 固化高频知识，以 Directives 和 Disposition 约束推理行为，并用 Operations、Monitoring、Storage、MCP allowlist 和认证机制保障生产运行。

这一定义只使用 Hindsight 官方文档中已经存在的能力，不假设官方文档之外的安全、合规或架构能力。

---

# 附录 A：官方依据索引

| 编号 | 来源文件 | 在本文中的作用 |
|---|---|---|
| S01 | `kb/00_core/faq.md` | Hindsight 与 RAG 区别、三大操作、用户隔离、延迟、zombie operations |
| S02 | `kb/00_core/best-practices.md` | Bank、taxonomy、missions、retain/recall/reflect 最佳实践、反模式 |
| S03 | `kb/00_core/templates.md` | Bank Templates Hub 与模板类型 |
| S04 | `kb/01_deployment_config/configuration.md` | 认证、MCP、LLM、embedding、reranker、recall/retain/reflect/consolidation 配置 |
| S05 | `kb/01_deployment_config/storage.md` | PostgreSQL、Oracle AI Database、pg0、生产数据库要求 |
| S06 | `kb/01_deployment_config/admin-cli.md` | migration、backup、restore、worker recovery、zombie operation 恢复 |
| S07 | `kb/02_memory_architecture/retain.md` | Retain 如何抽取事实、实体、关系、时间、因果连接 |
| S08 | `kb/02_memory_architecture/retrieval.md` | TEMPR 四策略、结果融合、token budget、chunks |
| S09 | `kb/02_memory_architecture/reflect.md` | Agentic loop、disposition、directives、evidence、hierarchical retrieval |
| S10 | `kb/02_memory_architecture/observations.md` | Observations 的证据、proof count、freshness trend、演化 |
| S11 | `kb/02_memory_architecture/rag-vs-hindsight.md` | RAG 与 Hindsight 的边界 |
| S12 | `kb/03_api/main-methods.md` | Retain/Recall/Reflect 对照 |
| S13 | `kb/03_api/memory-banks.md` | Bank 配置、entity_labels、directives、MCP tool allowlist |
| S14 | `kb/03_api/retain.md` | Retain API 参数、timestamp、document_id、update_mode、observation_scopes |
| S15 | `kb/03_api/recall.md` | Recall API 参数、types、budget、max_tokens、query_timestamp、include |
| S16 | `kb/03_api/reflect.md` | Reflect API 参数、response_schema、tags、include |
| S17 | `kb/03_api/documents.md` | Documents、chunks、document update/delete/source tracing |
| S18 | `kb/03_api/mental-models.md` | Mental Models、tags、refresh、detail levels、automatic refresh |
| S19 | `kb/03_api/operations.md` | Operation lifecycle、operation types、cancel/retry、webhook_delivery |
| S20 | `kb/03_api/bank-templates.md` | Declarative bank templates、manifest schema、import/export |
| S21 | `kb/06_operations/mcp-server.md` | MCP endpoint、tools、single-bank/multi-bank mode |
| S22 | `kb/06_operations/monitoring.md` | Prometheus metrics、OpenTelemetry tracing、Grafana dashboards |
| S23 | `kb/06_operations/performance.md` | Retain/Recall/Reflect 性能、读性能优先、预算与成本优化 |

# 附录 B：本文未做结论的事项

由于当前依据文档未明确给出，本文不对以下事项做肯定性结论：

1. Hindsight 是否内置完整 RBAC；
2. Hindsight 是否默认提供字段级加密或端到端加密；
3. Hindsight 是否通过特定行业合规认证；
4. Hindsight 是否默认提供固定数据保留周期策略；
5. Hindsight 是否对所有 MCP 工具默认采取最小权限暴露；
6. Hindsight 是否为所有部署模式自动配置生产级备份；
7. Hindsight 的云服务 SLA、区域、数据驻留和审计日志策略。

这些事项需要查阅对应的官方部署、安全、云服务或企业文档后才能定论。
