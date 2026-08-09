# D2 — Tika-only for Phase 1 (Docling deferred to Phase 2)

> **Audience:** Engineers, architects.
> **Purpose:** Record why Apache Tika is the sole document parser for the MVP, and why IBM Docling is explicitly deferred to Phase 2 for table and layout-heavy PDFs.

---

## Context

The StashRoom ingestion pipeline reads venue documents (PDF decks, floor plans, spec sheets, CAD files) and extracts structured metadata (capacity, catering policy, restrictions, etc.). Two production-grade document readers are available in the Java/self-hosted ecosystem:

- **Apache Tika** — battle-tested since 2007, ships as a first-class Spring AI `TikaDocumentReader`, supports 1000+ formats including DWG/AutoCAD, has forked-JVM isolation via Tika Pipes.
- **IBM Docling** — newer layout-aware parser with state-of-the-art table extraction and multi-column PDF reconstruction. Runs as a self-hosted Docker container (MIT license, no per-page cost).

The question is which parsers to ship in Phase 1 (MVP) and which to defer.

---

## Options considered

### Option A — Docling-first for all PDFs in Phase 1

All PDF assets route through a Docling sidecar container. Tika is used only for non-PDF formats (DOCX, XLSX, DWG) where Docling has no support.

**Pros:**

- Best-in-class table extraction for capacity configuration tables (banquet/theater/classroom) — the single most valuable structured field in venue decks
- Layout-aware chunking preserves multi-column reading order; Tika mangles two-column marketing decks
- TableFormer model reconstructs cell relationships; downstream GPT-4o extraction receives cleaner structured input

**Cons:**

- One new infrastructure dependency in MVP: Docling container must be deployed, monitored, and scaled separately from the ingestion worker
- Docling supports ~6 document formats vs. Tika's 1000+; a non-trivial amount of format-handling code still needs Tika as a fallback
- MVP scope creep: Docling HTTP client, retry/backoff against the sidecar, health checks, and per-tenant resource quotas for Docling throughput are all new code paths with no existing foundation code
- Spring AI has no first-class `DoclingDocumentReader` today; a custom implementation must be written and maintained
- For 60–70% of real-world venue PDFs (single-column, large tables with clear headers, no overlapping text boxes), Tika's plain text output + GPT-4o's ability to reason about tabular text produces acceptable extraction accuracy

### Option B — Tika-only for Phase 1; Docling in Phase 2

All document parsing in MVP goes through Apache Tika. The pipeline explicitly handles the known Tika quality gap by:

- Increasing GPT-4o token context window for extraction calls so more surrounding text is included per chunk
- Lowering confidence thresholds on capacity/catering fields that typically live in tables (more fields surface to the human-verification UI rather than being auto-accepted)
- Flagging asset-type `SPEC_SHEET` and `FLOOR_PLAN` for mandatory human review (skip auto-confirm on those types)

Docling is designed-in as a Phase 2 capability. The pipeline uses a pluggable `DocumentReader` strategy interface; a Docling reader is wired in Phase 2 without touching the extraction, embedding, or aggregation stages. A container deployment manifest is prepared but not enabled in staging/production.

**Pros:**

- Zero new infrastructure dependencies in MVP; reduces deployment surface, monitoring, and incident-response complexity for the first paying customers
- Spring AI ships `TikaDocumentReader` out of the box; zero custom parser code, zero client libraries to maintain
- Tika Pipes forked-JVM isolation is production-proven; a malformed or malicious PDF crashes a forked JVM, not the ingestion worker
- DWG/AutoCAD support exists today in Tika — a rare capability that Docling does not provide (CAD venue files appear in ~10% of agency uploads based on pre-MVP file inventory)
- Human-in-the-loop verification (split-screen confirm/correct UI) catches the gap: a human reviews every extracted field anyway in MVP, so parser quality gaps are corrected before they reach search or pitch generation

**Cons:**

- Table-heavy venue decks produce lower extraction accuracy in Phase 1; the human reviewer must correct capacity numbers more often for the first 2–3 months
- Marketing PDFs with dense two-column layouts produce mangled reading order; GPT-4o must disambiguate messy input, occasionally yielding wrong values with mis-attributed confidence scores
- Phase 2 switch to Docling for a subset of PDFs requires re-extraction of historical assets; a one-shot re-extraction job must be written and run

---

## Decision made

**Option B: Tika-only for Phase 1. Docling is explicitly designed-in as a Phase 2 DocumentReader implementation** with a reserved container slot in the docker-compose topology, but no MVP customer traffic routes through it.

---

## Rationale

- **MVP priority is correctness via human review, not full automation.** The split-screen human-verification flow is a first-class feature, not a backup. Every extracted field in MVP is confirmed or corrected by a human before being persisted as verified. Parser quality gaps are caught in the UI before they corrupt search or pitch output. Shipping Docling for MVP does not eliminate human review — it merely shifts the human's effort from correcting fields to reviewing near-perfect fields. The incremental user-visible quality gain does not justify the incremental infrastructure and code complexity for launch.
- **Tika's DWG support is irreplaceable in the short term.** Agencies regularly upload AutoCAD files for floor plans and technical layouts. Docling has no CAD parsing. A Tika-based MVP handles every asset type agencies upload today without a "format not supported" error path. A Docling-first MVP would still need Tika alongside it for CAD/DWG support — yielding both parsers in MVP with zero complexity saved.
- **Spring AI integration maturity matters for launch stability.** `TikaDocumentReader` is a maintained, tested first-class component with Pipes configuration, timeout controls, and memory caps exposed as simple builder properties. A custom Docling reader would need all of that implemented and battle-tested before launch — several weeks of work that is orthogonal to the venue-domain product.
- **Phase 2 Docling integration is architecturally free.** The `AssetExtractionOrchestrator` in `stashroom-data-intelligence` accepts any `DocumentReader` via Spring DI. Adding a Docling reader in Phase 2 is a `@Bean` registration, a content-type routing predicate (use Docling for `application/pdf` when `asset_type = FLOOR_PLAN | SPEC_SHEET`; fall back to Tika otherwise), and a container in the manifest. No extraction, embedding, or aggregation code changes. The one-shot re-extraction job for historical PDFs is the same job infrastructure we already need for extraction-prompt version bumps.

---

## Consequences

- MVP accuracy benchmarking (§12 of architecture.md) calibrates expected accuracy using Tika-only extraction on a 50-PDF real-world corpus. Marketing materials and product documentation do not promise table-level accuracy that Tika cannot deliver.
- The human-verification UI marks capacity.configuration fields and catering.policy fields with a "Table source — please verify" hint when the asset is a multi-page PDF. This compensates for Tika's known table-mangling weakness in the interim.
- A backlog item exists for Phase 2: Docling sidecar deployment, custom DocumentReader implementation, content-type routing, and the historical re-extraction job. It is tracked under the Phase 2 milestone.
- When Docling is enabled, confidence scores for table-derived fields must be recalibrated against the new parser output. Existing confidence thresholds for auto-accept vs. human-review are explicitly parser-versioned.

---

## Status

**Accepted.** Phase 1 (MVP) ships with Tika only. Docling integration designed-in for Phase 2.

---

**Docs:** [Vision](../vision.md) · [Intelligence Layer](../../platform/intelligence.md) · [Architecture](../../platform/architecture.md) · [Epics](../epics/README.md)
