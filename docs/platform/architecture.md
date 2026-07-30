# Venue Intelligence Platform (iQ BENE) — Architecture Reference

> **Audience:** Engineers, architects.
> **Purpose:** Single source of truth for all technical decisions before development starts.

**Docs:** [What is iQ BENE?](../README.md) · [Business Proposal](../business/proposal.md) · [Competitive Landscape](../business/comparison.md) · [Architecture](architecture.md)

---

## 1. Platform Context

iQ BENE is a new product service built **on top of the IQKV foundation**. It does not replace or fork any existing service. It reuses:

| Foundation service           | What iQ BENE inherits                                                                   |
| ---------------------------- | --------------------------------------------------------------------------------------- |
| `foundation-gateway-service` | JWT validation, tenant routing, header propagation — no changes needed                  |
| `foundation-iam-service`     | Auth, multi-tenancy, team invitations, SSO, presigned S3 upload pattern                 |
| `foundation-billing-service` | Plan entitlements (`max_venues`, `ai_extraction_enabled`, etc.), subscription lifecycle |
| `foundation-audit-service`   | Compliance log — consumes iQ BENE events passively, no code changes                     |
| `foundation-ui-app`          | Extended (not forked) with new `/venues/*` routes under FSD architecture                |
| `foundation-tenancy`         | Schema-per-tenant isolation library reused directly                                     |

**New services introduced by iQ BENE:**

- `iqbene-venue-model` — shared library (JAR). Canonical domain model, event contracts, enums, and Liquibase migrations. No Spring beans, no business logic — pure model and schema. Imported by both services.
- `iqbene-venue-service` — core domain: venues, assets, metadata, search, plan enforcement, venue registry lookup. Synchronous request/response only.
- `iqbene-venue-ingestion-worker` — async sidecar: document ETL pipeline, extraction orchestration, embedding generation, registry matching, scheduled jobs. No inbound HTTP — event-driven only. Shares the same PostgreSQL schema as `iqbene-venue-service`.

**New infrastructure introduced by iQ BENE:**

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

#### `registry/` — Platform Venue Registry

**Not tenant-owned. Lives in `public` schema. Read-only to tenants.**

The venue registry is a platform-level reference dataset — a growing catalogue of known venues, seeded during development (see [cold-start.md](../business/cold-start.md)) and enriched over time. It is not a source of truth; it is a starting point. Tenant data always wins over registry data.

**`VenueRegistryEntry`**

| Field          | Type             | Notes                                         |
| -------------- | ---------------- | --------------------------------------------- |
| `id`           | UUID             | PK                                            |
| `name`         | varchar(255)     |                                               |
| `address`      | text             |                                               |
| `city`         | varchar(100)     |                                               |
| `country_code` | varchar(2)       | ISO 3166-1 alpha-2                            |
| `location`     | geography(point) | PostGIS, lat/lng                              |
| `metadata`     | jsonb            | Same field shape as `venues.metadata`         |
| `confidence`   | numeric(3,2)     | Overall quality score 0.0–1.0                 |
| `source`       | varchar(50)      | `platform_seed`, `web_scrape`, `admin_import` |
| `created_at`   | timestamp        |                                               |
| `updated_at`   | timestamp        |                                               |

**`VenueRegistryAlias`** — alternative names for the same venue (e.g. "The Bowery Hotel" / "Bowery Hotel NYC"):

| Field                     | Type         | Notes               |
| ------------------------- | ------------ | ------------------- |
| `id`                      | UUID         | PK                  |
| `venue_registry_entry_id` | UUID         | FK → venue_registry |
| `alias`                   | varchar(255) |                     |

---

#### `venue_groups/` — Tenant Library Organisation (Phase 2)

**Tenant-owned. Lives in `t_{tenantKey}` schema.**

Event managers organise their venue library into groups — by city, event type, client, season, or any other taxonomy they choose. Groups are purely a UI/navigation concern; they do not affect search, extraction, or metadata.

**`VenueGroup`**

| Field        | Type         | Notes                                             |
| ------------ | ------------ | ------------------------------------------------- |
| `id`         | UUID         | PK                                                |
| `name`       | varchar(255) |                                                   |
| `parent_id`  | UUID         | FK → venue_groups (self-referential, for nesting) |
| `created_by` | UUID         |                                                   |
| `created_at` | timestamp    |                                                   |

Venues are assigned to groups via a join table `venue_group_members(venue_id, group_id)`. A venue can belong to multiple groups.

This is a Phase 2 feature. The `venue_groups` table is not included in the Phase 1 (MVP) Liquibase migrations.

---

The `venues.metadata` column stores the **consolidated view** of all extracted and manually entered data. The `venues.metadata_sources` column stores **provenance** per field.

**Canonical field set:**

```
_schema_version (int)        MANDATORY — schema version for migration, starts at 1. Never absent
                                 Incremented on every canonical schema change. Read by
                                 VenueMetadataMigrator (see §2a).

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

## 2a. Metadata Schema Versioning & Migration

The canonical `venues.metadata` shape evolves over time — new nested fields are added, types are widened, obsolete keys are retired. Without an explicit version marker and migration pipeline, JSONB documents produced by older extraction prompts or older code versions deserialize incompletely (null pointers, type coercion failures, silent data drops) when read by newer backend code.

### Contract

Every `venues.metadata` JSONB document must carry an integer key `_schema_version` at the top level. The value is the version of the **canonical field set** (§2) that the document was produced against. Documents without `_schema_version` are treated as version `0` — the "legacy, pre-versioning" shape.

| Rule                | Value                                                                                                                                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Initial version     | `1` (matches the first canonical field set shipped in MVP)                                                                                                                                        |
| Absent key fallback | `0` (triggers full migration chain from v0)                                                                                                                                                       |
| Bump condition      | Any backwards-incompatible change to canonical fields, or any addition of a required nested field                                                                                                 |
| Bump ownership      | `iqbene-venue-model` library — only the shared model may define `CURRENT_SCHEMA_VERSION`                                                                                                          |
| Write enforcement   | Every write path (aggregation, manual override, bulk import, registry copy) runs the migrator and sets `_schema_version = CURRENT_SCHEMA_VERSION` before persisting                               |
| Read enforcement    | `VenueMetadataMigrator.migrateToCurrent(JsonNode)` is called on **every read** from `venues.metadata` (via MyBatis ResultMap handler or wrapper mapper) — deserialization never sees stale shapes |

### Migration pipeline — `VenueMetadataMigrator`

The migrator lives in `iqbene-venue-model` (shared library, zero Spring deps — pure Java). Both services use the **same** migrator instance, so read and write paths agree on schema shape with no drift.

```
                 ┌──────────────────────────────────┐
  raw JSONB  ──► │  VenueMetadataMigrator           │
  (any version)  │                                  │
                 │  1. Extract _schema_version      │
                 │     (absent → 0)                 │
                 │                                  │
                 │  2. Chain ordered migrations:    │
                 │     v0→v1, v1→v2, v2→v3, …      │
                 │     (sequential apply, skip if   │
                 │      at or above target)         │
                 │                                  │
                 │  3. Apply defaults for new keys: │
                 │     null → sensible default /    │
                 │     Optional.empty (never set to │
                 │     sentinel "unknown")          │
                 │                                  │
                 │  4. Validate against current     │
                 │     canonical schema (type-only, │
                 │     no business rules)           │
                 │                                  │
                 │  5. Stamp _schema_version = N    │
                 │     and return JsonNode          │
                 └──────────────┬───────────────────┘
                                │
                                ▼
                     JsonNode at CURRENT_VERSION
                     (safe to deserialize into
                      VenueMetadata POJO)
```

### Migration classes

Each `N → N+1` step is a single-responsibility class implementing `MetadataMigration` interface. Migrations are registered in an ordered list inside `VenueMetadataMigrator` — the order is the source of truth.

```java
// iqbene-venue-model: no Spring, no framework deps
public interface MetadataMigration {
    int fromVersion();
    int toVersion();
    JsonNode apply(JsonNode input, ObjectMapper mapper);
}

// Example: v1 → v2 adds catering.vegan_available and renames
// capacity.configurations.banquet → capacity.configurations.round_banquet
public final class MetadataMigrationV1ToV2 implements MetadataMigration {
    @Override public int fromVersion() { return 1; }
    @Override public int toVersion()   { return 2; }

    @Override
    public JsonNode apply(JsonNode input, ObjectMapper mapper) {
        var root = (ObjectNode) input.deepCopy();
        var catering = root.withObject("catering");
        if (!catering.has("vegan_available")) {
            catering.putNull("vegan_available");  // explicit null = unknown, NOT false
        }
        var caps = root.path("capacity").path("configurations");
        if (caps.isObject() && caps.has("banquet") && !caps.has("round_banquet")) {
            ((ObjectNode) root.path("capacity").path("configurations"))
                .set("round_banquet", caps.get("banquet"));
            ((ObjectNode) root.path("capacity").path("configurations"))
                .remove("banquet");
        }
        return root;
    }
}
```

Rules for migration authors:

- Migrations are **pure functions**: same input → same output, no side effects, no I/O. Testable with plain unit tests — no Spring context required.
- Migrations never drop keys they do not recognise. Unknown keys are preserved verbatim so that forward-compatible shapes (e.g. a v3 writer writing while v2 readers are still live) do not lose data.
- Numeric type widening (int → long, int → double) is done by value coercion, not by dropping the value.
- Enum string rename: produce a mapping table. Missing values default to `null` with no exception thrown — callers decide how to render an unknown enum value in UI/API.
- Migration list is append-only. Once a migration class is merged, it is never deleted or reordered. Superseded logic lives on because tenant schemas can contain arbitrarily old JSONB documents (e.g. a venue last updated 18 months ago).

### Write path — stamping the version

All code paths that produce or mutate `venues.metadata` call `migrator.ensureCurrent(node)` **before** the `UPDATE` / `INSERT`. This:

1. Applies any pending migrations from the document's current version.
2. Sets `_schema_version = CURRENT_SCHEMA_VERSION`.
3. Returns the node ready to be persisted.

Write paths affected: `MetadataAggregationConsumer` (after conflict resolution), `PATCH /metadata/{field}` handler, bulk import job, registry gap-fill copy, `CreateVenueRequest` default metadata initializer.

### Read path — safe deserialization

Reading `venues.metadata` from the database must never return a raw JSONB value to the caller. The MyBatis result map for `Venue` applies the migrator inside the column handler:

```xml
<!-- MyBatis mapper: venues result map -->
<resultMap id="VenueResultMap" type="com.iqkv.iqbene.model.venue.Venue">
  <id     property="id"        column="id"/>
  <result property="name"      column="name"/>
  <!-- …other columns… -->
  <result property="metadata"  column="metadata"
          typeHandler="com.iqkv.iqbene.model.metadata.VenueMetadataTypeHandler"/>
</resultMap>
```

`VenueMetadataTypeHandler` runs `VenueMetadataMigrator.migrateToCurrent()` between the raw `PGobject` JSONB and `VenueMetadata` POJO construction. If a document is several versions behind, all intermediate migrations run in a single pass inside one handler call. No extra SQL, no batch background job required.

### Backfill: online-only, no offline migration job

Because the migrator runs on every read and every write, schema convergence is incremental and online:

- **First read** of a stale document → migrator upgrades in-memory, returns correct shape to the caller. The DB copy remains stale (lowest cost, no write amplification).
- **Next write** (aggregation, manual edit, anything that touches the row) → `ensureCurrent()` stamps the DB copy to the latest version permanently.
- **Warm cache effect**: venues actively used by tenants converge quickly; idle dormant venues are upgraded on their first access and permanently fixed at their next write.

No scheduled backfill job, no downtime, no ALTER TABLE on JSONB. If we later want to force-converge all rows for a tenant (e.g. before a heavy schema jump), a one-shot admin endpoint iterates `SELECT id FROM venues` and issues a no-op `UPDATE venues SET metadata = metadata` to trigger the write-path stamping.

### Version distribution observability

The migrator records the pre-migration version of every document it sees via Micrometer (see §12). This lets us observe how many legacy versions are still in the wild and when it is safe to delete very old migration classes from the chain (typically after all tenants have had their last stale document touched and stamped).

### Schema version in registry metadata

`public.venue_registry.metadata` follows the same `_schema_version` contract. Registry import jobs run `VenueMetadataMigrator.ensureCurrent()` before `INSERT/UPDATE public.venue_registry`. When a tenant copies registry fields into its own venue record (gap-fill, §5 Stage 3), both sides are at known versions and the copy logic merges field-by-field rather than JSON-blob-blit — no cross-version contamination.

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
REGISTRY            → platform registry seed (lowest priority — fills gaps only)
```

### Array Fields (amenities, restrictions)

Set-union across all sources. An entry is included if at least one source reports it with confidence ≥ 0.6.

### Trigger Points

Aggregation runs (async, via RabbitMQ) when:

- An `asset.uploaded` event triggers extraction → extraction completes → `extraction.completed` triggers aggregation
- A user submits a manual override → immediate re-aggregation
- A scheduled job catches stale venues (24h without re-aggregation)

Aggregation is debounced (5s) to batch rapid successive events.

### Concurrency Control and Race Condition Prevention

Metadata aggregation is a read-modify-write operation: `SELECT venues.metadata` → merge extracted fields → `UPDATE venues.metadata`. If three extraction jobs for the same venue complete in parallel, three workers can simultaneously read stale metadata, each merge one PDF's fields, and each write back — two of the three writes are lost (Lost Update anomaly). The result is a `metadata` JSONB that contains only a subset of the three PDFs' extracted fields.

The chosen approach eliminates the race at the messaging layer, before the message reaches the consumer, using RabbitMQ's built-in capabilities. No distributed locks, no optimistic locking retry loops, no row-level `SELECT … FOR UPDATE` contention.

#### Why RabbitMQ FIFO routing per venue_id

| Constraint satisfied      | How FIFO routing delivers it                                                                                                                                               |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Zero new dependencies     | Uses existing RabbitMQ (§14, already in platform stack). No Redis, no distributed lock library.                                                                            |
| Minimal code footprint    | One-line routing key computation on publish; consumer configuration only. No retry-loop code, no deadlock corner cases.                                                    |
| Extraction stays parallel | Three PDFs extract concurrently (CPU/IO-bound work — fast). Only the final metadata merge step (microseconds of in-memory merge + 1 SQL `UPDATE`) is serialised per venue. |
| No infrastructure cost    | Same RabbitMQ cluster, same number of consumer threads. No extra services or sidecars.                                                                                     |
| Horizontal scalability    | As venue count grows, increase the hash slot count and consumer pool size. Different venues always process in parallel.                                                    |

