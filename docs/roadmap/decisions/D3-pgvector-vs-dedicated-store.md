# D3 — pgvector vs. dedicated vector store

> **Audience:** Engineers, architects.
> **Purpose:** Record why vector embeddings are stored in PostgreSQL via the pgvector extension rather than in a dedicated vector database (Pinecone, Weaviate, Qdrant, Milvus, Chroma).

---

## Context

Shortlisty uses semantic search over venue descriptions and extracted document chunks. Each embedding is 1536 dimensions (OpenAI text-embedding-3-small). At MVP scale, the expected vector volume is per-tenant: 100–1,000 venues × 10–50 chunks each = 1,000–50,000 vectors per tenant. At mid-term scale (Post-MVP, 12–18 months), a large tenant may reach 500K vectors. The question is where to store and query them.

---

## Options considered

### Option A — Dedicated vector database (Pinecone / Weaviate / Qdrant / Milvus)

Stand up a purpose-built vector database. Vectors live outside PostgreSQL. Tenant search queries route to the vector DB for semantic similarity results, then join with PostgreSQL for metadata filtering.

**Pros:**

- Best-in-class HNSW index performance at very large scale (10M+ vectors per tenant)
- Built-in hybrid search (sparse + dense) operators in some products
- Some products offer multi-tenancy namespaces, per-tenant isolation, and per-tenant resource quotas
- Managed offerings (Pinecone, Weaviate Cloud) reduce ops burden at large scale

**Cons:**

