# Industrial Document Intelligence Platform — Architecture

## 0. Framing

Naive RAG (PDF → chunk → embed → retrieve) fails here for one reason: **it treats a document as a bag of text**, when industrial docs are actually a bag of *typed objects* (spec tables, wiring diagrams, procedures, warnings) glued together by layout position, not by semantic proximity. So the fix isn't a better chunker — it's inserting a **normalization layer** between raw extraction and retrieval. Everything below is built around that one decision.

Four layers, each independently testable and independently scalable:

```
L0  Ingestion & Structure Extraction   (PDF → typed, hierarchical objects)
L1  Canonical Knowledge Model          (typed objects → normalized entities)
L2  Retrieval Substrate                (entities → indexes: vector + keyword + graph + metadata)
L3  Agentic Reasoning Layer            (query → plan → iterative retrieval → grounded answer)
```

Tenant isolation is a cross-cutting concern applied identically at all four layers (Section 5), not a separate module.

---

## 1. L0 — Ingestion & Document Structure Extraction

### 1.1 Why layout-preserving parsing, not text extraction

Standard PDF text extraction (pdfplumber, PyPDF) gives you a stream of characters with x/y coordinates at best. You need a **layout-aware parser** that outputs typed blocks with bounding boxes and reading order, so relationships can be reconstructed geometrically before anything is thrown away.

**Recommended stack:**
- **Docling** (IBM, open-source) or **Unstructured.io** for layout detection + block typing (heading/paragraph/table/figure/caption). Docling has stronger table structure recovery (TableFormer) which matters a lot here — spec tables are your highest-value content.
- **LayoutLMv3 / Donut** as a fallback/secondary pass for scanned or image-heavy pages (wiring diagrams, CAD exports) where Docling's layout model is uncertain.
- **Camelot / img2table** as a table-specific second opinion when Docling's confidence on a table is low — run both, reconcile.
- Vision-capable LLM (Claude) as a **verification and repair pass**, not primary extraction — cheaper and more reliable to use it to check/caption/repair than to extract from scratch on every page.

### 1.2 Reconstructing relationships (the core technical challenge)

This is answered with **three signals combined**, not one:

1. **Geometric proximity + reading order** — a table directly below a heading with no intervening block, on the same page, is provisionally owned by that heading.
2. **Explicit textual references** — "see Table 3", "Figure 2 shows...", "refer to Section 4.2" are extracted via regex + NER and used to *override* geometric guesses when they conflict (explicit reference always wins).
3. **Hierarchical containment** — every block gets a `parent_id` forming a tree (Document → Chapter → Section → Block), built from heading level detection (font size/weight/numbering patterns like "4.2.1"), not from page boundaries. This is what lets a table on page 5 still belong to the section that started on page 3.

Store this as an explicit graph, not implicit ordering:

```json
{
  "block_id": "b_0091",
  "type": "table",
  "page": 5,
  "bbox": [x0,y0,x1,y1],
  "parent_section_id": "s_003",
  "explicit_refs_to": [],
  "explicit_refs_from": ["b_0088"],
  "raw_content": {...},
  "confidence": 0.94
}
```

Cross-page continuation (a table split across pages 5–6) is detected via matching column headers/structure on the next page's first block and merged before this record is finalized.

### 1.3 Output of L0

A **Document Object Model (DOM)** per source file: a tree of typed blocks with bounding boxes, parent/child links, explicit cross-references, and page provenance. This is stored as-is (e.g., in Postgres as JSONB, or S3 as JSON) — it is the traceability backbone. Every downstream entity keeps a pointer back into this DOM (`source_block_ids: [...]`), which is how you get citations later without re-deriving them.

---

## 2. L1 — Canonical Knowledge Model

### 2.1 The normalization problem

"Supply Voltage" / "Operating Voltage" / "Power Input" all meaning the same attribute is solved with a **two-tier approach**, because pure LLM normalization at scale is slow/expensive and pure dictionary mapping doesn't generalize to new manufacturers:

1. **Canonical attribute registry** — you maintain a schema of canonical attributes (`supply_voltage`, `ip_rating`, `operating_temp_range`, ...) each with a list of known synonyms/aliases, per attribute, growing over time.
2. **LLM-assisted mapping with human-in-the-loop for new terms** — when an extracted attribute label doesn't match any known alias (embedding similarity below threshold against the canonical registry), an LLM proposes a mapping with confidence + reasoning; above a confidence bar it's auto-accepted and the alias is added to the registry (so the system gets cheaper over time); below the bar it's queued for a human reviewer (you, or a tenant admin) to confirm once.

This turns "normalization" into a **self-improving lookup table with an LLM escape hatch**, not a per-document LLM call for every field forever.

### 2.2 Canonical entity schema

```
Product
  ├─ id, manufacturer, model_number, category, status (active/discontinued)
  ├─ specifications: [{ canonical_attribute, value, unit, source_block_id }]
  ├─ compatible_accessories: [product_id]
  ├─ replacement_for / replaced_by: [product_id]
Installation
  ├─ product_id, steps: [{ order, instruction_text, warning_ids, figure_ids, source_block_id }]
Troubleshooting
  ├─ product_id, symptom, possible_causes, diagnostic_steps, resolution, error_codes: [code]
Warning / SafetyNote
  ├─ severity, text, applies_to_block_ids
Figure (wiring diagram / CAD / flowchart)
  ├─ image_ref, caption, referenced_by_block_ids, extracted_labels (via vision model)
FAQ
  ├─ question, answer, related_product_ids
```

Every entity carries `source_block_ids` back to the L0 DOM and a `tenant_id`. This schema is manufacturer-agnostic by construction — the registry absorbs vocabulary differences; the schema itself never changes per tenant.

### 2.3 Entity construction pipeline

For each L0 DOM, walk the tree section by section and classify each section against the canonical entity types (a section titled "Installation" or one whose content pattern matches numbered steps + warnings → `Installation` entity type). This is a classification + extraction task, best done with an LLM given the section's block tree as structured input (not flattened text), producing entity JSON with `source_block_ids` populated per field. Validate against a schema (Pydantic/JSON Schema) before persisting; failed validations go to a review queue rather than being silently dropped.

---

## 3. L2 — Retrieval Substrate

Four indexes, queried together, not sequentially-or-separately-chosen:

| Index | Technology | Purpose |
|---|---|---|
| Vector | Qdrant or Weaviate (both support per-tenant namespace + payload filtering natively; pick Qdrant if you want simpler self-hosting/cost, Weaviate if you want built-in hybrid search) | Semantic search over entity text (spec descriptions, troubleshooting text, FAQ) |
| Keyword/BM25 | Same DB's hybrid mode (Qdrant/Weaviate both support this) or OpenSearch | Exact matches: model numbers, error codes, part numbers — semantic search is bad at these |
| Metadata | Postgres | Structured filters: product category, manufacturer, status, date — used to pre-filter before vector search |
| Knowledge Graph | Neo4j (justified — see below) | Relationship traversal: "compatible accessories", "replacement chain", "which warning applies to which step" |

**Is a graph DB justified?** Yes, but scoped narrowly: only for the *relationship* queries (compatibility, replacement chains, cross-references), not as a general retrieval store. Most queries are answered by vector+keyword+metadata; the graph gets invoked specifically when the agent's plan includes a relationship hop ("suggest a replacement", "what accessories work with this"). Don't over-invest here early — you can start with relationship data as foreign keys in Postgres and promote to Neo4j when traversal queries (multi-hop compatibility chains) get slow or complex enough to need Cypher.

**Retrieval strategy per query:**
1. Metadata filter first (tenant_id always; product/category if identifiable from query) — this scopes the search space before anything expensive runs.
2. Hybrid vector+keyword search within that scope.
3. If the query implies a relationship ("compatible with", "replacement for", "similar to") → graph traversal, results merged with step 2's results.
4. Re-rank combined results using a cross-encoder (e.g., Cohere rerank or a local BGE reranker) before handing to the agent — this materially improves precision over raw vector similarity ranking.

---

## 4. L3 — Agentic Reasoning Layer