#### Implementation variants

Two variants share the same conceptual model. Start with A1 for MVP; both use the same consumer code shape.

**Variant A1 — Simple (MVP).** One queue, one serialising consumer.

| Aspect                | Specification                                                                                                                                                                        |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Queue                 | Single queue `iqbene.metadata.aggregation`.                                                                                                                                          |
| Publisher routing key | `extraction.completed` unchanged. No slot computation.                                                                                                                               |
| Consumer              | `@RabbitListener` with `concurrency = 1`, `prefetchCount = 1`. Exactly one thread processes all aggregation events sequentially across all tenants and all venues.                   |
| Backlog envelope      | Aggregation per event is ~1 ms (merge + SQL `UPDATE`). Even 100 events/s sustained yields a 100 ms backlog, which is invisible to end users and well within the 5 s debounce window. |
| When to choose        | MVP. The product does not expect mass-parallel upload across many accounts simultaneously. When queue depth metrics breach threshold, migrate to A2.                                 |

**Variant A2 — Scalable (ready immediately, no later migration).** N hash-partitioned queues with per-queue single-threaded consumption.

| Aspect               | Specification                                                                                                                                                                                |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Queues               | `iqbene.metadata.aggregation.0` through `iqbene.metadata.aggregation.15` (16 slots by default; configurable via `application.yml`).                                                          |
| Publisher routing    | Slot = `Math.abs(venueId.hashCode() % SLOT_COUNT)`. Publisher appends slot suffix to routing key or binds queues via a consistent-hash exchange. Same venue_id always maps to the same slot. |
| Consumer pool        | 16 consumer threads. Each thread binds to exactly one slot queue with `prefetchCount = 1`.                                                                                                   |
| Parallelism property | Different venues process in parallel across slots. Same venue always routes to the same slot → strict FIFO ordering per venue.                                                               |
| When to choose       | If immediate horizontal headroom is desired, or to avoid an A1→A2 queue topology migration later. ~20 extra lines of code on the publisher side.                                             |

#### Consumer-side guarantees (both variants)

Even with FIFO ordering at the messaging layer, the consumer must enclose the entire aggregation step in a single database transaction to guard against consumer crash or restart mid-operation.

```
Consumer transaction boundary (single DB transaction):
  1. BEGIN
  2. SELECT venues.metadata, venues.metadata_aggregated_at
     FROM venues WHERE id = ?
  3. If metadata_aggregated_at within 5 s debounce window → no-op, ack message.
  4. Else → merge all unprocessed venue_metadata_events into metadata
     via VenueMetadataMigrator.ensureCurrent() + conflict resolution (§3)
  5. UPDATE venues SET metadata = ?, metadata_sources = ?,
     metadata_aggregated_at = NOW(), updated_at = NOW()
     WHERE id = ?
  6. DELETE / mark-consumed processed venue_metadata_events
  7. COMMIT
  8. RabbitMQ ack — only after successful COMMIT
```

Acknowledgement mode on the listener container must be `MANUAL` (or `AUTO` with `prefetchCount = 1`, which serialises in-flight messages to the same effect). A single worker must never pull a batch of messages and process them interleaved; one message at a time per queue.

#### Debounce + FIFO synergy

The existing 5-second debounce window at line 423 above composes naturally with FIFO ordering:

1. Three PDFs finish extraction almost simultaneously → three `extraction.completed` events published, all routed to the same slot queue for venue X.
2. Event 1 is consumed first. `metadata_aggregated_at` is older than 5 s → aggregation runs, `metadata_aggregated_at` is set to `NOW()`.
3. Event 2 is consumed next. `metadata_aggregated_at` is within the 5 s window → no-op, message acked without SQL `UPDATE`.
4. Event 3 is consumed next. Same no-op path.

Outcome: one SQL `UPDATE` instead of three. The redundant work is eliminated before it reaches the database, not contended inside it.

#### Secondary benefits of this approach

- **No retry loops or conflict handling.** In contrast to optimistic locking (`_version` column), there is no conflict exception, no exponential-backoff retry code, and no test surface for livelock or starvation edge cases.
- **Event sourcing friendly.** If a future feature replays `venue_metadata_events`, per-venue ordering is preserved at the messaging layer — the consumer sees the same sequence on replay as it did originally.
- **Straightforward integration testing.** Seed three `extraction.completed` events for the same `venue_id` into the queue, consume, assert that the final `venues.metadata` contains merged fields from all three sources. No `CountDownLatch` multi-threaded test harness for database-level locks.
- **Compatible with `_schema_version` (§2a).** The `VenueMetadataMigrator` runs inside step 5 of the consumer transaction, before the `UPDATE` commits. Because events are serialised per venue, the migration chain sees a monotonically increasing version sequence with no interleaved writes.

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
                    │            iqbene-venue-service             │
                    │  venues · assets · metadata · search · api │
                    └────────┬──────────────┬──────────┬─────────┘
                             │ RabbitMQ:    │ r/w       │ presigned URL
                             │ asset.uploa- │           │ issue + delete
                    ┌────────▼─────────┐  ┌─▼──────────▼──────┐
                    │ iqbene-ingestion- │  │   PostgreSQL        │
                    │    worker        │──►│   t_{tenant}        │
                    │ (async sidecar)  │  │   + pgvector        │
                    └────────┬─────────┘  │   + PostGIS         │
                             │ registry   └─────────────────────┘
                             │ match      ┌─────────────────────┐
                             ├───────────►│   public schema      │
                             │            │   venue_registry     │
                             │            └─────────────────────┘
                             │ read asset ┌─────────────────────┐
                             └───────────►│   S3 / MinIO         │◄── client (direct PUT)
                                          │   vip/tenants/{key}/ │
                                          │   vip/registry/      │
                                          └─────────────────────┘
```

### iqbene-venue-service

- **Responsibilities:** venue CRUD, asset upload flow (presigned URL), metadata read/write, search API, plan entitlement enforcement
- **Database:** owns the iQ BENE PostgreSQL schema. Tenancy is schema-level via `foundation-tenancy` — each tenant gets its own schema `t_{tenantKey}`. No `tenant_id` column on any table; schema routing is handled by `MyBatisSchemaInterceptor`. Shared with `iqbene-venue-ingestion-worker` — no cross-service API calls for data.
- **Exposes:** REST API at `/api/v1/venues`
- **Publishes:** `venue.created`, `venue.updated`, `asset.uploaded`, `asset.deleted` (RabbitMQ)
- **Consumes:** `extraction.completed`, `extraction.failed` (RabbitMQ) — triggers metadata aggregation

### iqbene-venue-ingestion-worker

- **Responsibilities:** document ETL pipeline (parse → chunk → extract → embed), extraction job lifecycle, registry matching and gap-fill, metadata aggregation, scheduled maintenance jobs (stale re-aggregation, cost reporting)
- **Nature:** async sidecar — no inbound HTTP, no REST API, no service discovery entry. Event-driven only.
- **Database:** shared PostgreSQL schema with `iqbene-venue-service`. Reads `venue_assets`, writes `extraction_jobs`, `venue_metadata_events`, `venue_vectors`, `ai_cost_tracking`. Also reads `public.venue_registry` for the registry match step.
- **Consumes:** `asset.uploaded` (RabbitMQ) — triggers ETL pipeline
- **Publishes:** `extraction.started`, `extraction.completed`, `extraction.failed` (RabbitMQ)
- **External calls:** OpenAI API (GPT-4o, text-embedding-3-small), optionally Docling sidecar (Phase 2)
- **Scaling:** replicas scaled independently based on RabbitMQ queue depth — no impact on `iqbene-venue-service`

### Table Ownership

Both services share one PostgreSQL schema. Ownership defines who may write to a table. Cross-boundary reads are permitted; cross-boundary writes are not.

| Table                   | Owner                           | The other service may…                                                       |
| ----------------------- | ------------------------------- | ---------------------------------------------------------------------------- |
| `venues`                | `iqbene-venue-service`          | read (ingestion-worker: resolve venue_id only)                               |
| `venue_assets`          | `iqbene-venue-service`          | read (ingestion-worker: fetch asset for processing)                          |
| `venue_metadata_events` | `iqbene-venue-service`          | write via event reaction (`extraction.completed` → venue-service aggregates) |
| `extraction_jobs`       | `iqbene-venue-ingestion-worker` | read (venue-service: expose job status to API)                               |
| `venue_vectors`         | `iqbene-venue-ingestion-worker` | read (venue-service: vector search queries)                                  |
| `ai_cost_tracking`      | `iqbene-venue-ingestion-worker` | read (venue-service: expose cost summary to API)                             |

The single legitimate cross-boundary read from `iqbene-venue-ingestion-worker` is a `SELECT` on `venue_assets` by `asset_id` (delivered in the `asset.uploaded` event payload). This is a foreign key lookup, not business logic — acceptable and intentional.

---

## 4a. Shared Library — iqbene-venue-model

`iqbene-venue-model` is a plain Java library (JAR, no Spring Boot, no `@SpringBootApplication`). Both `iqbene-venue-service` and `iqbene-venue-ingestion-worker` declare it as a compile dependency. It is the single source of truth for anything both services need to agree on.

**Contents:**

```
iqbene-venue-model/
├── model/
│   ├── venue/
│   │   ├── Venue.java                  Plain POJO (aggregate root, no JPA annotations)
│   │   ├── VenueStatus.java            enum: DRAFT, ACTIVE, ARCHIVED
│   │   └── VenueAsset.java             Plain POJO
│   ├── asset/
│   │   ├── AssetType.java              enum: PDF_DECK, FLOOR_PLAN, PHOTO, CAD_FILE…
│   │   └── ExtractionStatus.java       enum: PENDING, IN_PROGRESS, COMPLETED, FAILED
│   ├── extraction/
│   │   ├── ExtractionJob.java          Plain POJO
│   │   ├── ExtractorType.java          enum: TIKA_TEXT, GPT4O_DOCUMENT, GPT4O_VISION
│   │   └── VenueMetadataEvent.java     Plain POJO (append-only event log)
│   ├── metadata/
│   │   ├── VenueMetadata.java          Value object (mirrors venues.metadata JSONB)
│   │   ├── VenueCapacity.java          Capacity configurations value object
│   │   ├── MetadataSource.java         Provenance per field
│   │   ├── MetadataEventType.java      enum: ASSET_EXTRACTED, MANUAL_OVERRIDE, BULK_IMPORT, REGISTRY
│   │   ├── MetadataSchemaVersion.java  Single source of truth: CURRENT_SCHEMA_VERSION = 1
│   │   ├── MetadataMigration.java      Interface: fromVersion(), toVersion(), apply()
│   │   ├── VenueMetadataMigrator.java  Migration chain runner: migrateToCurrent(), ensureCurrent()
│   │   ├── VenueMetadataTypeHandler    MyBatis TypeHandler: auto-apply migrator on every read
│   │   └── migrations/                 Append-only ordered list of N→N+1 migration classes
│   │       ├── MetadataMigrationV0ToV1.java   (bootstraps legacy pre-versioned docs → v1)
│   │       └── MetadataMigrationV1ToV2.java   (reserved for next schema bump)
│   ├── registry/
│   │   ├── VenueRegistryEntry.java     Plain POJO — platform registry record (public schema)
│   │   └── VenueRegistryAlias.java     Plain POJO — alternative names for registry entries
│   └── events/                         RabbitMQ message contracts (POJOs, no framework deps)
│       ├── AssetUploadedEvent.java
│       ├── ExtractionStartedEvent.java
│       ├── ExtractionCompletedEvent.java
│       └── ExtractionFailedEvent.java
└── db/
    └── changelog/
        ├── system/                     System (public) schema migrations
        │   ├── master.xml
        │   └── 20250101000000-create-venue-registry.xml
        └── tenant/                     Tenant schema migrations — single source of truth
            ├── master.xml
            ├── 20250101000001-create-venues.xml
            ├── 20250101000002-create-venue-assets.xml
            ├── 20250101000003-create-extraction-jobs.xml
            ├── 20250101000004-create-metadata-events.xml
            ├── 20250101000005-create-venue-vectors.xml
            └── 20250101000006-create-ai-cost-tracking.xml
```

**Rules:**

- No `@Service`, `@Repository`, `@Component`, or any Spring bean annotation
- No business logic — plain Java domain classes, value objects, enums, event POJOs only. The `VenueMetadataMigrator` is the single exception: pure-function schema migration is a model-layer concern, not a service concern.
- No JPA annotations — the platform uses MyBatis, not JPA. Entities are plain POJOs, not `@Entity` classes. ORM annotations must not appear in this library.
- Liquibase migrations live here so schema changes are a compile-time dependency bump, not a coordination exercise between services
- Changing an event POJO field is a compile-time break in both services — intentional, prevents silent contract drift
- `MetadataSchemaVersion.CURRENT_SCHEMA_VERSION` is the **only** place the canonical schema version number is hardcoded. No service may define its own copy. Incrementing this constant is the single action that declares a schema bump; the corresponding `MetadataMigrationV{N}ToV{N+1}` class must be added to the `migrations/` package in the same commit.
- `VenueMetadataMigrator` has no Spring dependency. It uses Jackson `ObjectMapper` (shaded or provided) and exposes static methods or a singleton instance. Both services share the migrator classpath-identical; one service never lags behind because the library version is a compile dependency.
- Migration classes under `metadata/migrations/` are added, never removed or reordered. The order of migration registration inside `VenueMetadataMigrator` is the single source of truth for the chain.

**Dependency graph:**

```
iqbene-venue-model  (library, no runtime)
      ├── iqbene-venue-service     (Spring Boot, imports model)
      └── iqbene-venue-ingestion-worker  (Spring Boot, imports model)
