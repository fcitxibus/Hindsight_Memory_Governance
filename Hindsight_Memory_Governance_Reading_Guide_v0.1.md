---
title: "Hindsight Memory Governance Reading Guide"
subtitle: "v0.1 | Compiled from official documents in hindsight_final_production_kb_v2.zip"
author: "Compiled from Hindsight official documentation"
date: "2026-06-04"
lang: en-US
---

# Hindsight Memory Governance Reading Guide v0.1

## Version Statement

This document is a reading guide for production knowledge bases, long-term Agent memory, and Hindsight integration implementation. It is written only from the Hindsight official documents retained in `hindsight_final_production_kb_v2.zip`; it does not introduce community experience, personal assumptions, or configuration conclusions that do not appear in the documents.

In this document, "memory governance" is not a separately named Hindsight product module. It is a systematic organization of memory-system isolation, writing, retrieval, reasoning, auditing, operations, and maintenance based on capabilities that already exist in the official documentation. In other words, the governance framework in this document is composed from official capabilities; it is not a new concept added outside the documentation.

The main source files include:

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

# 1. Whitepaper Summary

The Hindsight official documentation positions the system as an Agent memory system, that is, a system that provides long-term memory for AI Agents. Unlike traditional RAG, which centers on document-chunk retrieval, Hindsight's official documentation emphasizes that it stores structured facts, builds relationships between entities and concepts, supports time-aware retrieval, and provides disposition-aware memory reasoning through `reflect()`.

In production environments, Hindsight memory governance can be summarized as seven control planes:

| Control Plane | Official Capability | Governance Goal |
|---|---|---|
| Isolation Control | Memory Bank, Tags, Tenant Extension | Prevent memory leakage among users, Agents, and scenarios |
| Write Control | Retain, Document ID, Context, Timestamp, Metadata, Observation Scopes | Make content entering the memory system traceable, updatable, and deduplicable |
| Extraction Control | Retain Mission, Extraction Mode, Entity Labels | Control what the system remembers, what it does not remember, and how it classifies memory |
| Consolidation Control | Observations, Observations Mission, Consolidation | Turn multiple facts into stable knowledge supported by evidence |
| Recall Control | Recall, Tags, Types, Budget, Max Tokens, Query Timestamp | Accurately recall relevant memories while controlling cost and latency |
| Reasoning Control | Reflect, Disposition, Directives, Mental Models, Response Schema | Make memory reasoning constrained, reusable, and auditable |
| Operations Control | Operations, Monitoring, Performance, Storage, MCP Tool Allowlist | Ensure asynchronous tasks, performance, observability, and integration boundaries |

The core conclusion of this document is: production-grade Hindsight is not "writing everything into memory." It is the construction of an isolatable, traceable, filterable, auditable memory system around Memory Bank, Retain, Recall, Reflect, Observations, Mental Models, and Directives.

Sources: `kb/00_core/faq.md`, `kb/00_core/best-practices.md`, `kb/02_memory_architecture/retain.md`, `kb/02_memory_architecture/retrieval.md`, `kb/02_memory_architecture/reflect.md`.

---

# 2. Core Term Definitions

## 2.1 Memory Bank

A Memory Bank is Hindsight's isolation unit. The official documentation describes it as an independent memory store. All `retain`, `recall`, and `reflect` operations point to a single bank. Banks do not share data with each other.

Governance implications:

- One bank per user is a common pattern for most multi-user applications.
- One bank per Agent is suitable for Agent-specific long-term memory.
- A single shared bank plus tags can serve scenarios that require cross-user aggregate analysis, but visibility must be controlled through strict tag filtering.
- Banks should be configured before data is written, because bank configuration affects extraction, observations, reasoning, and tool boundaries.

Sources: `kb/00_core/best-practices.md`, `kb/00_core/faq.md`, `kb/03_api/memory-banks.md`.

## 2.2 Retain

`retain()` is the write operation. The official documentation explains that when retain is called, Hindsight turns conversations and documents into structured, searchable memories while preserving meaning and context. Retain is not simple text storage; it extracts facts, entities, relationships, temporal information, and causal relationships.

After Retain completes, the system obtains:

- structured facts;
- unified entities;
- a knowledge graph composed of entity, temporal, semantic, and causal connections;
- a temporal basis that supports both historical queries and recency ranking.

Governance implication: Retain is the entry point of memory governance. Write quality determines the quality of subsequent recall and reasoning.

Sources: `kb/02_memory_architecture/retain.md`, `kb/03_api/retain.md`, `kb/03_api/main-methods.md`.

## 2.3 Recall

`recall()` is the retrieval operation. The official documentation explains that Recall runs four retrieval strategies in parallel: semantic similarity, keyword/BM25, graph traversal, and temporal retrieval. It then returns a single ranked result list through fusion and reranking. Recall returns structured facts rather than directly generating answers.

Governance implication: Recall is suitable for passing memory as context to an external Agent or RAG pipeline, where the external system continues reasoning. It emphasizes control, speed, and raw evidence.

Sources: `kb/02_memory_architecture/retrieval.md`, `kb/03_api/recall.md`, `kb/00_core/best-practices.md`.

## 2.4 Reflect

`reflect()` is the reasoning operation. The official documentation explains that Reflect runs an agentic loop, automatically searches the memory bank, uses mental models, observations, and raw facts to form layered evidence, reasons according to the bank's disposition traits, and finally returns a synthesized answer.

The boundary between Reflect and Recall:

- Recall returns facts;
- Reflect returns answers;
- Recall is suitable when external systems perform their own reasoning;
- Reflect is suitable when Hindsight should perform multi-step search, reasoning, and answering based on memory.

Sources: `kb/02_memory_architecture/reflect.md`, `kb/03_api/reflect.md`, `kb/00_core/faq.md`.

## 2.5 Observations

Observations are knowledge consolidated from multiple facts. The official documentation explains that they are not temporary summaries fabricated by an LLM, but deduplicated beliefs, preferences, and learnings backed by concrete source memories, containing a proof count, and updated as new evidence supports, contradicts, or extends them.

Observations provide:

- deduplication: multiple repeated facts become one persistent observation;
- evidence: each observation is connected to concrete supporting memories and citations;
- evolution: observations are strengthened, weakened, contradicted, or marked stale as new evidence appears;
- freshness signals: trends such as stable, strengthening, weakening, new, and stale;
- efficiency: more compact knowledge to support retrieval.

Sources: `kb/02_memory_architecture/observations.md`, `kb/03_api/recall.md`, `kb/00_core/best-practices.md`.

## 2.6 Mental Models

Mental Models are saved `reflect` responses. When a mental model is created, Hindsight runs reflect based on `source_query` and stores the result. Later reflect calls check these precomputed contents first, producing faster and more consistent answers.

The priority order given by the official documentation is:

