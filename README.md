# Design Review: Context Engineering, Memory & Retrieval for a Consulting Research Assistant

**Scope:** Production architecture for an AI research assistant supporting consultants on multi-week client engagements: large document volumes, multi-source retrieval, low token cost, explainable answers, multi-team scale.

**Core principle:** The context window is a scarce, expensive resource. The system keeps *everything* durable outside the model and assembles a small, ranked, cited context per call.

---

## 1. Context Engineering Strategy

Each LLM call is built by a **Context Assembler** from a fixed, budgeted template (example for a 32k working budget):

| Layer | Content | Budget | Cacheable |
|---|---|---|---|
| System | Role, policies, citation rules, tool schemas | ~2k | Yes (prompt cache) |
| Engagement brief | Client, goals, constraints, glossary, team conventions | ~1.5k | Yes |
| Plan state | Current plan, open tasks, decisions log (summary) | ~2k | Semi |
| Memory recall | Top-k relevant facts/episodes | ~3k | No |
| Retrieved evidence | Reranked chunks with source IDs | ~12k | No |
| Conversation | Last N turns + rolling summary | ~4k | No |
| Output reserve | Answer + reasoning | ~7k | n/a |

Optimization rules: stable content first (maximizes prefix caching), volatile content last; just-in-time retrieval instead of preloading; hard per-layer caps with priority-based eviction; tool outputs are trimmed or summarized before re-entering context; a **query router** classifies each request (chat, lookup, synthesis, deep research) to pick the cheapest sufficient context recipe.

## 2. Memory Architecture

| Memory | Scope / lifetime | Storage | Contents | Write policy |
|---|---|---|---|---|
| **Working** | Single LLM call/step | Context window | Assembled context, scratchpad | Rebuilt every step |
| **Session** | One conversation / work session | Redis (TTL) + DB | Recent turns, rolling summary, active task state | Summarize every ~10 turns |
| **Long-term (semantic)** | Engagement lifetime, scoped by `tenant/engagement/user` | Postgres + vector index | Verified facts, client preferences, definitions, decisions | Extracted by a background "memory writer"; deduplicated, confidence-scored, consultant can edit/delete |
| **Episodic** | Engagement lifetime | Event log + vector index | Timestamped records of past tasks: question, steps, sources used, outcome, feedback | Appended at task completion; retrieved by similarity to learn "what worked before" |

Memory is retrieved, never dumped: recall is scored on relevance × recency × importance. All memory is **namespaced per client engagement** (ethical walls); nothing crosses engagements. Stale facts carry `valid_from/valid_to` so contradictions resolve to the latest verified version.

## 3. Retrieval Architecture

- **Data sources:** SharePoint/Drive (decks, memos, models), client data rooms, internal knowledge base and past deliverables, email/meeting transcripts, licensed research and news APIs, structured data (SQL/warehouse, via text-to-SQL tool).
- **Ingestion:** Connectors → parsing (layout-aware PDF/PPTX/XLSX parsing, OCR where needed) → PII/permission tagging → chunking → embedding → indexing. Incremental sync with content hashing; ACLs copied into chunk metadata.
- **Chunking:** Structure-aware, not fixed-size. Split on headings/sections/slides/table boundaries, ~400–800 tokens with ~10–15% overlap; tables kept intact with caption; **parent-child chunks** (small child for matching, larger parent returned for context). Each chunk is prefixed with contextual metadata (doc title, section path, date, author) to improve matching ("contextual retrieval").
- **Embeddings:** A strong general embedding model (e.g., OpenAI text-embedding-3-large, Voyage, or Cohere embed) with Matryoshka/dimension reduction to cut storage; domain-tuned model considered if evals justify it. Versioned embeddings allow re-indexing without downtime.
- **Vector storage:** Qdrant or pgvector (HNSW, scalar/binary quantization) with per-tenant collections/partitions, metadata filters (engagement, date, doc type, ACL), plus BM25 index (OpenSearch/Postgres FTS).
- **Pipeline:** query rewrite/decomposition → hybrid retrieve (dense + BM25, top ~50 each) → fuse (RRF) → rerank → ACL/metadata filter → dedupe → context packing with citations.

## 4. Retrieval Optimization

