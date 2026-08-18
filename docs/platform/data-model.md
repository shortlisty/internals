# VenueMi — Data Model

> **Audience:** Engineers, architects.
> **Purpose:** Domain model, canonical metadata field set, JSONB schema versioning strategy, and full database schema with index rationale.

---

## Related Documents

- [architecture-overview.md](architecture-overview.md) — platform context and foundation reuse
- [services.md](services.md) — which service owns which table; shared library layout
- [aggregation.md](aggregation.md) — how conflicting field values are resolved at write time
- [master-catalog.md](master-catalog.md) — `public.master_venue` tables and MC_INHERIT provenance
- [etl-pipeline.md](etl-pipeline.md) — how extraction jobs produce `venue_metadata_events`

---

## 2. Domain Model

### Bounded Contexts

#### `venue/` — Core Profile

**Aggregate root: `Venue`**

| Field                          | Type             | Notes                                                                                            |
| ------------------------------ | ---------------- | ------------------------------------------------------------------------------------------------ |
| `id`                           | UUID             | PK                                                                                               |
| `name`                         | varchar(255)     | Extracted from files / AI-drafted. Source of truth for dedup and search.                         |
| `display_name`                 | varchar(255)     | Agent-assigned label for their collection (nullable). Shown in UI list/cards in place of `name`. |
| `address`                      | text             |                                                                                                  |
| `city`                         | varchar(100)     | Denormalised from address. Btree-indexed for fast city filters.                                  |
| `country_code`                 | char(2)          | ISO 3166-1 alpha-2. Denormalised. Nullable (not always resolvable from raw address).             |
| `location`                     | geography(point) | PostGIS, lat/lng                                                                                 |
| `website_url`                  | varchar(500)     | Primary venue website. Used in Sales Room card and enrichment. Nullable.                         |
| `description`                  | text             | Human-written or AI-drafted                                                                      |
| `primary_photo_asset_id`       | UUID             | App-level FK → `venue_assets`. Nullable. Fast cover photo resolution for list views.             |
| `status`                       | enum             | `DRAFT`, `ACTIVE`, `ARCHIVED`                                                                    |
| `profile_stage`                | enum             | `SEEDED`, `ENRICHED`, `CURATED`, `READY` — progressive enrichment state (see §2b)               |
| `source`                       | enum             | `MANUAL`, `FILE_IMPORT`, `MASTER_CATALOG`, `BULK_CSV` — how the record was created              |
| `metadata`                     | jsonb            | Consolidated extracted + manual fields                                                           |
| `metadata_sources`             | jsonb            | Provenance per field (see §2a)                                                                   |
| `metadata_aggregated_at`       | timestamp        | When consolidation last ran                                                                      |
| `description_embedding`        | vector(1536)     | pgvector, for semantic search                                                                    |
| `description_text`             | tsvector         | Auto-updated via trigger, full-text search                                                       |
| `master_venue_id`              | UUID             | App-level FK → `public.master_venue`. Nullable. Set on MC_INHERIT match or explicit import.      |
| `last_used_in_sales_room_at`   | timestamp        | Updated when venue is included in any Sales Room. Informs ranking and retention analytics.       |
| `created_by`                   | UUID             | IAM user id                                                                                      |
| `created_at`                   | timestamp        |                                                                                                  |
| `updated_at`                   | timestamp        |                                                                                                  |

**`profile_stage` transitions** (computed automatically inside `MetadataAggregationConsumer`, never set manually):

| Stage      | Condition to reach it                                                                                      |
| ---------- | ---------------------------------------------------------------------------------------------------------- |
| `SEEDED`   | Default on creation. Name present + at least one asset uploaded.                                          |
| `ENRICHED` | `capacity.max_total` known + `venue_type` non-empty + `catering.policy` known. Typically via extraction + MC_INHERIT. |
| `CURATED`  | At least one `venue_annotations` row exists for this venue (agent added a note, tag, or rating).           |
| `READY`    | All `ENRICHED` conditions met + `primary_photo_asset_id` set + `website_url` present.                     |

Transitions are monotone by default (SEEDED → ENRICHED → CURATED → READY) but can regress if a key field is removed.

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