1. Mental Models: user-curated summaries;
2. Observations: consolidated knowledge;
3. Raw Facts: original facts used for verification.

Governance implication: Mental Models are suitable for repeatedly read, slowly changing knowledge that requires stable output, such as user profiles, current projects, technology stacks, communication styles, and common FAQ answers.

Sources: `kb/03_api/mental-models.md`, `kb/02_memory_architecture/reflect.md`, `kb/00_core/best-practices.md`.

## 2.7 Directives

Directives are hard rules that must be followed in reflect. The official documentation distinguishes directives from disposition: disposition is a soft influence, while directives are explicit constraints that are injected into the prompt and must be followed in all reflect responses.

Applicable scenarios include:

- compliance rules;
- privacy restrictions;
- style requirements;
- domain guardrails;
- requirements to cite sources.

Governance implication: Disposition is used for persona and style, while Directives are used for boundaries that cannot be violated.

Sources: `kb/02_memory_architecture/reflect.md`, `kb/03_api/memory-banks.md`, `kb/03_api/bank-templates.md`.

## 2.8 Documents and Chunks

Documents are containers for retained content. They are used to track sources, update content, bulk-delete related memories, and organize facts by source. When Hindsight retains content, it first splits it into chunks and then extracts facts. Chunks are stored together with extracted memories and preserve original text fragments. When recall needs exact wording or richer context, chunks can be returned through include options.

Governance implication: Document ID is the key field for update, replacement, provenance, and bulk deletion. Chunks are the evidence layer for exact citation and context review.

Sources: `kb/03_api/documents.md`, `kb/03_api/retain.md`, `kb/02_memory_architecture/retrieval.md`.

---

# 3. Overall Model of Hindsight Memory Governance

## 3.1 Governance Objects

What needs to be governed in Hindsight is not a single "memory item," but a set of related objects:

| Object | Role | Governance Focus |
|---|---|---|
| Bank | Isolate memory and configure behavior | User/Agent/scenario boundaries |
| Document | Organize source content | Update, replacement, deletion, provenance |
| Chunk | Preserve original text fragments | Exact citation and context lookup |
| Memory Fact | Structured fact | Type, tags, entities, time, source |
| Entity | Person, organization, place, concept, etc. | Resolution, disambiguation, relationship linking |
| Observation | Consolidated knowledge | Deduplication, evidence, evolution, freshness |
| Mental Model | Precomputed reflect result | Stable, high-frequency, human-reviewed knowledge |
| Directive | Hard rule for Reflect | Compliance, privacy, style, guardrails |
| Operation | Asynchronous task | Status, failure, retry, cancellation, recovery |

Sources: `kb/03_api/memory-banks.md`, `kb/03_api/documents.md`, `kb/03_api/operations.md`, `kb/03_api/mental-models.md`.

## 3.2 Governance Lifecycle

The Hindsight memory governance lifecycle can be organized according to the official operation chain:

```text
Configure Bank
  ↓
Retain writes content
  ↓
Extract facts / entities / relationships / time / causality
  ↓
Documents and chunks preserve source traces
  ↓
Observations automatically consolidate durable knowledge
  ↓
Recall retrieves by query/tags/types/time/budget
  ↓
Reflect reasons according to mental models → observations → raw facts
  ↓
Operations / Monitoring / Admin CLI perform operational governance
```

In this lifecycle, governance is not centered on a single answer. It is centered on what content enters the system, which bank it enters, which tags it carries, who can recall it, how it is consolidated, how it is audited, and how it is updated.

Sources: `kb/00_core/best-practices.md`, `kb/02_memory_architecture/retain.md`, `kb/02_memory_architecture/observations.md`, `kb/02_memory_architecture/retrieval.md`, `kb/02_memory_architecture/reflect.md`.

---

# 4. Official Reading Path

## 4.1 Layer One: Read the Core Concepts First

Must-read:

1. `kb/00_core/faq.md`
2. `kb/00_core/best-practices.md`
3. `kb/03_api/main-methods.md`

Reading goal: clarify the difference between Hindsight and RAG; understand the boundaries among Retain, Recall, and Reflect; and understand the governance roles of Bank, Tags, Observations, Mental Models, and Directives.

## 4.2 Layer Two: Read the Memory Architecture

Continue with:

1. `kb/02_memory_architecture/retain.md`
2. `kb/02_memory_architecture/retrieval.md`
3. `kb/02_memory_architecture/reflect.md`
4. `kb/02_memory_architecture/observations.md`
5. `kb/02_memory_architecture/rag-vs-hindsight.md`

Reading goal: understand why Hindsight is not a simple vector database; how Retain extracts facts; how Recall performs multi-strategy retrieval; how Reflect reasons through an agentic loop; and how Observations consolidate facts into durable knowledge.

## 4.3 Layer Three: Read APIs and Governance Parameters

Continue with:

1. `kb/03_api/memory-banks.md`
2. `kb/03_api/retain.md`
3. `kb/03_api/recall.md`
4. `kb/03_api/reflect.md`
5. `kb/03_api/documents.md`
6. `kb/03_api/mental-models.md`
7. `kb/03_api/operations.md`
8. `kb/03_api/bank-templates.md`

Reading goal: map concepts to concrete parameters: `retain_mission`, `observations_mission`, `reflect_mission`, `entity_labels`, `document_id`, `timestamp`, `tags`, `tags_match`, `types`, `budget`, `max_tokens`, `response_schema`, `include.facts`, `include.tool_calls`, and so on.

## 4.4 Layer Four: Read Production Operations

Continue with:

1. `kb/01_deployment_config/configuration.md`
2. `kb/01_deployment_config/storage.md`
3. `kb/01_deployment_config/admin-cli.md`
4. `kb/06_operations/performance.md`
5. `kb/06_operations/monitoring.md`
6. `kb/06_operations/mcp-server.md`

Reading goal: understand storage, authentication, MCP exposure surface, performance budgets, monitoring metrics, asynchronous tasks, zombie operation recovery, and other production runtime boundaries.

---

# 5. Bank Isolation Governance

## 5.1 Bank Is the First Isolation Boundary

The official documentation clearly states that a Memory Bank is an isolated memory store. Banks do not share data. All retain, recall, and reflect operations specify a single bank.

Production governance rules:

| Scenario | Available Pattern in Official Documentation | Governance Implication |
|---|---|---|
| Multi-user application | One bank per user | Simple, strong isolation; suitable for personalization |
| Agent-specific memory | One bank per Agent | Prevents memory contamination among different Agents |
| Cross-user aggregate analysis required | Single bank + tags | Tags and tags_match must be strictly controlled |
| Team shared knowledge | Shared bank + team/topic/scope tags | Shared and private memories must be explicitly distinguished |

Sources: `kb/00_core/best-practices.md`, `kb/00_core/faq.md`.

## 5.2 Tags Are Visibility Control in Shared Banks