```

---

## 4b. S3 Storage Layout

S3 (MinIO for local dev) is already in the IQKV stack. iQ BENE adds its own prefix namespace inside the shared bucket (`iqkv-files`) — no new bucket needed in dev/staging. Production can isolate into a dedicated bucket (`iqkv-vip-files`) via a single config change; the key structure is identical either way.

---

### Bucket strategy

| Environment | Bucket           | Notes                                                                                     |
| ----------- | ---------------- | ----------------------------------------------------------------------------------------- |
| Dev / CI    | `iqkv-files`     | Shared with foundation services, MinIO default. VIP objects live under `vip/` prefix.     |
| Staging     | `iqkv-files`     | Same shared bucket, same prefix scheme. Isolated by prefix only.                          |
| Production  | `iqkv-vip-files` | Dedicated bucket. Separate IAM policy, separate lifecycle rules. Key structure identical. |

MinIO in local dev is configured in `docker-compose.yml` with `MINIO_DEFAULT_BUCKETS=iqkv-files`. No `iqkv-vip-files` bucket is needed until the production deployment config is introduced.

---

### Key naming convention

All VIP objects follow a deterministic, hierarchical key structure. Every segment is lowercase, no spaces.

#### Tenant asset files (uploaded by tenant users)

```
vip/tenants/{tenantKey}/venues/{venueId}/assets/{assetId}/{fileName}
```

| Segment       | Value                                                   | Example                            |
| ------------- | ------------------------------------------------------- | ---------------------------------- |
| `vip/`        | VIP namespace — separates from other foundation objects | (literal)                          |
| `tenants/`    | Tenant subtree root                                     | (literal)                          |
| `{tenantKey}` | 8-char nanoid from JWT `tenant_id` claim                | `acme0001`                         |
| `venues/`     | Venue subtree                                           | (literal)                          |
| `{venueId}`   | UUID of the venue (no hyphens — compact form)           | `550e8400e29b41d4a716446655440000` |
| `assets/`     | Asset subtree                                           | (literal)                          |
| `{assetId}`   | UUID of the asset (no hyphens)                          | `6ba7b8109dad11d180b400c04fd430c8` |
| `{fileName}`  | Original file name, URL-safe, max 255 chars             | `grand-ballroom-deck.pdf`          |

Full example:

```
vip/tenants/acme0001/venues/550e8400e29b41d4a716446655440000/assets/6ba7b8109dad11d180b400c04fd430c8/grand-ballroom-deck.pdf
```

**Key rules:**

- `{fileName}` is the original client-supplied file name, stripped of path separators (`/`, `\`), URL-encoded where necessary. It is stored as-is after sanitisation — no UUID substitution — so the key stays human-readable in MinIO console / S3 CLI.
- `{venueId}` and `{assetId}` are UUIDs without hyphens (compact 32-char hex). This keeps keys short and avoids double-encoding issues.
- The `asset_id` is generated server-side at `POST /assets/initiate` and written to `venue_assets.id` before the presigned URL is issued. The S3 key is computed from that ID and stored in `venue_assets.s3_key`. At confirm time no key re-computation happens — the stored `s3_key` is used directly.

#### Presigned URL issuance

`POST /assets/initiate` response returns:

```json
{
  "asset_id": "<uuid>",
  "upload_url": "https://minio.local/iqkv-files/vip/tenants/acme0001/venues/.../grand-ballroom-deck.pdf?X-Amz-Signature=...",
  "expires_at": "2025-06-01T12:15:00Z"
}
```

The presigned PUT URL is pre-signed server-side with the exact key. The client uploads directly. The service never proxies the file body.

Download presigned URLs (1h TTL) are generated on-demand at `GET /assets/{id}/download-url` — not stored. The key is always re-derived from `venue_assets.s3_key`.

---

#### Platform registry import files (admin / seeding)

Registry seed data and bulk import files are not tenant-owned. They live under a separate subtree:

```
vip/registry/imports/{importId}/{fileName}
vip/registry/exports/{date}/{snapshot}.jsonl.gz
```

| Path                               | Purpose                                                                           |
| ---------------------------------- | --------------------------------------------------------------------------------- |
| `vip/registry/imports/{importId}/` | One folder per import batch (admin-triggered). Contains raw CSV/JSON input files. |
| `vip/registry/exports/{date}/`     | Nightly compacted snapshots of `public.venue_registry` for downstream consumers.  |

`{importId}` is a UUID generated at import initiation. `{date}` is `YYYY-MM-DD`.

Registry import files are processed by a scheduled admin job (no inbound HTTP for import — admin drops files via the Registry Admin API, Phase 2). The import job reads from S3, populates `public.venue_registry` and `public.venue_registry_aliases`, then archives the source object by moving it to `vip/registry/imports/processed/{importId}/`.

---

### Tenant isolation

Tenant data isolation in S3 mirrors the schema-per-tenant approach in PostgreSQL:

- All tenant objects are scoped under `vip/tenants/{tenantKey}/`. Cross-tenant read is structurally impossible without knowing the other tenant's key.
- The service account used by `iqbene-venue-service` and `iqbene-venue-ingestion-worker` holds a single S3 IAM policy that allows `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on the full `vip/*` prefix. Presigned URLs are scoped to the exact object key — the client cannot enumerate or access any other key.
- Registry paths (`vip/registry/*`) are not accessible via tenant-issued presigned URLs. They are written only by the platform's internal job service account.

---

### Lifecycle rules and compaction

S3 lifecycle rules are configured on the bucket (not in application code). Two rules apply:

| Rule                       | Prefix                             | Action                                                                                                                                                                                  |
| -------------------------- | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Extraction artefact expiry | `vip/tenants/*/venues/*/assets/*/` | Transition to Glacier/IA after 90 days if `extraction_status = COMPLETED` and no re-extraction pending. Managed via tags set on object at confirm time (`extraction_status=completed`). |
| Registry import cleanup    | `vip/registry/imports/processed/`  | Delete after 30 days.                                                                                                                                                                   |
| Registry snapshot rotation | `vip/registry/exports/`            | Keep last 14 daily snapshots; delete older.                                                                                                                                             |

Object tags are set by `iqbene-venue-service` at `POST /assets/confirm` using `PutObjectTagging`. Tags used:

| Tag key             | Values                                          | Set by                          |
| ------------------- | ----------------------------------------------- | ------------------------------- |
| `extraction_status` | `pending`, `completed`, `failed`                | venue-service at confirm/update |
| `asset_type`        | `pdf_deck`, `floor_plan`, `photo`, `cad_file` … | venue-service at initiate       |
| `tenant_key`        | 8-char nanoid                                   | venue-service at initiate       |

Tags enable cost allocation reports per tenant and per asset type in AWS Cost Explorer / MinIO billing.

---

### Deletion cascade

When a tenant deletes an asset (`DELETE /assets/{id}`) or when a tenant account is terminated:

1. `iqbene-venue-service` deletes the `venue_assets` row (DB cascade drops extraction jobs, metadata events referencing the asset).
2. `iqbene-venue-service` issues `s3:DeleteObject` for `venue_assets.s3_key`.
3. A `asset.deleted` event is published → `iqbene-venue-ingestion-worker` deletes all `venue_vectors` rows where `metadata->>'asset_id' = :assetId`.

For full tenant deletion (GDPR right to erasure):

1. `DELETE FROM t_{tenantKey}.venues` cascades to all asset rows.
2. A separate `tenant.deleted` event triggers a background S3 sweep: `s3:DeleteObjects` with all keys matching `vip/tenants/{tenantKey}/*` (batched in 1000-object chunks to respect S3 API limits).
3. The pgvector sweep deletes all `venue_vectors` rows for the tenant schema (schema drop handles this implicitly if the schema is dropped).

---

### Registry population strategy (cold start for MVP)

The `public.venue_registry` table is the platform's canonical venue reference. It is never populated by tenant uploads. Registry is a secondary, gap-fill-only source. Tenant data always wins (see §3 conflict resolution priority: `REGISTRY` is lowest).

Registry population in MVP uses **three human-in-the-loop channels**. No automatic scraper-to-INSERT pipeline. No AI/embeddings in the population process.

| Channel                                                    | Mechanism                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Source in `venue_registry.source` |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------- |
| **1. Pre-provisioned seed migrations (MVP cold-start)**    | Hardcoded rows in Liquibase XML changesets under `iqbene-venue-model/src/main/resources/db/changelog/system/`. Curated shortlist of 50–200 high-signal venues (top convention centres, major hotel chains in target launch cities). Runs on first startup against `public` schema via `TenantLiquibaseRunner`. Zero code, zero S3, zero admin interaction.                                                                                                                                                         | `platform_seed`                   |
| **2. Platform admin manual entry (MVP)**                   | Registry Admin API (§17: Phase 2 design signal — pull forward for MVP, single-entity CRUD only, no bulk): `POST /api/v1/admin/registry/entries`, `PATCH /api/v1/admin/registry/entries/{id}`, `POST /api/v1/admin/registry/entries/{id}/aliases`. Authority `PLATFORM_ADMIN` only. Admin provides every field manually; deduplication check runs server-side as a pre-write validation and returns a list of candidate duplicates for human review (admin clicks "Confirm insert" or "Merge with existing #1234"). | `admin_import`                    |
| **3. Scraper scripts (Cvent et al.) + human review (MVP)** | Standalone scripts outside the service (cron, admin laptop, or optional scheduled container) produce CSV/JSONL output files, upload to S3 `vip/registry/imports/{importId}/` with manifest. `VenueRegistryImportOrchestrator` in `iqbene-venue-ingestion-worker` runs only when triggered by an admin RabbitMQ event (`admin.registry.import.dry-run`) → produces a CSV audit report → uploads to `vip/registry/imports/reports/{importId}_review.csv`. Admin reviews the report (each row: action `INSERT`        | `MERGE #id`                       | `SKIP` with name_sim + geo_distance + duplicate candidates list), edits the Action column, re-uploads reviewed CSV. Admin then fires `admin.registry.import.apply` → worker applies the reviewed actions exactly, never making its own merge/insert decision. | `web_scrape` |
| **Tenant-signal enrichment**                               | After `extraction.completed`, if tenant data has high-confidence fields not in registry → candidate event (Phase 3, no reverse flow in MVP)                                                                                                                                                                                                                                                                                                                                                                        | — (not in MVP)                    |

### Alias normalisation (shared by all population channels + extraction matcher)

Before any name comparison (INSERT-time dedup check, scraper dry-run report, extraction-time gap-fill), both sides of the comparison pass through the same normalisation function so that trivial differences do not break the match:

```
normalize(name):
  1. Trim leading/trailing whitespace
  2. Strip case-insensitive leading articles: "^the\s+", "^a\s+", "^an\s+"
  3. Lowercase
  4. Strip any character that is not alphanumeric or whitespace (keep ASCII spaces only)
  5. Collapse consecutive whitespace into single space
  6. Trim again
```

Example: `"The  Bowery-Hotel, NYC!"` → `"bowery hotel nyc"`.

Both the original name and the normalised form are stored. `venue_registry_aliases` holds the original-name alias, and the normalised form is computed on-the-fly during comparison (or stored redundantly in the same row for index-friendliness).

### Scraper import — dry-run dedup check

Channel 3 scraper dry-run and channel 2 admin INSERT pre-write validation share one deduplication rule set. **Never auto-apply.** Rule set:

```
For a candidate import row (name_raw, address_raw, country_code, location_raw):
  candidates = SELECT r.id, r.name, r.location
               FROM venue_registry r
               LEFT JOIN venue_registry_aliases a ON a.venue_registry_entry_id = r.id
               WHERE normalize(name_raw) % normalize(COALESCE(a.alias, r.name))
               ORDER BY similarity DESC
               LIMIT 5

  For each candidate:
    name_sim = similarity( normalize(name_raw), normalize(COALESCE(a.alias, r.name)) )
    if both locations available:
        geo_distance_m = ST_Distance( location_raw, r.location )::int
        geo_ok         = geo_distance_m <= 150
    else:
        geo_distance_m = null
        geo_ok         = null

  Action recommendation (for human review, non-binding):
    if (name_sim >= 0.90 AND geo_ok = true)
       OR (geo_distance_m is null AND name_sim >= 0.95):
        -> MERGE candidate #id (show confidence)
    else if name_sim >= 0.75 OR geo_distance_m <= 300:
        -> REVIEW CANDIDATE #id (list top 3, show sim + dist table)
    else:
        -> INSERT (new)
```

Admin always has the final say via the reviewed CSV action column or the confirm-insert API call.

### Extraction-time gap-fill (VenueRegistryMatcher) — algorithm and thresholds

After a tenant document finishes extraction, `VenueRegistryMatcher` runs as step 3 in §5 Stage 3 (Load). It compares the tenant's `Venue` record against `public.venue_registry` using the same `normalize()` function above. **No LLM calls. No embedding similarity. Pure PostgreSQL pg_trgm + PostGIS, zero external cost.**

```
Query strategy:
  1. Fetch top-5 registry candidates via trigram GIN index on
     venue_registry_aliases.alias (normalised match).
  2. For each candidate:
       name_sim = similarity( normalize(venue.name),
                              normalize(best alias OR registry.name) )
       if venue.location not null AND candidate.location not null:
           geo_within_200m = ST_DWithin( venue.location, candidate.location, 200 )
           geo_weight = 1.0
       else:
           geo_within_200m = null   // signal unavailable
           geo_weight = 0.0

Combined confidence:
  if geo_within_200m is not null:
      combined = 0.60 * name_sim  +  0.40 * (geo_within_200m ? 1.0 : 0.0)
  else:
      combined = 1.00 * name_sim   // name-only threshold is much stricter

Outcome and thresholds:
  MATCH (copy fields):
      (geo_within_200m is not null AND combined >= 0.75)
      OR
      (geo_within_200m is null     AND combined >= 0.90)
      AND
      the top candidate has >= 0.08 confidence gap over the 2nd-place
      candidate (not ambiguous).
    →
      For every canonical metadata field where:
        (a) tenant venue.metadata.{field} is null/empty AND
        (b) candidate.metadata.{field} is not null AND
        (c) the field is a leaf key (not a nested dict merge):
          copy field value
          set metadata_sources.{field} = { source: "REGISTRY",
                                            source_id: registry_entry.id,
                                            confidence: combined,
                                            applied_at: now() }

  NO MATCH / AMBIGUOUS (everything else, including < 0.08 delta
  between top-2 candidates above 0.60):
      → silent no-op. No venue_metadata_events row written.
        Registry is secondary; we do not surface candidates to UI in MVP.
```

After a successful copy, `venues.metadata_aggregated_at` is set to `NOW()` so the subsequent aggregation step (§3) sees the REGISTRY source and applies conflict-resolution priority correctly (`REGISTRY` lowest, overridden by any later extraction/user input).

### Observability for match quality

`VenueRegistryMatcher` records two Micrometer metrics (§12) on every extraction run:

| Metric                                            | Tags                            | Purpose                                                                             |
| ------------------------------------------------- | ------------------------------- | ----------------------------------------------------------------------------------- |
| `iqbene_registry_match_total`                     | `stage="extraction"`, `outcome` | `matched` vs `no_match` vs `ambiguous` counts. Tracks how often registry is useful. |
| `iqbene_registry_match_confidence_seconds` (hist) | `stage="extraction"`            | Distribution of `combined` score across all runs. Calibrate thresholds from this.   |

