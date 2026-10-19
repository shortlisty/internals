# Shortlisty — Service Architecture

> **Audience:** Engineers, architects.
> **Purpose:** Service decomposition, table ownership rules, self-contained service rule, and S3 storage layout.

---

## Related documents

- [architecture-overview.md](architecture-overview.md) — platform context, foundation reuse, implementation patterns
- [data-model.md](data-model.md) — full domain model and database schema
- [aggregation.md](aggregation.md) — metadata aggregation and FIFO race-condition prevention
- [etl-pipeline.md](etl-pipeline.md) — async ETL pipeline internals
- [events.md](events.md) — RabbitMQ event contracts and queue topology
- [master-catalog.md](master-catalog.md) — master catalog cold-start and MC_INHERIT merge

---

## 4. Service architecture

Two runtime services, each fully self-contained. Domain classes, event payloads, and Liquibase changelogs live inside the service that owns them, and neither service declares a compile dependency on the other. Where both services touch the same table, the ownership rules below govern access, and each service carries its own copy of the payload types it needs. Duplication across services is accepted deliberately in exchange for zero cross-service compile coupling.

```
                               ┌─────────────────────┐
  Browser / App         ─────► │   Gateway Service   │  (existing)
                               └──────────┬──────────┘
                                          │ JWT validated, X-Tenant-ID set
                               ┌──────────▼──────────┐     ┌─────────────────────┐
                               │     IAM Service     │     │   Billing Service   │  (existing)
                               └──────────┬──────────┘     └──────────┬──────────┘
                                          │                           │ plan entitlements
  ┌───────────────────────────────────────▼───────────────────────────▼───────────┐
  │                       docintelligence-catalog-service                       │
  │                   venues · assets · metadata · search · api                   │
  └───────────┬───────────────────────────┬───────────────────────────┬───────────┘
              │ RabbitMQ                  │ r/w                       │ presigned URL
              │ asset.uploaded            │                           │ issue + delete
   ┌──────────▼──────────┐     ┌──────────▼──────────┐     ┌──────────▼──────────┐
   │ docintelligence-  │     │ PostgreSQL          │     │ S3 / MinIO          │
   │ catalog-processing- │     │ t_{tenantKey}       │     │ shortlisty/tenants/ │
   │ worker (async)      │     │ public.master_venue │     │ shortlisty/master-  │
   │ (no inbound HTTP)   │     │ + pgvector/PostGIS  │     │ catalog/            │
   └─────────────────────┘     └─────────────────────┘     └─────────────────────┘
```

### `docintelligence-catalog-service`

- **Responsibilities:** venue CRUD, asset upload flow (presigned URL), metadata read/write, search API, plan entitlement enforcement, master catalog backdrop lookup
- **Database:** shares the docintelligence PostgreSQL instance with `docintelligence-catalog-processing-worker` — this is a deliberate architectural decision, not an exception (see [D13](../roadmap/decisions/D13-shared-database-per-product.md)). Tenancy is schema-level via `foundation-tenancy` — each tenant gets its own schema `t_{tenantKey}`. No `tenant_id` column on any table; schema routing is handled by `MyBatisSchemaInterceptor`.
- **Exposes:** REST API at `/api/v1/venues` (see [api.md](api.md))
- **Publishes:** `venue.created`, `venue.updated`, `asset.uploaded`, `asset.deleted` (RabbitMQ)
- **Consumes:** `extraction.completed`, `extraction.failed` (RabbitMQ) — triggers metadata aggregation

### `docintelligence-catalog-processing-worker`

- **Responsibilities:** document ETL pipeline (parse → chunk → extract → embed), extraction job lifecycle, master catalog match and MC_INHERIT merge, metadata aggregation, scheduled maintenance jobs (stale re-aggregation, cost reporting)
- **Nature:** async background worker — no inbound HTTP, no REST API, no service discovery entry. Event-driven only.
- **Database:** shares the docintelligence PostgreSQL instance with `docintelligence-catalog-service` (see [D13](../roadmap/decisions/D13-shared-database-per-product.md)). Reads `venue_assets`, writes `extraction_jobs`, `venue_metadata_events`, `item_vectors`, `ai_cost_tracking`. Also reads `public.master_venue` for the master catalog match step.
- **Consumes:** `asset.uploaded` (RabbitMQ) — triggers ETL pipeline (see [etl-pipeline.md](etl-pipeline.md))
- **Publishes:** `extraction.started`, `extraction.completed`, `extraction.failed` (RabbitMQ)
- **External calls:** self-hosted LLM inference endpoint (Phi-4 / Qwen2.5 via Ollama or vLLM), optionally Docling sidecar (Phase 2) — see [D12](../roadmap/decisions/D12-self-hosted-inference-stack.md)
- **Scaling:** replicas scaled independently based on RabbitMQ queue depth — no impact on `docintelligence-catalog-service`