In the official best practices, tags are defined as visibility scope. A memory tagged with `user:alice` will be returned only when recall/reflect calls include the corresponding tags and the matching mode allows it.

Recommended naming patterns in the official documentation include:

| Tag Pattern | Purpose |
|---|---|
| `user:<id>` | User isolation |
| `session:<id>` | Session scope |
| `team:<name>` | Team shared knowledge |
| `topic:<name>` | Domain filtering |
| `scope:<name>` | Visibility level |

Minimum multi-tenant requirement: user data must include at least the `user:<id>` tag when retained. The official documentation clearly states that omitting this tag makes the memory globally visible.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/recall.md`, `kb/03_api/reflect.md`.

## 5.3 tags_match Is the Leakage-Control Switch

The official documentation provides four tag matching modes:

| Mode | Includes Untagged Memories | Condition |
|---|---|---|
| `any` | Yes | At least one tag matches, or the memory is untagged |
| `all` | Yes | All specified tags exist, or the memory is untagged |
| `any_strict` | No | At least one tag matches |
| `all_strict` | No | All specified tags exist |

Production governance rules:

- If a bank contains both global knowledge and private user knowledge, `any` can be used to return user memory plus untagged global memory.
- If full partitioning is required and cross-user leakage is not allowed, use `any_strict` or `all_strict`.
- For combined user + topic filtering, use `all_strict`.
- For complex conditions, use the `and`, `or`, and `not` tree structure of `tag_groups`.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/recall.md`.

## 5.4 Entity Labels Can Turn Classification into Filterable Tags

The official documentation explains that `entity_labels` can define a controlled vocabulary. During retain, the LLM assigns `key:value` classification labels to facts. If `tag: true`, those labels are also written into the memory unit's tags, so they can be used for standard tag filtering in recall/reflect.

Governance implication: when a bank contains semantically similar memories with different purposes, such as rules and procedures, ranking alone cannot reliably distinguish them. After `entity_labels` are written into tags, filtering is executed at the database layer, and non-target memories do not enter the ranking pipeline.

Sources: `kb/00_core/best-practices.md`, `kb/02_memory_architecture/retain.md`, `kb/03_api/memory-banks.md`.

---

# 6. Retain Write Governance

## 6.1 Retain Is Not "The More, the Better"

The official best practices clearly list several anti-patterns: summarizing before retain, random document_id, missing context, using metadata for filtering, generic missions, immediately recalling after retain in the same request, missing timestamp, and more.

Production governance rule: before writing memory, the system must clearly define the content source, content type, owning user/session/topic, time, update method, and whether observations need to be consolidated.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/retain.md`.

## 6.2 content Should Preserve Original Context and Should Not Be Summarized First

The official anti-patterns state that pre-summarizing before retain loses entity relationships, temporal markers, and structural context. The official recommendation is to retain raw content and let Hindsight extract facts.

Governance implication: do not summarize conversations first in order to "save memory." Instead, pass context-rich original conversations, document passages, ticket content, or meeting content to retain, and control quality through context, tags, and document_id.

Sources: `kb/00_core/best-practices.md`, `kb/02_memory_architecture/retain.md`.

## 6.3 The context Field Must Describe the Nature and Source of the Content

The official best practices state that `context` has a high impact on extraction quality and should always be set. It describes the nature and source of the content.

Governance implications:

- Conversations should indicate whether they are user support conversations, product feedback, project meetings, or personal assistant conversations.
- Documents should indicate whether they are technical design documents, FAQs, tickets, contract excerpts, or research notes.
- `context` is not used for filtering; filtering uses tags.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/retain.md`.

## 6.4 document_id Is the Core of Deduplication, Updates, and Deletion

The official documentation explains that the same `document_id` means upsert: retaining the same document_id again deletes the old version and reprocesses it. `update_mode="replace"` is the default behavior. `update_mode="append"` appends new content to the existing document text and reprocesses the combined content. Delta retain skips unchanged chunks.

Production governance rules:

- Sessions, tickets, documents, and files must use stable document_id values.
- Do not use a random UUID for each retain call; the official documentation states that this creates duplicate documents.
- For continuously growing conversations, always retain the full conversation with the same ID.
- When source content needs to be deleted, documents can be used to bulk-delete the memories generated from that source.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/retain.md`, `kb/03_api/documents.md`.

## 6.5 timestamp Affects Temporal Retrieval

The official documentation explains that `timestamp` should be set when temporal context exists. It enables temporal retrieval strategies. For conversations, it can be set to the session start time. Omitting timestamp disables temporal ranking.

The Retain API also explains:

- omitted or `null`: use the current ingestion time;
- ISO 8601 string: use the given time;
- `"unset"`: no timestamp, suitable for timeless material such as reference documents, books, or fictional content.

Governance implication: time-sensitive content must carry the true event time. Materials without a true event time should use `"unset"` to avoid polluting temporal retrieval with incorrect time.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/retain.md`, `kb/02_memory_architecture/retain.md`.

## 6.6 metadata Is for Provenance, Not Filtering

The official best practices explicitly state: metadata is not filterable; filtering should use tags. Metadata is returned with recalled memories and is suitable for linking to source systems, UI deep links, and audit trails.

Governance implications:

- `source`, `channel`, `thread_id`, `ticket_id`, `url`, `version`, and similar fields are suitable for metadata.
- Search visibility conditions such as `user:<id>`, `topic:<name>`, and `scope:<name>` must be placed in tags.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/recall.md`.

## 6.7 observation_scopes Controls Consolidation Granularity

The official documentation provides four observation scopes:

| Value | Behavior | Use Case |
|---|---|---|
| `combined` | Run one observation pass over all tags together | Default; single-user bank and general usage |
| `per_tag` | Run an independent observation pass for each tag | User behavior observations that require isolation |
| `all_combinations` | Run all tag subset combinations | Complex multidimensional analysis; high cost |
| Custom list | Explicitly specify scope list | Precise multi-tenant control |

Production governance rule: in multi-tenant scenarios, tags should not be mixed for consolidation by default. When user-level, team-level, or combination-level observations are required, scopes should be explicitly defined.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/retain.md`, `kb/02_memory_architecture/observations.md`.

## 6.8 Sync and Async Boundaries

The official best practices explain:

- `async_=False` is the default for scenarios where confirmation is needed before continuing;
- `async_=True` is used for end-of-turn retain, end-of-session retain, and user-facing low-latency flows;
- retain should not be immediately followed by recall in the same turn, because retain is a write operation and extracted memories are not immediately available.

