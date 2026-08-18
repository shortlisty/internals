# VenueMi — Service Architecture

> **Audience:** Engineers, architects.
> **Purpose:** Service decomposition, table ownership rules, shared library internals (`mi-venue-model`, `mi-data-intelligence`), and S3 storage layout.

---

## Related Documents

- [architecture-overview.md](architecture-overview.md) — platform context, foundation reuse, implementation patterns
- [data-model.md](data-model.md) — full domain model and database schema
- [aggregation.md](aggregation.md) — metadata aggregation and FIFO race-condition prevention
- [etl-pipeline.md](etl-pipeline.md) — async ETL pipeline internals
- [events.md](events.md) — RabbitMQ event contracts and queue topology
- [master-catalog.md](master-catalog.md) — master catalog cold-start and MC_INHERIT merge

---

## 4. Service Architecture

```
                    ┌──────────────────┐
  Browser/App  ───► │  Gateway Service │  (existing)
                    └────────┬─────────┘
                             │ JWT validated, X-Tenant-ID set
                    ┌────────▼─────────┐     ┌──────────────────┐
                    │   IAM Service    │     │  Billing Service │  (existing)
                    └────────┬─────────┘     └────────┬─────────┘
                             │                        │ plan entitlements
                    ┌────────▼────────────────────────▼─────────┐
                    │            mi-venue-service                │
                    │  venues · assets · metadata · search · api │
                    └────────┬──────────────┬──────────┬─────────┘
                             │ RabbitMQ     │ r/w       │ presigned URL
                             │ asset.uploa- │           │ issue + delete
                    ┌────────▼─────────┐  ┌─▼──────────▼──────┐
                    │ mi-venue-        │  │   PostgreSQL        │
                    │ processing-worker│──►│   t_{tenant}        │
                    │ (async sidecar)  │  │   + pgvector        │
                    └────────┬─────────┘  │   + PostGIS         │
                             │            └─────────────────────┘
                             │            ┌─────────────────────┐
                             │            │   public schema      │
                             ├───────────►│   master_venue       │
                             │            │   master_venue_alias │
                             │            │   master_venue_extern│
                             │            └─────────────────────┘
                             │ read asset ┌─────────────────────┐
                             └───────────►│   S3 / MinIO         │◄── client (direct PUT)
                                          │   venuemi/tenants/   │
                                          │   venuemi/master-    │
                                          │   catalog/           │
                                          └─────────────────────┘
                                          ▲
                                          │ scraper output CSV/JSONL
                    ┌─────────────────────┤
                    │ mi-mc-ingest-        │
                    │ tagvenue-scraper     │──► mi-mc-loader ──► public.master_venue
                    │ (Node.js)            │     (Spring Boot)   master_venue_external
                    └─────────────────────┘
```

### `mi-venue-service`

- **Responsibilities:** venue CRUD, asset upload flow (presigned URL), metadata read/write, search API, plan entitlement enforcement, master catalog backdrop lookup
- **Database:** owns the VenueMi PostgreSQL schema. Tenancy is schema-level via `foundation-tenancy` — each tenant gets its own schema `t_{tenantKey}`. No `tenant_id` column on any table; schema routing is handled by `MyBatisSchemaInterceptor`. Shared with `mi-venue-processing-worker` — no cross-service API calls for data.
- **Exposes:** REST API at `/api/v1/venues` (see [api.md](api.md))
- **Publishes:** `venue.created`, `venue.updated`, `asset.uploaded`, `asset.deleted` (RabbitMQ)
- **Consumes:** `extraction.completed`, `extraction.failed` (RabbitMQ) — triggers metadata aggregation

### `mi-venue-processing-worker`

- **Responsibilities:** document ETL pipeline (parse → chunk → extract → embed), extraction job lifecycle, master catalog match and MC_INHERIT merge, metadata aggregation, scheduled maintenance jobs (stale re-aggregation, cost reporting)
- **Nature:** async sidecar — no inbound HTTP, no REST API, no service discovery entry. Event-driven only.
- **Database:** shared PostgreSQL schema with `mi-venue-service`. Reads `venue_assets`, writes `extraction_jobs`, `venue_metadata_events`, `item_vectors`, `ai_cost_tracking`. Also reads `public.master_venue` for the master catalog match step.
- **Consumes:** `asset.uploaded` (RabbitMQ) — triggers ETL pipeline (see [etl-pipeline.md](etl-pipeline.md))
- **Publishes:** `extraction.started`, `extraction.completed`, `extraction.failed` (RabbitMQ)
- **External calls:** OpenAI API (GPT-4o, text-embedding-3-small), optionally Docling sidecar (Phase 2)
- **Scaling:** replicas scaled independently based on RabbitMQ queue depth — no impact on `mi-venue-service`

