# Venue Intelligence Platform (IQ BENE) — Architecture Reference

> **Audience:** Engineers, architects.
> **Purpose:** Single source of truth for all technical decisions before development starts.

**Docs:** [What is IQ BENE?](what-is-vip.md) · [Business Overview](business-overview.md) · [Competitive Landscape](intelligence-and-competitive-landscape.md) · [Architecture](architecture.md)

---

## 1. Platform Context

IQ BENE is a new product service built **on top of the IQKV foundation**. It does not replace or fork any existing service. It reuses:

| Foundation service           | What IQ BENE inherits                                                                   |
| ---------------------------- | --------------------------------------------------------------------------------------- |
| `foundation-gateway-service` | JWT validation, tenant routing, header propagation — no changes needed                  |
| `foundation-iam-service`     | Auth, multi-tenancy, team invitations, SSO, presigned S3 upload pattern                 |
| `foundation-billing-service` | Plan entitlements (`max_venues`, `ai_extraction_enabled`, etc.), subscription lifecycle |
| `foundation-audit-service`   | Compliance log — consumes IQ BENE events passively, no code changes                     |
| `foundation-ui-app`          | Extended (not forked) with new `/venues/*` routes under FSD architecture                |
| `foundation-tenancy`         | Schema-per-tenant isolation library reused directly                                     |

**New services introduced by IQ BENE:**

- `vip-venue-model` — shared library (JAR). Canonical domain model, event contracts, enums, and Liquibase migrations. No Spring beans, no business logic — pure model and schema. Imported by both services.
- `vip-venue-service` — core domain: venues, assets, metadata, search, plan enforcement. Synchronous request/response only.
- `vip-venue-ingestion-worker` — async sidecar: document ETL pipeline, extraction orchestration, embedding generation, scheduled jobs. No inbound HTTP — event-driven only. Shares the same PostgreSQL schema as `vip-venue-service`.

**New infrastructure introduced by IQ BENE:**

- pgvector extension on existing PostgreSQL (not a new service)
- PostGIS extension on existing PostgreSQL (not a new service)
- IBM Docling (optional self-hosted container, Phase 2 only)

---

## 2. Domain Model

### Bounded Contexts

#### `venue/` — Core Profile

**Aggregate root: `Venue`**

| Field                    | Type             | Notes                                      |
| ------------------------ | ---------------- | ------------------------------------------ |
| `id`                     | UUID             | PK                                         |
| `name`                   | varchar(255)     |                                            |
| `address`                | text             |                                            |
| `location`               | geography(point) | PostGIS, lat/lng                           |
| `description`            | text             | Human-written or AI-drafted                |
| `status`                 | enum             | `DRAFT`, `ACTIVE`, `ARCHIVED`              |
| `metadata`               | jsonb            | Consolidated extracted + manual fields     |
| `metadata_sources`       | jsonb            | Provenance per field (see §4)              |
| `metadata_aggregated_at` | timestamp        | When consolidation last ran                |
| `description_embedding`  | vector(1536)     | pgvector, for semantic search              |
| `description_text`       | tsvector         | Auto-updated via trigger, full-text search |
| `created_by`             | UUID             | IAM user id                                |
| `created_at`             | timestamp        |                                            |
| `updated_at`             | timestamp        |                                            |

**Operations:** create, update, archive, restore.

---

#### `asset/` — File Attachments

**Aggregate root: `VenueAsset`**

| Field                      | Type         | Notes                                                                        |
| -------------------------- | ------------ | ---------------------------------------------------------------------------- |
| `id`                       | UUID         | PK                                                                           |
| `venue_id`                 | UUID         | FK → venues                                                                  |
| `asset_type`               | enum         | `PDF_DECK`, `FLOOR_PLAN`, `PHOTO`, `VIDEO`, `CAD_FILE`, `SPEC_SHEET`, `MISC` |
| `file_name`                | varchar(255) |                                                                              |
| `content_type`             | varchar(100) | MIME type                                                                    |
| `size_bytes`               | bigint       |                                                                              |
| `s3_key`                   | text         | Storage path                                                                 |
| `extracted_text`           | text         | Raw text extracted by parser                                                 |
| `extracted_text_embedding` | vector(1536) | pgvector, chunk-level search                                                 |
| `extraction_status`        | enum         | `PENDING`, `IN_PROGRESS`, `COMPLETED`, `FAILED`                              |
| `uploaded_by`              | UUID         |                                                                              |
| `uploaded_at`              | timestamp    |                                                                              |

**Upload flow:** two-phase presigned URL (same pattern as IAM avatar upload).