Governance implication: recall at the beginning of an Agent turn; retain at the end of the turn. Do not treat retain as immediate context writing.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/retain.md`, `kb/06_operations/performance.md`.

---

# 7. Extraction Governance: Missions and Entity Labels

## 7.1 Boundaries of the Three Mission Types

The official best practices emphasize that all missions accept natural language, but they must be specific. Vague missions produce vague results.

| Mission | Scope of Impact | Role |
|---|---|---|
| `retain_mission` | Retain | Tells the LLM what to extract and what to ignore |
| `observations_mission` | Observations consolidation | Controls what types of durable patterns are consolidated |
| `reflect_mission` | Reflect | Sets the bank's identity, reasoning frame, and response orientation |

Governance rule: do not use one generic description to cover all stages. Writing, consolidation, and reasoning are different control planes and must be configured separately.

Sources: `kb/00_core/best-practices.md`, `kb/02_memory_architecture/retain.md`, `kb/02_memory_architecture/observations.md`, `kb/02_memory_architecture/reflect.md`.

## 7.2 retain_mission Must Define Both "What to Remember" and "What Not to Remember"

The official documentation states that `retain_mission` is injected into the fact extraction prompt and is used to tell the LLM what to extract and what to ignore. An effective mission should list needed fact types, such as preferences, decisions, errors, and commitments, and also list content to ignore, such as greetings, small talk, and scheduling logistics.

Governance implication: a high-quality retain_mission is not "extract all information." It is a domain-specific extraction rule.

Sources: `kb/00_core/best-practices.md`, `kb/02_memory_architecture/retain.md`, `kb/03_api/memory-banks.md`.

## 7.3 observations_mission Must Emphasize Durable Patterns

The official documentation explains that `observations_mission` can replace the default durable-knowledge rule and control the shape of observations. The official recommendation is to emphasize durable patterns, avoid ephemeral observation noise, and explicitly require contradiction detection when history tracking is needed.

Governance implication: observations should not become a pile of temporary states. They should carry stable preferences, skills, relationships, recurring patterns, and contradictory changes.

Sources: `kb/00_core/best-practices.md`, `kb/02_memory_architecture/observations.md`.

## 7.4 reflect_mission and Disposition Jointly Control Reasoning Style

The official documentation explains that `reflect_mission` provides identity context, describes who the agent is and what it cares about, and keeps reasoning consistent across conversations. Disposition traits include skepticism, literalism, and empathy, with values from 1 to 5.

Governance implications:

- `reflect_mission` is not an extraction rule; it only affects reflect.
- `disposition_skepticism` controls the tendency to trust or question.
- `disposition_literalism` controls the tendency to interpret flexibly or literally.
- `disposition_empathy` controls the tendency to focus on facts or emotional context.

Sources: `kb/02_memory_architecture/reflect.md`, `kb/03_api/memory-banks.md`.

## 7.5 entity_labels Is Structured Classification Capability

The official Memory Banks API defines `entity_labels` as label group configuration. Fields include `key`, `description`, `type`, `values`, `fields`, `optional`, and `tag`. When `tag: true`, the extracted label is written as tags and supports recall/reflect filtering.

Governance implication: Entity Labels are suitable for turning business categories, knowledge forms, priority, status, roles, and similar dimensions into structured memory labels that the system can recognize.

Sources: `kb/03_api/memory-banks.md`, `kb/00_core/best-practices.md`.

---

# 8. Observations Consolidation Governance

## 8.1 Observations Are an Evidence-Driven Consolidation Layer

The official definition of Observations is consolidated knowledge built from multiple facts. They differ from single raw facts and are not real-time temporary summaries. Each observation has supporting memories, a proof count, and a freshness trend.

Governance implication: in production use, Observations are an important layer for reducing repeated facts and improving long-term preference recognition and pattern induction. However, they must be supported by factual evidence.

Sources: `kb/02_memory_architecture/observations.md`, `kb/03_api/recall.md`.

## 8.2 Observations Are Maintained Automatically After Retain

The official documentation explains that Observations are maintained by background consolidation after retain operations complete, rather than being part of the retain call itself.

Governance implication: at the system design level, the write path and consolidation path are separate. Production monitoring must track the status of both retain operations and consolidation operations.

Sources: `kb/02_memory_architecture/retain.md`, `kb/02_memory_architecture/observations.md`, `kb/03_api/operations.md`.

## 8.3 Observations Can Be Filtered by Recall types

The `types` parameter of Recall supports `world`, `experience`, and `observation`. The official documentation explains that querying only `observation` can return consolidated patterns, while querying `world` and `experience` can return raw facts.

Governance implications:

- For high-level user profiles, long-term preferences, and behavior patterns: prefer querying `observation`.
- For scenarios requiring raw evidence and citation sensitivity: query `world` and `experience`.
- If `types` is not set: all three types are queried.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/recall.md`.

---

# 9. Recall Governance

## 9.1 Recall Uses Four Parallel Retrieval Strategies

The official Retrieval document names Recall's retrieval strategy TEMPR, including:

| Strategy | Problem Solved |
|---|---|
| Semantic Search | Semantic similarity, synonymous expressions, natural-language questions |
| Keyword Search / BM25 | Proper nouns, technical terms, unique identifiers, exact phrases |
| Graph Traversal | Indirect relationships and multi-hop connections between entities |
| Temporal Search | Historical time, time ranges, relative time, before/after relationships |

After retrieval, results are fused. Memories that appear in multiple strategies rank higher. Rank is more important than raw score, and a neural model is used for final reranking.

Sources: `kb/02_memory_architecture/retrieval.md`, `kb/03_api/recall.md`.

## 9.2 Recall query Is the Only Required Field

In the Recall API, `query` is the only required field. It drives all four retrieval strategies at the same time: semantic embedding, BM25 tokenization, graph traversal seed, and temporal parsing. After retrieval, it is also passed to the cross-encoder reranker. The official configuration also states that recall queries longer than 500 tokens are rejected.

Governance implication: query should not be stuffed with excessive context. Long context should be organized through tags, types, metadata provenance, document_id, and external prompts, rather than putting all content into the recall query.

Sources: `kb/03_api/recall.md`, `kb/01_deployment_config/configuration.md`.

## 9.3 Budget Controls Retrieval Depth and Is Not the Same as max_tokens

The official documentation clearly distinguishes:

- `budget` controls search depth, candidate pool, graph traversal, and reranking work;
- `max_tokens` controls the total token amount of returned memory text.

Recall budget can be `low`, `mid`, or `high`. The latency ranges provided in Best Practices are:

| Budget | Latency | Use Case |
|---|---|---|
| `low` | 50-100ms | Simple fact queries, single-hop questions |
| `mid` | 100-300ms | Multi-hop reasoning and relationship queries; default |
| `high` | 300-500ms | Deep exploration and complex cross-domain patterns |

Production governance rule: use `mid` by default; use `low` for high-frequency agent loops; use `high` only for explicit deep recall.

Sources: `kb/00_core/best-practices.md`, `kb/02_memory_architecture/retrieval.md`, `kb/03_api/recall.md`.

## 9.4 max_tokens Is Governed by the Agent Context Budget