Defined in `mi-data-intelligence`. Owned by `mi-venue-processing-worker`. See [services.md](services.md) §4c.

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

How this table feeds aggregation: see [aggregation.md](aggregation.md).

---

#### `venue_annotations/` — Agent Personal Context

**Tenant-owned. Lives in `t_{tenantKey}` schema.**

Personal context layer that agents attach to venues — notes, tags, ratings, and colored labels. This data is **never mixed with extracted metadata or provenance**. It exists solely in the curation layer and is the primary signal that moves a venue from `ENRICHED` to `CURATED` stage.

Annotations are owned per-member (not per-tenant) and can be scoped as private (visible only to the creator) or shared across the team. When building a Sales Room, the agent explicitly controls which annotations — if any — are exposed to the client.

**`VenueAnnotation`**

| Field              | Type          | Notes                                                                                         |
| ------------------ | ------------- | --------------------------------------------------------------------------------------------- |
| `id`               | UUID          | PK                                                                                            |
| `venue_id`         | UUID          | FK → venues                                                                                   |
| `annotation_type`  | enum          | `NOTE`, `TAG`, `RATING`, `BOOKMARK`, `INTERNAL_FLAG`                                          |
| `text_value`       | text          | For `NOTE` and `TAG`. `TAG` values are short labels (max 50 chars). Nullable for other types. |
| `color_hex`        | varchar(7)    | For `TAG` and `INTERNAL_FLAG`. Hex color for colored badge in UI (e.g. `#F59E0B`). Nullable. |
| `numeric_value`    | numeric(3,1)  | For `RATING` (0.0–5.0 scale). Nullable for other types.                                      |
| `is_private`       | boolean       | `true` = visible to creator only; `false` = shared with the whole team. Default: `true`.      |
| `created_by`       | UUID          | IAM user id of the member who created this annotation.                                        |
| `created_at`       | timestamp     |                                                                                               |
| `updated_at`       | timestamp     |                                                                                               |

**Design rules:**

- A venue can have multiple annotations of any type from multiple members.
- `TAG` values are free-form strings, not a controlled vocabulary. Normalisation (lowercase, trim) is applied at write time so that `"Rooftop"` and `"rooftop"` are treated as the same tag for filtering.
- `BOOKMARK` has no `text_value` or `numeric_value` — it is a presence flag only (the annotation row itself is the bookmark).
- `RATING` is per-member. If aggregated team rating is needed in search, it is computed as an average at query time — not stored separately.
- Annotations are **soft-delete only** (set `updated_at`, mark deleted via `annotation_type = NULL` or a future `deleted_at` column). Hard deletes are avoided to preserve audit trail.
- The `profile_stage` aggregation step in `MetadataAggregationConsumer` queries `COUNT(*) FROM venue_annotations WHERE venue_id = ?` to decide the `CURATED` transition — no other service logic depends on annotation content.

**Operations:** create, update (text/color/privacy only), soft-delete.

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

> Phase 2 feature. The `venue_groups` table is not included in the Phase 1 (MVP) Liquibase migrations.

---

### Canonical Field Set

The `venues.metadata` column stores the **consolidated view** of all extracted and manually entered data. The `venues.metadata_sources` column stores **provenance** per field.

