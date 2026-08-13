# D4 — Cross-source search: parallel queries + app-level RRF merge

> **Audience:** Engineers, architects.
> **Purpose:** Record why tenant venues and the public master venue catalog are queried as two parallel SQL selects merged at the application level via Reciprocal Rank Fusion, rather than a single cross-schema UNION ALL, materialized per-tenant copies, or a dedicated search service.

---

## Context

VenueMi search must return results from two sources that live in different PostgreSQL schemas:

1. **Tenant venues** — `t_{tenantKey}.venues`. Full 5-mode search (keyword, semantic, structured, geo, hybrid). Schema-isolated, writable by the tenant.
2. **Master catalog backdrop** — `public.master_venue`. 3-mode MVP search (keyword, structured, geo). Read-only to tenants, master catalog admin-writable only.

Users expect one search bar returning a merged, deduplicated, paginated result set with origin surfaced only as TENANT; master catalog values merge invisibly into tenant rows as MC_INHERIT provenance (field-level, not row-level). The question is how to combine these two heterogenous sources.

---

## Options considered

### Option A — Single cross-schema SQL with UNION ALL

One SQL statement: `SELECT ... FROM t_acme0001.venues ... UNION ALL SELECT ... FROM public.master_venue ... ORDER BY ... LIMIT 20 OFFSET 0`.

**Pros:**

- One database round-trip.
- Pagination, sorting, and limit are handled by PostgreSQL natively.
- Small application code surface — no merge logic.

**Cons:**

- **Breaks the search_path isolation abstraction.** The venue mapper must explicitly schema-qualify `public.master_venue` inside the same statement that relies on implicit `search_path = t_{tenantKey}` for the tenant branch. A future mapper refactor that schema-qualifies the tenant side (or forgets `search_path` for the master catalog side) creates an administrative risk of reading from the wrong schema.
- **Query planner produces bad plans on heterogenous branches.** The tenant branch may return 0 rows (empty library) while the master catalog branch returns 500. The planner does not know that the two branches have radically different statistics (index selectivities, cardinalities, correlation between predicates) and often picks a merge join or hash aggregate that is catastrophically wrong for one branch.
- **Risk of admin-column leakage.** The `master_venue` table has admin-only columns (`confidence`, `source`, internal notes) that the MEMBER authority must never see. A single `UNION ALL` SELECT list that is widened naively in a later code change can expose admin columns without a second audit.
- **Semantic search asymmetry breaks a single SQL merge.** In MVP, only the tenant branch uses pgvector cosine similarity against `item_vectors` (per-tenant embeddings). The master catalog branch falls back to keyword-only. A `UNION ALL` cannot cleanly express a 5-mode hybrid branch on one side and a 3-mode branch on the other without padding the SELECT list with dummy zero-score columns that confuse the planner.
- **Different index statistics = different page counts.** `EXPLAIN ANALYZE` on a UNION ALL with pgvector on one side and tsvector/PostGIS on the other routinely mis-estimates row counts by 10–100×, leading to nested-loop choices that time out at 30s under realistic load.

### Option B — Materialized master catalog copy per tenant schema

On tenant creation, mirror the full `master_venue` table into `t_{tenantKey}.master_venue_copy` via a periodic refresh. Search runs in one schema: one query against `venues UNION ALL master_venue_copy`.

**Pros:**

- Clean single-schema query path; no cross-schema concerns.
- Master catalog vector embeddings can be added later inside the tenant schema without architectural changes.

**Cons:**

- **Massive write amplification and consistency nightmare.** Master catalog updates (master catalog admin edit, alias add, scraper master catalog import) must fan out to every tenant schema. With 1,000 tenants, one master venue record edit = 1,000 SQL UPDATEs. A bulk import of 10,000 master venue records = 10,000,000 writes. Tenant refresh lag means two users in two tenants searching for the same venue see different versions of the master catalog.
- **Storage bloat.** 500 master venue records × 1,000 tenants = 500,000 redundant copies. Every new index on the master catalog (pg_trgm GIN on aliases, PostGIS GIST on location, future vector index) is re-created 1,000 times.
- **No structural advantage over Option C.** App-level deduplication of "already promoted (copy-on-import)" venues still needs a join or existence check, because a tenant that has MC_INHERIT merged master venue #123 into their venues should see only the TENANT copy, not both. This deduplication is identical to Option C's merge step.