Additionally, the scraper dry-run report CSV is retained in S3 for 90 days as an audit trail of every registry population decision.

### Before Sprint 1: threshold calibration dry-run

Before any production tenant has access, run a mandatory calibration pass to de-risk the 0.75/0.90 thresholds:

1. Assemble a fixture set of **50 real-world venue PDFs** from target launch cities. Manually attach the "correct" `venue_registry_entry.id` ground truth to each row (or "no registry match" if none applies).
2. Run `VenueRegistryMatcher` in **dry-run mode**: no writes to tenant schema. For every fixture, log: top-5 candidates with individual `name_sim`, `geo_within_200m`, `combined`, final outcome `matched|ambiguous|no_match`.
3. Build confusion matrix against ground truth: TP, FP, FN counts.
4. **Acceptance criterion: FP rate ≤ 1 %.** A wrong copy poisons tenant data; an FN is just a missed gap-fill that the user fills manually. If FP > 1 %, raise thresholds incrementally (e.g. 0.75 → 0.78 → 0.80) until criterion passes and note the final calibrated thresholds in CHANGELOG.md.
5. Delete the calibration run log from any environment that stores real tenant PII.

---

## 5. ETL Pipeline (iqbene-venue-ingestion-worker)

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
3. **Registry match** — `VenueRegistryMatcher` runs the full gap-fill algorithm documented above (trigram name similarity via `pg_trgm` GIN index on `venue_registry_aliases`, PostGIS `ST_DWithin` 200m radius, combined confidence formula, thresholds 0.75 with geo / 0.90 name-only, ambiguity delta guard ≥ 0.08). Registry is secondary only. No LLM calls, no embedding similarity. Full algorithm, thresholds, and field-copy semantics in the "Extraction-time gap-fill" subsection above.
4. **Aggregate** — publishes `extraction.completed` event → `MetadataAggregationConsumer` updates `venues.metadata`.

### Processing SLA

| Asset type           | Target latency |
| -------------------- | -------------- |
| PDF / DOCX (text)    | < 30s          |
| Images / floor plans | < 60s          |
| CAD files            | < 2 min        |

Retry on failure: 3 attempts with exponential backoff. After 3 failures → `extraction.failed` event → user notification.

---

## 6. Search Architecture

All search is served by `iqbene-venue-service` querying PostgreSQL directly. No separate search service.

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

### Cross-source Search (tenant venues + public registry)

**Problem.** Registry lives in `public.venue_registry`, tenant venues live in `t_{tenantKey}.venues`. Users need one search bar that returns _both_ "my venues" and "registry venues I can import". A single cross-schema SQL `UNION ALL` would collapse this gap technically, but it breaks the schema-per-tenant isolation abstraction and couples two data-sets that have different index statistics, different signal sets, and different column-level permission policies.

**Decided (Approach 2: two parallel SQL selects + app-level merge).** — Single PostgreSQL instance, no external search service. No cross-schema joins or `UNION ALL` in one statement.

```
            ┌──────────────────────────────┐
            │  GET /api/v1/venues/?scope=  │
            │  BOTH | TENANT_ONLY          │
            │  |REGISTRY_ONLY (default)    │
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
   ┌──────────────▼───────┐  ┌──────▼──────────────────┐
   │  Branch A: tenant    │  │  Branch B: registry     │
   │  venues scope        │  │  entries scope         │
   │  (VenueMapper)       │  │  (RegistryQueryMapper) │
   │  — no schema         │  │  — explicitly schema-  │
   │    qualify (implicit │  │    qualified: FROM     │
   │    search_path)      │  │    public.venue_registry│
   │  — all 5 modes:      │  │  — 3 modes MVP:        │
   │    keyword, semantic,│  │    keyword, structured,│
   │    structured, geo,  │  │    geo only (semantic  │
   │    hybrid RRF        │  │    TBD Phase 2)        │
   │  — top-50 + scores   │  │  — top-50 + scores     │
   └──────────────┬───────┘  └──────┬──────────────────┘
                  │                  │
          ┌───────▼──────────────────▼───────┐
          │  App-level merge + dedup         │
          │  — same-id check (registry entry │
          │    already imported → show as    │
          │    origin=TENANT only)           │
          │  — Reciprocal Rank Fusion        │
          │    (equal weight A:B = 0.5:0.5)  │
          │  — slice by (page*size, size)    │
          │  — append origin: TENANT|REGISTRY│
          └──────────────────┬───────────────┘
                             ▼
              VenueSummaryListResponse(items: [...])
```

**Rejected alternatives.**

- _(Rejected)_ Materialized registry copy per tenant schema. Duplicates N×registry rows; scraper/admin updates would need fan-out propagation — consistency nightmare.
- _(Rejected)_ Single `UNION ALL` or JOIN in one SQL statement with `public.` qualified names. Breaks search_path isolation abstraction (planner cardinality on two heterogenous branches produces bad plans), risks admin-only column leakage if the mapper SELECT list is later widened naively, and couples two sources that may scale independently.
- _(Rejected for MVP)_ Dedicated OpenSearch/Elasticsearch cluster. Over-engineering at MVP scales (< 500 registry entries, < 1000 tenant venues per tenant). PostgreSQL keyword + geo + JSONB filters is sufficient; promote to dedicated search if registry count breaches 100 K rows or tenant-side vector recall becomes a bottleneck (see §17 Phase 2 signal).

**Failure isolation.** Branch B (registry) timeout or SQL exception → search does _not_ fail the whole request. Returns branch A results with an RFC 7807 `Warning: 299 - "Registry search unavailable, results incomplete"` response header and increments `iqbene_search_failures_total{branch="registry"}`.

**Search parameters.**

- New parameter `scope` (enum `TENANT_ONLY`, `REGISTRY_ONLY`, `BOTH`). Default: `BOTH` so the default search bar shows the union immediately.
- Semantic mode with `search` natural-language query: only branch A uses cosine similarity against `venue_vectors` per-tenant. Branch B falls back to keyword + structured for MVP; registry semantic similarity and entry embeddings are a Phase 2 decision.

**Response shape additions.**

- Each `VenueSummaryView` record includes a new `origin: "TENANT" | "REGISTRY"` field. Consumers route UX actions accordingly (see `from-registry` import endpoint in §7 Venues API).
- `totalElements` in `VenueSummaryListResponse` is deliberately _approximate_ for `scope=BOTH` (equal ranks across two top-K → the real total is unknown without issuing two `COUNT(*)`). Clients show "1K+" or "Load more" buttons rather than rendering a page 50 pagination bar.

**Import-on-click semantics.** User clicks a REGISTRY-origin result → client calls `POST /api/v1/venues/from-registry/{registryEntryId}` (see §7). Creates a fresh tenant-owned `venues` row copying registry fields into `metadata` with `metadata_sources[*].source = "REGISTRY"` lowest priority — exactly like extraction-time gap-fill, but user-initiated. Subsequent searches show the result as origin `TENANT` and the merge step deduplicates (de-dup key: `t_{tenant}.venues.registry_entry_id = registryEntry.id` stored on import; see §10 schema).

---

## 7. API Surface (iqbene-venue-service)

All endpoints follow platform conventions based on the actual implementation in `foundation-cms-service` and `foundation-iam-service`:

- Base path: `/api/v1/venues` — always versioned
- All endpoints require `Bearer` JWT — no public routes
- `POST` (create) → `201 Created` with resource body
- `GET` (read) → `200 OK`
- `PUT` (full replace) → `200 OK`
- `PATCH` (partial update) → `200 OK`
- `DELETE` → `204 No Content`
- Paginated responses: custom wrapper records (e.g. `VenueSummaryListResponse(items, totalElements)`) — not Spring's `Page<T>`
- Error responses: RFC 7807 `ProblemDetail`, `type` = `about:blank`, includes `correlationId` and `requestId` extension properties
- Authority strings are bare — `USER`, `ADMIN`, `TENANT_OWNER` (never `ROLE_` prefixed)
- Tenant context is set automatically by `TenantExtractionFilter` from JWT `tenant_id` claim — no tenant path variable needed on regular tenant-scoped endpoints

---

### Venues

Base: `/api/v1/venues`

| Method   | Path                               | Authority               | Status | Request / Response                                                    | Notes                                                                                                                                                                                                                                                              |
| -------- | ---------------------------------- | ----------------------- | ------ | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `GET`    | `/`                                | `MEMBER`                | 200    | Query: `page`, `size`, `sort`, `status`, `search`, `scope` → response | Hybrid search + filter when `search` param present; `scope=TENANT_ONLY                                                                                                                                                                                             | REGISTRY_ONLY | BOTH`(default`BOTH`) |
| `POST`   | `/`                                | `MEMBER`                | 201    | `CreateVenueRequest` → `VenueResponse`                                | Enforces `max_venues` plan limit before insert                                                                                                                                                                                                                     |
| `GET`    | `/{id}`                            | `MEMBER`                | 200    | → `VenueResponse` (with consolidated metadata)                        | 404 if not found or belongs to different tenant                                                                                                                                                                                                                    |
| `PATCH`  | `/{id}`                            | `MEMBER`                | 200    | `UpdateVenueRequest` → `VenueResponse`                                | Partial update — ignores null fields                                                                                                                                                                                                                               |
| `DELETE` | `/{id}`                            | `ADMIN`, `TENANT_OWNER` | 204    | → empty body                                                          | Soft delete: sets `status = ARCHIVED`. Hard delete: separate admin endpoint (Phase 3)                                                                                                                                                                              |
| `POST`   | `/from-registry/{registryEntryId}` | `MEMBER`                | 201    | → `VenueResponse`                                                     | Copy-on-import: clones registry entry fields into new tenant `venues` row with `metadata_sources[*].source = "REGISTRY"` (lowest priority). Stores `venues.registry_entry_id` for cross-source dedup. Idempotent: re-import returns the existing tenant venue row. |

#### Search

Search is served via the list endpoint (`GET /`) with query parameters — no separate `POST /search` needed at this scale. Complex multi-field queries are passed as structured query params.

If query complexity grows beyond what URL params can express cleanly (Phase 2), introduce `POST /search` with a `VenueSearchRequest` body. `GET` with a body is non-standard and must not be used.

| Parameter    | Type              | Description                                                    |
| ------------ | ----------------- | -------------------------------------------------------------- |
| `search`     | string            | Natural language or keyword query — triggers hybrid mode       |
| `scope`      | enum              | `TENANT_ONLY`, `REGISTRY_ONLY`, `BOTH` (default)               |
| `status`     | enum              | `DRAFT`, `ACTIVE`, `ARCHIVED`                                  |
| `capacity`   | integer           | Minimum total capacity                                         |
| `lat`, `lng` | decimal           | Centre point for geo-spatial search                            |
| `radius_km`  | decimal           | Radius from lat/lng (requires both to be set)                  |
| `amenities`  | string[]          | Required amenity codes (comma-separated)                       |
| `catering`   | enum              | Catering policy filter                                         |
| `page`       | integer (0-based) | Default: 0                                                     |
| `size`       | integer           | Default: 20, max: 100                                          |
| `sort`       | string            | e.g. `name,asc` or `relevance` (default when `search` present) |

#### DTOs

```
CreateVenueRequest  — name (required), address, description, tags
UpdateVenueRequest  — all fields optional; null fields ignored (PATCH semantics)
VenueResponse       — id, name, address, location, status, metadata (consolidated),
                       metadata_aggregated_at, asset_count, registry_entry_id (nullable),
                       created_by, created_at, updated_at
VenueSummaryView    — id, name, address, location, summary metadata slice, origin
                       (origin: "TENANT" | "REGISTRY")
```

---

### Registry Entries (MEMBER-read, ADMIN-write via admin API)

Base: `/api/v1/registry/entries`

Registry entry GET endpoints are always MEMBER-authenticated (JWT-required, same authority as tenant venues). Write / edit / delete endpoints live under the Platform Admin API scope (§13; `PLATFORM_ADMIN` authority only).

| Method | Path    | Authority | Status | Request / Response                               | Notes                                                                                                                                                                                                    |
| ------ | ------- | --------- | ------ | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET`  | `/{id}` | `MEMBER`  | 200    | → `RegistryEntryResponse` (safe projection only) | Read-only public-facing projection of `venue_registry` + aliases. Never returns admin-only fields (confidence scores, private notes, source audit). 404 if entry does not exist or is `status=ARCHIVED`. |

```
RegistryEntryResponse — id, name, aliases[], address, location, metadata safe projection,
                         created_at, last_synced_at
```

---

### Assets

Base: `/api/v1/venues/{venueId}/assets`

Upload uses the two-phase presigned URL pattern (same as IAM avatar upload — no multipart to the service).

| Method   | Path            | Authority               | Status | Request / Response                                 | Notes                                                        |
| -------- | --------------- | ----------------------- | ------ | -------------------------------------------------- | ------------------------------------------------------------ |
| `GET`    | `/`             | `MEMBER`                | 200    | → `List<AssetResponse>`                            | All assets for venue; not paginated (reasonable upper bound) |
| `POST`   | `/initiate`     | `MEMBER`                | 201    | `InitiateUploadRequest` → `InitiateUploadResponse` | Returns `asset_id` + presigned S3 PUT URL (15 min TTL)       |
| `POST`   | `/{id}/confirm` | `MEMBER`                | 200    | → `AssetResponse`                                  | Marks asset ready, publishes `asset.uploaded` event          |
| `DELETE` | `/{id}`         | `ADMIN`, `TENANT_OWNER` | 204    | → empty body                                       | Deletes asset record + S3 object + associated vectors        |

Enforces `max_assets_per_venue` plan limit at `POST /initiate`.

#### DTOs

```
InitiateUploadRequest  — file_name (required), content_type (required), size_bytes (required),
                          asset_type (required: PDF_DECK | FLOOR_PLAN | PHOTO | VIDEO | CAD_FILE | SPEC_SHEET | MISC)
InitiateUploadResponse — asset_id (UUID), upload_url (presigned S3 PUT, 15 min), expires_at
AssetResponse          — id, venue_id, asset_type, file_name, content_type, size_bytes,
                          extraction_status, uploaded_by, uploaded_at