The official documentation explains that Hindsight is designed for Agents and does not mainly use a top-k model. Instead, it returns results according to a token budget. The default value of `max_tokens` is 4096. It counts only the `text` field of each fact; metadata, tags, entities, and similar fields are not counted in this budget. After reranking, facts are filled in relevance order until the budget is exhausted.

Governance implications:

- High-quality retrieval is not a fixed top-k; it is the amount of memory context allocated to the Agent.
- Smaller token budgets can be used for concise Q&A.
- Larger token budgets can be used for comprehensive summaries.

Sources: `kb/02_memory_architecture/retrieval.md`, `kb/03_api/recall.md`.

## 9.5 include Should Be Enabled Only When the Evidence Layer Is Needed

The include options given in the official best practices are:

| Option | Default | When to Enable |
|---|---|---|
| `include.entities` | Enabled | Keep enabled to provide graph traversal entity context |
| `include.chunks` | Disabled | When original wording or source citation is needed |
| `include.source_facts` | Disabled | When auditing observation provenance |

Governance implication: return structured facts by default. Expand returned content only when original text, exact citation, or source audit is required.

Sources: `kb/00_core/best-practices.md`, `kb/02_memory_architecture/retrieval.md`, `kb/03_api/recall.md`.

## 9.6 query_timestamp Is Used for Time-Sensitive Queries

The official documentation explains that `query_timestamp` anchors relative time expressions and recency scoring. For example, a query such as "what was the team working on in January this year" needs to anchor the query time to a concrete point in time.

Governance implication: any recall containing relative time expressions such as "recently," "last week," "this January," "before," or "after" should explicitly set `query_timestamp` to avoid unstable time interpretation.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/recall.md`.

---

# 10. Reflect Reasoning Governance

## 10.1 Reflect Is an Agentic Loop with Evidence Search

The official documentation explains that the Reflect agent has the following tools:

| Tool | Role | Priority |
|---|---|---|
| `search_mental_models` | User-curated summaries | Highest |
| `search_observations` | Consolidated knowledge | High |
| `recall` | Raw facts | Fallback |
| `expand` | Get more memory context | As needed |
| `done` | Complete the final answer | When ready |

The Reflect agent must gather evidence before answering, runs at most 10 iterations, and may cite only IDs that were actually retrieved.

Source: `kb/02_memory_architecture/reflect.md`.

## 10.2 Reflect Is Suitable for Synthesized Answers, Not as a Replacement for All Recall

The official FAQ and best practices both explain: use recall when raw facts, maximum control, simple fact queries, low latency, or a custom answer synthesis layer is required; use reflect when a ready-to-use answer, disposition-aware response, multi-step reasoning, structured output, or citations are required.

The official FAQ also gives a latency comparison: Recall is approximately 50-500ms; Reflect is approximately 1-10s.

Governance implication: production systems should not turn every memory query into reflect. High-frequency, low-latency, controllable context injection should prioritize recall; complex judgment, recommendations, profiles, and synthesized answers should use reflect.

Sources: `kb/00_core/faq.md`, `kb/00_core/best-practices.md`, `kb/03_api/reflect.md`.

## 10.3 Disposition Is a Soft Constraint

The official documentation defines disposition traits as three characteristics that affect reflect interpretation and reasoning:

| Trait | Range | Low Value Means | High Value Means |
|---|---|---|---|
| Skepticism | 1-5 | Trusting, accepts surface information | Skeptical, questions claims |
| Literalism | 1-5 | Flexible interpretation, reads implications | Literal interpretation, understands by facts |
| Empathy | 1-5 | Detached, focuses on factual logic | Considers emotional context |

Governance implication: Disposition is used to form a consistent role, not to enforce compliance rules. Mandatory rules must use Directives.

Sources: `kb/02_memory_architecture/reflect.md`, `kb/03_api/memory-banks.md`.

## 10.4 Directives Are Hard Constraints

The official documentation clearly distinguishes Directives from Disposition: Disposition is a soft influence, while Directives are hard rules. Directives affect only reflect, are injected into the prompt, and require the agent to follow them in responses.

Production governance rules:

- Put non-violable requirements such as compliance, privacy, citation, and style into Directives.
- Put personality, caution level, and expression orientation into Disposition.
- Directives can be created, listed, updated, and deleted through the API, and can also be copied through bank templates.

Sources: `kb/02_memory_architecture/reflect.md`, `kb/03_api/memory-banks.md`, `kb/03_api/bank-templates.md`.

## 10.5 response_schema Is for Programmatic Output

The Reflect API explains that `response_schema` is an optional JSON Schema. When provided, the LLM generates a response that conforms to the schema and returns the parsed result through `structured_output`. This mode is suitable for programmatic consumption rather than displaying natural-language paragraphs.

Governance implication: when reflect results enter workflows, automated approval, routing, scoring, or structured UI display, use response_schema instead of asking the model to freely output a fixed format.

Sources: `kb/03_api/reflect.md`, `kb/00_core/best-practices.md`.

## 10.6 include.facts and include.tool_calls Are Audit Switches

The official best practices explain:

| Option | Purpose |
|---|---|
| `include.facts=True` | Exposes which memories and mental models were used, supporting transparency and audit |
| `include.tool_calls=True` | Exposes the full execution trace of the internal search loop, suitable for debugging |

The official recommendation: enable `include.facts` in production for audit; enable `include.tool_calls` only during development.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/reflect.md`.

---

# 11. Mental Models Governance

## 11.1 Mental Models Are for High-Frequency, Stable, Consistency-Required Knowledge

The official best practices recommend creating mental models in the following situations:

- common repeated queries that need consistent answers;
- high-frequency Agents that need sub-100ms responses;
- user profiles or personas read on every request;
- knowledge summaries already reviewed or approved by humans;
- cross-session and slowly changing state, such as preferences, skills, and background.

Governance implication: a Mental Model is a "curated memory summary" in production systems. It is not a grab bag for all knowledge.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/mental-models.md`.

## 11.2 Tags Affect Both Mental Model Construction and Visibility

The official best practices explain that tags on a Mental Model both filter "the memories used to build it" and "which recall/reflect calls can see it." The Mental Model API also explains that tags use `all_strict` matching by default, so only memories carrying all specified tags are read.

Governance implication: user-level mental models must carry user tags; team-level mental models must carry team or scope tags; global mental models are globally visible when untagged.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/mental-models.md`.

## 11.3 Refresh Strategy Must Be Chosen by Knowledge Change Rate

The official Mental Models API supports `trigger`, where `refresh_after_consolidation` can automatically refresh after observation consolidation. Refresh modes include:

- `full`: regenerate the full content from scratch each time; simple and predictable;
- `delta`: output typed operations to add, remove, or modify the existing document structure; suitable for long-lived skills, playbooks, or onboarding summaries.

Boundaries given by the official documentation: real-time dashboards and user preferences can enable automatic refresh; policy summaries and FAQ answers are more suitable for manual refresh because they change infrequently or require review.