> The worker reads the uploaded object from S3 and writes its extraction results to the tenant schema. Those edges are omitted from the diagram above for clarity.
>
> Scraping (e.g. Tagvenue) is extracted to standalone Node.js scrapers: `docintelligence-mc-ingest-<source>-scraper`. Master Catalog population runs in `docintelligence-master-venue-loader` (Spring Boot). Full cold-start strategy in [master-catalog.md](master-catalog.md).

### Table ownership

Both services share one PostgreSQL schema. Ownership defines who may write to a table. Cross-boundary reads are permitted; cross-boundary writes are not.

| Table                   | Owner                                       | The other service may…                                                                         |
| ----------------------- | ------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `venues`                | `docintelligence-catalog-service`           | read (processing-worker: resolve venue_id only)                                                |
| `venue_assets`          | `docintelligence-catalog-service`           | read (processing-worker: fetch asset for processing)                                           |
| `venue_metadata_events` | `docintelligence-catalog-service`           | write via event reaction (`extraction.completed` → docintelligence-catalog-service aggregates) |
| `extraction_jobs`       | `docintelligence-catalog-processing-worker` | read (venue-service: expose job status to API)                                                 |
| `item_vectors`          | `docintelligence-catalog-processing-worker` | read (venue-service: vector search queries)                                                    |
| `ai_cost_tracking`      | `docintelligence-catalog-processing-worker` | read (venue-service: expose cost summary to API)                                               |

The single legitimate cross-boundary read from `docintelligence-catalog-processing-worker` is a `SELECT` on `venue_assets` by `asset_id` (delivered in the `asset.uploaded` event payload). This is a foreign key lookup, not business logic — acceptable and intentional.

---

## 4a. S3 storage layout

S3 (MinIO for local dev) is already in the iQ Key Value stack. Shortlisty adds its own prefix namespace inside the shared bucket (`iqkv-files`).

### Bucket strategy

| Environment | Bucket           | Notes                                                                                        |
| ----------- | ---------------- | -------------------------------------------------------------------------------------------- |
| Dev / CI    | `iqkv-files`     | Shared with foundation services, MinIO default. VIP objects live under `shortlisty/` prefix. |
| Staging     | `iqkv-files`     | Same shared bucket, same prefix scheme. Isolated by prefix only.                             |
| Production  | `iqkv-vip-files` | Dedicated bucket. Separate IAM policy, separate lifecycle rules. Key structure identical.    |

MinIO in local dev is configured in `docker-compose.yml` with `MINIO_DEFAULT_BUCKETS=iqkv-files`.

### Key naming convention

#### Tenant asset files

```
shortlisty/tenants/{tenantKey}/venues/{venueId}/assets/{assetId}/{fileName}
```

| Segment       | Value                                         | Example                            |
| ------------- | --------------------------------------------- | ---------------------------------- |
| `shortlisty/` | VIP namespace                                 | (literal)                          |
| `tenants/`    | Tenant subtree root                           | (literal)                          |
| `{tenantKey}` | 8-char nanoid from JWT `tenant_id` claim      | `acme0001`                         |
| `venues/`     | Venue subtree                                 | (literal)                          |
| `{venueId}`   | UUID of the venue (no hyphens — compact form) | `550e8400e29b41d4a716446655440000` |
| `assets/`     | Asset subtree                                 | (literal)                          |
| `{assetId}`   | UUID of the asset (no hyphens)                | `6ba7b8109dad11d180b400c04fd430c8` |
| `{fileName}`  | Original file name, URL-safe, max 255 chars   | `grand-ballroom-deck.pdf`          |

Full example:

```
shortlisty/tenants/acme0001/venues/550e8400e29b41d4a716446655440000/assets/6ba7b8109dad11d180b400c04fd430c8/grand-ballroom-deck.pdf
```

**Key rules:**