```
_schema_version (int)        MANDATORY — schema version for migration, starts at 1. Never absent.
                                 Incremented on every canonical schema change.
                                 Read by VenueMetadataMigrator (see §2a).

capacity
  └─ max_total (int)
  └─ configurations: banquet, theater, classroom, cocktail, conference (int each)

venue_type (string[])         e.g. ["conference_center", "hotel_ballroom"]

location_notes (string)       e.g. "3 blocks from Grand Central"

catering
  └─ policy (enum)            in_house_exclusive | in_house_preferred | outside_allowed | no_catering
  └─ kosher_available (bool)
  └─ halal_available (bool)
  └─ vegan_available (bool)

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
  └─ setup_hours_before (int)   hours available for setup before event start
  └─ teardown_hours_after (int) hours available for teardown after event end

outdoor_space
  └─ available (bool)
  └─ covered (bool)
  └─ sqm (int)                  outdoor area in square metres

exclusivity
  └─ exclusive_use (bool)       venue can be rented as a whole, no other events simultaneously
  └─ shared_space (bool)        venue may run concurrent events in separate areas

restrictions (string[])       e.g. ["no_open_flame", "no_confetti"]

amenities (string[])          e.g. ["WiFi", "AV_equipment", "parking"]

contacts (object[])
  └─ name, role, email, phone

pricing
  └─ minimum_spend (int)
  └─ currency (string)
  └─ rental_fee_indicative (int)

social
  └─ instagram_handle (string)  e.g. "the_plaza_hotel"
  └─ google_place_id (string)   Google Places API ID — key for cheap structured enrichment
                                 (address, photos, rating, hours) without AI cost.
                                 Populated via MC_INHERIT or manual entry.

ratings
  └─ google_rating (decimal)    e.g. 4.7
  └─ google_review_count (int)
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

Every `venues.metadata` JSONB document must carry an integer key `_schema_version` at the top level. The value is the version of the **canonical field set** that the document was produced against. Documents without `_schema_version` are treated as version `0` — the "legacy, pre-versioning" shape.

| Rule                | Value                                                                                                        |
| ------------------- | ------------------------------------------------------------------------------------------------------------ |
| Initial version     | `1` (matches the first canonical field set shipped in MVP)                                                   |
| Absent key fallback | `0` (triggers full migration chain from v0)                                                                  |
| Bump condition      | Any backwards-incompatible change to canonical fields, or any addition of a required nested field            |
| Bump ownership      | `mi-venue-model` library — only the shared model may define `CURRENT_SCHEMA_VERSION`                         |
| Write enforcement   | Every write path runs the migrator and sets `_schema_version = CURRENT_SCHEMA_VERSION` before persisting     |
| Read enforcement    | `VenueMetadataMigrator.migrateToCurrent(JsonNode)` is called on **every read** via MyBatis ResultMap handler |

### Migration Pipeline — `VenueMetadataMigrator`

Lives in `mi-venue-model` (shared library, zero Spring deps — pure Java). Both services use the **same** migrator instance.

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

### Migration Classes

Each `N → N+1` step is a single-responsibility class implementing `MetadataMigration` interface (defined in `mi-data-intelligence`). Migrations are registered in an ordered list inside `VenueMetadataMigrator` — the order is the source of truth.

```java
// mi-venue-model: no Spring, no framework deps
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

**Rules for migration authors:**

- Migrations are **pure functions**: same input → same output, no side effects, no I/O. Testable with plain unit tests — no Spring context required.
- Never drop keys they do not recognise. Unknown keys are preserved verbatim.
- Numeric type widening (int → long, int → double) is done by value coercion, not by dropping the value.
- Enum string rename: produce a mapping table. Missing values default to `null` — callers decide how to render an unknown enum value in UI/API.
- Migration list is **append-only**. Once a migration class is merged, it is never deleted or reordered.

### Write Path — Stamping the Version

All code paths that produce or mutate `venues.metadata` call `migrator.ensureCurrent(node)` **before** the `UPDATE` / `INSERT`. This:

1. Applies any pending migrations from the document's current version.
2. Sets `_schema_version = CURRENT_SCHEMA_VERSION`.
3. Returns the node ready to be persisted.

Write paths affected: `MetadataAggregationConsumer`, `PATCH /metadata/{field}` handler, bulk import job, MC_INHERIT merge, `CreateVenueRequest` default metadata initializer.

### Read Path — Safe Deserialization

Reading `venues.metadata` from the database never returns a raw JSONB value. The MyBatis result map applies the migrator inside the column handler:

```xml
<resultMap id="VenueResultMap" type="com.iqkv.mi.model.venue.Venue">
  <id     property="id"        column="id"/>
  <result property="name"      column="name"/>
  <result property="metadata"  column="metadata"
          typeHandler="com.iqkv.mi.model.metadata.VenueMetadataTypeHandler"/>
</resultMap>
```

`VenueMetadataTypeHandler` runs `VenueMetadataMigrator.migrateToCurrent()` between the raw `PGobject` JSONB and `VenueMetadata` POJO construction. If a document is several versions behind, all intermediate migrations run in a single pass.