1. `POST /assets/initiate` → returns presigned S3 PUT URL + `asset_id`
2. Client uploads directly to S3
3. `POST /assets/{id}/confirm` → marks asset ready, publishes `asset.uploaded` event

---

#### `extraction/` — AI Processing Jobs

**Aggregate root: `ExtractionJob`**

| Field               | Type      | Notes                                         |
| ------------------- | --------- | --------------------------------------------- |
| `id`                | UUID      | PK                                            |
| `asset_id`          | UUID      | FK → venue_assets                             |
| `status`            | enum      | `QUEUED`, `PROCESSING`, `COMPLETED`, `FAILED` |
| `extractor_type`    | enum      | `TIKA_TEXT`, `GPT4O_DOCUMENT`, `GPT4O_VISION` |
| `extracted_data`    | jsonb     | Raw extraction result                         |
| `confidence_scores` | jsonb     | Per-field confidence (0.0–1.0)                |
| `started_at`        | timestamp |                                               |
| `completed_at`      | timestamp |                                               |
| `error_message`     | text      | On failure                                    |

---

#### `metadata_events/` — Event Log (for aggregation)

**Not an aggregate root — append-only event log.**

| Field         | Type      | Notes                                               |
| ------------- | --------- | --------------------------------------------------- |
| `id`          | UUID      | PK                                                  |
| `venue_id`    | UUID      | FK → venues                                         |
| `event_type`  | enum      | `ASSET_EXTRACTED`, `MANUAL_OVERRIDE`, `BULK_IMPORT` |
| `source_type` | enum      | `PDF_DECK`, `FLOOR_PLAN`, `PHOTO`, `USER_INPUT`     |
| `source_id`   | UUID      | asset_id or user_id                                 |
| `event_data`  | jsonb     | Fields with values and confidence scores            |
| `occurred_at` | timestamp |                                                     |
| `created_by`  | UUID      |                                                     |

---

### Metadata Schema (JSONB)

The `venues.metadata` column stores the **consolidated view** of all extracted and manually entered data. The `venues.metadata_sources` column stores **provenance** per field.

**Canonical field set:**

```
capacity
  └─ max_total (int)
  └─ configurations: banquet, theater, classroom, cocktail, conference (int each)

venue_type (string[])         e.g. ["conference_center", "hotel_ballroom"]

location_notes (string)       e.g. "3 blocks from Grand Central"

catering
  └─ policy (enum)            in_house_exclusive | in_house_preferred | outside_allowed | no_catering
  └─ kosher_available (bool)
  └─ halal_available (bool)

av_tech
  └─ built_in_av (bool)
  └─ projector_lumens (int)
  └─ screens (int)
  └─ rigging_points (bool)
  └─ internet_bandwidth_mbps (int)

accessibility
  └─ ada_compliant (bool)
  └─ elevator_access (bool)
  └─ accessible_restrooms (bool)
  └─ wheelchair_stage_access (bool)

logistics
  └─ load_in_access (string)
  └─ parking_spaces (int)
  └─ valet_available (bool)
  └─ curfew_time (time)

restrictions (string[])       e.g. ["no_open_flame", "no_confetti"]

amenities (string[])          e.g. ["WiFi", "AV_equipment", "parking"]

contacts (object[])
  └─ name, role, email, phone

pricing
  └─ minimum_spend (int)
  └─ currency (string)
  └─ rental_fee_indicative (int)
```

**Provenance per field** (stored in `metadata_sources`):

```json
"capacity.max_total": {
  "value": 500,
  "confidence": 0.94,
  "source_type": "PDF_DECK",
  "source_id": "<asset-uuid>",
  "updated_at": "...",
  "alternatives": [
    { "value": 480, "confidence": 0.65, "source_type": "FLOOR_PLAN" }
  ]
}
```

---

## 3. Metadata Aggregation

Multiple assets per venue produce multiple extraction events, potentially with conflicting values. The aggregation service resolves conflicts and maintains the consolidated `metadata` column.

### Conflict Resolution Priority

```
MANUAL_OVERRIDE     → always wins (user explicitly set this)
VERIFIED_EXTRACTION → admin confirmed the AI result
HIGH_CONFIDENCE_AI  → confidence ≥ 0.9
MEDIUM_CONFIDENCE_AI→ confidence 0.7–0.9
LOW_CONFIDENCE_AI   → confidence < 0.7
```

### Array Fields (amenities, restrictions)

Set-union across all sources. An entry is included if at least one source reports it with confidence ≥ 0.6.

### Trigger Points

Aggregation runs (async, via RabbitMQ) when:

- An `asset.uploaded` event triggers extraction → extraction completes → `extraction.completed` triggers aggregation
- A user submits a manual override → immediate re-aggregation
- A scheduled job catches stale venues (24h without re-aggregation)

Aggregation is debounced (5s) to batch rapid successive events.

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
                    │              vip-venue-service              │
                    │  venues · assets · metadata · search · api │
                    └────────┬─────────────────────────┬─────────┘
                             │ RabbitMQ: asset.uploaded │ read/write
                    ┌────────▼─────────┐     ┌─────────▼────────┐
                    │ vip-ingestion-   │     │   PostgreSQL      │
                    │    worker        │────►│   + pgvector      │
                    │ (async sidecar)  │     │   + PostGIS       │
                    └──────────────────┘     └──────────────────┘
                             │
                    ┌────────▼─────────┐
                    │   S3 / MinIO     │  (existing)
                    └──────────────────┘
```

### vip-venue-service

- **Responsibilities:** venue CRUD, asset upload flow (presigned URL), metadata read/write, search API, plan entitlement enforcement
- **Database:** owns the IQ BENE PostgreSQL schema (schema-per-tenant via `foundation-tenancy`). Shared with `vip-venue-ingestion-worker` — no cross-service API calls for data.
- **Exposes:** REST API at `/api/v1/venues`
- **Publishes:** `venue.created`, `venue.updated`, `asset.uploaded`, `asset.deleted` (RabbitMQ)
- **Consumes:** `extraction.completed`, `extraction.failed` (RabbitMQ) — triggers metadata aggregation

### vip-venue-ingestion-worker

- **Responsibilities:** document ETL pipeline (parse → chunk → extract → embed), extraction job lifecycle, metadata aggregation, scheduled maintenance jobs (stale re-aggregation, cost reporting)
- **Nature:** async sidecar — no inbound HTTP, no REST API, no service discovery entry. Event-driven only.
- **Database:** shared PostgreSQL schema with `vip-venue-service`. Reads `venue_assets`, writes `extraction_jobs`, `venue_metadata_events`, `venue_vectors`, `ai_cost_tracking`.
- **Consumes:** `asset.uploaded` (RabbitMQ) — triggers ETL pipeline
- **Publishes:** `extraction.started`, `extraction.completed`, `extraction.failed` (RabbitMQ)
- **External calls:** OpenAI API (GPT-4o, text-embedding-3-small), optionally Docling sidecar (Phase 2)
- **Scaling:** replicas scaled independently based on RabbitMQ queue depth — no impact on `vip-venue-service`

### Table Ownership

Both services share one PostgreSQL schema. Ownership defines who may write to a table. Cross-boundary reads are permitted; cross-boundary writes are not.

| Table                   | Owner                        | The other service may…                                                       |
| ----------------------- | ---------------------------- | ---------------------------------------------------------------------------- |
| `venues`                | `vip-venue-service`          | read (ingestion-worker: resolve venue_id only)                               |
| `venue_assets`          | `vip-venue-service`          | read (ingestion-worker: fetch asset for processing)                          |
| `venue_metadata_events` | `vip-venue-service`          | write via event reaction (`extraction.completed` → venue-service aggregates) |
| `extraction_jobs`       | `vip-venue-ingestion-worker` | read (venue-service: expose job status to API)                               |
| `venue_vectors`         | `vip-venue-ingestion-worker` | read (venue-service: vector search queries)                                  |
| `ai_cost_tracking`      | `vip-venue-ingestion-worker` | read (venue-service: expose cost summary to API)                             |

The single legitimate cross-boundary read from `vip-venue-ingestion-worker` is a `SELECT` on `venue_assets` by `asset_id` (delivered in the `asset.uploaded` event payload). This is a foreign key lookup, not business logic — acceptable and intentional.

---

## 4a. Shared Library — vip-venue-model

`vip-venue-model` is a plain Java library (JAR, no Spring Boot, no `@SpringBootApplication`). Both `vip-venue-service` and `vip-venue-ingestion-worker` declare it as a compile dependency. It is the single source of truth for anything both services need to agree on.

**Contents:**

```
vip-venue-model/
├── model/
│   ├── venue/
│   │   ├── Venue.java                  JPA entity (aggregate root)
│   │   ├── VenueStatus.java            enum: DRAFT, ACTIVE, ARCHIVED
│   │   └── VenueAsset.java             JPA entity
│   ├── asset/
│   │   ├── AssetType.java              enum: PDF_DECK, FLOOR_PLAN, PHOTO, CAD_FILE…
│   │   └── ExtractionStatus.java       enum: PENDING, IN_PROGRESS, COMPLETED, FAILED
│   ├── extraction/
│   │   ├── ExtractionJob.java          JPA entity
│   │   ├── ExtractorType.java          enum: TIKA_TEXT, GPT4O_DOCUMENT, GPT4O_VISION
│   │   └── VenueMetadataEvent.java     JPA entity (append-only event log)
│   ├── metadata/
│   │   ├── VenueMetadata.java          value object (mirrors venues.metadata JSONB)
│   │   ├── VenueCapacity.java          capacity configurations value object
│   │   ├── MetadataSource.java         provenance per field
│   │   └── MetadataEventType.java      enum: ASSET_EXTRACTED, MANUAL_OVERRIDE, BULK_IMPORT
│   └── events/                         RabbitMQ message contracts (POJOs, no framework deps)
│       ├── AssetUploadedEvent.java
│       ├── ExtractionStartedEvent.java
│       ├── ExtractionCompletedEvent.java
│       └── ExtractionFailedEvent.java
└── db/
    └── changelog/
        └── tenant/                     Liquibase migrations — single source of truth
            ├── 001-venues.sql
            ├── 002-venue-assets.sql
            ├── 003-extraction-jobs.sql
            ├── 004-metadata-events.sql
            ├── 005-venue-vectors.sql
            └── 006-ai-cost-tracking.sql