- `{fileName}` is the original client-supplied file name, stripped of path separators, URL-encoded where necessary. Stored as-is after sanitisation — no UUID substitution — so the key stays human-readable.
- `{venueId}` and `{assetId}` are UUIDs without hyphens (compact 32-char hex).
- The `asset_id` is generated server-side at `POST /api/v1/venues/{venueId}/assets/initiate` and written to `venue_assets.id` before the presigned URL is issued. The S3 key is stored in `venue_assets.s3_key` — at confirm time no key re-computation happens.

#### Presigned URL issuance

`POST /api/v1/venues/{venueId}/assets/initiate` response:

```json
{
  "asset_id": "<uuid>",
  "upload_url": "https://minio.local/iqkv-files/shortlisty/tenants/acme0001/venues/.../grand-ballroom-deck.pdf?X-Amz-Signature=...",
  "expires_at": "2025-06-01T12:15:00Z"
}
```

The presigned PUT URL is scoped to the exact key. The client uploads directly. The service never proxies the file body. Download presigned URLs (1h TTL) are generated on-demand at `GET /api/v1/venues/{venueId}/assets/{id}/download-url` — not stored.

#### Master catalog import files

```
shortlisty/master-catalog/imports/{importId}/{fileName}
shortlisty/master-catalog/exports/{date}/{snapshot}.jsonl.gz
```

| Path                                            | Purpose                                                                           |
| ----------------------------------------------- | --------------------------------------------------------------------------------- |
| `shortlisty/master-catalog/imports/{importId}/` | One folder per import batch (admin-triggered). Contains raw CSV/JSON input files. |
| `shortlisty/master-catalog/exports/{date}/`     | Nightly compacted snapshots of `public.master_venue` for downstream consumers.    |

### Tenant isolation

- All tenant objects scoped under `shortlisty/tenants/{tenantKey}/`. Cross-tenant read is structurally impossible without knowing the other tenant's key.
- The service account holds a single S3 IAM policy allowing `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on the full `shortlisty/*` prefix. Presigned URLs are scoped to the exact object key.
- Master catalog paths (`shortlisty/master-catalog/*`) are not accessible via tenant-issued presigned URLs. Written only by the platform's internal job service account (`docintelligence-master-venue-loader`).

### Lifecycle rules

| Rule                             | Prefix                                         | Action                                                                                                 |
| -------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Extraction artefact expiry       | `shortlisty/tenants/*/venues/*/assets/*/`      | Transition to Glacier/IA after 90 days if `extraction_status = COMPLETED`. Managed via S3 object tags. |
| Master catalog import cleanup    | `shortlisty/master-catalog/imports/processed/` | Delete after 30 days.                                                                                  |
| Master catalog snapshot rotation | `shortlisty/master-catalog/exports/`           | Keep last 14 daily snapshots; delete older.                                                            |

Object tags set by `docintelligence-catalog-service` at `POST /api/v1/venues/{venueId}/assets/{id}/confirm`:

| Tag key             | Values                                          | Set by                                            |
| ------------------- | ----------------------------------------------- | ------------------------------------------------- |
| `extraction_status` | `pending`, `completed`, `failed`                | docintelligence-catalog-service at confirm/update |
| `asset_type`        | `pdf_deck`, `floor_plan`, `photo`, `cad_file` … | docintelligence-catalog-service at initiate       |
| `tenant_key`        | 8-char nanoid                                   | docintelligence-catalog-service at initiate       |

### Deletion cascade

When a tenant deletes an asset (`DELETE /api/v1/venues/{venueId}/assets/{id}`) or when a tenant account is terminated:

1. `docintelligence-catalog-service` deletes the `venue_assets` row (DB cascade drops extraction jobs and metadata events).
2. `docintelligence-catalog-service` issues `s3:DeleteObject` for `venue_assets.s3_key`.
3. `asset.deleted` event published → `docintelligence-catalog-processing-worker` deletes all `item_vectors` rows where `metadata->>'asset_id' = :assetId`.

For full tenant deletion (GDPR right to erasure):

1. `DELETE FROM t_{tenantKey}.venues` cascades to all asset rows.
2. A `tenant.deleted` event triggers a background S3 sweep: `s3:DeleteObjects` with all keys matching `shortlisty/tenants/{tenantKey}/*` (batched in 1000-object chunks).
3. The pgvector sweep deletes all `item_vectors` rows for the tenant schema (schema drop handles this implicitly if the schema is dropped).

---

**Docs:** [Architecture Index](README.md) · [Architecture Overview](architecture-overview.md) · [Data Model](data-model.md) · [ETL Pipeline](etl-pipeline.md)
