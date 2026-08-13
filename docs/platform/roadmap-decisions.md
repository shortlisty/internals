# VenueMi — Open Decisions & Roadmap

> **Audience:** Engineers, product, founders.
> **Purpose:** All architecture decisions (resolved and pending), pre-Sprint 1 implementation tasks, and Phase 2/3 design backlog.

---

## Related Documents

- [architecture-overview.md](architecture-overview.md) — platform context and stack
- [data-model.md](data-model.md) — schema versioning contract (§2a)
- [aggregation.md](aggregation.md) — FIFO metadata aggregation (§3)
- [master-catalog.md](master-catalog.md) — MC_INHERIT merge algorithm and threshold calibration
- [search.md](search.md) — cross-source search architecture (§6)
- [services.md](services.md) — shared library split (`mi-data-intelligence` + `mi-venue-model`)
- [observability.md](observability.md) — metrics referenced in Phase 2 triggers

---

## 15. Open Decisions

### Resolved

- [x] **One service or two?** Decided: Two deployments — `mi-venue-service` (synchronous API, data-tied) and `mi-venue-processing-worker` (async sidecar, shared schema, no inbound HTTP). Full design in [services.md](services.md).

- [x] **Naming convention.** Service names reflect domain/purpose, not implementation technology. `mi-venue-processing-worker` describes what it does (ingest and process assets), not how (AI/ML).

- [x] **Master Venue Catalog (MC).** A `public.master_venue` table (with `master_venue_alias`, `master_venue_external` tables) serves as invisible backdrop for gap-filling tenant venue data at extraction time. Copy-on-match (MC_INHERIT provenance, priority 7), not link — tenant record is fully independent after copy. No reverse flow from tenant to master catalog in MVP. Full design in [master-catalog.md](master-catalog.md).

- [x] **Metadata schema versioning and JSONB drift.** Every `venues.metadata` and `public.master_venue.metadata` JSONB document carries a top-level integer `_schema_version` (initial: 1; absent = 0 "legacy"). `mi-data-intelligence` contains the `MetadataMigration` interface and `MetadataMigrator` chain runner. `mi-venue-model` contains the venue-specific migration classes, `VenueMetadataMigrator`, and `VenueMetadataSchemaVersion.CURRENT_SCHEMA_VERSION`. Every read upgrades the shape in memory; every write stamps the document to `CURRENT_SCHEMA_VERSION` before persist. Full design in [data-model.md § 2a](data-model.md).

- [x] **Metadata aggregation race condition prevention.** RabbitMQ FIFO routing per `venue_id` using hash-partitioned queues. Eliminates the race at the messaging layer before the consumer runs. Zero new dependencies, no retry code. Variant A1 (single queue, concurrency=1, prefetch=1) for MVP; variant A2 (16 hash-slot queues) when throughput requires. Full design in [aggregation.md](aggregation.md).

- [x] **Cross-source search architecture (tenant venues + public `MasterVenue` across schema boundary).** Two parallel SQL branches → app-level merge, no PostgreSQL cross-schema JOIN/UNION in a single statement. Default scope is `TENANT_ONLY` (branch B: master catalog backdrop merges invisibly via field-level MC_INHERIT provenance, not as separate visible rows). Full design in [search.md](search.md).

- [x] **Master Catalog match threshold and algorithm.** Fuzzy name + PostGIS proximity, no embeddings/LLM in the match path. Combined confidence formula, thresholds 0.75 with geo / 0.90 name-only, ambiguity delta guard ≥ 0.08. FP rate ≤ 1% acceptance criterion before Sprint 1. Full design in [master-catalog.md](master-catalog.md).

- [x] **Chunking table placement and naming.** One `item_vectors` table per tenant schema (defined in `mi-data-intelligence` changelog). Table name `item_vectors` — generic, not `vector_store`. IVFFlat index at MVP volumes. Full design in [data-model.md § 10](data-model.md).

- [x] **Shared library split: `mi-data-intelligence` + `mi-venue-model`.** Generic extraction pipeline contracts, event POJOs, metadata versioning mechanism, and provenance model live in `mi-data-intelligence`. Venue canonical field set and migrations live in `mi-venue-model`. Full design in [services.md](services.md).

### Pending

- [ ] **Docling in Phase 1?** Start with pure Tika (simpler). Add Docling sidecar in Phase 2 when floor plan / table fidelity is needed. **Lean: Tika-only for Phase 1.**