```

**Rules:**

- No `@Service`, `@Repository`, `@Component`, or any Spring bean annotation
- No business logic — entities, value objects, enums, event POJOs only
- JPA annotations on entities are acceptable (`@Entity`, `@Table`, `@Column` etc.)
- Liquibase migrations live here so schema changes are a compile-time dependency bump, not a coordination exercise between services
- Changing an event POJO field is a compile-time break in both services — intentional, prevents silent contract drift

**Dependency graph:**

```
vip-venue-model  (library, no runtime)
      ├── vip-venue-service     (Spring Boot, imports model)
      └── vip-venue-ingestion-worker  (Spring Boot, imports model)
```

---

## 5. ETL Pipeline (vip-venue-ingestion-worker)

Built on **Spring AI's ETL framework**. Three composable stages:

```
DocumentReader  →  DocumentTransformer  →  DocumentWriter
  (parse)            (chunk + enrich)        (embed + store)
```

### Stage 1 — Parse (per asset type)

| Asset type                   | Reader                                    | Notes                                   |
| ---------------------------- | ----------------------------------------- | --------------------------------------- |
| PDF (text-based)             | `TikaDocumentReader`                      | Apache Tika, ships with Spring AI       |
| PDF (scanned)                | `TikaDocumentReader` + Tesseract OCR      | Tika bundles OCR                        |
| PDF (complex layout, tables) | Docling sidecar → custom `DocumentReader` | Phase 2; better table fidelity          |
| DOCX / XLSX / PPTX           | `TikaDocumentReader`                      | Same reader, 1000+ formats              |
| Images (JPG, PNG)            | GPT-4o vision direct                      | No text reader needed                   |
| Floor plan (PDF/image)       | Docling layout-aware → GPT-4o vision      | Phase 2                                 |
| DWG / DXF (CAD)              | `TikaDocumentReader` (AutoCAD parser)     | Extracts metadata; visual in Phase 2    |
| Video                        | Out of scope Phase 1                      | Phase 2: keyframe extraction via ffmpeg |

### Stage 2 — Transform

1. **Chunk** — `TokenTextSplitter` (512 tokens, 50-token overlap). Spec-sheet tables use 256-token chunks to preserve row precision.
2. **Tag** — attach `venue_id`, `asset_id`, `asset_type`, `tenant_id` as Document metadata.
3. **Extract** — `VenueMetadataEnricher` (custom `DocumentTransformer`): calls GPT-4o with structured output schema, returns `VenueMetadata` POJO with confidence scores per field.

### Stage 3 — Load

1. **Embed** — `EmbeddingModel` (`text-embedding-3-small`, 1536 dims).
2. **Store** — `TenantAwarePgVectorStore` writes chunks + embeddings to `venue_vectors` table in the tenant's schema.
3. **Aggregate** — publishes `extraction.completed` event → `MetadataAggregationConsumer` updates `venues.metadata`.

### Processing SLA

| Asset type           | Target latency |
| -------------------- | -------------- |
| PDF / DOCX (text)    | < 30s          |
| Images / floor plans | < 60s          |
| CAD files            | < 2 min        |

Retry on failure: 3 attempts with exponential backoff. After 3 failures → `extraction.failed` event → user notification.

---

## 6. Search Architecture

All search is served by `vip-venue-service` querying PostgreSQL directly. No separate search service.

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
- Upgrade path: switch to HNSW when tenant exceeds ~500K venues (rare)

---

## 7. API Surface (vip-venue-service)

Base path: `/api/v1/venues`

### Venues

| Method   | Path      | Auth       | Description                                  |
| -------- | --------- | ---------- | -------------------------------------------- |
| `GET`    | `/`       | JWT Member | List venues (paginated, filterable)          |
| `POST`   | `/`       | JWT Member | Create venue profile                         |
| `GET`    | `/{id}`   | JWT Member | Get venue with consolidated metadata         |
| `PATCH`  | `/{id}`   | JWT Admin  | Update venue fields                          |
| `DELETE` | `/{id}`   | JWT Owner  | Archive venue                                |
| `GET`    | `/search` | JWT Member | Hybrid search (keyword + semantic + filters) |

### Assets

| Method   | Path                             | Auth       | Description                             |
| -------- | -------------------------------- | ---------- | --------------------------------------- |
| `POST`   | `/{venueId}/assets/initiate`     | JWT Member | Start upload — returns presigned S3 URL |
| `POST`   | `/{venueId}/assets/{id}/confirm` | JWT Member | Confirm upload, trigger extraction      |
| `GET`    | `/{venueId}/assets`              | JWT Member | List assets for venue                   |
| `DELETE` | `/{venueId}/assets/{id}`         | JWT Member | Delete asset and S3 object              |

### Metadata

| Method | Path                                   | Auth       | Description                            |
| ------ | -------------------------------------- | ---------- | -------------------------------------- |
| `GET`  | `/{venueId}/metadata`                  | JWT Member | Get consolidated metadata + provenance |
| `POST` | `/{venueId}/metadata/{field}/override` | JWT Member | Manual override for a field            |
| `GET`  | `/{venueId}/metadata/{field}/history`  | JWT Member | Extraction event history for field     |

### Extraction Jobs (read-only for clients)

| Method | Path                             | Auth       | Description                    |
| ------ | -------------------------------- | ---------- | ------------------------------ |
| `GET`  | `/{venueId}/extractions`         | JWT Member | List extraction jobs for venue |
| `GET`  | `/{venueId}/extractions/{jobId}` | JWT Member | Get job status and result      |

---

## 8. Event Contracts (RabbitMQ)

Exchange: `iqkv.events` (Topic) — same exchange used by all foundation services.

### Published by vip-venue-service

| Routing key      | Payload fields                                                  | Description                           |
| ---------------- | --------------------------------------------------------------- | ------------------------------------- |
| `venue.created`  | venue_id, tenant_id, created_by                                 | New venue profile created             |
| `venue.updated`  | venue_id, tenant_id, changed_fields                             | Venue fields updated                  |
| `asset.uploaded` | asset_id, venue_id, tenant_id, asset_type, s3_key, content_type | Asset confirmed, ready for extraction |
| `asset.deleted`  | asset_id, venue_id, tenant_id                                   | Asset removed                         |

### Published by vip-venue-ingestion-worker

| Routing key            | Payload fields                                | Description           |
| ---------------------- | --------------------------------------------- | --------------------- |
| `extraction.started`   | job_id, asset_id, venue_id, tenant_id         | Processing began      |
| `extraction.completed` | job_id, asset_id, venue_id, tenant_id         | Extraction succeeded  |
| `extraction.failed`    | job_id, asset_id, venue_id, tenant_id, reason | All retries exhausted |

### Consumed by vip-venue-ingestion-worker

| Routing key      | Queue                                  | Action                                |
| ---------------- | -------------------------------------- | ------------------------------------- |
| `asset.uploaded` | `vip.extraction.priority` (Enterprise) | Trigger ETL pipeline immediately      |
| `asset.uploaded` | `vip.extraction.standard` (Free/Pro)   | Trigger ETL pipeline (standard queue) |

### Consumed by vip-venue-service

| Routing key            | Queue                      | Action                                |
| ---------------------- | -------------------------- | ------------------------------------- |
| `extraction.completed` | `vip.metadata.aggregation` | Run metadata aggregation for venue    |
| `extraction.failed`    | `vip.extraction.dlq`       | Mark asset extraction_status = FAILED |

### Consumed by foundation-audit-service (passive, no changes)

| Routing key                          | Notes                                                      |
| ------------------------------------ | ---------------------------------------------------------- |
| `venue.#`, `asset.#`, `extraction.#` | Automatically captured by audit service's wildcard binding |