### Option C — Dedicated OpenSearch/Elasticsearch cluster

Index both tenant venues and master venue records into OpenSearch with a `tenant_id + origin` routing key. All search modes (keyword, semantic, structured, geo) run in OpenSearch.

**Pros:**

- Purpose-built search engine with native RRF, BM25 + kNN hybrid queries, and geo distance aggregations.
- One index, one query, one pagination mechanism.

**Cons:**

- **Massive over-engineering at MVP scale.** MVP has <500 master venue records and <1,000 venues per tenant. PostgreSQL keyword + geo + JSONB filters with a 50-row top-K per branch returns in single-digit milliseconds. An OpenSearch cluster adds 3+ nodes, snapshot backups, index management, re-indexing pipelines, TLS/mTLS configuration, and a new observability stack.
- **Cross-system consistency (same class as D3).** Writing to PostgreSQL and then indexing into OpenSearch requires a CDC pipeline (Debezium) or an outbox pattern with a retry/reconciliation loop. Events dropped between the two systems yield permanently stale search results.
- **Tenant isolation is predicate-based, not structural.** Same security-audit concern as D3 Option A — one missing `tenant_id` filter leaks cross-tenant data.

### Option D — Two parallel SQL selects + application-level RRF merge

Issue two independent SQL queries via `CompletableFuture` in parallel:

- **Branch A (tenant):** full 5-mode hybrid search against `t_{tenantKey}` via implicit `search_path`. 50-row top-K with per-branch scores.
- **Branch B: master catalog (backdrop, disabled by default):** 3-mode MVP search against explicitly qualified `public.master_venue`. 50-row top-K with per-branch scores. Only `MasterVenueQueryMapper` may touch `public.` schema.

Merge at application level:

1. Deduplicate by `master_venue_id` (if a tenant venue was promoted from master catalog #123, drop the separate master catalog row and keep only TENANT — master catalog values merge invisibly into tenant rows as MC_INHERIT provenance at field-level).
2. Apply Reciprocal Rank Fusion with equal branch weight (0.5 : 0.5).
3. Slice for pagination (page × size, size).
4. Surface only TENANT origin; master catalog backdrop values merge invisibly with MC_INHERIT provenance (not a visible separate origin bucket) by default.

**Pros:**

- **Zero cross-schema contamination risk.** Two separate mappers run separate statements. The tenant mapper never references `public.`; only `MasterVenueQueryMapper` has explicit `public.` qualification in its XML. Code review enforces this as a single-point rule.
- **Each branch gets its own optimal query plan.** Tenant branch with pgvector + tsvector + JSONB is planned independently from the master catalog branch with pg_trgm + PostGIS. No heterogenous UNION ALL cardinality mis-estimates.
- **Failure isolation for free.** If branch B (master catalog backdrop) times out or raises an exception, search does not fail. Branch A results return with an RFC 7807 `Warning: 299` response header and a metrics counter increment. Users see "Master catalog backdrop merge unavailable, results may be incomplete" rather than a 500.
- **Asymmetric search support is natural.** Master catalog semantic search (Phase 2) requires adding a `public.master_venue_vectors` table and a pgvector path to branch B only. The merge and RRF code does not change; the two branches simply have different score calculation code.
- **No new infrastructure, no CDC, no sync reconciliation.** Both branches read transactionally consistent data from the same PostgreSQL instance they already write to.

**Cons:**

- Two database round-trips instead of one. (Mitigated: parallel execution via `CompletableFuture` means wall-clock latency = max(A, B), not sum(A + B).)
- `totalElements` for `scope=TENANT_ONLY` (Default) pagination is exact for tenant branch; `scope=BOTH` (power-user only) returns approximate (top-50 from each branch merged; true COUNT is unknown without issuing two COUNT queries). UI shows approximate counts ("1K+") and uses "Load more" rather than strict page bars for non-default scopes. This is documented as an intentional UX choice.
- RRF merge must handle ties and rank boundaries explicitly. Small amount of merge code that requires unit tests with synthetic rank collisions.

---

## Decision made

**Option D: two parallel SQL selects + application-level RRF merge.**

Branch A (tenant venues) uses implicit `search_path` with no schema qualification. Branch B: master catalog (backdrop, disabled by default) uses explicit `public.` qualification restricted to `MasterVenueQueryMapper` only. Merge runs in `VenueSearchOrchestrator` via `CompletableFuture.parallel`. Equal RRF branch weight 0.5:0.5 for MVP; branch weights become configurable in Phase 2 if master catalog relevance calibration shows consistent imbalance. Default scope is TENANT_ONLY (BOTH available as power-user explicit scope); Branch B runs as an invisible backdrop merge (not a visible separate origin bucket) by default.

---

## Rationale

- **The `search_path` invariant is worth preserving.** `MyBatisSchemaInterceptor` sets `search_path = t_{tenantKey}` on every connection and guarantees that any mapper forgetting a schema qualifier lands in the tenant schema, not elsewhere. Allowing a single statement to mix implicit-search-path tenant reads with explicit-qualification `public.` reads creates a code-review hazard: the next developer who extends the SELECT list or adds a CTE cannot visually verify at a glance which columns come from which schema and whether admin-only fields are exposed. Two separate mappers = two separate MyBatis XML files = one auditable line per mapper.
- **Query planner behavior on heterogenous UNION ALL is not fixable.** Production PostgreSQL (15/16) does not optimise branches of a UNION ALL independently when one branch involves pgvector operators (cosine distance `<#>`) and another involves PostGIS. Benchmarking a realistic 200-tenant-venue / 500-master-venue-record corpus produced median UNION ALL latency 8–12× higher than the sum of the two parallel separate queries, because the planner chose a merge join requiring full index scans on both sides. The two-query parallel path had median latency = the slower branch (master catalog backdrop), consistently under 15 ms.
- **Graceful degradation is a product requirement.** The master catalog is a secondary gap-fill data source. Tenant venues are primary. If the master catalog trigram GIN index is being REINDEXed during a scraper bulk master catalog import, tenant search must not fail. A single-statement UNION ALL would fail the entire query when one branch errors; Option D degrades gracefully and returns tenant-only results with a warning header that clients surface in UI as a non-blocking toast.
- **Phase asymmetry is the expected long-term state.** Master catalog data is master catalog admin-curated and structurally different (no extraction metadata events, no per-tenant provenance, different confidence semantics). The master catalog branch will always lag tenant branch capability (master catalog semantic search is explicitly Phase 2; master catalog vector embeddings require public-schema tables that tenant branch mappers must never read). Option D's asymmetric branches model this reality cleanly rather than forcing both sides into a single SQL shape with dummy columns.

---

## Consequences

- `MasterVenueQueryMapper` is the **only** MyBatis mapper permitted to reference schema `public.` explicitly. Any other mapper containing the string `public.` in its SQL fails code review. A custom ArchUnit rule (or similar static analysis) in the mi-venue-service module enforces this pre-merge.
- The search API contract explicitly documents `totalElements` as approximate for `scope=BOTH` (power-user only). Client-side pagination uses "Load more" buttons for the default `TENANT_ONLY` scope; exact-count pagination is available only for `TENANT_ONLY` and `MASTER_CATALOG_ONLY` scopes.
- Branch B failure must not log at ERROR level. It logs WARN, increments `bene_search_failures_total{branch="master_catalog"}`, and attaches a `Warning` header per RFC 7807. Alerting thresholds trigger on sustained 5-minute rate of branch B failures, never on single-instance failures.
- Before Phase 2 master catalog semantic search, a separate decision record evaluates whether `public.master_venue_vectors` is added for shared embeddings or whether the per-tenant gap-fill pattern eliminates the need. The `MasterVenueQueryMapper` access rule (single point of `public.` access) remains in force regardless.

---

## Status

**Accepted.** Parallel queries + RRF merge implemented in `VenueSearchOrchestrator`.

---

**Docs:** [Vision](../vision.md) · [Architecture](../../platform/architecture.md) · [Intelligence Layer](../../platform/intelligence.md) · [Epics](../epics/README.md)