### 4.1 Orchestration framework

**LangGraph** is the right fit over plain function-calling loops or CrewAI/AutoGen, because this workflow is a **state machine with conditional branches and loops** (identify product → retrieve → check sufficiency → maybe ask clarifying question → retrieve again), which is exactly what LangGraph models as a graph with explicit state, not an implicit agent loop. It also gives you checkpointing (resume a conversation mid-reasoning) which matters for the "ask follow-up questions" requirement — the conversation needs to pause for a human reply and resume with state intact.

### 4.2 Reasoning graph (concrete nodes)

```
[Identify Product/Context] → [Plan Retrieval] → [Retrieve (parallel: vector/keyword/graph)]
        → [Assess Sufficiency] ──(insufficient)──> [Generate Clarifying Question] → (wait for user) → back to Retrieve
        → [Assess Sufficiency] ──(sufficient)────> [Generate Grounded Answer + Citations] → [Suggest Next Steps]
```

- **Identify Product/Context**: resolve model number / product mentioned, using conversation history + metadata index. If ambiguous, this itself can trigger a clarifying question before retrieval even starts.
- **Assess Sufficiency**: an explicit LLM-judged check — "given retrieved evidence, can I answer confidently and specifically, or is there a gap?" — not just "did retrieval return >0 results." This is the node that makes retrieval iterative instead of single-shot.
- **Generate Grounded Answer**: answer is constructed with mandatory citation of `source_block_ids` → resolved to page/section references shown to the user. Refuse to state a fact not traceable to a retrieved entity.

### 4.3 Guardrails specific to this domain

- Safety/warning entities attached to a procedure are **always surfaced**, even if not directly asked for — an agent answering "how do I install X" must include applicable warnings, not just steps, regardless of retrieval ranking. Enforce this as a hard rule in the answer-generation node (check for attached `warning_ids` on any Installation/Troubleshooting entity used), not as something the LLM "should remember."
- Discontinued products: if `Product.status == discontinued`, the agent must proactively mention it and surface `replaced_by` before answering the original question.

---

## 5. Multi-Tenant Isolation Strategy

Applied identically at every layer — the rule is **tenant_id is a mandatory, non-optional filter at every read and write, enforced at the data layer, not the application layer**, so a bug in agent code can't leak cross-tenant data.

| Layer | Isolation mechanism |
|---|---|
| Object storage (raw PDFs) | S3 prefix per tenant + bucket policy, or separate buckets for enterprise tenants needing hard isolation |
| Postgres (DOM, entities, metadata) | Row-level security (RLS) policies keyed on `tenant_id`, not just an app-level WHERE clause — this is the difference between "isolation by convention" and "isolation enforced by the database" |
| Vector DB | Native per-tenant collection/namespace (Qdrant collections, Weaviate multi-tenancy feature — both support this directly) rather than a shared collection with a metadata filter, for hard isolation and independent scaling/deletion |
| Graph DB (Neo4j) | Separate database per tenant (Neo4j supports multi-database in Enterprise) for larger tenants; label-based partitioning + query-time filtering for smaller ones |
| Conversation/agent memory | Scoped by `tenant_id` + `user_id`, stored in Postgres or Redis, never shared across the LangGraph checkpoint namespace |
| LLM calls | No cross-tenant context ever placed in the same prompt — even for internal analytics/benchmarking, aggregate only, never raw content across tenants |

For compliance-sensitive tenants, offer **dedicated infrastructure tier** (separate vector DB instance, separate Postgres schema or instance) vs. the default shared-infra-with-hard-partitioning tier — this is a pricing/tiering lever, not just a technical one.

---

## 6. Validation & QA Mechanisms

This is where most RAG platforms are weakest and where you can differentiate:

1. **Extraction confidence scoring** at L0 (per block) and L1 (per entity/field) — anything below threshold routes to a review queue instead of silently entering the retrieval index.
2. **Golden dataset regression testing** — maintain a small set of manually-verified Q&A pairs per document type (spec lookup, troubleshooting, installation) and run them against the pipeline on every ingestion pipeline change to catch silent quality regressions.
3. **Citation verifiability check** — automated check that every claim in a generated answer maps to a real `source_block_id` that actually supports it (can be done with a cheap LLM-as-judge pass comparing answer sentence to cited source text) — this catches hallucinated citations, which are worse than no citations.
4. **Canonical attribute registry drift monitoring** — track how often new aliases get auto-accepted vs. queued for review; a spike signals either a new manufacturer's terminology needs attention or the confidence threshold needs tuning.

---

## 7. Recommended Technology Stack (summary)

| Concern | Choice | Why |
|---|---|---|
| Layout extraction | Docling (primary), Unstructured.io (fallback) | Best open-source table structure recovery |
| Orchestration (ingestion) | Temporal or Prefect | Long-running, retryable, resumable pipelines — ingestion jobs are exactly the failure-prone, multi-step workloads these are built for |
| Entity extraction LLM | Claude (Sonnet-tier for volume, escalate to a stronger tier for low-confidence cases) | Strong structured extraction + long context for whole-DOM reasoning |
| Metadata/relational store | Postgres (+ RLS) | Mature, RLS gives real enforced isolation |
| Vector DB | Qdrant or Weaviate | Native multi-tenancy, hybrid search built in |
| Graph DB | Neo4j | Only once relationship queries justify it (see §3) |
| Agent orchestration | LangGraph | State-machine model fits iterative, clarifying-question-driven retrieval |
| Reranking | Cohere Rerank or self-hosted BGE reranker | Meaningful precision gain over raw vector similarity |

---

## 8. Scalability Considerations

- **Ingestion is the bottleneck, not retrieval** — layout parsing and LLM-based entity extraction are the expensive steps. Design ingestion as an async, queue-based pipeline (per-document jobs, parallelizable across documents, checkpointed per stage) so a single large/malformed document doesn't block a tenant's whole corpus.
- **Cache aggressively at the canonical attribute registry level** — this is shared read-heavy, rarely-written data; cache in-memory/Redis and only hit Postgres on registry misses.
- **Retrieval scales horizontally trivially** (stateless queries against sharded vector/keyword indexes); the harder scaling problem is the **agentic reasoning loop's LLM cost** for multi-step iterative retrieval — track cost per conversation and set a hard cap on retrieval iterations (e.g., 3) before forcing an answer or an escalation to a human.

---

## 9. Key Trade-offs to Be Explicit About

- **LLM-in-the-loop entity extraction is expensive and slower than pure rule-based extraction**, but rule-based approaches cannot generalize across manufacturers — this is a deliberate cost/generalization trade-off, mitigated by the self-improving alias registry reducing LLM calls over time.
- **Per-tenant vector namespace vs. shared collection with filtering**: namespace-per-tenant costs more at small scale (many small collections) but is simpler to reason about for isolation and easier to delete-on-offboarding cleanly. Recommend starting here and only moving to shared-with-filtering if operational overhead of many collections becomes the actual bottleneck (unlikely before hundreds of tenants).
- **Graph DB adds operational complexity** — don't introduce Neo4j until relationship-traversal queries are demonstrably underserved by foreign-key joins in Postgres.

---

## 10. Future Enhancements (not in initial scope)

- ERP/inventory API integration for real-time stock/pricing on top of the Product entity (the schema already has the right join key: `model_number`).
- Predictive maintenance: requires time-series sensor data ingestion, out of scope for document-based knowledge but the Troubleshooting entity schema can extend to accept structured sensor thresholds later.
- Product recommendation: the compatibility/replacement graph in Neo4j is already the substrate for this — a recommendation feature is mostly a new query pattern over existing data, not new infrastructure.

---

## Summary

The single architectural decision that makes everything else work: **don't retrieve from documents — retrieve from a canonical knowledge model that was built from documents.** L0 (structure) and L1 (canonical entities) are where nearly all the engineering difficulty and differentiation lives; L2 (retrieval) and L3 (agent) are comparatively standard once L0/L1 are solid. Prioritize build order accordingly — a mediocre agent over a great knowledge model beats a great agent over raw PDF chunks, every time.