> Scraping (e.g. Tagvenue) is extracted to standalone Node.js scrapers: `mi-mc-ingest-<source>-scraper`. Master Catalog population runs in `mi-mc-loader` (Spring Boot). Full cold-start strategy in [master-catalog.md](master-catalog.md).

### Table Ownership

Both services share one PostgreSQL schema. Ownership defines who may write to a table. Cross-boundary reads are permitted; cross-boundary writes are not.

| Table                   | Owner                        | The other service may…                                                          |
| ----------------------- | ---------------------------- | ------------------------------------------------------------------------------- |
| `venues`                | `mi-venue-service`           | read (processing-worker: resolve venue_id only)                                 |
| `venue_assets`          | `mi-venue-service`           | read (processing-worker: fetch asset for processing)                            |
| `venue_metadata_events` | `mi-venue-service`           | write via event reaction (`extraction.completed` → mi-venue-service aggregates) |
| `extraction_jobs`       | `mi-venue-processing-worker` | read (venue-service: expose job status to API)                                  |
| `item_vectors`          | `mi-venue-processing-worker` | read (venue-service: vector search queries)                                     |
| `ai_cost_tracking`      | `mi-venue-processing-worker` | read (venue-service: expose cost summary to API)                                |

The single legitimate cross-boundary read from `mi-venue-processing-worker` is a `SELECT` on `venue_assets` by `asset_id` (delivered in the `asset.uploaded` event payload). This is a foreign key lookup, not business logic — acceptable and intentional.

---

## 4a. Shared Library — `mi-venue-model`

`mi-venue-model` is a plain Java library (JAR, no Spring Boot). It is the **venue-domain layer** containing only venue-specific entities, field definitions, metadata migrations, and schema changelogs. Generic extraction infrastructure lives in `mi-data-intelligence` (§4c), which this library imports as a compile dependency.

Both `mi-venue-service` and `mi-venue-processing-worker` declare `mi-venue-model` as a compile dependency and receive `mi-data-intelligence` transitively.

**Contents:**

```
mi-venue-model/
├── venue/
│   ├── Venue.java                       Plain POJO — aggregate root, no JPA annotations
│   ├── VenueStatus.java                 enum: DRAFT, ACTIVE, ARCHIVED
│   └── VenueAsset.java                  Plain POJO — file attachment owned by a venue
├── metadata/
│   ├── VenueMetadata.java               Value object — mirrors venues.metadata JSONB
│   ├── VenueCapacity.java               Capacity configurations value object
│   ├── VenueMetadataSchemaVersion.java  Single source of truth: CURRENT_SCHEMA_VERSION = 1
│   │                                    Extends MetadataSchemaVersion contract from
│   │                                    mi-data-intelligence.
│   ├── VenueMetadataMigrator.java       Extends MetadataMigrator (mi-data-intelligence):
│   │                                    supplies the ordered venue migration list and
│   │                                    CURRENT_SCHEMA_VERSION.
│   ├── VenueMetadataTypeHandler.java    Extends MetadataTypeHandler (mi-data-intelligence):
│   │                                    wires VenueMetadataMigrator into MyBatis result maps.
│   └── migrations/
│       ├── VenueMetadataMigrationV0ToV1.java   bootstraps legacy pre-versioned docs → v1
│       └── VenueMetadataMigrationV1ToV2.java   reserved for next schema bump
├── mastercatalog/
│   ├── MasterVenue.java                 Plain POJO — platform curated venue record (public schema)
│   ├── MasterVenueAlias.java            Plain POJO — alternative names for deduplication
│   └── MasterVenueExternal.java         Plain POJO — external provider records (Tagvenue, Cvent, etc.)
└── db/
    └── changelog/
        ├── system/                      System (public) schema — master venue catalog tables
        │   ├── master.xml
        │   └── 20260901000000-create-master-venue.xml
        └── tenant/                      Tenant schema — venue domain tables only
            ├── master.xml               includes mi-data-intelligence/intelligence/master.xml
            │                            first, then venue-specific changesets
            ├── 20260901000001-create-venues.xml
            └── 20260901000002-create-venue-assets.xml
```