- One entirely new infrastructure dependency with its own connection pooling, auth model, backup/restore, and incident-response playbooks. The iQ Key Value foundation has no existing vector DB contracts.
- **Cross-system joins destroy transactional consistency.** When a `venue_metadata_events` row is written to PostgreSQL and a vector is written to the vector DB, the two writes are not part of the same transaction. One can succeed and the other fail, yielding search results that do not match the source of truth. A compensating reconciliation/sync job must be written, tested, and monitored permanently.
- Per-tenant isolation is structurally harder. Schema-per-tenant PostgreSQL gives hard structural isolation (one tenant cannot read another's rows via any misconfigured query). A vector DB namespace or partition key relies on query-time filter logic — a single missing `tenant_id = ?` predicate leaks cross-tenant embeddings.
- Metadata filtering (capacity >= 200, amenities contains WiFi, ST_DWithin geo radius) must either:
  - be pushed into the vector DB's metadata filter engine (varying support per product, some require re-indexing on schema changes), or
  - be applied as a post-filter in application code after the vector recall, which destroys recall when the structured filter is highly selective.
- PostGIS geo-search and JSONB structured filtering already run in PostgreSQL. Splitting search into two systems means the hybrid RRF merge (keyword + semantic + structured + geo) now runs across two query engines — a distributed merge instead of an in-process merge over a single PostgreSQL result set.
- Cost: a managed vector DB at MVP scale (10 tenants × 10K vectors each) is comparable to PostgreSQL cost but becomes significantly more expensive at mid-scale before the performance benefit is needed.

### Option B — pgvector, PostgreSQL extension

Vectors are stored in `item_vectors` table inside each tenant's schema (`t_{tenantKey}.item_vectors`). pgvector provides IVFFlat and HNSW index types. PostgreSQL's existing query planner handles joins between vector results, `venues` table rows, PostGIS geo predicates, and JSONB structured filters in one transactional query.

**Pros:**

- **Zero new infrastructure dependencies.** pgvector is a PostgreSQL `CREATE EXTENSION` — one command on the existing database instance. No new hosts, no new network paths, no new auth or TLS configuration. The iQ Key Value foundation already runs PostgreSQL.
- **Transactional atomicity with the source of truth.** Writing `extraction_jobs`, `venue_metadata_events`, and vectors all happen in one PostgreSQL transaction. There is no cross-system eventual-consistency gap, no sync/reconciliation job, and no failure mode where search has embeddings without metadata or vice versa.
- **Hard structural tenant isolation.** Vectors live inside each tenant's schema, behind the same `search_path` interceptor and schema grants that protect `venues` and `venue_assets`. There is no tenant-filter predicate that could be accidentally omitted.
- **Unified query surface.** Keyword (tsvector), structured (JSONB GIN), semantic (pgvector cosine), and geo (PostGIS GIST) predicates all run inside one PostgreSQL query plan. The query planner optimizes predicate order; the application performs the RRF merge in-process over already-filtered result sets.
- **Existing backup/restore, replication, and observability tooling works for vectors.** `pg_dump`, WAL-based replication, PITR, pg_stat_statements — all apply without any new adapters.
- **Sufficient performance at expected scale.** pgvector IVFFlat delivers sub-10ms semantic search at 1M vectors per instance with reasonable recall. HNSW (pgvector 0.5+) scales beyond 5M vectors per tenant before dedicated-store performance meaningfully diverges. MVP is 3+ orders of magnitude below that threshold.

**Cons:**

- At very large scale (5M+ vectors per tenant with strict p95 latency SLA), pgvector HNSW requires more memory tuning than dedicated vector databases. Tenant per-schema vector tables mean shared buffer cache contention when many large tenants coexist on one instance.
- pgvector does not ship a built-in sparse-dense hybrid operator (BM25 + cosine in one index). Hybrid search is implemented as two separate PostgreSQL queries (keyword tsvector, semantic cosine) with an in-process RRF merge — acceptable at MVP, one extra hop per search.
- Vertical vector-only features (multi-vector per document reranking, quantisation-aware ANN, built-in data loaders) are not available out of the box; some require custom SQL or extension-level work.

---

## Decision made

**Option B: pgvector as the sole vector store for Phase 1 and Phase 2.**

Vectors live in per-tenant `item_vectors` tables via the pgvector extension. Index type: IVFFlat for MVP. HNSW is available as a drop-in re-index when individual tenants breach ~500K vectors. A dedicated vector store is explicitly not planned before any single tenant sustains 5M+ vectors and a 20 ms p95 semantic search SLA — expected no earlier than 18 months post-launch.

---

## Rationale

- **Transactional consistency is non-negotiable for a knowledge base.** A search result that points to a venue whose metadata has been deleted or whose extraction job has rolled back is a correctness bug that erodes user trust. Achieving this correctly with an external vector DB is a distributed-systems problem (two-phase commit or saga with compensation) that adds months of engineering work for zero product benefit at MVP scale. pgvector gives transactional atomicity by default.
- **Schema-per-tenant structural isolation is an existing platform invariant.** The foundation architecture uses schema-per-tenant, not tenant_id column filtering, because it eliminates an entire class of misconfiguration leakage bugs. pgvector tables live inside the tenant schema; there is nothing to audit. A dedicated vector DB would be the first cross-system component that relies on predicate-level tenant isolation — a security review risk and an ongoing operational audit burden.
- **PostgreSQL already provides three of the five search modes.** Keyword (tsvector GIN), structured (JSONB GIN), and geo (PostGIS GIST) all run in PostgreSQL today. The cost of splitting only the semantic vector component into a second system is a permanent cross-system RRF merge that must handle partial failure of one branch, schema evolution in two systems, and observability aggregation in two query engines. Keeping all five search modes in PostgreSQL removes this complexity at zero incremental cost.
- **Scale inflection point is well beyond product-market-fit thresholds.** Shortlisty's unit economics target 100 paying agencies at $150/month for sustainability. 100 tenants × 1,000 venues × 20 chunks = 2M vectors total across the entire platform. pgvector IVFFlat on a single reasonably-sized PostgreSQL instance handles this volume with sub-10ms latency. The dedicated-vector-store scale horizon is at least 50–100× that.

---

## Consequences

- PostgreSQL instances in all environments must have the `pgvector` extension available in their base image. This is a one-line change to the foundation PostgreSQL Dockerfile.
- The IVFFlat `lists` parameter is tenant-specific. Per-tenant index creation sets `lists = max(ceil(sqrt(vector_count)), 10)` via an admin re-index endpoint that runs after bulk imports. Default MVP `lists = 10` is explicitly documented.
- HNSW index creation is a documented manual procedure for tenants above 500K vectors. Before reaching that scale, platform capacity planning must evaluate whether to vertically scale the PostgreSQL node or shard tenants across multiple PostgreSQL instances.
- A future decision record evaluates Pinecone / Qdrant migration only when: (a) at least one tenant has sustained 5M+ vectors for 90 days, and (b) pgvector HNSW tuning cannot meet a 20 ms p95 latency SLA that the tenant is contractually entitled to. That decision record must explicitly cover cross-system consistency, tenant isolation, and hybrid-search merge paths.

---

## Status

**Accepted.** pgvector is the vector store. IVFFlat for MVP. HNSW available at scale.

---

**Docs:** [Vision](../vision.md) · [Intelligence Layer](../../platform/intelligence.md) · [Architecture](../../platform/README.md) · [Epics](../epics/README.md)