- [ ] **Cost tracking granularity:** per-asset or per-tenant-per-month? Both are in schema; decide which is surfaced in UI.

- [ ] **Old migration class retirement policy.** After how many consecutive months of zero hits on the `schema_version < N` Prometheus counter do we delete the oldest migration classes from the chain? Define the guardrail before the first schema bump.

---

## 17. Next Steps & Design Iterations

### Before Sprint 1 — Resolve These First

- [ ] **Implement `VenueMetadataMigrator` v0 + v1 chain** in `mi-venue-model` before any service code reads or writes `venues.metadata`. Wire `VenueMetadataTypeHandler` into the `VenueResultMap`. Add Micrometer counter inside `migrateToCurrent()`. Add JUnit tests for `MetadataMigrationV0ToV1` with 5+ fixture JSON shapes (empty `{}`, `{}` without `_schema_version`, full v1 shape, partial v1 shape, master-catalog-copy shape) to cover legacy-bootstrapping edge cases.

- [ ] **Implement metadata aggregation FIFO routing (A1 for MVP)** in `mi-venue-service` `MetadataAggregationConsumer`: configure `@RabbitListener` on `venuemi.metadata.aggregation` with `concurrency=1`, `prefetchCount=1`, `acknowledgeMode=MANUAL`. Wrap the full SELECT → debounce check → merge via `VenueMetadataMigrator` → UPDATE → `venue_metadata_events` consume cycle in a single `@Transactional` DB transaction. Ack the RabbitMQ message only after the transaction commits. Add integration test: publish three `extraction.completed` events for the same `venue_id` in quick succession, consume, assert final `venues.metadata` contains merged fields from all three sources AND exactly one SQL `UPDATE` was executed. When queue depth metrics show sustained backlog > 1 s, promote from A1 to A2.

- [ ] **Implement `MasterVenueMatcher` + dry-run threshold calibration.** Code: `MasterVenueMatcher` class in `mi-venue-processing-worker` — pure PostgreSQL `pg_trgm` + PostGIS, shared `normalize()` 6-step function, fetch top-5 trigram candidates via GIN index, PostGIS `ST_DWithin` 200m radius, combined confidence formula, MATCH thresholds 0.75 with geo / 0.90 name-only, ambiguity guard delta ≥ 0.08, field copy only for leaf keys that are null on tenant side + MC_INHERIT provenance tag (priority 7). Populate `venues.master_venue_id` on unambiguous MATCH. Micrometer counters matched/ambiguous/no_match + confidence histogram. Unit tests: 10+ synthetic pairs. Dry-run calibration: 50 real PDFs with ground truth, confusion matrix, acceptance FP-rate ≤ 1%.

- [ ] **Implement cross-source search path (`?scope` + RRF merge + master catalog detail + from-master-catalog import).** Code in `mi-venue-service`: (1) new `scope` enum param on `GET /api/v1/venues/`; (2) `VenueSearchOrchestrator` with `CompletableFuture` parallel branch A (tenant mapper, full 5 modes) + branch B (`MasterVenueQueryMapper` explicitly qualified `public.master_venue … LEFT JOIN public.master_venue_alias`, 3 MVP modes: keyword + structured + geo only). Branch timeout 2s on master catalog backdrop. (3) App-level merge: dedup by `master_venue_id` → drop separate `MASTER_CATALOG` row if already imported → Reciprocal Rank Fusion equal weights 0.5:0.5 → top-50 each → slice by page/size. (4) Failure isolation: branch B exception/timeout → return A only + `Warning` header + Micrometer counter. (5) DTOs: add `master_venue_id` nullable UUID to `VenueResponse`, add `origin` to `VenueSummaryView`. (6) `GET /api/v1/master-catalog/entries/{id}` (MEMBER authority, fixed column safe projection). (7) `POST /api/v1/venues/from-master-catalog/{masterVenueId}` (MEMBER): idempotent — if venue with `master_venue_id` exists → return existing. (8) Unit tests: 8 scenarios (BOTH scopes dedup imported, BOTH scopes dedup not imported, `MASTER_CATALOG_ONLY` scope returns only MC origin, `TENANT_ONLY` scope never queries `MasterVenueQueryMapper`, RRF interleaving order, Warning header on branch B timeout, graceful branch B degradation, from-master-catalog POST idempotent on second call). (9) Enforce cross-schema rule at code review: no other MyBatis mapper emits `public.` SQL except three permitted mappers; cross-schema JOIN/UNION forbidden.