```

Plan gate: `cad_support` feature checked at `POST /initiate` when `asset_type = CAD_FILE`. Returns `403 Forbidden` with `ProblemDetail` (feature not on current plan) if not on qualifying plan.

---

### Metadata

Base: `/api/v1/venues/{venueId}/metadata`

| Method  | Path               | Authority | Status | Request / Response                                  | Notes                                                                      |
| ------- | ------------------ | --------- | ------ | --------------------------------------------------- | -------------------------------------------------------------------------- |
| `GET`   | `/`                | `MEMBER`  | 200    | → `MetadataResponse` (consolidated + provenance)    | Includes confidence scores and source attribution                          |
| `PATCH` | `/{field}`         | `MEMBER`  | 200    | `MetadataOverrideRequest` → `MetadataFieldResponse` | Manual override — sets source = `MANUAL_OVERRIDE`, triggers re-aggregation |
| `GET`   | `/{field}/history` | `MEMBER`  | 200    | → `List<MetadataEventResponse>`                     | Full extraction + override history for a single field                      |

`PATCH /{field}` is used for manual override (partial update of a single field) — `POST` would imply creating a new resource. `PATCH` semantics are correct here: "update this field to this value."

#### DTOs

```
MetadataResponse       — fields (map of field → value + confidence + source), aggregated_at,
                          conflict_count (int: fields with unresolved competing values)
MetadataOverrideRequest — value (required), reason (optional free text)
MetadataFieldResponse  — field, value, confidence, source_type, source_id, updated_at
MetadataEventResponse  — event_type, source_type, source_id, value, confidence, occurred_at, created_by
```

---

### Extraction Jobs

Base: `/api/v1/venues/{venueId}/extractions`

Read-only for API clients. Jobs are created internally when `asset.uploaded` is consumed.

| Method | Path       | Authority | Status | Response                      | Notes                                             |
| ------ | ---------- | --------- | ------ | ----------------------------- | ------------------------------------------------- |
| `GET`  | `/`        | `MEMBER`  | 200    | `List<ExtractionJobResponse>` | All jobs for venue, ordered by `started_at DESC`  |
| `GET`  | `/{jobId}` | `MEMBER`  | 200    | `ExtractionJobResponse`       | Single job status + extracted data (if completed) |

#### DTOs

```
ExtractionJobResponse — id, asset_id, status, extractor_type, confidence_scores (map),
                         started_at, completed_at, error_message
```

---

### Error responses

All errors use Spring's `ProblemDetail` (RFC 7807). The `type` field is `about:blank` for all errors (matches actual platform implementation in `GlobalExceptionHandler`). Every error response includes `correlationId` and `requestId` as extension properties.

| Scenario                             | Status | Title                   |
| ------------------------------------ | ------ | ----------------------- |
| Venue not found                      | 404    | `Not Found`             |
| Asset not found                      | 404    | `Not Found`             |
| Venue quota reached (plan limit)     | 402    | `Plan upgrade required` |
| Asset quota reached (plan limit)     | 402    | `Plan upgrade required` |
| Feature not on plan (e.g. CAD files) | 403    | `Plan upgrade required` |
| Validation failure                   | 400    | `Validation Failed`     |
| Tenant context missing               | 400    | `Bad Request`           |
| Access denied                        | 403    | `Forbidden`             |
| Token revoked / expired              | 401    | `Unauthorized`          |

For plan-gated features (`PlanFeatureNotAvailableException`): status `403`, plus `featureCode` extension property naming the specific feature code (e.g. `cad_support`, `semantic_search`).

For quota limits (`PlanMemberQuotaException` equivalent for venues/assets): status `402`.

---

## 8. Event Contracts (RabbitMQ)

Exchange: `iqkv.events` (Topic) — same exchange used by all foundation services.

### Published by iqbene-venue-service

| Routing key      | Payload fields                                                  | Description                           |
| ---------------- | --------------------------------------------------------------- | ------------------------------------- |
| `venue.created`  | venue_id, tenant_id, created_by                                 | New venue profile created             |
| `venue.updated`  | venue_id, tenant_id, changed_fields                             | Venue fields updated                  |
| `asset.uploaded` | asset_id, venue_id, tenant_id, asset_type, s3_key, content_type | Asset confirmed, ready for extraction |
| `asset.deleted`  | asset_id, venue_id, tenant_id                                   | Asset removed                         |

### Published by iqbene-venue-ingestion-worker

| Routing key            | Payload fields                                | Description           |
| ---------------------- | --------------------------------------------- | --------------------- |
| `extraction.started`   | job_id, asset_id, venue_id, tenant_id         | Processing began      |
| `extraction.completed` | job_id, asset_id, venue_id, tenant_id         | Extraction succeeded  |
| `extraction.failed`    | job_id, asset_id, venue_id, tenant_id, reason | All retries exhausted |

For the scalable topology (§3, variant A2), the publisher appends a hash-slot suffix to the `extraction.completed` routing key: `extraction.completed.{slot}` where `slot = Math.abs(venueId.hashCode() % SLOT_COUNT)`. Same `venue_id` always produces the same slot.

### Consumed by iqbene-venue-ingestion-worker

| Routing key      | Queue                                     | Action                                |
| ---------------- | ----------------------------------------- | ------------------------------------- |
| `asset.uploaded` | `iqbene.extraction.priority` (Enterprise) | Trigger ETL pipeline immediately      |
| `asset.uploaded` | `iqbene.extraction.standard` (Free/Pro)   | Trigger ETL pipeline (standard queue) |

### Consumed by iqbene-venue-service

| Routing key            | Queue (MVP A1)                               | Queue (Scalable A2)                                                                                                         | Action                                |
| ---------------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| `extraction.completed` | `iqbene.metadata.aggregation` (single queue) | `iqbene.metadata.aggregation.0` … `iqbene.metadata.aggregation.15` (16 slots by default, slot-bound via routing key suffix) | Run metadata aggregation for venue    |
| `extraction.failed`    | `iqbene.extraction.dlq`                      | `iqbene.extraction.dlq`                                                                                                     | Mark asset extraction_status = FAILED |

### Metadata aggregation queue — consumer configuration

The `MetadataAggregationConsumer` listener container in `iqbene-venue-service` must be configured to serialise processing per queue so that per-venue events never execute concurrently:

| Configuration    | Value (A1)                                          | Value (A2)  | Notes                                                                                      |
| ---------------- | --------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------ |
| `concurrency`    | `1`                                                 | `16`        | A2: one thread per slot queue. Total threads = `SLOT_COUNT`.                               |
| `prefetchCount`  | `1`                                                 | `1`         | Critical. A worker must never prefetch a batch; one in-flight message per consumer thread. |
| Acknowledge mode | `MANUAL`                                            | `MANUAL`    | Ack sent only after the wrapping DB transaction commits (see §3).                          |
| Error handler    | Reject + requeue on transient (≤3x) → DLQ on fatal. | Same as A1. | Aggregation is idempotent via `metadata_aggregated_at` debounce (§3) → safe requeues.      |

The consumer listens either on one queue (A1) or on all 16 slot queues (A2) via a queue array or wildcard in `@RabbitListener`. The consumer handler code is identical across variants — only the listener container configuration differs.

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

Migrations live in `iqbene-venue-model` under `src/main/resources/db/changelog/tenant/` — the shared library is the single source of truth for schema. Both `iqbene-venue-service` and `iqbene-venue-ingestion-worker` include the library on their classpath; `iqbene-venue-service` runs the migrations on startup (or a dedicated init container applies them on tenant provisioning via `TenantProvisionedEvent` listener, same pattern as IAM).

### Naming and format conventions

Follows the platform coding guidelines (§12):

- Format: **XML only** — no SQL scripts, no YAML
- File naming: `YYYYMMDDhhmmss-description.xml` (timestamp prefix + kebab-case description)
- Changeset ID: matches filename without `.xml` extension
- Author: `iqkv`
- Every changeset must include a `<rollback>` block
- Never modify an existing changeset — add a new one

### Migration files

```
db/changelog/system/
├── master.xml
└── 20250101000000-create-venue-registry.xml   ← public schema, platform-owned

db/changelog/tenant/
├── master.xml
├── 20250101000001-create-venues.xml
├── 20250101000002-create-venue-assets.xml
├── 20250101000003-create-extraction-jobs.xml
├── 20250101000004-create-metadata-events.xml
├── 20250101000005-create-venue-vectors.xml
└── 20250101000006-create-ai-cost-tracking.xml
```

The `TenantLiquibaseRunner` from `foundation-tenancy` applies `system/master.xml` first (to the `public` schema), then `tenant/master.xml` to each `t_{tenantKey}` schema. The `venue_registry` and `venue_registry_aliases` tables are created in `public` once, not per-tenant.

### Changeset structure (example: venues table)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.33.xsd">

  <changeSet id="20250101000001-create-venues" author="iqkv">

    <!-- Extensions applied once per tenant schema -->
    <sql>CREATE EXTENSION IF NOT EXISTS vector;</sql>
    <sql>CREATE EXTENSION IF NOT EXISTS postgis;</sql>

    <createTable tableName="venues">
      <column name="id" type="UUID">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <column name="name" type="VARCHAR(255)">
        <constraints nullable="false"/>
      </column>
      <column name="address" type="TEXT"/>
      <column name="location" type="GEOGRAPHY(POINT, 4326)"/>
      <column name="description" type="TEXT"/>
      <column name="description_embedding" type="VECTOR(1536)"/>
      <column name="description_text" type="TSVECTOR"/>
      <column name="status" type="VARCHAR(20)" defaultValue="DRAFT">
        <constraints nullable="false"/>
      </column>
      <column name="metadata" type="JSONB" defaultValue="{&quot;_schema_version&quot;:1}">
        <constraints nullable="false"/>
      </column>
      <column name="metadata_sources" type="JSONB" defaultValue="{}">
        <constraints nullable="false"/>
      </column>
      <column name="metadata_aggregated_at" type="TIMESTAMP"/>
      <column name="registry_entry_id" type="UUID">
        <remarks>FK to public.venue_registry.id. Populated when venue is created via user-initiated
        copy-on-import (POST /venues/from-registry/{id}) or by extraction-time gap-fill when
        matcher produces MATCH with no ambiguity. NULL when venue was created directly
        (admin POST /venues or extraction with no registry match). Used by search orchestrator
        for cross-source dedup (registry-origin result already imported → show as TENANT only).
        No DB-level FK constraint (cross-schema FKs are not supported in PostgreSQL across
        tenant schemas + public). Referential integrity is enforced at application level;
        sweeping orphan references on registry.admin.entry.deleted event is a Phase 2 task.</remarks>
      </column>
      <column name="created_by" type="UUID">
        <constraints nullable="false"/>
      </column>
      <column name="created_at" type="TIMESTAMP" defaultValueComputed="NOW()">
        <constraints nullable="false"/>
      </column>
      <column name="updated_at" type="TIMESTAMP" defaultValueComputed="NOW()">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <createIndex tableName="venues" indexName="idx_venues_embedding" using="ivfflat">
      <column name="description_embedding vector_cosine_ops"/>
    </createIndex>
    <createIndex tableName="venues" indexName="idx_venues_fts" using="gin">
      <column name="description_text"/>
    </createIndex>
    <createIndex tableName="venues" indexName="idx_venues_metadata" using="gin">
      <column name="metadata jsonb_path_ops"/>
    </createIndex>
    <createIndex tableName="venues" indexName="idx_venues_location" using="gist">
      <column name="location"/>
    </createIndex>

    <sql>
      CREATE TRIGGER trg_venues_tsvector BEFORE INSERT OR UPDATE ON venues
        FOR EACH ROW EXECUTE FUNCTION
        tsvector_update_trigger(description_text, 'pg_catalog.english', name, description, address);
    </sql>

    <rollback>
      <sql>DROP TRIGGER IF EXISTS trg_venues_tsvector ON venues;</sql>
      <dropIndex tableName="venues" indexName="idx_venues_location"/>
      <dropIndex tableName="venues" indexName="idx_venues_metadata"/>
      <dropIndex tableName="venues" indexName="idx_venues_fts"/>
      <dropIndex tableName="venues" indexName="idx_venues_embedding"/>
      <dropTable tableName="venues"/>
    </rollback>

  </changeSet>
</databaseChangeLog>
```

### Schema overview (all tables)

**Public schema (platform-owned, not tenant-scoped):**

| Table                    | Key columns                                                                                                                                                     | Notes                                                                           |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `venue_registry`         | `id` UUID PK, `name` VARCHAR(255), `address` TEXT, `city` VARCHAR(100), `location` GEOGRAPHY, `metadata` JSONB, `confidence` NUMERIC(3,2), `source` VARCHAR(50) | Platform seed data. Read-only to tenants. `metadata._schema_version` mandatory. |
| `venue_registry_aliases` | `id` UUID PK, `venue_registry_entry_id` UUID FK, `alias` VARCHAR(255)                                                                                           | Alternative names for registry deduplication                                    |

**Tenant schema `t_{tenantKey}` (tenant-owned):**

| Table                   | Key columns                                                                                                                                                         | Owner                                                                             |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `venues`                | `id` UUID PK, `status` VARCHAR(20), `metadata` JSONB, `description_embedding` VECTOR(1536), `location` GEOGRAPHY, `registry_entry_id` UUID (nullable, app-level FK) | `iqbene-venue-service`. `metadata._schema_version` mandatory, default 1 on insert |
| `venue_assets`          | `id` UUID PK, `venue_id` UUID FK, `asset_type` VARCHAR(50), `extraction_status` VARCHAR(20), `extracted_text_embedding` VECTOR(1536)                                | `iqbene-venue-service`                                                            |
| `extraction_jobs`       | `id` UUID PK, `asset_id` UUID FK, `status` VARCHAR(20), `extractor_type` VARCHAR(50), `extracted_data` JSONB, `confidence_scores` JSONB                             | `iqbene-venue-ingestion-worker`                                                   |
| `venue_metadata_events` | `id` UUID PK, `venue_id` UUID FK, `event_type` VARCHAR(50), `event_data` JSONB — append-only                                                                        | `iqbene-venue-service`                                                            |
| `venue_vectors`         | `id` UUID PK, `content` TEXT, `metadata` JSONB, `embedding` VECTOR(1536) — Spring AI PgVectorStore table                                                            | `iqbene-venue-ingestion-worker`                                                   |
| `ai_cost_tracking`      | `id` UUID PK, `provider` VARCHAR(50), `model` VARCHAR(100), `tokens_used` INTEGER, `cost_usd` NUMERIC(10,6)                                                         | `iqbene-venue-ingestion-worker`                                                   |

### Index strategy summary