---

## 9. Plan Entitlement Mapping

Feature codes used in `foundation-billing-service` plan config:

| Feature code             | Free | Pro | Enterprise | Enforcement point                    |
| ------------------------ | ---- | --- | ---------- | ------------------------------------ |
| `max_venues`             | 10   | 500 | unlimited  | venue-service: before create         |
| `max_assets_per_venue`   | 20   | 100 | unlimited  | venue-service: before upload         |
| `basic_extraction`       | ✅   | ✅  | ✅         | ai-service: PDF text only            |
| `advanced_extraction`    | ⛔   | ✅  | ✅         | ai-service: all asset types          |
| `cad_support`            | ⛔   | ✅  | ✅         | venue-service: reject DWG/DXF upload |
| `semantic_search`        | ⛔   | ✅  | ✅         | venue-service: search endpoint       |
| `priority_ai_processing` | ⛔   | ⛔  | ✅         | RabbitMQ: route to priority queue    |
| `api_access`             | ⛔   | ✅  | ✅         | gateway: API key route               |
| `white_label`            | ⛔   | ⛔  | ✅         | ui-app: branding config              |

Enforcement via `PlanFeatureGuard` (same pattern as IAM service's existing implementation).

---

## 10. Database Schema (Liquibase, tenant schema)

Migrations live in `vip-venue-model` under `src/main/resources/db/changelog/tenant/` — the shared library is the single source of truth for schema. Both `vip-venue-service` and `vip-venue-ingestion-worker` include the library on their classpath; `vip-venue-service` runs the migrations on startup (or a dedicated init container applies them on tenant provisioning via `TenantProvisionedEvent` listener, same pattern as IAM).

```sql
-- extensions (applied once per tenant schema)
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS postgis;

-- venues
CREATE TABLE venues (
  id                      UUID PRIMARY KEY,
  name                    VARCHAR(255) NOT NULL,
  address                 TEXT,
  location                GEOGRAPHY(POINT, 4326),
  description             TEXT,
  description_embedding   VECTOR(1536),
  description_text        TSVECTOR,
  status                  VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
  metadata                JSONB NOT NULL DEFAULT '{}',
  metadata_sources        JSONB NOT NULL DEFAULT '{}',
  metadata_aggregated_at  TIMESTAMP,
  created_by              UUID NOT NULL,
  created_at              TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at              TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_venues_embedding  ON venues USING ivfflat (description_embedding vector_cosine_ops);
CREATE INDEX idx_venues_fts        ON venues USING GIN (description_text);
CREATE INDEX idx_venues_metadata   ON venues USING GIN (metadata jsonb_path_ops);
CREATE INDEX idx_venues_location   ON venues USING GIST (location);

CREATE TRIGGER trg_venues_tsvector BEFORE INSERT OR UPDATE ON venues
  FOR EACH ROW EXECUTE FUNCTION
  tsvector_update_trigger(description_text, 'pg_catalog.english', name, description, address);

-- venue_assets
CREATE TABLE venue_assets (
  id                       UUID PRIMARY KEY,
  venue_id                 UUID NOT NULL REFERENCES venues(id) ON DELETE CASCADE,
  asset_type               VARCHAR(50) NOT NULL,
  file_name                VARCHAR(255) NOT NULL,
  content_type             VARCHAR(100) NOT NULL,
  size_bytes               BIGINT NOT NULL,
  s3_key                   TEXT NOT NULL,
  extracted_text           TEXT,
  extracted_text_embedding VECTOR(1536),
  extraction_status        VARCHAR(20) NOT NULL DEFAULT 'PENDING',
  uploaded_by              UUID NOT NULL,
  uploaded_at              TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_assets_venue       ON venue_assets (venue_id);
CREATE INDEX idx_assets_type        ON venue_assets (asset_type);
CREATE INDEX idx_assets_embedding   ON venue_assets USING ivfflat (extracted_text_embedding vector_cosine_ops);

-- extraction_jobs
CREATE TABLE extraction_jobs (
  id               UUID PRIMARY KEY,
  asset_id         UUID NOT NULL REFERENCES venue_assets(id) ON DELETE CASCADE,
  status           VARCHAR(20) NOT NULL,
  extractor_type   VARCHAR(50) NOT NULL,
  extracted_data   JSONB,
  confidence_scores JSONB,
  started_at       TIMESTAMP,
  completed_at     TIMESTAMP,
  error_message    TEXT
);
CREATE INDEX idx_jobs_asset  ON extraction_jobs (asset_id);
CREATE INDEX idx_jobs_status ON extraction_jobs (status);

-- venue_metadata_events (append-only)
CREATE TABLE venue_metadata_events (
  id           UUID PRIMARY KEY,
  venue_id     UUID NOT NULL REFERENCES venues(id) ON DELETE CASCADE,
  event_type   VARCHAR(50) NOT NULL,
  source_type  VARCHAR(50) NOT NULL,
  source_id    UUID,
  event_data   JSONB NOT NULL,
  occurred_at  TIMESTAMP NOT NULL DEFAULT NOW(),
  created_by   UUID
);
CREATE INDEX idx_metadata_events_venue  ON venue_metadata_events (venue_id, occurred_at DESC);
CREATE INDEX idx_metadata_events_source ON venue_metadata_events (source_id, source_type);

-- venue_vectors (Spring AI PgVectorStore table)
CREATE TABLE venue_vectors (
  id        UUID PRIMARY KEY,
  content   TEXT NOT NULL,
  metadata  JSONB,
  embedding VECTOR(1536)
);
CREATE INDEX idx_vectors_embedding ON venue_vectors USING ivfflat (embedding vector_cosine_ops);

-- ai_cost_tracking
CREATE TABLE ai_cost_tracking (
  id             UUID PRIMARY KEY,
  provider       VARCHAR(50) NOT NULL,
  operation_type VARCHAR(50) NOT NULL,
  model          VARCHAR(100) NOT NULL,
  tokens_used    INTEGER,
  cost_usd       NUMERIC(10, 6) NOT NULL,
  asset_id       UUID REFERENCES venue_assets(id),
  created_at     TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_ai_cost_month ON ai_cost_tracking (DATE_TRUNC('month', created_at));
```

---

## 11. UI Integration (foundation-ui-app)

Extend `foundation-ui-app` — do **not** fork. New IQ BENE features live under:

```
src/features/venue-management/
├── create-venue/
├── upload-asset/
├── view-venue/          (profile + metadata card + asset gallery)
├── edit-metadata/       (manual override, confidence badges, conflict alerts)
└── search-venues/       (search bar, filters, semantic results)
```

New routes added to TanStack Router:

| Path                   | Auth   | Description          |
| ---------------------- | ------ | -------------------- |
| `/venues`              | Member | Venue list / search  |
| `/venues/new`          | Member | Create venue         |
| `/venues/:id`          | Member | Venue profile        |
| `/venues/:id/assets`   | Member | Asset gallery        |
| `/venues/:id/metadata` | Member | Metadata view + edit |

Reuse without modification:

- Auth flows, session management, token refresh
- Team management (`/team`)
- Billing / entitlements (`FeatureGate`, `useEntitlements`)
- Notification bell (subscribe to `extraction.completed` notifications)

---

## 12. Observability

Both IQ BENE services follow foundation patterns exactly.

**Prometheus metrics to add:**

| Metric                            | Labels                            | Notes                       |
| --------------------------------- | --------------------------------- | --------------------------- |
| `vip_venues_total`                | tenant_id, status                 | Venue count by state        |
| `vip_assets_uploaded_total`       | tenant_id, asset_type             | Upload volume               |
| `vip_extractions_total`           | tenant_id, extractor_type, status | Success/failure rates       |
| `vip_extraction_duration_seconds` | extractor_type                    | Latency histogram           |
| `vip_ai_cost_usd_total`           | tenant_id, model                  | Cost tracking               |
| `vip_search_requests_total`       | search_mode                       | keyword / semantic / hybrid |
| `vip_search_latency_seconds`      | search_mode                       | Search latency              |

Grafana dashboard added to `docker/grafana/provisioning/dashboards/VipService.json`.

---

## 13. Security

| Concern                 | Approach                                                                                                                       |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Tenant data isolation   | Schema-per-tenant (PostgreSQL + pgvector); S3 key prefix per tenant                                                            |
| Asset access            | Presigned S3 URLs only (15 min upload, 1h download). No public bucket.                                                         |
| AI data handling        | Documents sent to OpenAI API per their data processing terms. Enterprise option: Azure OpenAI (data stays in tenant's region). |
| GDPR / right to erasure | `DELETE tenant` cascades to venues → assets → S3 objects → vector embeddings                                                   |
| Audit trail             | All `venue.*`, `asset.*`, `extraction.*` events passively consumed by Audit Service                                            |
| PII in documents        | Warn on upload. Do not log extracted text.                                                                                     |

---

## 14. Technology Decisions (summary)

| Concern             | Decision                                       | Rationale                                                                     |
| ------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------- |
| Document parsing    | Apache Tika via Spring AI `TikaDocumentReader` | 1000+ formats, DWG support, fault-tolerant Tika Pipes, zero extra infra       |
| PDF layout / tables | IBM Docling (Phase 2, self-hosted)             | State-of-the-art table reconstruction, MIT license, zero per-page cost        |
| AI framework        | Spring AI 1.0                                  | Java-native, provider-agnostic, ETL pipeline built-in, Micrometer integration |
| LLM (extraction)    | OpenAI GPT-4o                                  | Best structured output + multimodal (vision for images/floor plans)           |
| Embeddings          | OpenAI text-embedding-3-small                  | 1536 dims, $0.02/1M tokens, good quality/cost ratio                           |
| Vector store        | pgvector (PostgreSQL extension)                | No new service, transactional, tenant-isolated via schema                     |
| Full-text search    | PostgreSQL tsvector                            | Unified with relational data, no new service                                  |
| Geo search          | PostGIS (PostgreSQL extension)                 | No new service                                                                |
| Async processing    | RabbitMQ (existing foundation)                 | Priority queues, DLQ, already in platform                                     |
| File storage        | S3 / MinIO (existing foundation)               | Presigned URL pattern already proven in IAM                                   |

Full rationale and competitor analysis: see `venue-intelligence-platform-intelligence.md`.

---

## 15. Open Decisions (resolve before Sprint 1)

- [x] **One service or two?** ~~`vip-venue-service` + `vip-ai-service` vs. a single `vip-venue-service` with an internal AI module.~~ **Decided:** Two deployments — `vip-venue-service` (synchronous API, data-tied) and `vip-venue-ingestion-worker` (async sidecar, shared schema, no inbound HTTP). Services are tied to data; ingestion is a processing concern, not a peer service.
- [x] **Naming convention.** Service names reflect domain/purpose, not implementation technology. `vip-venue-ingestion-worker` describes what it does (ingest and process assets), not how (AI/ML).
- [ ] **Docling in Phase 1?** Start with pure Tika (simpler). Add Docling sidecar in Phase 2 when floor plan / table fidelity is needed. **Lean: Tika-only for Phase 1.**
- [ ] **Chunking table** in separate schema or same as venue tables? Spring AI's `PgVectorStore` defaults to a `vector_store` table. IQ BENE uses `venue_vectors` to be explicit. Confirm naming before first migration.
- [ ] **Cost tracking granularity:** per-asset or per-tenant-per-month? Both are in schema; decide which is surfaced in UI.

---

## 16. Next Steps & Design Iterations

### Before Sprint 1 — Resolve These First

- [ ] **Docling in Phase 1?** No — Tika-only for MVP. Add Docling sidecar in Phase 2 for floor plan / table fidelity.
- [ ] **MVP scope cut** — Phase 1 is: venue profiles, asset upload, basic extraction (PDF only), keyword + semantic search, team collaboration. Everything else is Phase 2+.

### Phase 2 Design (post-MVP signal)

- Conflict resolution UX — API shape + state machine for resolving competing extracted values
- Bulk import / CSV ingestion — for concierge onboarding and agency migrations
- Saved searches + alerts — schema and delivery mechanism
- Notification delivery — decide WebSocket vs. polling for extraction status (can reuse IAM's WebSocket infra)
- CAD visual extraction — convert DWG/DXF to image, then GPT-4o vision
- Video walkthroughs — keyframe extraction via ffmpeg, vision-based amenity detection

### Phase 3 Design

- Export / sharing — shareable branded links, PDF/Excel report generation, venue comparison view
- Deduplication — detect and merge duplicate venue records across teams
- Verification workflow — API + UX for promoting AI-extracted fields to "human-verified" status
- CRM integrations — Salesforce, HubSpot webhook connectors

### Cross-Cutting Concerns (design before Phase 2 builds)

- **AI resilience** — fallback behavior when OpenAI is unavailable; graceful degradation (queue for retry, notify user)
- **Per-tenant AI cost controls** — budget caps, monthly usage alerts, what happens at limit
- **Storage / retention policy** — S3 lifecycle rules, old embedding versions, extracted text retention (GDPR angle)
- **Testing strategy** — test doubles for OpenAI, fixture documents for ETL pipeline, contract tests between services

### Strategic Bets to Validate with First Users

- Is semantic search ("find venues like this one") actually used, or do planners prefer structured filters?
- What's the real extraction accuracy on real-world venue PDFs? Run benchmark on 50 sample documents before committing to accuracy claims.
- Is the "aha moment" the extraction result, or the search finding something instantly? Shapes onboarding flow design.
- Which asset type matters most to upload first — venue decks or floor plans? Informs parser priority.

---

**Docs:** [What is IQ BENE?](what-is-vip.md) · [Business Overview](business-overview.md) · [Competitive Landscape](intelligence-and-competitive-landscape.md) · [Architecture](architecture.md)