### Backfill: Online-Only, No Offline Migration Job

- **First read** of a stale document → migrator upgrades in-memory, returns correct shape. The DB copy remains stale.
- **Next write** → `ensureCurrent()` stamps the DB copy to the latest version permanently.

No scheduled backfill job, no downtime, no `ALTER TABLE` on JSONB. If all rows must be force-converged (e.g. before a heavy schema jump), a one-shot admin endpoint iterates `SELECT id FROM venues` and issues a no-op `UPDATE venues SET metadata = metadata` to trigger write-path stamping.

### Version Distribution Observability

The migrator records the pre-migration version of every document it sees via Micrometer. See [observability.md](observability.md) for the `venuemi_metadata_schema_version_seen_total` metric.

### Schema Version in Master Catalog Metadata

`public.master_venue.metadata` follows the same `_schema_version` contract. Master catalog import jobs run `VenueMetadataMigrator.ensureCurrent()` before `INSERT/UPDATE public.master_venue`. See [master-catalog.md](master-catalog.md).

---

## 10. Database Schema

Migrations live in `mi-venue-model` under `src/main/resources/db/changelog/tenant/`. Both services include the library on their classpath; `mi-venue-service` runs the migrations on startup (same pattern as IAM).

### Naming and Format Conventions

- Format: **XML only** — no SQL scripts, no YAML
- File naming: `YYYYMMDDhhmmss-description.xml`
- Changeset ID: matches filename without `.xml` extension
- Author: `iqkv`
- Every changeset must include a `<rollback>` block
- Never modify an existing changeset — add a new one

### Migration Files

```
db/changelog/system/
├── master.xml
└── 20260901000000-create-master-venue.xml   ← public schema, platform-owned

db/changelog/tenant/
├── master.xml
├── 20260901000001-create-venues.xml
├── 20260901000002-create-venue-assets.xml
├── 20260901000003-create-extraction-jobs.xml
├── 20260901000004-create-metadata-events.xml
├── 20260901000005-create-venue-vectors.xml
├── 20260901000006-create-ai-cost-tracking.xml
└── 20260901000007-create-venue-annotations.xml
```

The `TenantLiquibaseRunner` from `foundation-tenancy` applies `system/master.xml` first (to the `public` schema), then `tenant/master.xml` to each `t_{tenantKey}` schema. The `master_venue`, `master_venue_alias`, and `master_venue_external` tables are created in `public` once, not per-tenant.