Sources: `kb/03_api/mental-models.md`, `kb/00_core/best-practices.md`.

## 11.4 Mental Model Granularity Must Be Narrow

The official best practices clearly state that narrow, scoped models should be created, with one model per knowledge dimension. Models such as "Everything about the user" have low accuracy, slow refresh, and poor scoping.

Available granularity examples:

- User Profile;
- Current Projects;
- Technical Stack;
- Communication Style.

Sources: `kb/00_core/best-practices.md`, `kb/03_api/mental-models.md`.

---

# 12. Operations and Asynchronous Task Governance

## 12.1 Operations Have a Clear Lifecycle

The official Operations API defines the following statuses:

| Status | Meaning |
|---|---|
| `pending` | Task is queued, not yet taken by a worker, or delayed due to backpressure |
| `processing` | Worker has taken the task and is executing it |
| `completed` | Handler returned successfully |
| `failed` | Handler raised an error; `error_message` contains the reason and the operation can be retried |
| `cancelled` | Cancelled before a worker took it; operations already in processing cannot be cancelled |

Governance implication: production systems must treat retain, file convert, consolidation, mental model refresh, graph maintenance, and webhook delivery as observable background tasks, not as synchronous function calls.

Source: `kb/03_api/operations.md`.

## 12.2 Operation Types Correspond to Different Maintenance Tasks

The operation types listed in the official documentation include:

- `retain`;
- `file_convert_retain`;
- `consolidation`;
- `refresh_mental_model`;
- `graph_maintenance`;
- `webhook_delivery`.

Governance implication: different task failures have different impacts. Retain failure affects writing; consolidation failure affects observations; refresh_mental_model failure affects precomputed answers; graph_maintenance failure affects graph-structure maintenance; webhook_delivery failure affects external system notifications.

Source: `kb/03_api/operations.md`.

## 12.3 Zombie Operations Require Stable worker_id

The official FAQ defines a zombie operation as a background task stuck in `processing` because the worker that claimed it has disappeared, usually after a Docker container restart. The root cause is usually an unstable `HINDSIGHT_API_WORKER_ID`; the default worker uses the container hostname, and Docker may change it on every restart.

Recovery methods:

```bash
hindsight-admin decommission-worker <old-worker-id>
hindsight-admin decommission-workers
```

Prevention method: set a stable `HINDSIGHT_API_WORKER_ID` for the worker. The official documentation explains that the Helm chart handles this by binding the pod name through a StatefulSet.

Sources: `kb/00_core/faq.md`, `kb/01_deployment_config/admin-cli.md`.

## 12.4 Webhooks Are the External Notification Mechanism for Operations

The official Operations document explains that after certain operations complete, for example after consolidation completes and the bank is configured with a webhook, Hindsight enqueues a `webhook_delivery` task. The handler POSTs a payload to the configured URL and retries on transient failures.

Configuration items include:

- `HINDSIGHT_API_WEBHOOK_URL`;
- `HINDSIGHT_API_WEBHOOK_SECRET`;
- `HINDSIGHT_API_WEBHOOK_EVENT_TYPES`;
- webhook delivery poll interval, batch size, max attempts, and more.

Sources: `kb/03_api/operations.md`, `kb/01_deployment_config/configuration.md`, `kb/03_api/webhooks.md`.

---

# 13. MCP and Tool Exposure Governance

## 13.1 MCP Server Is Enabled by Default

The official MCP Server documentation explains that Hindsight has a built-in Model Context Protocol server, enabled by default, mounted at `/mcp` on the API server. Each memory bank has its own MCP endpoint:

```text
http://localhost:8888/mcp/{bank_id}/
```

It can be disabled through an environment variable:

```bash
export HINDSIGHT_API_MCP_ENABLED=false
```

Sources: `kb/06_operations/mcp-server.md`, `kb/01_deployment_config/configuration.md`.

## 13.2 MCP Tools Must Be Exposed Minimally

The official configuration document provides `HINDSIGHT_API_MCP_ENABLED_TOOLS`, which limits which MCP tools are registered at the server level. The Memory Banks API also provides a bank-level `mcp_enabled_tools` allowlist. Tool calls not included in the list return an error.

Governance implications:

- Read-only deployments can expose only `recall`.
- When only retrieval and reasoning are allowed, expose `recall` and `reflect`.
- Expose `retain` only when writing is allowed.
- High-privilege tools such as `clear_memories`, `delete_bank`, and `update_bank` should not be exposed by default to untrusted Agents.

The last item is a governance usage derived from the official allowlist capability. The official documentation does not provide a verbatim requirement that "delete tools are forbidden by default," so this document treats it as a security-use guideline based on the allowlist, not as a mandatory default behavior of Hindsight.

Sources: `kb/01_deployment_config/configuration.md`, `kb/03_api/memory-banks.md`, `kb/06_operations/mcp-server.md`.

## 13.3 MCP Authentication and Transport Configuration

The official configuration includes:

- `HINDSIGHT_API_MCP_AUTH_TOKEN`: MCP Bearer token;
- `HINDSIGHT_API_MCP_STATELESS`: whether to use stateless HTTP transport;
- `HINDSIGHT_API_MCP_LOCAL_BANK_ID`: bank ID used by local MCP;
- `HINDSIGHT_API_MCP_INSTRUCTIONS`: additional instructions appended to retain/recall tool descriptions.

Governance implication: when MCP is used publicly or across tools, the authentication token, transport mode, and available tool set should be explicitly defined.

Sources: `kb/01_deployment_config/configuration.md`, `kb/06_operations/mcp-server.md`.

---

# 14. Deployment, Storage, and Authentication Governance

## 14.1 Storage Backends

The official Storage document explains that Hindsight uses PostgreSQL as the primary storage backend and provides Oracle AI Database as an enterprise deployment alternative. PostgreSQL provides:

- pgvector vector search;
- full-text search;
- relational data;
- JSONB;
- recursive CTE support for graph queries.

Production environments use PostgreSQL 15+ and require pgvector 0.5.0+. Local development can use pg0 when `HINDSIGHT_API_DATABASE_URL` is not configured, with data stored in `~/.hindsight/pg0/`.

Source: `kb/01_deployment_config/storage.md`.

## 14.2 Authentication Is Disabled by Default; Production Should Enable API Key Authentication

The official Configuration document explains that Hindsight has no authentication by default. Production deployments can enable the built-in API key tenant extension:

```bash
export HINDSIGHT_API_TENANT_EXTENSION=hindsight_api.extensions.builtin.tenant:ApiKeyTenantExtension
export HINDSIGHT_API_TENANT_API_KEY=your-secret-api-key
```

After enabling it, requests must include:

```text
Authorization: Bearer your-secret-api-key
```

