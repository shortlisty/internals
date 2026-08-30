# Shortlisty — Search Architecture

> **Audience:** Engineers.
> **Purpose:** All search modes, the cross-source orchestration pattern for merging tenant venues with the master catalog backdrop, and how the `scope` parameter controls what is returned.

---

## Related Documents

- [data-model.md](data-model.md) — index strategy (`IVFFlat`, `GIN`, `GIST`), `item_vectors` table, `venues.master_venue_id`
- [services.md](services.md) — `mi-venue-service` owns the search API; cross-schema access rules for `public.master_venue`
- [master-catalog.md](master-catalog.md) — MC_INHERIT provenance; `master_venue_id` populated at extraction time
- [api.md](api.md) — `GET /api/v1/venues/` endpoint, `scope` query param, response DTOs; `GET /api/v1/venues/master-venues` MEMBER search endpoint
- [observability.md](observability.md) — `shortlisty_search_requests_total`, `shortlisty_search_latency_seconds`, `shortlisty_search_failures_total`

---

## 6. Search Architecture

All search is served by `mi-venue-service` querying PostgreSQL directly. No separate search service.

### Search Modes

| Mode               | Implementation                                 | Use case                               |
| ------------------ | ---------------------------------------------- | -------------------------------------- |
| Keyword            | `tsvector` / `to_tsquery` (GIN index)          | "conference downtown AV"               |
| Structured filters | JSONB queries (GIN index)                      | capacity ≥ 200, amenities include WiFi |
| Semantic           | pgvector cosine distance (IVFFlat index)       | "modern loft space with natural light" |
| Geo-spatial        | PostGIS `ST_DWithin` (GIST index)              | venues within 10 miles of zip code     |
| Hybrid             | Reciprocal Rank Fusion over keyword + semantic | Default search bar                     |

### Vector Index Strategy

- Index type: IVFFlat (faster build, good recall up to ~5M vectors)
- Distance: cosine
- Dimensions: 1536
- Scope: per-tenant schema (no cross-tenant leakage)
- Upgrade path: switch to HNSW when tenant exceeds ~500K venues (rare) — see [roadmap-decisions.md](roadmap-decisions.md) Phase 2

---

## Cross-Source Search (Tenant Venues + Master Catalog Backdrop)

**Problem.** Master catalog lives in `public.master_venue`, tenant venues live in `t_{tenantKey}.venues`. A single cross-schema SQL `UNION ALL` would technically work but it breaks the schema-per-tenant isolation abstraction and couples two datasets that have different index statistics, different signal sets, and different column-level permission policies.

**Decided: Approach 2 — two parallel SQL selects + app-level field-level merge.** Single PostgreSQL instance, no external search service. No cross-schema joins or `UNION ALL` in one statement.

```
            ┌──────────────────────────────┐
            │  GET /api/v1/venues?scope=  │
            │  BOTH | TENANT_ONLY          │
            │  | MASTER_CATALOG_ONLY       │
            │  (default: TENANT_ONLY)      │
            │  &search=... &page=0 &size=20│
            └──────────────┬───────────────┘
                           │ MyBatisSchemaInterceptor
                           │ search_path = t_acme0001, public
                           ▼
          ┌────────────────┴─────────────────┐
          │  VenueSearchOrchestrator         │
          │  (CompletableFuture.parallel)    │
          └───────┬──────────────────┬───────┘
                  │                  │
   ┌──────────────▼───────┐  ┌──────▼──────────────────────────┐
   │  Branch A: tenant    │  │  Branch B: master catalog       │
   │  venues              │  │  (backdrop, disabled by default)│
   │  (VenueMapper)       │  │  (MasterVenueQueryMapper)       │
   │  — no schema qualify │  │  — explicitly schema-qualified: │
   │    (implicit         │  │    FROM public.master_venue     │
   │    search_path)      │  │  — 3 modes MVP: keyword,        │
   │  — all 5 modes:      │  │    structured, geo only         │
   │    keyword, semantic,│  │    (semantic TBD Phase 2)       │
   │    structured, geo,  │  │  — top-50 + scores              │
   │    hybrid RRF        │  │                                 │
   │  — top-50 + scores   │  │                                 │
   └──────────────┬───────┘  └──────┬───────────────────────────┘
                  │                  │
          ┌───────▼──────────────────▼───────┐
          │  App-level field-level merge     │
          │  — dedup: if venues.master_      │
          │    venue_id == masterVenue.id    │
          │    → drop separate MC row,       │
          │    keep TENANT origin only       │
          │  — MC values merge INTO tenant   │
          │    rows invisibly via MC_INHERIT │
          │    provenance (not separate rows)│
          │  — RRF equal weights 0.5:0.5     │
          │    for scope=BOTH                │
          │  — slice by (page*size, size)    │
          └──────────────────┬───────────────┘
                             ▼
              VenueSummaryListResponse(items: [...])
```