### Changeset Structure Example — `venues` Table

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.33.xsd">

  <changeSet id="20260901000001-create-venues" author="iqkv">

    <sql>CREATE EXTENSION IF NOT EXISTS vector;</sql>
    <sql>CREATE EXTENSION IF NOT EXISTS postgis;</sql>

    <createTable tableName="venues">
      <column name="id" type="UUID">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <!-- identity -->
      <column name="name" type="VARCHAR(255)">
        <constraints nullable="false"/>
      </column>
      <column name="display_name" type="VARCHAR(255)"/>
      <!-- location -->
      <column name="address" type="TEXT"/>
      <column name="city" type="VARCHAR(100)"/>
      <column name="country_code" type="CHAR(2)"/>
      <column name="location" type="GEOGRAPHY(POINT, 4326)"/>
      <column name="website_url" type="VARCHAR(500)"/>
      <!-- content -->
      <column name="description" type="TEXT"/>
      <column name="primary_photo_asset_id" type="UUID"/>
      <column name="description_embedding" type="VECTOR(1536)"/>
      <column name="description_text" type="TSVECTOR"/>
      <!-- lifecycle -->
      <column name="status" type="VARCHAR(20)" defaultValue="DRAFT">
        <constraints nullable="false"/>
      </column>
      <column name="profile_stage" type="VARCHAR(20)" defaultValue="SEEDED">
        <constraints nullable="false"/>
      </column>
      <column name="source" type="VARCHAR(20)" defaultValue="MANUAL">
        <constraints nullable="false"/>
      </column>
      <!-- metadata -->
      <column name="metadata" type="JSONB" defaultValue="{&quot;_schema_version&quot;:1}">
        <constraints nullable="false"/>
      </column>
      <column name="metadata_sources" type="JSONB" defaultValue="{}">
        <constraints nullable="false"/>
      </column>
      <column name="metadata_aggregated_at" type="TIMESTAMP"/>
      <!-- relations -->
      <column name="master_venue_id" type="UUID">
        <remarks>App-level FK to public.master_venue.id. No DB-level FK constraint
        (cross-schema FKs not supported in PostgreSQL across tenant schemas + public).
        Populated on explicit promote (POST /venues/from-master-catalog/{id}) or on
        extraction-time MC_INHERIT unambiguous MATCH.</remarks>
      </column>
      <!-- analytics -->
      <column name="last_used_in_sales_room_at" type="TIMESTAMP"/>
      <!-- audit -->
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
    <createIndex tableName="venues" indexName="idx_venues_city">
      <column name="city"/>
    </createIndex>
    <createIndex tableName="venues" indexName="idx_venues_country_code">
      <column name="country_code"/>
    </createIndex>
    <createIndex tableName="venues" indexName="idx_venues_profile_stage">
      <column name="profile_stage"/>
    </createIndex>
    <createIndex tableName="venues" indexName="idx_venues_master_venue_id">
      <column name="master_venue_id"/>
    </createIndex>

    <sql>
      CREATE TRIGGER trg_venues_tsvector BEFORE INSERT OR UPDATE ON venues
        FOR EACH ROW EXECUTE FUNCTION
        tsvector_update_trigger(description_text, 'pg_catalog.english', name, display_name, description, address);
    </sql>

    <rollback>
      <sql>DROP TRIGGER IF EXISTS trg_venues_tsvector ON venues;</sql>
      <dropIndex tableName="venues" indexName="idx_venues_master_venue_id"/>
      <dropIndex tableName="venues" indexName="idx_venues_profile_stage"/>
      <dropIndex tableName="venues" indexName="idx_venues_country_code"/>
      <dropIndex tableName="venues" indexName="idx_venues_city"/>
      <dropIndex tableName="venues" indexName="idx_venues_location"/>
      <dropIndex tableName="venues" indexName="idx_venues_metadata"/>
      <dropIndex tableName="venues" indexName="idx_venues_fts"/>
      <dropIndex tableName="venues" indexName="idx_venues_embedding"/>
      <dropTable tableName="venues"/>
    </rollback>

  </changeSet>
</databaseChangeLog>
```

### Changeset Structure Example — `venue_annotations` Table

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.33.xsd">

  <changeSet id="20260901000007-create-venue-annotations" author="iqkv">

    <createTable tableName="venue_annotations">
      <column name="id" type="UUID">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <column name="venue_id" type="UUID">
        <constraints nullable="false" foreignKeyName="fk_annotations_venue" references="venues(id)"/>
      </column>
      <column name="annotation_type" type="VARCHAR(20)">
        <constraints nullable="false"/>
      </column>
      <column name="text_value" type="TEXT"/>
      <column name="color_hex" type="VARCHAR(7)"/>
      <column name="numeric_value" type="NUMERIC(3,1)"/>
      <column name="is_private" type="BOOLEAN" defaultValueBoolean="true">
        <constraints nullable="false"/>
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

    <createIndex tableName="venue_annotations" indexName="idx_annotations_venue">
      <column name="venue_id"/>
    </createIndex>
    <createIndex tableName="venue_annotations" indexName="idx_annotations_created_by">
      <column name="venue_id"/>
      <column name="created_by"/>
    </createIndex>
    <createIndex tableName="venue_annotations" indexName="idx_annotations_type">
      <column name="annotation_type"/>
    </createIndex>
    <!-- Case-insensitive index for tag lookup and dedup -->
    <sql>
      CREATE INDEX idx_annotations_tag_value
        ON venue_annotations (LOWER(text_value))
        WHERE annotation_type = 'TAG';
    </sql>

    <rollback>
      <sql>DROP INDEX IF EXISTS idx_annotations_tag_value;</sql>
      <dropIndex tableName="venue_annotations" indexName="idx_annotations_type"/>
      <dropIndex tableName="venue_annotations" indexName="idx_annotations_created_by"/>
      <dropIndex tableName="venue_annotations" indexName="idx_annotations_venue"/>
      <dropTable tableName="venue_annotations"/>
    </rollback>

  </changeSet>
</databaseChangeLog>
```