---

### Phase 2 Design (post-MVP signal)

- **Conflict resolution UX** — API shape + state machine for resolving competing extracted values
- **Bulk import / CSV ingestion** — for concierge onboarding and agency migrations
- **Saved searches + alerts** — schema and delivery mechanism
- **Notification delivery** — decide WebSocket vs. polling for extraction status (can reuse IAM's WebSocket infra)
- **CAD visual extraction** — convert DWG/DXF to image, then GPT-4o vision
- **Video walkthroughs** — keyframe extraction via ffmpeg, vision-based amenity detection
- **Venue groups** — `venue_groups` and `venue_group_members` tables, tenant-owned library organisation by city / event type / client / season. Tree/folder navigation in the tenant app. No impact on search or extraction.
- **Master Catalog admin API** — internal platform endpoint for bulk-importing seed data, managing master catalog entries, reviewing match quality. `PLATFORM_ADMIN` authority only.
- **pgvector index strategy: IVFFlat → HNSW evaluation.** When any single tenant's `item_vectors` count exceeds 1M rows (Micrometer gauge `venuemi_item_vectors_rows_total{tenant_id}`), schedule a maintenance window to benchmark HNSW on that tenant's data. HNSW delivers better recall/performance at high volumes but has higher build cost and write amplification; only promote tenants that actually breach the size threshold.
- **Master Catalog semantic search + unified search index evaluation.** Two triggers to revisit search: (1) `venuemi_search_latency_seconds{branch="tenant"}` p99 > 500ms sustained for 10 min and per-tenant venues count > 10K, OR (2) Master Catalog admin wants semantic on master catalog entries enabled. Trigger path: introduce `master_venue.metadata.description_embedding` VECTOR(1536) in `public` schema, generate embeddings during scraper/admin apply, add cosine branch to `MasterVenueQueryMapper`. If either branch's p99 > 500ms or `venuemi_venue_import_from_master_catalog_total{status=created}` is above 10 per user per week, consider a dedicated shared OpenSearch cluster. For MVP: PostgreSQL stays search engine of record.
- **Sweep orphan `venues.master_venue_id` values after `master_catalog.admin.entry.deleted`.** When Master Catalog Admin deletes or archives an entry, run a per-tenant sweeping job to set `master_venue_id = NULL` on affected tenant venue rows. Deferred because seed is small in MVP and deletions are rare. Log with counter `venuemi_master_catalog_entry_sweep_total{outcome=updated|no_match|error}`.

---

### Phase 3 Design

- **Export / sharing** — shareable branded links, PDF/Excel report generation, venue comparison view
- **Deduplication** — detect and merge duplicate venue records across teams
- **Verification workflow** — API + UX for promoting AI-extracted fields to "human-verified" status
- **CRM integrations** — Salesforce, HubSpot webhook connectors

---

### Cross-Cutting Concerns (design before Phase 2 builds)

- **AI resilience** — fallback behavior when OpenAI is unavailable; graceful degradation (queue for retry, notify user)
- **Per-tenant AI cost controls** — budget caps, monthly usage alerts, what happens at limit
- **Storage / retention policy** — S3 lifecycle rules, old embedding versions, extracted text retention (GDPR angle)
- **Testing strategy** — test doubles for OpenAI, fixture documents for ETL pipeline, contract tests between services

---

### Strategic Bets to Validate with First Users

- Is semantic search ("find venues like this one") actually used, or do planners prefer structured filters?
- What is the real extraction accuracy on real-world venue PDFs? Run benchmark on 50 sample documents before committing to accuracy claims.
- Is the "aha moment" the extraction result, or the search finding something instantly? Shapes onboarding flow design.
- Which asset type matters most to upload first — venue decks or floor plans? Informs parser priority.

---

**Docs:** [Architecture Index](README.md) · [Architecture Overview](architecture-overview.md) · [Data Model](data-model.md) · [Services](services.md) · [Aggregation](aggregation.md) · [Master Catalog](master-catalog.md) · [ETL Pipeline](etl-pipeline.md) · [Search](search.md) · [API](api.md) · [Events](events.md) · [Observability](observability.md)
