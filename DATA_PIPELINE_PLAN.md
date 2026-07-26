# Implementation Plan — Data Pipeline Only, 8 Weeks

## Scope change from previous plan

Dropping the agent (L3) and UI entirely changes the calculus: you now have 8 weeks for what was previously squeezed into weeks 1-2. That's not "less work," it's "same core work, done properly" — the things that got cut or manually-hacked in the MVP plan (self-improving alias registry, confidence-based review routing, multi-document-type robustness, real validation tooling) are now in scope because they're the entire deliverable, not a means to an agent demo.

**Deliverable at week 8:** A pipeline that takes raw industrial PDFs from any tenant, in any of your target document types, and reliably outputs validated, normalized, traceable canonical entities — with measured accuracy, a working review workflow for low-confidence extractions, and indexes ready for a retrieval layer to consume (even though you're not building that layer now). This is a sellable/demoable artifact on its own: "upload messy PDFs, get structured, cited, queryable knowledge out."

---

## Week 1 — Foundations + Layout Extraction

- Repo, Postgres schema (DOM storage, entity tables, registry tables) — `tenant_id` on everything from migration #1.
- Docling integration for layout detection (heading/paragraph/table/figure/caption/warning types + bbox + page + reading order).
- Test against a deliberately varied sample set: 5-6 documents spanning your target types (catalog, manual, datasheet, wiring diagram, troubleshooting guide, SOP) from at least 2 different manufacturers — variety matters more than volume right now, since layout inconsistency is the whole problem you're solving.
- Table-specific fallback: run Camelot or img2table alongside Docling on table blocks, reconcile when they disagree (flag disagreements for manual check this week rather than auto-resolving — you need to see real disagreement patterns before automating resolution).

**Deliverable:** Raw layout extraction working across 5-6 structurally distinct documents, with a visible reconciliation log for tables.

---

## Week 2 — DOM Construction (Hierarchy + Relationships)

- Heading-level detection → parent/child tree construction (numbering pattern matching like "4.2.1", font-size/weight heuristics).
- Explicit cross-reference extraction ("see Table 3", "Figure 2 shows", "refer to Section 4.2") via regex + light NER, stored as `explicit_refs_to`/`explicit_refs_from` on blocks.
- Cross-page table/section continuation detection and merging (matching column headers or heading continuation across a page break).
- Conflict resolution rule implementation: explicit reference overrides geometric/proximity guess when they disagree — write this as an explicit, testable function, not an ad hoc heuristic, since it's the answer to the hardest problem in the whole system.

**Deliverable:** Full DOM tree per document — hierarchy, explicit refs, cross-page merges — persisted and inspectable (a simple script that prints/visualizes the tree structure for manual review is worth building here, since you'll be staring at this a lot).

---

## Week 3 — Canonical Schema + Attribute Registry

- Finalize Pydantic entity schemas: Product, Specification, Installation, Troubleshooting, Warning, Figure, FAQ (from the architecture doc — no changes needed, it's already manufacturer-agnostic by design).
- Build the canonical attribute registry properly this time (not hand-seeded and frozen): store as a real table (`canonical_attribute`, `aliases: [text]`, `source_examples`), seed with ~50-70 attributes from your sample set.
- Build the matching pipeline: embedding-similarity lookup against registry first, LLM fallback for unmatched terms, **and this time build the confidence-threshold auto-accept/review-queue split properly** — this was cut from the MVP plan, but you have the time for it now and it's core to the pipeline's long-term value (gets cheaper/better per document processed).
- Review queue as a real (simple) interface — even a basic table view is fine — not a JSON file, since you'll actually be using it iteratively this week and next.

**Deliverable:** Attribute registry with auto-accept/review split working, measurably reducing LLM calls on repeat/similar terms across documents.

---

## Week 4 — Entity Extraction Pipeline

- Section-classification step: walk DOM, classify sections against entity types.
- LLM-based extraction per section (Claude, structured output), given the block subtree (not flattened text) as input.
- Validate every extraction against Pydantic schema; failed validations → review queue, not silent drop.
- `source_block_ids` populated on every entity field — build a quick verification check that these actually resolve back to real blocks (an automated integrity check, since a broken pointer here silently breaks all future citation work).
- Run full pipeline (Week 1 → Week 4) end-to-end on your 5-6 sample documents.

**Deliverable:** Entities extracted, validated, and traceable for your full sample set, first end-to-end pipeline run.

---

## Week 5 — Confidence Scoring + Validation Framework

- Per-block confidence score (Week 1 output already has this from Docling; formalize it into a stored field used downstream).
- Per-entity/per-field confidence score (combination of extraction-step LLM confidence + schema validation pass/fail + source block confidence) — this determines review-queue routing.
- Build the golden dataset now, early, not in week 7 like the MVP plan — since this is the whole deliverable, you want a regression baseline as early as possible to catch pipeline changes that silently degrade quality. 10-15 manually-verified extraction examples (not Q&A pairs this time — "given this document section, these are the correct entities/fields") across your document types.
- Automated citation-integrity check: does every entity's `source_block_ids` actually contain the text/data that supports the extracted field? (LLM-as-judge comparison, cheap and catches hallucinated extraction early.)

**Deliverable:** A measured baseline (e.g., "X% of fields extracted correctly against golden set, Y% correctly routed to review when they should be") — your first real quality number.

---

## Week 6 — Second Manufacturer + Robustness Pass

- Onboard a genuinely different manufacturer's documents (different terminology, different layout conventions, ideally a document type you haven't tested yet — e.g., if week 1 covered catalogs/manuals/datasheets, add SOPs or safety documents now).
- This is your real stress test for the canonical registry: measure how many new terms hit the review queue vs. auto-match against existing aliases from manufacturer #1.
- Fix whatever breaks — layout edge cases, new heading numbering conventions, table structures Docling/Camelot both mishandle, entity types that don't cleanly classify.
- Re-run the golden dataset from Week 5 to confirm no regression from these fixes.

**Deliverable:** Pipeline proven across 2+ manufacturers with measured registry generalization (e.g., "70% of manufacturer #2's attributes auto-matched via existing aliases").

---

## Week 7 — Retrieval-Ready Indexing (Stretch, but High-Value)

Since the pipeline's output needs to actually be consumable by a retrieval layer eventually, spend this week making that trivial for whoever builds it next (you, later, or a collaborator):

- Postgres metadata index (product category, manufacturer, status) with RLS on `tenant_id`.
- Embed entity text and load into a vector DB (Qdrant, per-tenant namespace) — you don't need to build search *logic*, just prove entities are embedded, indexed, and filterable by tenant, which is the last mile of "pipeline output is retrieval-ready."
- Basic relationship data (compatible accessories, replacements) as Postgres FK joins, ready for a graph layer later if needed.

**Deliverable:** Pipeline output isn't just "structured JSON sitting in Postgres" — it's live in queryable, tenant-isolated indexes, one retrieval-layer sprint away from a working agent.

---

## Week 8 — Documentation, Tooling, Buffer

- Buffer for whatever slipped (there will be something — table reconciliation edge cases and heading-hierarchy edge cases are the most likely culprits, same as before).
- Write up the pipeline's measured accuracy numbers, known limitations, and the review-queue workflow as actual documentation — this is your artifact for pitching this as a component/product on its own, separate from the eventual agent.
- Package ingestion as a clean, runnable service (BullMQ job queue, API endpoint to submit a document, status polling) rather than a collection of scripts, since "just the pipeline" as a deliverable implies it needs to be operable by someone who isn't you reading code.

**Deliverable:** A documented, tested, operable data pipeline — the hard 80% of the whole platform — ready to have a retrieval layer and agent built on top of it whenever that's the next phase.

---

## What changed vs. the full-product plan, and why it's better use of the same 8 weeks

The MVP plan spent weeks 1-2 on a *rushed* version of this exact scope, because it also had to leave room for retrieval, agent, and UI. Here, the review-queue/auto-accept split, the golden dataset, the second-manufacturer stress test, and the citation-integrity automation all get built properly instead of stubbed — and those are precisely the things that determine whether this pipeline is actually reusable across real customers later, versus a one-off that only worked on your test documents.