### Schema Overview

**Public schema (platform-owned, not tenant-scoped):**

| Table                   | Key columns                                                                                                                                                     | Notes                                                                           |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `master_venue`          | `id` UUID PK, `name` VARCHAR(255), `address` TEXT, `city` VARCHAR(100), `location` GEOGRAPHY, `metadata` JSONB, `confidence` NUMERIC(3,2), `source` VARCHAR(50) | Platform seed data. Read-only to tenants. `metadata._schema_version` mandatory. |
| `master_venue_alias`    | `id` UUID PK, `master_venue_id` UUID FK, `alias` VARCHAR(255)                                                                                                   | Alternative names for master catalog deduplication                              |
| `master_venue_external` | `id` UUID PK, `master_venue_id` UUID FK, `provider` VARCHAR, `provider_external_id` VARCHAR, `raw_payload` JSONB, `confidence` NUMERIC, `crawled_at` TIMESTAMP  | 1 MasterVenue ↔ N external provider records from Tagvenue, Cvent, etc.          |

**Tenant schema `t_{tenantKey}` (tenant-owned):**

| Table                   | Key columns                                                                                                                                                                                                                          | Owner                        |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------- |
| `venues`                | `id` UUID PK, `name` VARCHAR(255), `display_name` VARCHAR(255), `city` VARCHAR(100), `country_code` CHAR(2), `website_url` VARCHAR(500), `primary_photo_asset_id` UUID, `status` VARCHAR(20), `profile_stage` VARCHAR(20), `source` VARCHAR(20), `metadata` JSONB, `description_embedding` VECTOR(1536), `location` GEOGRAPHY, `master_venue_id` UUID nullable, `last_used_in_sales_room_at` TIMESTAMP | `mi-venue-service`           |
| `venue_annotations`     | `id` UUID PK, `venue_id` UUID FK, `annotation_type` VARCHAR(20), `text_value` TEXT, `color_hex` VARCHAR(7), `numeric_value` NUMERIC(3,1), `is_private` BOOLEAN, `created_by` UUID                                                   | `mi-venue-service`           |
| `venue_assets`          | `id` UUID PK, `venue_id` UUID FK, `asset_type` VARCHAR(50), `extraction_status` VARCHAR(20), `extracted_text_embedding` VECTOR(1536)                                                                                                 | `mi-venue-service`           |
| `extraction_jobs`       | `id` UUID PK, `asset_id` UUID FK, `status` VARCHAR(20), `extractor_type` VARCHAR(50), `extracted_data` JSONB, `confidence_scores` JSONB                                                                                             | `mi-venue-processing-worker` |
| `venue_metadata_events` | `id` UUID PK, `venue_id` UUID FK, `event_type` VARCHAR(50), `event_data` JSONB — append-only                                                                                                                                        | `mi-venue-service`           |
| `item_vectors`          | `id` UUID PK, `content` TEXT, `metadata` JSONB, `embedding` VECTOR(1536) — Spring AI PgVectorStore table. Defined in `mi-data-intelligence` changelog.                                                                              | `mi-venue-processing-worker` |
| `ai_cost_tracking`      | `id` UUID PK, `provider` VARCHAR(50), `model` VARCHAR(100), `tokens_used` INTEGER, `cost_usd` NUMERIC(10,6)                                                                                                                         | `mi-venue-processing-worker` |

### Index Strategy