Requests without a valid API key return `401 Unauthorized`. Advanced authentication such as JWT, OAuth, and multi-tenant schemas requires a custom `TenantExtension`.

Sources: `kb/01_deployment_config/configuration.md`, `kb/06_operations/extensions.md`.

## 14.3 This Document Does Not Assert Security Capabilities Not Confirmed by Official Documentation

Within the current source documents, this document does not make factual assertions about:

- whether built-in RBAC exists;
- whether built-in end-to-end encryption exists;
- whether specific compliance certifications are provided;
- whether fixed data retention periods are provided;
- whether all stored fields are encrypted by default.

This document only confirms what the official documentation explicitly states: authentication is disabled by default, production can enable the API key tenant extension; MCP can configure an auth token and tool allowlist; Bank and tags can be used for memory isolation and visibility control; Documents and metadata support source tracing.

Sources: `kb/01_deployment_config/configuration.md`, `kb/03_api/memory-banks.md`, `kb/06_operations/mcp-server.md`.

---

# 15. Performance Governance

## 15.1 Hindsight Prioritizes Read Performance

The official Performance document explains that Hindsight has three performance priorities:

- Retain: batch processing and async operations for large-scale memory storage;
- Recall: sub-second semantic search and configurable thinking budgets;
- Reflect: disposition-aware answer generation and controllable compute.

The official documentation also explains that the system architecture prioritizes read performance because the typical usage pattern of a memory system is write once and read many times.

Source: `kb/06_operations/performance.md`.

## 15.2 The Main Retain Performance Bottleneck Is the LLM

The official Models and Performance documents state that the LLM is the main bottleneck for retain operations. The Performance document also explains that retain does not require the smartest model; fact extraction can usually be handled by a smaller, faster, and cheaper model.

Governance implication: in production, the retain LLM, reflect LLM, and consolidation LLM should be treated as different control planes. The configuration document provides separate variables for LLM provider, model, API key, and similar settings for retain, reflect, and consolidation.

Sources: `kb/01_deployment_config/models.md`, `kb/01_deployment_config/configuration.md`, `kb/06_operations/performance.md`.

## 15.3 Recall Controls Cost and Latency Through budget and max_tokens

Recall's `budget` affects retrieval depth, while `max_tokens` affects returned context size. The official documentation emphasizes that the two are independent.

Governance rules:

- High-frequency chat replies: low budget and small max_tokens;
- Ordinary document Q&A: mid budget and default-level context;
- Research queries: high budget and larger max_tokens;
- Do not make high budget the default for all queries; the official best practices list this as an anti-pattern.

Sources: `kb/02_memory_architecture/retrieval.md`, `kb/00_core/best-practices.md`, `kb/06_operations/performance.md`.

## 15.4 Monitoring Is a Required Production Control Plane

The official Monitoring document explains that Hindsight provides Prometheus metrics, OpenTelemetry distributed tracing, and Grafana dashboards. Available metrics include:

| Metric Category | Examples |
|---|---|
| Operation Metrics | `hindsight.operation.duration`, `hindsight.operation.total` |
| LLM Metrics | `hindsight.llm.duration`, `hindsight.llm.calls.total`, input/output tokens |
| HTTP Metrics | `hindsight.http.duration`, `hindsight.http.requests.total` |
| Database Pool Metrics | Database connection pool metrics |
| Process Metrics | Process resource metrics |

Governance implication: production environments should at least monitor retain/recall/reflect latency, operation success rate, LLM call latency, token consumption, HTTP 5xx, and database pool utilization.

Sources: `kb/06_operations/monitoring.md`, `kb/06_operations/performance.md`.

---

# 16. Bank Templates and Configuration Reuse Governance

## 16.1 Bank Template Is a Declarative JSON Manifest

The official Bank Templates API defines a bank template as a JSON manifest describing the complete bank setup, including configuration overrides, mental models, directives, and more. Importing a manifest can configure a bank in one operation.

Templates are suitable for:

- copying consistent configuration to multiple users or Agents;
- onboarding new users;
- sharing recommended configurations;
- providing recommended templates together with framework integrations.

Sources: `kb/03_api/bank-templates.md`, `kb/00_core/templates.md`.

## 16.2 Template Schema Is a Governance Baseline

The official schema includes:

- `version`, currently `"1"`;
- `bank`: reflect_mission, retain_mission, retain_extraction_mode, retain_chunk_size, disposition, enable_observations, observations_mission, entity_labels, and more;
- `mental_models`: id, name, source_query, tags, max_tokens, trigger;
- `directives`: name, content, priority, is_active, tags.

Governance implication: production systems should not scatter configuration manually. Different Agent types, business domains, and tenant types should have template manifests as auditable, reproducible, dry-runnable configuration baselines.

Source: `kb/03_api/bank-templates.md`.

## 16.3 Import Behavior

The official documentation explains: when importing a template, if the bank does not exist, it is automatically created; config fields are applied as per-bank overrides; mental models are matched by id, existing ones are updated and missing ones are created; directives are matched by name; mental model content is generated asynchronously, and the response contains operation_ids.

Sources: `kb/03_api/bank-templates.md`, `kb/03_api/operations.md`.

---

# 17. Anti-Pattern List

The following anti-patterns all come from the official Best Practices and are reorganized by governance layer:

| Anti-Pattern | Consequence | Correct Practice |
|---|---|---|
| Summarizing before retain | Loses entity relationships, temporal markers, and structural context | Retain raw content and let Hindsight extract facts |
| Using a random document_id for each retain | Creates a new document every time and causes duplication | Use stable session/ticket/document IDs |
| Omitting context | Significantly reduces extraction quality | Always describe data type and source |
| Using metadata for filtering | metadata is not filterable | Filtering conditions must use tags |
| Overly generic mission | Extraction noise and low-value memory | Specify domain, data type, and ignored items |
| Using `tags_match="any"` in a multi-tenant bank | Possible cross-user leakage | Use `any_strict` or `all_strict` for user partitioning |
| Immediately recalling after retain in the same request | Newly written memory is not yet indexed | Recall at turn start; retain at turn end |
| One mental model covering everything | Low accuracy, slow refresh, difficult scoping | One model per knowledge dimension |
| Using high budget for all recall | Slow and expensive | Simple queries use low; default uses mid; deep queries use high |
| Retain missing timestamp | Disables temporal retrieval strategies | Set timestamp when true time exists |

Source: `kb/00_core/best-practices.md`.

---

# 18. Production Implementation Checklist

## 18.1 Bank Design Checklist

- Has it been clearly defined which bank corresponds to each user, Agent, team, or scenario?
- If a shared bank is used, have `user:<id>`, `team:<name>`, `topic:<name>`, and `scope:<name>` tag conventions been defined?
- Has the tradeoff between per-user banks and shared banks been clarified?
- Has bank configuration been turned into templates instead of manual configuration?

Sources: `kb/00_core/best-practices.md`, `kb/03_api/bank-templates.md`.

