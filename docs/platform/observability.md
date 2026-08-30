# Shortlisty — Observability & Security

> **Audience:** Engineers, SREs.
> **Purpose:** All Prometheus metrics with labels and purpose, Grafana dashboard location, and the security model for tenant data isolation, asset access, and GDPR compliance.

---

## Related Documents

- [services.md](services.md) — tenant isolation in PostgreSQL and S3; GDPR deletion cascade
- [data-model.md](data-model.md) — schema versioning metrics; `_schema_version` contract
- [aggregation.md](aggregation.md) — `shortlisty_metadata_schema_version_seen_total` on every `migrateToCurrent()` call
- [master-catalog.md](master-catalog.md) — `shortlisty_mc_match_total` and confidence histogram
- [search.md](search.md) — `shortlisty_search_requests_total`, branch-level latency, failure isolation
- [architecture-overview.md](architecture-overview.md) — security config patterns, JWT claims, authority strings

---

## 12. Observability

Both Shortlisty services follow foundation patterns exactly.

### Prometheus Metrics

| Metric                                           | Labels                                  | Notes                                                                                                                                                                                                                                                                                                     |
| ------------------------------------------------ | --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `shortlisty_venues_total`                           | `tenant_id`, `status`                   | Venue count by state                                                                                                                                                                                                                                                                                      |
| `shortlisty_assets_uploaded_total`                  | `tenant_id`, `asset_type`               | Upload volume                                                                                                                                                                                                                                                                                             |
| `shortlisty_extractions_total`                      | `tenant_id`, `extractor_type`, `status` | Success/failure rates                                                                                                                                                                                                                                                                                     |
| `shortlisty_extraction_duration_seconds`            | `extractor_type`                        | Latency histogram                                                                                                                                                                                                                                                                                         |
| `shortlisty_ai_cost_usd_total`                      | `tenant_id`, `model`                    | Cost tracking                                                                                                                                                                                                                                                                                             |
| `shortlisty_search_requests_total`                  | `search_mode`, `scope`                  | keyword / semantic / hybrid; `scope=TENANT_ONLY\|MASTER_CATALOG_ONLY\|BOTH`                                                                                                                                                                                                                               |
| `shortlisty_search_latency_seconds`                 | `search_mode`, `branch`                 | Branch-level latency histogram. `branch=tenant` / `master_catalog` / `orchestrator_total` — lets us attribute slowdowns to one of the two parallel SQL branches or the app-level merge step.                                                                                                              |
| `shortlisty_search_failures_total`                  | `branch`                                | Counter incremented when a branch query fails. For `branch=master_catalog` the orchestrator returns partial tenant results with a `Warning` header. For `branch=tenant` the whole request 5xx.                                                                                                            |
| `shortlisty_metadata_schema_version_seen_total`     | `tenant_id`, `schema_version`, `op`     | Counter incremented on every `migrateToCurrent()` or `ensureCurrent()` call — labels the schema version the document had _before_ migration. `op` = `read` or `write`. Lets us see how many legacy v0/v1/vN docs are still in the hot paths and when old migration classes are candidates for retirement. |
| `shortlisty_metadata_migration_duration_seconds`    | `target_version`                        | Latency histogram for the full migration chain. Alerts if a new migration step is unexpectedly slow for large JSONB payloads.                                                                                                                                                                             |
| `shortlisty_mc_match_total`                         | `stage`, `outcome`                      | `stage` = `"extraction"` (tenant MC_INHERIT merge) or `"import_dedup"` (scraper/admin pre-write). `outcome` = `matched` / `ambiguous` / `no_match`. Tracks how often master catalog produces useful hits.                                                                                                 |
| `shortlisty_mc_match_confidence_seconds`            | `stage`                                 | Histogram of the `combined` confidence score per matcher invocation. Buckets 0.5–0.6, 0.6–0.7, 0.7–0.8, 0.8–0.9, 0.9–1.0. Used during post-launch threshold recalibration.                                                                                                                                |
| `shortlisty_venue_import_from_master_catalog_total` | `tenant_id`, `status`                   | User-initiated explicit promote via `POST /api/v1/venues/from-master-catalog/{id}`. `status=created\|duplicate\|error`. Baseline "MC ROI" signal.                                                                                                                                                         |
| `shortlisty_item_vectors_rows_total`                | `tenant_id`                             | Gauge — per-tenant `item_vectors` row count. Trigger for IVFFlat → HNSW evaluation when > 1 M rows (see [roadmap-decisions.md](roadmap-decisions.md)).                                                                                                                                                    |
| `shortlisty_master_catalog_entry_sweep_total`       | `outcome`                               | Counter incremented when orphan `venues.master_venue_id` values are swept after a master catalog entry is deleted. `outcome` = `updated` / `no_match` / `error`. Phase 2 — see [roadmap-decisions.md](roadmap-decisions.md).                                                                              |

### Grafana Dashboard

Added to `docker/grafana/provisioning/dashboards/VipService.json`.

---

## 13. Security

| Concern                 | Approach                                                                                                                                                                |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tenant data isolation   | Schema-per-tenant (PostgreSQL + pgvector); S3 key prefix `shortlisty/tenants/{tenantKey}/` per tenant — see [services.md § 4b](services.md) for full key layout            |
| Asset access            | Presigned S3 URLs only (15 min upload, 1h download). No public bucket. Master catalog paths (`shortlisty/master-catalog/*`) inaccessible via tenant-issued presigned URLs. |
| AI data handling        | Documents sent to OpenAI API per their data processing terms. Enterprise option: Azure OpenAI (data stays in tenant's region).                                          |
| GDPR / right to erasure | `DELETE tenant` cascades to venues → assets → S3 objects → vector embeddings. Full cascade detail in [services.md § Deletion Cascade](services.md).                     |
| Audit trail             | All `venue.*`, `asset.*`, `extraction.*` events passively consumed by `foundation-audit-service` (see [events.md](events.md)).                                          |
| PII in documents        | Warn on upload. Do not log extracted text.                                                                                                                              |

### Security Config

Full `SecurityConfig` pattern, JWT claim names, and authority strings are in [architecture-overview.md § Security](architecture-overview.md).

Summary of authority strings for Shortlisty endpoints:

| Endpoint scope                                 | Required authority        |
| ---------------------------------------------- | ------------------------- |
| Regular venue/asset/metadata/search operations | `MEMBER`                  |
| Venue delete, asset delete                     | `ADMIN` or `TENANT_OWNER` |
| Admin cross-tenant read                        | `PLATFORM_ADMIN`          |
| Master Catalog admin write                     | `PLATFORM_ADMIN`          |

---

**Docs:** [Architecture Index](README.md) · [Architecture Overview](architecture-overview.md) · [Services](services.md) · [Data Model](data-model.md) · [Search](search.md) · [Master Catalog](master-catalog.md) · [Roadmap & Decisions](roadmap-decisions.md)