- **Hybrid retrieval:** Dense catches semantics; BM25 catches exact terms (client names, acronyms, figures, clause numbers). Fused with Reciprocal Rank Fusion.
- **Reranking:** Cross-encoder (Cohere Rerank / bge-reranker) cuts ~100 candidates to top 8–12. Optional LLM-based rerank only for deep-research mode.
- **Context ranking:** Final ordering by rerank score, source authority (client-provided > internal > public), and freshness; MMR to reduce redundancy; most relevant items placed at start and end of the evidence block to mitigate "lost in the middle."
- **Source attribution:** Every chunk carries a stable `source_id`, document, page/slide, and version. The model must cite inline `[S3]`; a **citation verifier** checks each claim against the cited chunk (entailment check) and flags unsupported claims. UI links each citation to the exact passage. Answers state confidence and "not found in sources" explicitly.

## 5. Context Window Management

- **Tiered reading:** Corpus → doc summaries → section summaries → raw chunks. The agent drills down only as needed.
- **Map-reduce / hierarchical summarization** for very large inputs (e.g., 500-page data room): parallel sub-agents summarize slices with focused questions and return compact, cited findings; the orchestrator only sees the results.
- **Sub-agent isolation:** Each sub-agent runs in a clean context and returns ≤1–2k tokens.
- **Compression:** Tool-output clearing, extractive snippets over full pages, rolling conversation summaries, dedupe of repeated evidence.
- **Budget enforcement:** The assembler counts tokens before each call and degrades gracefully (fewer chunks → shorter history → smaller memory recall) rather than truncating blindly.

## 6. Long-Running Workflow Support

- **Planning persistence:** The plan (goals, tasks, status, assumptions, decisions, open questions) is a structured JSON object stored in Postgres, re-injected as a compact summary each step and updated by the agent after every task.
- **External memory:** Agent writes notes, findings, and intermediate artifacts to a per-engagement workspace (files/DB) and references them by ID, so progress survives context resets.
- **Checkpointing:** An orchestration framework with durable state (e.g., LangGraph with Postgres checkpointer, or Temporal) saves state after each node. Workflows resume after failures, restarts, or days of inactivity; human-in-the-loop approval gates for high-impact steps.
- **Context compaction:** When a session nears ~70% of budget, the system summarizes history into a structured digest (decisions, findings, open items, source IDs), writes durable facts to long-term memory, and starts a fresh context seeded with brief + plan + digest. At session start, "weeks of work" are rehydrated from plan state and episodic memory, not raw transcripts.

## 7. Cost & Performance Optimization

| Area | Strategies |
|---|---|
| **Tokens** | Prompt caching for static layers; tight top-k after reranking; summaries instead of raw text; output length limits and structured outputs; model routing (small model for routing, extraction, summarization; large model for synthesis). |
| **Retrieval cost** | Quantized vectors, reduced embedding dimensions, tiered storage (hot/warm), incremental re-embedding by content hash, embedding cache for repeated queries, rerank only top candidates. |
| **Latency** | Parallel retrieval across sources, streaming responses, semantic cache for frequent queries (with ACL-aware keys), async memory writes and ingestion, precomputed document summaries, batch APIs for offline work. |
| **Observability** | Per-request tracing (tokens, cost, latency by layer) with budgets and alerts per team. |

## 8. Risk Assessment

| Risk | Mitigation |
|---|---|
| Hallucination / unsupported claims | Mandatory citations, entailment verifier, "insufficient evidence" behavior, regression evals on a golden set |
| Retrieval misses or poor ranking | Hybrid search + reranker, query decomposition, offline evals (recall@k, nDCG), user feedback loop |
| Cross-client data leakage | Hard tenant/engagement namespaces, ACL filters at query time, no shared memory across engagements, audit logs |
| Prompt injection via documents/web | Treat retrieved content as untrusted data, delimiters, tool permission scoping, output filtering, human approval for actions |
| Memory drift / stale or wrong facts | Provenance + confidence on memories, validity windows, consultant review/edit, periodic consolidation |
| Context rot / information loss in compaction | Structured summaries, keep source IDs, store raw history externally for re-retrieval, test compaction fidelity |
| Cost blow-up at scale | Budgets per call/team, model routing, caching, usage dashboards |
| Scalability / noisy neighbors | Sharded per-tenant indexes, autoscaled ingestion workers/queues, rate limits |
| Compliance (GDPR, client contracts) | Data residency, retention policies, deletion propagation to vectors and memories, PII redaction |
| Vendor lock-in / model changes | Abstraction layer over LLM, embedding, and vector DB; versioned embeddings; eval suite to validate swaps |

---

**Summary:** A budgeted, cache-friendly context assembler fed by a four-tier memory system and a hybrid, reranked, permission-aware retrieval pipeline, with durable plan state and checkpointing for multi-week work. Cost control comes from retrieving less but better, routing to the right model, and compacting history instead of carrying it.