### Rejected Alternatives

- _(Rejected)_ Materialised master catalog copy per tenant schema. Duplicates N×master catalog rows; scraper/admin updates would need fan-out propagation — consistency nightmare.
- _(Rejected)_ Single `UNION ALL` or JOIN in one SQL statement with `public.` qualified names. Breaks `search_path` isolation abstraction, risks admin-only column leakage if the mapper SELECT list is later widened naively, and couples two sources that may scale independently.
- _(Rejected for MVP)_ Dedicated OpenSearch/Elasticsearch cluster. Over-engineering at MVP scales (< 500 master catalog entries, < 1000 tenant venues per tenant). Promote if master catalog count breaches 100K rows or tenant-side vector recall becomes a bottleneck (see [roadmap-decisions.md](roadmap-decisions.md) Phase 2).

### Failure Isolation

Branch B (master catalog backdrop) timeout or SQL exception → search does **not** fail the whole request. Returns branch A results with:

```
Warning: 299 - "Master catalog backdrop merge unavailable, results incomplete"
```

response header and increments `shortlisty_search_failures_total{branch="master_catalog"}`.

### Search Parameters

| Parameter    | Type              | Description                                                           |
| ------------ | ----------------- | --------------------------------------------------------------------- |
| `search`     | string            | Natural language or keyword query — triggers hybrid mode              |
| `scope`      | enum              | `TENANT_ONLY`, `MASTER_CATALOG_ONLY`, `BOTH` (default: `TENANT_ONLY`) |
| `status`     | enum              | `DRAFT`, `ACTIVE`, `ARCHIVED`                                         |
| `capacity`   | integer           | Minimum total capacity                                                |
| `lat`, `lng` | decimal           | Centre point for geo-spatial search                                   |
| `radius_km`  | decimal           | Radius from lat/lng (requires both to be set)                         |
| `amenities`  | string[]          | Required amenity codes (comma-separated)                              |
| `catering`   | enum              | Catering policy filter                                                |
| `page`       | integer (0-based) | Default: 0                                                            |
| `size`       | integer           | Default: 20, max: 100                                                 |
| `sort`       | string            | e.g. `name,asc` or `relevance` (default when `search` present)        |

### Response Shape

- Each `VenueSummaryView` record includes `origin: "TENANT"` only by default. Old `"REGISTRY"` as a surfaced origin to UI is removed. MC values merge INTO tenant rows invisibly; provenance tracked only internally in `metadata_sources` per field with `MC_INHERIT` source.
- `totalElements` in `VenueSummaryListResponse` is deliberately _approximate_ for `scope=BOTH` (equal ranks across two top-K → real total is unknown without issuing two `COUNT(*)`). Clients show "1K+" or "Load more" buttons rather than rendering a page-50 pagination bar.

### Explicit Promote from Master Catalog

Power user/admin explicitly promotes a master catalog entry → client calls `POST /api/v1/venues/from-master-catalog/{masterVenueId}` (see [api.md](api.md)). Creates a fresh tenant-owned `venues` row copying master catalog fields into `metadata` with `metadata_sources[*].source = "MC_INHERIT"` priority 7. Dedup key: `t_{tenant}.venues.master_venue_id = masterVenue.id`. Subsequent searches show the result as origin `TENANT`.

---

**Docs:** [Architecture Index](README.md) · [Data Model](data-model.md) · [Services](services.md) · [Master Catalog](master-catalog.md) · [API](api.md) · [Observability](observability.md)