> Infrastructure table changelogs (`extraction_jobs`, `item_metadata_events`, `item_vectors`, `ai_cost_tracking`) live in `mi-data-intelligence/db/changelog/intelligence/` and are included via the `tenant/master.xml` reference. They must not be duplicated here.

**Rules:**

- No `@Service`, `@Repository`, `@Component`, or any Spring bean annotation.
- No business logic — plain Java domain classes, value objects, enums only. `VenueMetadataMigrator` is the single exception: pure-function schema migration is a model-layer concern.
- No JPA annotations. Plain POJOs only — the platform uses MyBatis.
- `VenueMetadataSchemaVersion.CURRENT_SCHEMA_VERSION` is the **only** place the venue schema version number is hardcoded. No service may define its own copy.
- Migration classes under `metadata/migrations/` are added, never removed or reordered.
- Event POJOs (`AssetUploadedEvent`, `ExtractionCompletedEvent`, etc.) come from `mi-data-intelligence`. Do not redefine or shadow them here.

---

## 4b. S3 Storage Layout

S3 (MinIO for local dev) is already in the iQ Key Value stack. VenueMi adds its own prefix namespace inside the shared bucket (`iqkv-files`).

### Bucket Strategy

| Environment | Bucket           | Notes                                                                                     |
| ----------- | ---------------- | ----------------------------------------------------------------------------------------- |
| Dev / CI    | `iqkv-files`     | Shared with foundation services, MinIO default. VIP objects live under `venuemi/` prefix. |
| Staging     | `iqkv-files`     | Same shared bucket, same prefix scheme. Isolated by prefix only.                          |
| Production  | `iqkv-vip-files` | Dedicated bucket. Separate IAM policy, separate lifecycle rules. Key structure identical. |

MinIO in local dev is configured in `docker-compose.yml` with `MINIO_DEFAULT_BUCKETS=iqkv-files`.

### Key Naming Convention

#### Tenant Asset Files

```
venuemi/tenants/{tenantKey}/venues/{venueId}/assets/{assetId}/{fileName}
```

| Segment       | Value                                         | Example                            |
| ------------- | --------------------------------------------- | ---------------------------------- |
| `venuemi/`    | VIP namespace                                 | (literal)                          |
| `tenants/`    | Tenant subtree root                           | (literal)                          |
| `{tenantKey}` | 8-char nanoid from JWT `tenant_id` claim      | `acme0001`                         |
| `venues/`     | Venue subtree                                 | (literal)                          |
| `{venueId}`   | UUID of the venue (no hyphens — compact form) | `550e8400e29b41d4a716446655440000` |
| `assets/`     | Asset subtree                                 | (literal)                          |
| `{assetId}`   | UUID of the asset (no hyphens)                | `6ba7b8109dad11d180b400c04fd430c8` |
| `{fileName}`  | Original file name, URL-safe, max 255 chars   | `grand-ballroom-deck.pdf`          |

Full example:

```
venuemi/tenants/acme0001/venues/550e8400e29b41d4a716446655440000/assets/6ba7b8109dad11d180b400c04fd430c8/grand-ballroom-deck.pdf
```

**Key rules:**

- `{fileName}` is the original client-supplied file name, stripped of path separators, URL-encoded where necessary. Stored as-is after sanitisation — no UUID substitution — so the key stays human-readable.
- `{venueId}` and `{assetId}` are UUIDs without hyphens (compact 32-char hex).
- The `asset_id` is generated server-side at `POST /assets/initiate` and written to `venue_assets.id` before the presigned URL is issued. The S3 key is stored in `venue_assets.s3_key` — at confirm time no key re-computation happens.

#### Presigned URL Issuance

`POST /assets/initiate` response:

```json
{
  "asset_id": "<uuid>",
  "upload_url": "https://minio.local/iqkv-files/venuemi/tenants/acme0001/venues/.../grand-ballroom-deck.pdf?X-Amz-Signature=...",
  "expires_at": "2025-06-01T12:15:00Z"
}
```

The presigned PUT URL is scoped to the exact key. The client uploads directly. The service never proxies the file body. Download presigned URLs (1h TTL) are generated on-demand at `GET /assets/{id}/download-url` — not stored.

#### Master Catalog Import Files

```
venuemi/master-catalog/imports/{importId}/{fileName}
venuemi/master-catalog/exports/{date}/{snapshot}.jsonl.gz
```

| Path                                         | Purpose                                                                           |
| -------------------------------------------- | --------------------------------------------------------------------------------- |
| `venuemi/master-catalog/imports/{importId}/` | One folder per import batch (admin-triggered). Contains raw CSV/JSON input files. |
| `venuemi/master-catalog/exports/{date}/`     | Nightly compacted snapshots of `public.master_venue` for downstream consumers.    |

### Tenant Isolation

- All tenant objects scoped under `venuemi/tenants/{tenantKey}/`. Cross-tenant read is structurally impossible without knowing the other tenant's key.
- The service account holds a single S3 IAM policy allowing `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on the full `venuemi/*` prefix. Presigned URLs are scoped to the exact object key.
- Master catalog paths (`venuemi/master-catalog/*`) are not accessible via tenant-issued presigned URLs. Written only by the platform's internal job service account (`mi-mc-loader`).

### Lifecycle Rules

| Rule                             | Prefix                                      | Action                                                                                                 |
| -------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Extraction artefact expiry       | `venuemi/tenants/*/venues/*/assets/*/`      | Transition to Glacier/IA after 90 days if `extraction_status = COMPLETED`. Managed via S3 object tags. |
| Master catalog import cleanup    | `venuemi/master-catalog/imports/processed/` | Delete after 30 days.                                                                                  |
| Master catalog snapshot rotation | `venuemi/master-catalog/exports/`           | Keep last 14 daily snapshots; delete older.                                                            |

Object tags set by `mi-venue-service` at `POST /assets/confirm`:

| Tag key             | Values                                          | Set by                             |
| ------------------- | ----------------------------------------------- | ---------------------------------- |
| `extraction_status` | `pending`, `completed`, `failed`                | mi-venue-service at confirm/update |
| `asset_type`        | `pdf_deck`, `floor_plan`, `photo`, `cad_file` … | mi-venue-service at initiate       |
| `tenant_key`        | 8-char nanoid                                   | mi-venue-service at initiate       |

### Deletion Cascade

When a tenant deletes an asset (`DELETE /assets/{id}`) or when a tenant account is terminated:

1. `mi-venue-service` deletes the `venue_assets` row (DB cascade drops extraction jobs and metadata events).
2. `mi-venue-service` issues `s3:DeleteObject` for `venue_assets.s3_key`.
3. `asset.deleted` event published → `mi-venue-processing-worker` deletes all `item_vectors` rows where `metadata->>'asset_id' = :assetId`.

For full tenant deletion (GDPR right to erasure):

1. `DELETE FROM t_{tenantKey}.venues` cascades to all asset rows.
2. A `tenant.deleted` event triggers a background S3 sweep: `s3:DeleteObjects` with all keys matching `venuemi/tenants/{tenantKey}/*` (batched in 1000-object chunks).
3. The pgvector sweep deletes all `item_vectors` rows for the tenant schema (schema drop handles this implicitly if the schema is dropped).

---

## 4c. Shared Library — `mi-data-intelligence`

`mi-data-intelligence` is a plain Java library (JAR, no Spring Boot). It is the **domain-agnostic, vertical-independent layer** — containing everything that would be reused verbatim if the platform were applied to a different vertical (medical records, agro assets, legal documents, etc.). Neither venue-specific fields nor venue-specific migration logic belong here.

**Contents:**

```
mi-data-intelligence/
├── extraction/
│   ├── ExtractionJob.java          Plain POJO — AI processing job record
│   ├── ExtractionStatus.java       enum: QUEUED, PROCESSING, COMPLETED, FAILED
│   └── ExtractorType.java          enum: TIKA_TEXT, GPT4O_DOCUMENT, GPT4O_VISION
├── asset/
│   └── AssetType.java              enum: PDF_DECK, FLOOR_PLAN, PHOTO, VIDEO, CAD_FILE, SPEC_SHEET, MISC
├── metadata/
│   ├── MetadataSource.java         Provenance per field — generic structure, field names are strings
│   ├── MetadataEventType.java      enum: ASSET_EXTRACTED, MANUAL_OVERRIDE, BULK_IMPORT, MC_INHERIT, SCRAPE_PROVIDER
│   ├── MetadataSchemaVersion.java  Versioning contract: defines _schema_version semantics and absent-key fallback (→ 0)
│   ├── MetadataMigration.java      Interface: fromVersion(), toVersion(), apply(JsonNode, ObjectMapper)
│   └── MetadataMigrator.java       Generic chain runner — pure Java, zero Spring deps
├── typehandler/
│   └── MetadataTypeHandler.java    Abstract MyBatis TypeHandler base — applies MetadataMigrator on every DB read
├── events/
│   ├── AssetUploadedEvent.java     asset_id, item_id, tenant_id, asset_type, s3_key, content_type
│   ├── ExtractionStartedEvent.java job_id, asset_id, item_id, tenant_id
│   ├── ExtractionCompletedEvent.java job_id, asset_id, item_id, tenant_id
│   └── ExtractionFailedEvent.java  job_id, asset_id, item_id, tenant_id, reason
└── db/
    └── changelog/
        └── intelligence/
            ├── master.xml
            ├── 20260901000003-create-extraction-jobs.xml
            ├── 20260901000004-create-item-metadata-events.xml
            ├── 20260901000005-create-item-vectors.xml
            └── 20260901000006-create-ai-cost-tracking.xml
```

**Rules:**

- No `@Service`, `@Repository`, `@Component`, or any Spring bean annotation.
- No domain-specific field names. `MetadataSource` stores field names as plain `String` keys — the canonical field set is defined in `mi-venue-model`.
- No venue, medical, agro, or any other vertical concept. If a class name contains a vertical noun, it does not belong here.
- `MetadataMigrator` (the chain runner) is here. Concrete `MetadataMigrationV{N}ToV{N+1}` classes are in the domain library.
- `MetadataTypeHandler` is an abstract base class. Domain libraries extend it once, passing their concrete migrator and target POJO class.
- Event POJOs use `item_id` as the generic field name. Domain services map their aggregate root ID (`venue_id`) to `item_id` when publishing and back when consuming.
- Infrastructure Liquibase changelogs live here so that `extraction_jobs`, `item_vectors`, `item_metadata_events`, and `ai_cost_tracking` tables are created identically regardless of which vertical is deployed.

**What does NOT go here — common mistakes to avoid at code review:**

| Tempting addition                             | Why it does not belong                              |
| --------------------------------------------- | --------------------------------------------------- |
| `VenueMetadata` or any domain POJO            | Venue-specific — lives in `mi-venue-model`          |
| `CURRENT_SCHEMA_VERSION = 1` constant         | Domain-version-specific — lives in domain library   |
| `MetadataMigrationV0ToV1`                     | Venue field renames — domain migration, not generic |
| `MasterVenue`                                 | Field shape is venue-specific — domain library      |
| `capacity`, `catering`, `av_tech` field names | Canonical field set — domain library                |
| Extraction prompt templates                   | Domain config — externalised per vertical           |

### Dependency Graph

```
mi-data-intelligence   (platform library, no runtime, no vertical deps)
      │
      ▼
mi-venue-model         (venue-domain library, imports mi-data-intelligence)
      │
      ├── mi-venue-service          (Spring Boot)
      └── mi-venue-processing-worker (Spring Boot)


Future vertical example:

mi-data-intelligence
      │
      ▼
mi-med-model           (medical-domain library)
      │
      ├── mi-med-service
      └── mi-med-processing-worker
```

---

**Docs:** [Architecture Index](README.md) · [Architecture Overview](architecture-overview.md) · [Data Model](data-model.md) · [ETL Pipeline](etl-pipeline.md) · [Events](events.md) · [Aggregation](aggregation.md) · [Master Catalog](master-catalog.md)