| Table                | Index name                        | Type      | Column(s)                          | Purpose                                           |
| -------------------- | --------------------------------- | --------- | ---------------------------------- | ------------------------------------------------- |
| `venues`             | `idx_venues_embedding`            | IVFFlat   | `description_embedding`            | Semantic search                                   |
| `venues`             | `idx_venues_fts`                  | GIN       | `description_text`                 | Full-text search (name, display_name, description, address) |
| `venues`             | `idx_venues_metadata`             | GIN       | `metadata jsonb_path_ops`          | JSONB attribute filters                           |
| `venues`             | `idx_venues_location`             | GIST      | `location`                         | Geo-spatial queries                               |
| `venues`             | `idx_venues_city`                 | btree     | `city`                             | City filter in search and list views              |
| `venues`             | `idx_venues_country_code`         | btree     | `country_code`                     | Country filter                                    |
| `venues`             | `idx_venues_profile_stage`        | btree     | `profile_stage`                    | Filter by enrichment stage; onboarding dashboards |
| `venues`             | `idx_venues_master_venue_id`      | btree     | `master_venue_id`                  | Cross-source search dedup                         |
| `venue_annotations`  | `idx_annotations_venue`           | btree     | `venue_id`                         | FK lookup, all annotations for a venue            |
| `venue_annotations`  | `idx_annotations_created_by`      | btree     | `venue_id, created_by`             | Member's own annotations per venue                |
| `venue_annotations`  | `idx_annotations_type`            | btree     | `annotation_type`                  | Filter by type (TAG, NOTE, etc.)                  |
| `venue_annotations`  | `idx_annotations_tag_value`       | btree     | `LOWER(text_value)` WHERE TAG      | Case-insensitive tag lookup and dedup             |
| `venue_assets`       | `idx_assets_venue`                | btree     | `venue_id`                         | FK lookup                                         |
| `venue_assets`       | `idx_assets_type`                 | btree     | `asset_type`                       | Filter by type                                    |
| `venue_assets`       | `idx_assets_embedding`            | IVFFlat   | `extracted_text_embedding`         | Chunk-level vector search                         |
| `extraction_jobs`    | `idx_jobs_asset`                  | btree     | `asset_id`                         | FK lookup                                         |
| `extraction_jobs`    | `idx_jobs_status`                 | btree     | `status`                           | Queue polling                                     |
| `metadata_events`    | `idx_metadata_events_venue`       | btree     | `venue_id, occurred_at DESC`       | Timeline queries                                  |
| `metadata_events`    | `idx_metadata_events_source`      | btree     | `source_id, source_type`           | Provenance lookup                                 |
| `item_vectors`       | `idx_vectors_embedding`           | IVFFlat   | `embedding`                        | Vector similarity search                          |
| `item_vectors`       | `idx_vectors_asset`               | btree     | `(metadata->>'asset_id')`          | Fast sweep on `asset.deleted` event               |
| `ai_cost_tracking`   | `idx_ai_cost_month`               | btree     | `DATE_TRUNC('month', created_at)`  | Monthly cost rollup                               |

### Cross-Schema Access Rules

PostgreSQL schema-per-tenant means `public.master_venue` lives in a different schema than `t_{tenantKey}.venues`. The `MyBatisSchemaInterceptor` sets `SET search_path TO t_{tenantKey}, public` before every statement.

**Rules (enforced at code review + MyBatis mapper audit):**

1. **Only three mapper interfaces may reference `public.` schema-qualified names explicitly:**
   - `MasterVenueQueryMapper` (search branch B): read-only SELECT with a fixed column whitelist
   - `MasterVenueMatcherMapper` (extraction-time MC_INHERIT merge): same qualified reads
   - `MasterCatalogAdminMapper` (`PLATFORM_ADMIN` only): reads + writes `public.master_venue` / aliases / external

2. **Single-statement cross-schema `JOIN` / `UNION ALL` is forbidden.** Cross-source merging always happens in the `VenueSearchOrchestrator` app-layer (see [search.md](search.md)), never in SQL.

3. **No cross-schema DDL-level FK constraints.** The `venues.master_venue_id → master_venue.id` reference is application-enforced only.

4. **Role-level hardening (pre-production):** The `mi-venue-service` connection pool holds `SELECT` on `public.master_venue` / aliases / external. `INSERT / UPDATE / DELETE` on `public` tables is revoked; only the `master_catalog_admin` connection holds write grants.

---

**Docs:** [Architecture Index](README.md) · [Architecture Overview](architecture-overview.md) · [Services](services.md) · [Aggregation](aggregation.md) · [Master Catalog](master-catalog.md) · [ETL Pipeline](etl-pipeline.md) · [Search](search.md)