| Table              | Index name                   | Type    | Column(s)                         | Purpose                                                                                  |
| ------------------ | ---------------------------- | ------- | --------------------------------- | ---------------------------------------------------------------------------------------- |
| `venues`           | `idx_venues_embedding`       | IVFFlat | `description_embedding`           | Semantic search                                                                          |
| `venues`           | `idx_venues_fts`             | GIN     | `description_text`                | Full-text search                                                                         |
| `venues`           | `idx_venues_metadata`        | GIN     | `metadata jsonb_path_ops`         | JSONB attribute filters                                                                  |
| `venues`           | `idx_venues_location`        | GIST    | `location`                        | Geo-spatial queries                                                                      |
| `venue_assets`     | `idx_assets_venue`           | btree   | `venue_id`                        | FK lookup                                                                                |
| `venue_assets`     | `idx_assets_type`            | btree   | `asset_type`                      | Filter by type                                                                           |
| `venue_assets`     | `idx_assets_embedding`       | IVFFlat | `extracted_text_embedding`        | Chunk-level vector search                                                                |
| `extraction_jobs`  | `idx_jobs_asset`             | btree   | `asset_id`                        | FK lookup                                                                                |
| `extraction_jobs`  | `idx_jobs_status`            | btree   | `status`                          | Queue polling                                                                            |
| `metadata_events`  | `idx_metadata_events_venue`  | btree   | `venue_id, occurred_at DESC`      | Timeline queries                                                                         |
| `metadata_events`  | `idx_metadata_events_source` | btree   | `source_id, source_type`          | Provenance lookup                                                                        |
| `venue_vectors`    | `idx_vectors_embedding`      | IVFFlat | `embedding`                       | Vector similarity search                                                                 |
| `venue_vectors`    | `idx_vectors_asset`          | btree   | `(metadata->>'asset_id')`         | Fast sweep on asset.deleted event                                                        |
| `ai_cost_tracking` | `idx_ai_cost_month`          | btree   | `DATE_TRUNC('month', created_at)` | Monthly cost rollup                                                                      |
| `venues`           | `idx_venues_registry_entry`  | btree   | `registry_entry_id`               | Cross-source search dedup (find tenant venues already imported from a registry entry id) |

### Cross-schema Access Rules (registry in public vs tenant schema)

PostgreSQL schema-per-tenant tenancy model means `public.venue_registry` lives in a different schema than `t_{tenantKey}.venues`. The `MyBatisSchemaInterceptor` sets `SET search_path TO t_{tenantKey}, public` before every statement, so `public` _objects are_ accessible from tenant connection — but access is tightly restricted to prevent abstraction leakage and accidental cross-tenant writes.

**Rules (enforced at code review + MyBatis mapper audit):**

1. **Only three mapper interfaces are permitted to reference `public.` schema-qualified names explicitly.** Every other mapper MUST use unqualified table names so that `search_path` resolves them _inside_ the tenant schema. Permitted qualified readers:
   - `RegistryEntryQueryMapper` (search-read branch B): `FROM public.venue_registry r LEFT JOIN public.venue_registry_aliases a …` — read-only SELECT list with a fixed column whitelist (no `SELECT *`), never admin-only fields.
   - `VenueRegistryMatcherMapper` (extraction-time gap-fill in §5 Stage 3): same qualified reads.
   - The `RegistryAdminMapper` (PLATFORM_ADMIN only, §13): reads + writes `public.venue_registry` / aliases.

2. **Single-statement cross-schema `JOIN` / `UNION ALL` inside one MyBatis `<select>` is forbidden.** It couples planner statistics across two heterogenous tables, risks admin-only column leakage if a SELECT list is widened later, and breaks the "future-proof: swap schema-per-tenant for row-level tenancy" invariant. If we ever move to row-level tenancy (per-table `tenant_id` instead of schemas), every statement needs to re-qualify tables; the three qualified mappers above are the only audit surface. Cross-source merging always happens in the `VenueSearchOrchestrator` app-layer (§6), never in SQL.

3. **No cross-schema DDL-level FK constraints.** PostgreSQL does not support FKs across schemas owned by different role-level isolation; the `venues.registry_entry_id → venue_registry.id` reference is application-enforced only. Sweeper for orphaned `registry_entry_id` values after `registry.admin.entry.deleted` events is deferred to Phase 2.

4. **Role-level hardening (optional pre-production):** The connection pool `iqbene-venue-service` runs as is `t_{tenantKey}` owner only; grant `SELECT` on `public.venue_registry` / aliases to the app role explicitly. `INSERT / UPDATE / DELETE` on `public` tables to this role is revoked; only the `registry_admin` connection pool / PLATFORM_ADMIN user holds write grants on public registry tables.

---

## 11. UI Integration (foundation-ui-app)

Extend `foundation-ui-app` — do **not** fork. New iQ BENE features live under:

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

Both iQ BENE services follow foundation patterns exactly.

**Prometheus metrics to add:**

| Metric                                       | Labels                            | Notes                                                                                                                                                                                                                                                                                                                                        |
| -------------------------------------------- | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `iqbene_venues_total`                        | tenant_id, status                 | Venue count by state                                                                                                                                                                                                                                                                                                                         |
| `iqbene_assets_uploaded_total`               | tenant_id, asset_type             | Upload volume                                                                                                                                                                                                                                                                                                                                |
| `iqbene_extractions_total`                   | tenant_id, extractor_type, status | Success/failure rates                                                                                                                                                                                                                                                                                                                        |
| `iqbene_extraction_duration_seconds`         | extractor_type                    | Latency histogram                                                                                                                                                                                                                                                                                                                            |
| `iqbene_ai_cost_usd_total`                   | tenant_id, model                  | Cost tracking                                                                                                                                                                                                                                                                                                                                |
| `iqbene_search_requests_total`               | search_mode, scope                | keyword / semantic / hybrid. `scope=TENANT_ONLY                                                                                                                                                                                                                                                                                              | REGISTRY_ONLY | BOTH`.                                                                                                                                                                                                                        |
| `iqbene_search_latency_seconds`              | search_mode, branch               | Branch-level latency histogram. `branch=tenant` / `registry` / `orchestrator_total` — lets us attribute slowdowns to one of the two parallel SQL branches or the app-level merge step.                                                                                                                                                       |
| `iqbene_search_failures_total`               | branch                            | Counter incremented when a branch query fails (SQL exception, timeout). For `branch=registry` the orchestrator returns partial tenant results with a `Warning` header (see §6). For `branch=tenant` the whole request 5xx.                                                                                                                   |
| `iqbene_metadata_schema_version_seen_total`  | tenant_id, schema_version, op     | Counter incremented on every `migrateToCurrent()` or `ensureCurrent()` call — labels the schema version the document had _before_ migration. `op` = `read` or `write`. Lets us see how many legacy v0/v1/vN docs are still in the read/write hot paths so we know when old migration classes are candidates for retirement.                  |
| `iqbene_metadata_migration_duration_seconds` | target_version                    | Latency histogram for the full migration chain (per target version — always CURRENT_SCHEMA_VERSION in steady state). Alerts if a new migration step is unexpectedly slow for large JSONB payloads.                                                                                                                                           |
| `iqbene_registry_match_total`                | stage, outcome                    | `stage` = `"extraction"` (tenant gap-fill) or `"import_dedup"` (scraper/admin pre-write). `outcome` = `matched` / `ambiguous` / `no_match`. Tracks how often the registry produces useful hits. If `matched` plateaus near 0, the seed set is too small and scraper runs are overdue.                                                        |
| `iqbene_registry_match_confidence_seconds`   | stage                             | Histogram of the `combined` confidence score per matcher invocation. Buckets 0.5–0.6, 0.6–0.7, 0.7–0.8, 0.8–0.9, 0.9–1.0. Used during post-launch threshold recalibration — drift above the 0.75 MATCH bar indicates thresholds can be tightened; a spike of 0.72–0.75 near-boundary hits suggests a small bump to 0.78 would eliminate FPs. |
| `iqbene_venue_import_from_registry_total`    | tenant_id, status                 | User-initiated copy-on-import via `POST /venues/from-registry/{id}`. `status=created                                                                                                                                                                                                                                                         | duplicate     | error`. Measures how often the registry seed set is directly useful to end users. Baseline "registry ROI" signal — if `created`is flat vs`matched` (extraction-time), users prefer implicit gap-fill over explicit import UI. |

Grafana dashboard added to `docker/grafana/provisioning/dashboards/VipService.json`.

---

## 13. Security