## 18.2 Retain Checklist

- Does every item of user data carry a `user:<id>` tag?
- Is a stable `document_id` set?
- Is `context` set?
- Is the correct `timestamp` set, or is `"unset"` explicitly used?
- Are metadata and tags distinguished?
- Are `observation_scopes` clearly defined?
- Is immediate recall after retain in the same turn avoided?

Sources: `kb/00_core/best-practices.md`, `kb/03_api/retain.md`.

## 18.3 Recall Checklist

- Has the correct `tags_match` been selected?
- Is `types` filtering needed?
- Are `budget` and `max_tokens` configured separately?
- Do time-sensitive queries set `query_timestamp`?
- Are `include.chunks` or `include.source_facts` enabled only when needed?
- Are complex conditions expressed with `tag_groups`?

Sources: `kb/03_api/recall.md`, `kb/00_core/best-practices.md`.

## 18.4 Reflect Checklist

- Is reflect used only when synthesized answers are needed?
- Is `reflect_mission` configured?
- Are appropriate skepticism, literalism, and empathy values set?
- Are compliance, privacy, citation, and style requirements written as directives?
- Is `response_schema` set for programmatic output?
- Is `include.facts` enabled for production audit?
- Is `include.tool_calls` enabled only for development debugging?

Sources: `kb/02_memory_architecture/reflect.md`, `kb/03_api/reflect.md`, `kb/00_core/best-practices.md`.

## 18.5 Operations Checklist

- Is PostgreSQL 15+ with pgvector 0.5.0+ configured?
- Is API key authentication or a custom TenantExtension enabled in production?
- Does MCP use a minimal tool allowlist?
- Are operation, LLM, HTTP, DB pool, and process metrics monitored?
- Is a stable `HINDSIGHT_API_WORKER_ID` set for workers?
- Is an admin CLI recovery process prepared for zombie operations?
- Are mental model refresh, consolidation, and webhook delivery monitored as operations?

Sources: `kb/01_deployment_config/storage.md`, `kb/01_deployment_config/configuration.md`, `kb/01_deployment_config/admin-cli.md`, `kb/06_operations/monitoring.md`, `kb/03_api/operations.md`.

---

# 19. Minimum Viable Reading Order

If reading only 10 documents, use this order:

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

For production deployment, add:

11. `kb/03_api/mental-models.md`
12. `kb/03_api/operations.md`
13. `kb/03_api/bank-templates.md`
14. `kb/01_deployment_config/configuration.md`
15. `kb/01_deployment_config/storage.md`
16. `kb/06_operations/mcp-server.md`
17. `kb/06_operations/monitoring.md`
18. `kb/06_operations/performance.md`

---

# 20. One-Sentence Summary

Based on the official documentation, production-grade Hindsight memory governance can be defined as:

> Using Memory Bank as the isolation boundary; Retain to control write quality; Tags, Types, Budget, and Timestamp to control recall; Observations to consolidate long-term knowledge; Mental Models to stabilize high-frequency knowledge; Directives and Disposition to constrain reasoning behavior; and Operations, Monitoring, Storage, MCP allowlist, and authentication mechanisms to support production operation.

This definition uses only capabilities that already exist in the Hindsight official documentation. It does not assume security, compliance, or architecture capabilities outside the official documents.

---

# Appendix A: Official Source Index

| No. | Source File | Role in This Document |
|---|---|---|
| S01 | `kb/00_core/faq.md` | Difference between Hindsight and RAG, three main operations, user isolation, latency, zombie operations |
| S02 | `kb/00_core/best-practices.md` | Bank, taxonomy, missions, retain/recall/reflect best practices, anti-patterns |
| S03 | `kb/00_core/templates.md` | Bank Templates Hub and template types |
| S04 | `kb/01_deployment_config/configuration.md` | Authentication, MCP, LLM, embedding, reranker, recall/retain/reflect/consolidation configuration |
| S05 | `kb/01_deployment_config/storage.md` | PostgreSQL, Oracle AI Database, pg0, production database requirements |
| S06 | `kb/01_deployment_config/admin-cli.md` | migration, backup, restore, worker recovery, zombie operation recovery |
| S07 | `kb/02_memory_architecture/retain.md` | How Retain extracts facts, entities, relationships, time, and causal connections |
| S08 | `kb/02_memory_architecture/retrieval.md` | TEMPR four strategies, result fusion, token budget, chunks |
| S09 | `kb/02_memory_architecture/reflect.md` | Agentic loop, disposition, directives, evidence, hierarchical retrieval |
| S10 | `kb/02_memory_architecture/observations.md` | Observation evidence, proof count, freshness trend, evolution |
| S11 | `kb/02_memory_architecture/rag-vs-hindsight.md` | Boundary between RAG and Hindsight |
| S12 | `kb/03_api/main-methods.md` | Retain/Recall/Reflect comparison |
| S13 | `kb/03_api/memory-banks.md` | Bank configuration, entity_labels, directives, MCP tool allowlist |
| S14 | `kb/03_api/retain.md` | Retain API parameters, timestamp, document_id, update_mode, observation_scopes |
| S15 | `kb/03_api/recall.md` | Recall API parameters, types, budget, max_tokens, query_timestamp, include |
| S16 | `kb/03_api/reflect.md` | Reflect API parameters, response_schema, tags, include |
| S17 | `kb/03_api/documents.md` | Documents, chunks, document update/delete/source tracing |
| S18 | `kb/03_api/mental-models.md` | Mental Models, tags, refresh, detail levels, automatic refresh |
| S19 | `kb/03_api/operations.md` | Operation lifecycle, operation types, cancel/retry, webhook_delivery |
| S20 | `kb/03_api/bank-templates.md` | Declarative bank templates, manifest schema, import/export |
| S21 | `kb/06_operations/mcp-server.md` | MCP endpoint, tools, single-bank/multi-bank mode |
| S22 | `kb/06_operations/monitoring.md` | Prometheus metrics, OpenTelemetry tracing, Grafana dashboards |
| S23 | `kb/06_operations/performance.md` | Retain/Recall/Reflect performance, read-performance priority, budget and cost optimization |

# Appendix B: Matters Not Concluded in This Document

Because the current source documents do not explicitly state them, this document does not make affirmative conclusions on the following matters:

1. Whether Hindsight has built-in full RBAC;
2. Whether Hindsight provides field-level encryption or end-to-end encryption by default;
3. Whether Hindsight has specific industry compliance certifications;
4. Whether Hindsight provides a default fixed data retention policy;
5. Whether Hindsight exposes all MCP tools with least privilege by default;
6. Whether Hindsight automatically configures production-grade backups for all deployment modes;
7. The SLA, regions, data residency, and audit log policies of Hindsight cloud service.

These matters require corresponding official deployment, security, cloud service, or enterprise documents before conclusions can be made.