| Concern                 | Approach                                                                                                                                                |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tenant data isolation   | Schema-per-tenant (PostgreSQL + pgvector); S3 key prefix `vip/tenants/{tenantKey}/` per tenant — see §4b for full key layout                            |
| Asset access            | Presigned S3 URLs only (15 min upload, 1h download). No public bucket. Registry paths (`vip/registry/*`) inaccessible via tenant-issued presigned URLs. |
| AI data handling        | Documents sent to OpenAI API per their data processing terms. Enterprise option: Azure OpenAI (data stays in tenant's region).                          |
| GDPR / right to erasure | `DELETE tenant` cascades to venues → assets → S3 objects → vector embeddings                                                                            |
| Audit trail             | All `venue.*`, `asset.*`, `extraction.*` events passively consumed by Audit Service                                                                     |
| PII in documents        | Warn on upload. Do not log extracted text.                                                                                                              |

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

Full rationale and competitor analysis: see `../business/comparison.md`.

---

## 15. Open Decisions (resolve before Sprint 1)

- [x] **One service or two?** ~~`iqbene-venue-service` + `iqbene-ai-service` vs. a single `iqbene-venue-service` with an internal AI module.~~ **Decided:** Two deployments — `iqbene-venue-service` (synchronous API, data-tied) and `iqbene-venue-ingestion-worker` (async sidecar, shared schema, no inbound HTTP). Services are tied to data; ingestion is a processing concern, not a peer service.
- [x] **Naming convention.** Service names reflect domain/purpose, not implementation technology. `iqbene-venue-ingestion-worker` describes what it does (ingest and process assets), not how (AI/ML).
- [x] **Platform venue registry.** A `public.venue_registry` table seeds new tenant venues with known data at extraction time. Copy-on-match, not link — tenant record is independent after copy. Source tagged `REGISTRY` in `metadata_sources`, lowest priority in conflict resolution. No reverse flow from tenant to registry in MVP.
- [x] **Metadata schema versioning and JSONB drift.** Every `venues.metadata` and `venue_registry.metadata` JSONB document carries a top-level integer `_schema_version` (initial: 1; absent = 0 "legacy"). The shared library `iqbene-venue-model` contains an append-only chain of pure-function `MetadataMigration` classes (`v0→v1`, `v1→v2`, …) and a `VenueMetadataMigrator` runner. Every read upgrades the shape in memory via a MyBatis `VenueMetadataTypeHandler`; every write stamps the document to `CURRENT_SCHEMA_VERSION` before persist. No offline backfill job, incremental online convergence. Same migrator classpath-identical in both services, zero drift. Full design in §2a.
- [x] **Metadata aggregation race condition prevention.** How to prevent Lost Update when N extraction jobs for the same venue publish `extraction.completed` concurrently? **Rejected:** optimistic locking with retry-loop (complex retry code, hard to test livelock scenarios, conflicts hit the database first). **Rejected:** distributed locks (Redis or advisory locks — new dependency, deadlock surface, operational complexity). **Rejected:** `SELECT … FOR UPDATE` row locks (serialises at the DB, works but requires explicit transaction scripting and still contends on hot venues). **Decided:** RabbitMQ FIFO routing per `venue_id` using hash-partitioned queues (§3). Eliminates the race at the messaging layer before the consumer runs. Zero new dependencies, no retry code. Variant A1 (single queue, concurrency=1, prefetch=1) for MVP; variant A2 (16 hash-slot queues) when throughput requires. Consumer wraps the SELECT+merge+UPDATE in a single DB transaction + MANUAL ack only after COMMIT. Composes naturally with the existing 5 s debounce window — three rapid events become one aggregation. Full design in §3 "Concurrency Control and Race Condition Prevention" and §8 queue configuration.
- [x] **Cross-source search architecture (tenant venues + public registry across schema boundary).** **Decided (Approach 2):** two parallel SQL branches → app-level merge, no PostgreSQL cross-schema JOIN/UNION in a single statement. Search bar on the tenant's venues page returns the union by default. **Rejected:** schema-level registry materialised copy per tenant (N×row duplication, scraper admin updates become fan-out consistency nightmare). **Rejected:** single SQL `UNION ALL` between `public.venue_registry` and `t_tenant.venues` in one MyBatis statement — couples planner statistics on two heterogenous tables, risks leaking admin-only registry columns if the SELECT whitelist is widened later, breaks the "swap schema-per-tenant ↔ row-level tenancy" future-proof invariant (would have to rewrite all UNION queries). **Rejected for MVP:** dedicated OpenSearch/Elasticsearch unified index. New operational service, over-engineering at MVP scale (< 500 registry rows, < 1000 tenant venues per tenant). Promote to OpenSearch only if registry breaches 100 K rows _or_ registry semantic search switches on (§17 Phase 2 signal). **Flow:** `VenueSearchOrchestrator` issues branch A (tenant mapper `VenueMapper`, implicit search_path, full 5-mode hybrid incl. semantic pgvector cosine) and branch B (`RegistryEntryQueryMapper`, **explicitly schema-qualified `FROM public.venue_registry … LEFT JOIN public.venue_registry_aliases`**, MVP 3-mode keyword + structured + geo only — no semantic on registry in MVP) via `CompletableFuture.supplyAsync` independent threads. App-level: dedup (if `venues.registry_entry_id == registryEntry.id` — registry result already imported → drop the REGISTRY origin, keep TENANT origin only) → reciprocal rank fusion A:B weight 0.5:0.5 (equal weight, re-tune post-launch if bias is observed) → slice page 20 → append `origin="TENANT"|"REGISTRY"` on each summary record. **Failure isolation:** branch B timeout/exception → return branch A only with HTTP `Warning: 299 - "Registry search unavailable"` header + Micrometer `iqbene_search_failures_total{branch="registry"}`. **New API surface (§7):** `GET /api/v1/venues/?scope=TENANT_ONLY|REGISTRY_ONLY|BOTH` (default `BOTH`); new DTO fields `VenueResponse.registry_entry_id` (nullable UUID, app-level FK populated on `from-registry` import or unambiguous extraction-time MATCH) and `VenueSummaryView.origin` enum; `GET /api/v1/registry/entries/{id}` (MEMBER auth, safe projection, never admin-only fields); `POST /api/v1/venues/from-registry/{registryEntryId}` (copy-on-import, REGISTRY lowest-priority source tagged, idempotent — re-import returns existing). **Cross-schema access rules (§10):** only three MyBatis mappers are ever allowed to emit `public.`-qualified SQL — `RegistryEntryQueryMapper` (read search), `VenueRegistryMatcherMapper` (read gap-fill), `RegistryAdminMapper` (admin writes). All other mappers use unqualified names via search_path. Single-statement JOIN/UNION across schemas is forbidden. **Metrics (§12):** `iqbene_search_requests_total{search_mode, scope}`, `iqbene_search_latency_seconds{search_mode, branch=tenant|registry|orchestrator_total}`, `iqbene_venue_import_from_registry_total{tenant_id, status=created|duplicate|error}`. **Semantic search on registry entries:** NOT in MVP. Registry entry description embeddings (generation during scraper/admin apply + cosine branch in the orchestrator) is explicitly deferred to Phase 2 decision — see §17. Full design diagram and edge cases: see §6 "Cross-source Search (tenant venues + public registry)".
- [ ] **Docling in Phase 1?** Start with pure Tika (simpler). Add Docling sidecar in Phase 2 when floor plan / table fidelity is needed. **Lean: Tika-only for Phase 1.**
- [x] **Registry match threshold and algorithm.** Fuzzy name + PostGIS proximity, **no embeddings/LLM in the match path.** Registry is explicitly secondary — FP rate must stay ≤ 1 %, even at cost of elevated FN. Shared `normalize()` function strips leading articles, lowercases, strips non-alnum, collapses whitespace — applied identically to both sides of every comparison. **Cold-start population paths:** (1) hardcoded Liquibase XML seed rows (50–200 high-signal venues) in `iqbene-venue-model/src/main/resources/db/changelog/system/` → source `platform_seed`; (2) Platform Admin single-entity CRUD API (no bulk MVP) with pre-write dedup candidate list → admin confirms Merge/Insert manually → source `admin_import`; (3) standalone scraper scripts (Cvent etc.) → S3 `vip/registry/imports/{id}/` → admin-triggered `admin.registry.import.dry-run` RabbitMQ event → `VenueRegistryImportOrchestrator` produces a review CSV with `name_sim` + `geo_distance` + per-row action recommendation (MERGE/REVIEW/INSERT thresholds: 0.90+150m → MERGE rec, 0.75/300m → REVIEW rec, else INSERT rec) → admin edits Action column, re-uploads, fires `admin.registry.import.apply` → worker applies verbatim with no independent decisions → source `web_scrape`. **Tenant extraction-time gap-fill:** combined confidence = geo available ? `0.60·name_trigram_sim + 0.40·(geo_within_200m ? 1 : 0)` : `1.00·name_trigram_sim`. MATCH threshold = combined ≥ 0.75 (with geo) OR ≥ 0.90 (name-only) AND top-2 candidate delta ≥ 0.08 (not ambiguous). Below thresholds → silent no-op; REGISTRY is lowest conflict-resolution priority so tenant extraction/user input always overrides the copy. **Before Sprint 1 dry-run calibration on 50 real PDFs:** ground-truth each fixture, run `VenueRegistryMatcher` in no-write mode, build confusion matrix, accept only if FP ≤ 1 %, else incrementally raise thresholds. Full design: see "Registry population strategy (cold start for MVP)" section above. Micrometer metrics `iqbene_registry_match_total{stage,outcome}` + `iqbene_registry_match_confidence_seconds{stage}` histogram added to §12.
- [x] **Chunking table placement and naming.** Schema: inside each tenant schema `t_{tenantKey}` (same schema as venues / venue_assets, NOT public, NOT a separate cross-tenant vector schema). **Rejected:** shared cross-tenant `vector_store` in `public`. Rejection rationale: a shared table requires a `tenant_id` column plus an extra sweep job on `tenant.deleted`, loses the `DROP SCHEMA … CASCADE` implicit-vector-cleanup path we already rely on for GDPR erasure (§4b deletion cascade, line 786), opens a cross-tenant leak bug surface any time a WHERE clause forgets to filter by `tenant_id`, forces the venue+vectors write path across two resources that are no longer atomically transactional, and would produce one giant shared IVFFlat index whose REINDEX blocks every tenant on the cluster simultaneously. **Decided:** one `venue_vectors` table per tenant schema, routed by the same `MyBatisSchemaInterceptor` that sets `search_path` before every statement. All vectors live next to their venue data; one connection, one commit, per-schema compact IVFFlat indexes that can be reindexed per-tenant independently, schema-drop erasure works implicitly. **Table name:** `venue_vectors` — explicit `venue_` prefix that matches the rest of the venue-domain table family (`venues`, `venue_assets`, `venue_metadata_events`, `venue_groups` in Phase 2). **Rejected:** Spring AI's default name `vector_store` because it is generic, not self-documenting in a codebase that already carries venue-level `venues.description_embedding` and asset-level `venue_assets.extracted_text_embedding` as separate vector columns. Columns (already in §10 schema overview): `id UUID PK`, `content TEXT` (raw chunk), `metadata JSONB` (Spring AI tags: `venue_id`, `asset_id`, `asset_type`, `token_count`, `chunk_index`), `embedding VECTOR(1536)`. Spring AI integration is trivial — `TenantAwarePgVectorStore` (§5 Stage 3) wraps the default `PgVectorStore` and passes `"venue_vectors"` as the table-name constructor arg. Initial vector index: IVFFlat (IVFFlat `idx_vectors_embedding` in §10, cosine-distance operator), which is cheaper than HNSW at MVP volumes. Added `idx_vectors_asset` btree expression index on `(metadata->>'asset_id')` to make the `asset.deleted` sweep an index scan instead of a full-table seq scan (see §10 index strategy). See §17 Phase 2 design signal for the 1 M-row IVFFlat→HNSW evaluation trigger.
- [ ] **Cost tracking granularity:** per-asset or per-tenant-per-month? Both are in schema; decide which is surfaced in UI.
- [ ] **Old migration class retirement policy.** After how many consecutive months of zero hits on the `schema_version < N` Prometheus counter do we delete the oldest migration classes from the chain? Define the guardrail before the first schema bump so we do not accumulate deprecated code indefinitely.

---

## 16. Implementation patterns (grounded in actual platform code)

This section records how core cross-cutting concerns are implemented in the existing foundation services. iQ BENE must follow these patterns exactly — they are not aspirational, they are the actual running code.

---

### Stack

| Concern          | Technology                                 | Notes                                                      |
| ---------------- | ------------------------------------------ | ---------------------------------------------------------- |
| Web tier         | Spring MVC (`spring-boot-starter-web`)     | Blocking servlet stack — no WebFlux except in the gateway  |
| Persistence      | MyBatis (`mybatis-spring-boot-starter`)    | No JPA, no Hibernate, no `@Entity` annotations anywhere    |
| Messaging        | RabbitMQ (`spring-boot-starter-amqp`)      | No Kafka, no Spring Cloud Stream                           |
| Schema migration | Liquibase XML changesets                   | No Flyway, no SQL scripts, no YAML changesets              |
| Domain objects   | Plain Java POJOs                           | No `@Entity`, no `@Column`, no ORM annotations of any kind |
| SQL mapping      | MyBatis `@Mapper` interfaces + XML mappers | `ResultMap` per entity, `SET search_path` via interceptor  |
| Security         | Spring Security OAuth2 Resource Server     | JWT validation via `NimbusJwtDecoder`, RS256 only          |

---

### Multi-tenancy: how it actually works

Tenancy is **schema-level**, not row-level. There is no `tenant_id` column on any table.

**Schema naming:** each tenant gets a PostgreSQL schema named `t_{tenantKey}` (e.g. `t_acme0001`). The `public` schema holds system-level tables (tenants registry, platform users).

**Routing mechanism — `MyBatisSchemaInterceptor`:**
Before every MyBatis statement execution, the interceptor sets:

```sql
SET search_path TO t_{tenantKey}, public
```

This is done via a MyBatis `StatementHandler.prepare` interceptor that reads `TenantContext.getCurrentTenant()`. If no tenant context is active (system-level operation), the interceptor skips and leaves the default `public` search path in place.

**TenantContext:**
`ThreadLocal<String>` holder in `foundation-tenancy`. Must be set before any DB call and cleared in a `finally` block after every request.

```java
// Set by TenantExtractionFilter before the request reaches the controller
TenantContext.setCurrentTenant(tenantKey);
try {
    filterChain.doFilter(request, response);
} finally {
    TenantContext.clear();  // always, even on exception
}
```

**`TenantExtractionFilter`** — `@Order(HIGHEST_PRECEDENCE + 1)`, present in every service:

1. Check `X-Tenant-ID` header (set by the Gateway — priority 1)
2. Decode Bearer JWT, read `tenant_id` claim (priority 2)
3. If neither resolves → `400 Bad Request` with inline `application/problem+json` body, request stops
4. Set `TenantContext`, continue filter chain
5. `finally` → `TenantContext.clear()`

Paths excluded from tenant extraction (`shouldNotFilter`):

- `/actuator/**`, `/api-docs/**`, `/swagger-ui/**`
- Admin cross-tenant paths (e.g. `/api/v1/venues/admin/**`)
- Any path where the controller sets `TenantContext` manually

**`TenantLiquibaseRunner`** — `ApplicationRunner` startup sequence:

1. Migrate `public` schema (system changelog)
2. If `iqkv.liquibase.upgrade-existing-tenants=true`: discover all `t_*` schemas from `information_schema.schemata` and apply pending tenant changesets to each — failures are logged and skipped, they do not abort startup
3. Apply tenant migrations to any `iqkv.liquibase.bootstrap-tenants` list (idempotent)
4. Always migrate the `platform` tenant (landing zone for new users before org creation)

---

### When to use public schema instead of tenant schema

Schema-per-tenant isolation (`t_{tenantKey}`) is the default for tenant-owned business data. It is not a hard rule for every table. The right choice depends on data granularity, volume, and isolation requirements.

**Use `public` schema when:**

- The data is platform-level and shared across all tenants — e.g. the tenants registry itself, platform users, platform-wide announcements, locale lists, token denylist
- The data volume is small and the cost of per-tenant schema duplication outweighs the benefit
- The data requires cross-tenant queries — a row-level `tenant_key` column is faster and simpler than union queries across schemas
- Isolation is non-critical — the data carries no per-tenant secrets or PII that would be exposed by cross-tenant reads

**Use `t_{tenantKey}` schema when:**

- The data is tenant-owned business content — venues, assets, CMS pages, extraction jobs
- Data volume per tenant can grow independently and benefits from isolated storage
- Hard isolation is required — one tenant must never be able to read another tenant's rows even in the event of a query bug

**IAM service as the canonical example:**
All provisioned SaaS tenants, users, memberships, and invitations live in `public` with a `tenant_key` column providing logical separation:

```xml
<!-- 20260115120000-create-tenants-table.xml -->
<createTable tableName="tenants" schemaName="public">
    <column name="tenant_key" type="VARCHAR(12)"/>
    ...
</createTable>

<!-- 20260115120100-create-users-table.xml -->
<createTable tableName="users" schemaName="public">
    <column name="id" type="UUID"/>
    <column name="email" type="VARCHAR(255)"/>
    ...
</createTable>
```

Users are not duplicated per tenant schema — they are global identities. The `users` table has no `tenant_key` column; membership records in a separate `tenant_memberships` table link users to tenants. This is correct: a user can belong to multiple tenants, and storing them in a per-tenant schema would require duplicating the user record across schemas.

**iQ BENE decision:**
All venue intelligence data (venues, assets, extraction jobs, vectors, metadata events) lives in `t_{tenantKey}` schemas — correct, because this is per-tenant business content with potentially large volume and a hard isolation requirement. If a future feature needs a cross-tenant platform table (e.g. a public venue directory or a shared taxonomy of venue categories), that table goes in `public` with a `tenant_key` column, not in per-tenant schemas.

---

### Security: how it actually works

**`SecurityConfig` pattern** (identical structure in every consumer service):

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**").permitAll()
                .requestMatchers("/api-docs/**", "/swagger-ui/**").permitAll()
                // service-specific public paths...
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .addFilterBefore(correlationIdFilter, BearerTokenAuthenticationFilter.class)
            .addFilterAfter(tenantExtractionFilter, CorrelationIdFilter.class);
        return http.build();
    }

    @Bean
    public JwtDecoder jwtDecoder() {
        // Deployed: JWKS URI (Nimbus caches keys, handles rotation transparently)
        if (jwksUri is configured) {
            return NimbusJwtDecoder.withJwkSetUri(jwksUri).build();
        }
        // Local dev / tests: RSA public key from PEM file (classpath: or file: path)
        return NimbusJwtDecoder.withPublicKey(loadRsaPublicKey(publicKeyPath)).build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        var converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            List<String> authorities = jwt.getClaimAsStringList(JwtClaimNames.AUTHORITIES);
            if (authorities == null) return List.of();
            return authorities.stream()
                .map(SimpleGrantedAuthority::new)
                .map(a -> (GrantedAuthority) a)
                .toList();
        });
        return converter;
    }
}
```

Key points:

- RS256 only — no HS256, no symmetric keys
- Authority strings are read directly from JWT `authorities` claim — no `ROLE_` prefix, no `JwtGrantedAuthoritiesConverter.setAuthorityPrefix`
- No `UserContext` class — authority is read from Spring Security's `Authentication` object
- Consumer services validate using JWKS URI (deployed) or local PEM (dev), never a shared secret

**JWT claim names** (actual wire names from `JwtClaimNames.java`):

| Constant               | Wire key               | Notes                                             |
| ---------------------- | ---------------------- | ------------------------------------------------- |
| `USER_ID`              | `user_id`              | snake_case — not `userId`                         |
| `EMAIL`                | `email`                |                                                   |
| `FIRST_NAME`           | `first_name`           | snake_case — not `firstName`                      |
| `LAST_NAME`            | `last_name`            | snake_case — not `lastName`                       |
| `TENANT_ID`            | `tenant_id`            | 8-char nanoid `tenant_key`, not the internal UUID |
| `AUTHORITIES`          | `authorities`          | `List<String>`, bare strings                      |
| `PLAN_CODE`            | `plan_code`            | Active plan code, used for entitlement checks     |
| `EMAIL_VERIFIED`       | `email_verified`       |                                                   |
| `ONBOARDING_COMPLETED` | `onboarding_completed` |                                                   |
| `PROFILE_COMPLETED`    | `profile_completed`    |                                                   |
| `TYPE`                 | `type`                 | `"access"` or `"refresh"`                         |

**Authority strings in use** (actual, from `UserServiceConstants.java`):

| Authority        | Scope          | Who holds it                                                                                                       |
| ---------------- | -------------- | ------------------------------------------------------------------------------------------------------------------ |
| `PLATFORM_ADMIN` | Platform-level | Platform operator. Cross-tenant. Never auto-assigned — must be granted explicitly by a platform admin out-of-band. |
| `TENANT_OWNER`   | Tenant-scoped  | Tenant owner — full control within their tenant                                                                    |
| `ADMIN`          | Tenant-scoped  | Tenant admin — elevated access within their tenant                                                                 |
| `MEMBER`         | Tenant-scoped  | Regular authenticated tenant member                                                                                |

`USER` does not exist on this platform. Use `MEMBER` for regular authenticated users.

Use `hasAnyAuthority(...)` in `@PreAuthorize`. Never `hasRole(...)`.

**`JwtAuthenticationFilter` in IAM** is a denylist checker — it is IAM-specific, not a pattern to replicate in iQ BENE. Consumer services like iQ BENE do not need a denylist filter; revocation is enforced by token TTL and the IAM's Redis denylist at source.

---

### REST controllers: how they actually work

Three access patterns exist in the platform. iQ BENE uses the first two:

**Pattern 1 — Tenant-scoped (standard).**
Controller at `/api/v1/venues/**`. `TenantExtractionFilter` sets `TenantContext` from JWT before the request hits the controller. No tenant path variable. No manual `TenantContext.setCurrentTenant()` in the controller.

```java
@RestController
@RequestMapping("/api/v1/venues")
@Tag(name = "Venues", description = "Venue management")
@SecurityRequirement(name = "bearerAuth")
@PreAuthorize("hasAnyAuthority('TENANT_OWNER', 'ADMIN', 'MEMBER')")
public class VenueRestResource {

    @PostMapping
    public ResponseEntity<VenueDtos.VenueResponse> create(
            @Valid @RequestBody VenueDtos.CreateVenueRequest request) {
        // TenantContext is already set by TenantExtractionFilter
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(venueService.create(request));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAnyAuthority('TENANT_OWNER', 'ADMIN')")
    public ResponseEntity<Void> delete(@PathVariable UUID id) {
        venueService.archive(id);
        return ResponseEntity.noContent().build();
    }
}
```

**Pattern 2 — Admin cross-tenant.**
Controller at `/api/v1/venues/admin/{tenantKey}/**`. `TenantExtractionFilter` is excluded for `/admin/` paths. Controller sets `TenantContext` manually in try/finally per method.

```java
@RestController
@RequestMapping("/api/v1/venues/admin/{tenantKey}")
@PreAuthorize("hasAuthority('PLATFORM_ADMIN')")
public class AdminVenueRestResource {

    @GetMapping
    public ResponseEntity<VenueDtos.VenueSummaryListResponse> getAll(
            @PathVariable String tenantKey,
            @RequestParam(defaultValue = "20") int limit,
            @RequestParam(defaultValue = "0") int offset) {
        try {
            TenantContext.setCurrentTenant(tenantKey);
            return ResponseEntity.ok(venueService.getAllSummary(limit, offset));
        } finally {
            TenantContext.clear();
        }
    }
}
```

**What `@PreAuthorize` at class level means:** it applies to every method. Override per-method for stricter access (e.g. `DELETE` requiring `ADMIN` when the class allows `USER`).

---

### DTO pattern

All DTOs are Java **records**, grouped in a single `{Domain}Dtos.java` container class. Response wrappers for lists include `items` and `totalElements` — no Spring `Page<T>` in API responses.

```java
public final class VenueDtos {
    private VenueDtos() {}

    public record CreateVenueRequest(
        @NotBlank @Size(max = 255) String name,
        String address,
        String description) {}

    public record UpdateVenueRequest(
        @Size(max = 255) String name,
        String address,
        String description) {}

    public record VenueResponse(
        UUID id,
        String name,
        String address,
        VenueStatus status,
        Object metadata,
        LocalDateTime createdAt,
        LocalDateTime updatedAt) {}

    public record VenueSummaryListResponse(
        List<VenueResponse> items,
        long totalElements) {}
}
```

---

### MyBatis mapper pattern

```java
@Mapper
public interface VenueMapper {
    Optional<Venue> findById(UUID id);
    List<Venue> findAll(@Param("limit") int limit, @Param("offset") int offset);
    long countAll();
    void insert(Venue venue);
    void update(Venue venue);
    void updateStatus(@Param("id") UUID id, @Param("status") String status);
    boolean existsById(UUID id);
}
```

SQL in XML mapper. `search_path` is set to `t_{tenantKey}` by `MyBatisSchemaInterceptor` before every statement — mappers write plain table names with no schema prefix.

---

### GlobalExceptionHandler pattern

One `@RestControllerAdvice` per service. Every handler uses the same `problem(type, title, status, detail, request)` helper that sets `type = "about:blank"`, adds `correlationId` (from MDC) and `requestId` (generated short UUID). No `https://api.iqkv.site/errors/` URIs — the actual implementation uses `about:blank` throughout.

---

## 17. Next Steps & Design Iterations

### Before Sprint 1 — Resolve These First

- [ ] **Docling in Phase 1?** No — Tika-only for MVP. Add Docling sidecar in Phase 2 for floor plan / table fidelity.
- [ ] **MVP scope cut** — Phase 1 is: venue profiles, asset upload, basic extraction (PDF only), keyword + semantic search, team collaboration. Everything else is Phase 2+.
- [ ] **Implement `VenueMetadataMigrator` v0 + v1 chain** in `iqbene-venue-model` before any service code reads or writes `venues.metadata`. Wire `VenueMetadataTypeHandler` into the `VenueResultMap`. Add a 1-line counter Micrometer call inside `migrateToCurrent()` so version distribution metrics work from day zero. Add JUnit tests for `MetadataMigrationV0ToV1` with 5+ fixture JSON shapes (empty `{}`, `{}` without `_schema_version`, full v1 shape, partial v1 shape, registry-copy shape) to cover legacy-bootstrapping edge cases.
- [ ] **Implement metadata aggregation FIFO routing (A1 for MVP)** in `iqbene-venue-service` `MetadataAggregationConsumer`: configure `@RabbitListener` on `iqbene.metadata.aggregation` with `concurrency=1`, `prefetchCount=1`, `acknowledgeMode=MANUAL`. Wrap the full SELECT → debounce check → merge via VenueMetadataMigrator → UPDATE → venue_metadata_events consume cycle in a single `@Transactional` DB transaction. Ack the RabbitMQ message only after the transaction commits; nack with requeue (up to 3x) on transient exceptions, then DLQ. Add an integration test that publishes three `extraction.completed` events for the same `venue_id` in quick succession, consumes the queue, and asserts that the final `venues.metadata` contains merged fields from all three sources AND that exactly one SQL `UPDATE` was executed (debounce + FIFO coalescing). When queue depth metrics show sustained backlog > 1 s, promote from A1 to A2 (16 hash-slot queues + publisher-side slot computation); consumer handler code is unchanged.
- [ ] **Implement VenueRegistryMatcher + dry-run threshold calibration.** Code: `VenueRegistryMatcher` class in `iqbene-venue-ingestion-worker` (pure PostgreSQL pg_trgm + PostGIS, shared `normalize()` 6-step function (strip articles, lowercase, strip non-alnum, collapse whitespace, trim both sides of every comparison), fetch top-5 trigram candidates via GIN index, PostGIS ST_DWithin 200m radius, combined confidence formula with 0.60·name + 0.40·geo (or pure name when geo missing), MATCH thresholds 0.75 with geo / 0.90 name-only, ambiguity guard delta ≥ 0.08, field copy only for leaf keys that are null on tenant side + REGISTRY source tag in metadata_sources, set metadata_aggregated_at = NOW() after copy. Populate `venues.registry_entry_id` on unambiguous MATCH so cross-source search dedup works. Micrometer counters matched/ambiguous/no_match counter + confidence histogram. Unit tests: 10+ synthetic pairs (exact, fuzzy, geo cross-city mismatch, no geo strict, ambiguous 2-cand delta=0.05 ambiguous guard). Dry-run calibration: 50 real PDFs with ground truth, build confusion matrix, acceptance FP-rate ≤ 1 %, adjust thresholds and document final numbers → release. Admin single-entity CRUD MVP: `POST/PATCH /api/v1/admin/registry/entries` + dedup candidate endpoint returning review list. Scraper dry-run review CSV + `VenueRegistryImportOrchestrator` RabbitMQ events `admin.registry.import.dry-run` → report → `admin.registry.import.apply` verbatim apply (never autonomous decisions on the worker's side).
- [ ] **Implement cross-source search path (?scope + RRF merge + registry detail + from-registry import).** Code in `iqbene-venue-service`: (1) new `scope` enum param wired on `GET /api/v1/venues/`; (2) `VenueSearchOrchestrator` with `CompletableFuture` parallel branch A (tenant mapper `VenueMapper` full 5 modes) + branch B (`RegistryEntryQueryMapper` explicitly qualified `public.venue_registry … LEFT JOIN public.venue_registry_aliases`, 3 MVP modes keyword+structured+geo only, fixed column whitelist). Branch timeout 2 s on registry. (3) App-level merge: dedup by `registry_entry_id` → drop REGISTRY origin if already imported → Reciprocal Rank Fusion equal weights 0.5:0.5 → top-50 each → slice page by page/size → append `origin: "TENANT" | "REGISTRY"` per record. (4) Failure isolation: branch B exception/timeout → return A only + `Warning` header + Micrometer `iqbene_search_failures_total{branch="registry"}`. (5) DTOs: add `registry_entry_id` nullable UUID to `VenueResponse`, add `origin` to `VenueSummaryView`. (6) `GET /api/v1/registry/entries/{id}` (MEMBER authority, fixed column safe projection — never confidence, private notes, or source audit fields). (7) `POST /api/v1/venues/from-registry/{registryEntryId}` (MEMBER): idempotent (if venue with `registry_entry_id` exists → return existing), else INSERT copying registry fields into metadata with `REGISTRY` lowest-priority `metadata_sources` source tag + set `registry_entry_id` + `metadata_aggregated_at = NOW()`. (8) Unit tests: 8 scenarios (BOTH scopes equal weight dedup imported, BOTH scopes dedup not imported, REGISTRY_ONLY scope returns only registry origin, TENANT_ONLY scope never queries RegistryEntryQueryMapper, RRF interleaving order, Warning header on branch B timeout, 3-seconds branch B timeout degrades gracefully, from-registry POST idempotent on second call). (9) Enforce cross-schema rule at code review: no other MyBatis mapper emits `public.` SQL except RegistryEntryQueryMapper / VenueRegistryMatcherMapper / RegistryAdminMapper; cross-schema JOIN/UNION in one statement forbidden (§10).

### Phase 2 Design (post-MVP signal)

- Conflict resolution UX — API shape + state machine for resolving competing extracted values
- Bulk import / CSV ingestion — for concierge onboarding and agency migrations
- Saved searches + alerts — schema and delivery mechanism
- Notification delivery — decide WebSocket vs. polling for extraction status (can reuse IAM's WebSocket infra)
- CAD visual extraction — convert DWG/DXF to image, then GPT-4o vision
- Video walkthroughs — keyframe extraction via ffmpeg, vision-based amenity detection
- **Venue groups** — `venue_groups` and `venue_group_members` tables, tenant-owned library organisation by city / event type / client / season. Tree/folder navigation in the tenant app. No impact on search or extraction.
- **Registry admin API** — internal platform endpoint for bulk-importing seed data, managing registry entries, reviewing match quality. `PLATFORM_ADMIN` authority only.
- **pgvector index strategy: IVFFlat → HNSW evaluation.** Start with IVFFlat (indexed `idx_vectors_embedding`, cosine distance, 1536 dims, MVCC-friendly). When any single tenant's `venue_vectors` count exceeds 1 M rows (Micrometer gauge `iqbene_venue_vectors_rows_total{tenant_id}`), schedule a maintenance window to benchmark HNSW on that tenant's data. HNSW delivers better recall/performance at high volumes but has a higher build cost and write amplification; only promote tenants that actually breach the size threshold, keep IVFFlat as default for small tenants.
- **Registry semantic search + unified search index evaluation.** Two triggers to revisit search: (1) If Prometheus `iqbene_search_latency_seconds{branch="tenant"}` p99 > 500 ms sustained for 10 min on concurrent usage _and_ per-tenant venues count > 10 K (pgvector + tsvector hit plan-scaling wall) _or_ (2) Registry admin wants semantic on registry entries enabled ("venue atmosphere" queries on seed data). Trigger path: introduce `venue_registry.metadata.description_embedding` (VECTOR 1536) in `public` schema, generate embeddings during scraper/admin apply on `registry_entry.created` events, add cosine branch to RegistryEntryQueryMapper (now 4 modes), run 6-month production latency/cost profiling on `iqbene_search_latency_seconds` + `iqbene_ai_cost_usd_total` + `iqbene_search_requests_total{scope=REGISTRY_ONLY}`. If either branch's p99 > 500 ms or `iqbene_venue_import_from_registry_total{status=created}` (ROI signal) is above 10 per user per week, consider a dedicated shared OpenSearch cluster (single index with `tenant_id` + `registry_entry_id` fields) for both sources — before building, run a 2-week shadow benchmark on the same corpus using OpenSearch Neural Search plugin + field-mapped hybrid. For MVP: PostgreSQL stays search engine of record; no new infrastructure.
- **Sweep orphan `venues.registry_entry_id` values after `registry.admin.entry.deleted`.** When Registry Admin deletes or archives an entry, run a per-tenant sweeping job to set `registry_entry_id = NULL` on any tenant-owned venue rows that still referenced it. Prevents the cross-source dedup from silently "showing the already imported but now gone" behavior (the TENANT origin still works; only the dedup filter becomes a no-op). Deferred because registry seed is small in MVP and deletions are a rare admin action. When scheduled: log with counter `iqbene_registry_entry_sweep_total{outcome=updated|no_match|error}` and run in a background worker thread pool, not in the admin API request cycle.

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

**Docs:** [What is iQ BENE?](../README.md) · [Business Proposal](../business/proposal.md) · [Competitive Landscape](../business/comparison.md) · [Architecture](architecture.md)
