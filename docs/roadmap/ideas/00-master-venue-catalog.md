# Master venue catalog

> **Audience:** Product, engineering.
> **Status:** Promoted to [E1 Master Venue Catalog](../epics/E1-master-venue-catalog.md)

---

## Summary

The master venue catalog is a platform-wide reference dataset of venues that powers autocomplete suggestions, deduplication assistance, and metadata gap-filling when a tenant creates a venue. It is not tenant-facing — it is a platform service layer that every tenant venue creation draws from. Building it establishes the per-service domain classes, database schema, admin tooling, and seed data that tenant venue profiles depend on.

## Why it matters

Without a curated master catalog, every tenant retypes the same venue data from scratch and duplicate venue records proliferate across tenants. A platform-maintained reference set lets a planner pick a known venue in one click, inherit verified base fields, and focus on their agency-specific notes instead of re-keying an address. The catalog's domain classes and schema contracts are also the foundation that the document intelligence and search layers build on.

## Where it could land

- Likely epic: TBD (was E1 in the removed v0.1 milestone)
- Likely milestone: TBD
- Related decisions: [D7 — JSONB schema versioning](../decisions/D7-jsonb-schema-versioning.md), [D8 — Master venue seeding infrastructure first](../decisions/D8-master-venue-seeding-infrastructure-first.md)

## In scope

Scope markers: `[docintelligence-catalog-service]` `[docintelligence-catalog-processing-worker]` `[admin-ui]` `[tenant-ui]` `[infra]`

Tasks are ordered by dependency. Each is sized for 2–3 h including tests and CI/CD to UAT.

### Service Domain and Pipeline Types

- [ ] `[docintelligence-catalog-processing-worker]` Create inside the service: enums (`ExtractionStatus`, `ExtractorType`, `AssetType`) and event POJOs (`AssetUploadedEvent`, `ExtractionCompletedEvent`, `ExtractionFailedEvent`). No Spring beans.
- [ ] `[docintelligence-catalog-service]` Create inside the service: `MetadataMigration`, `MetadataMigrator`, `MetadataTypeHandler`, `MetadataSchemaVersion`, `MetadataSource`, `MetadataEventType`, `Venue`, `VenueStatus`, `VenueAsset`, `VenueMetadata`, `VenueCapacity`, `VenueMetadataSchemaVersion` (`CURRENT_SCHEMA_VERSION = 1`), `VenueMetadataMigrator`, `VenueMetadataTypeHandler`, `VenueMetadataMigrationV0ToV1`, plus `MasterVenue`, `MasterVenueAlias`, `MasterVenueExternal` POJOs. Unit tests for the migrator chain and the V0→V1 migration.

### Database Schema

- [ ] `[docintelligence-catalog-service]` Liquibase `system/20260901000000-create-master-venue.xml`: tables `master_venue`, `master_venue_alias`, `master_venue_external` in `public` schema. pg_trgm extension. GIN trigram index on `master_venue_alias.alias`. Full rollback block.
- [ ] `[docintelligence-catalog-service]` Liquibase `tenant/20260901000001-create-venues.xml`: `venues` table with pgvector + PostGIS extensions, IVFFlat/GIN/GIST/btree indexes, `trg_venues_tsvector` trigger. Full rollback block.
- [ ] `[docintelligence-catalog-service]` Liquibase `tenant/20260901000002-create-venue-assets.xml`: `venue_assets` table with btree + IVFFlat indexes. Full rollback block.
- [ ] `[docintelligence-catalog-service]` Liquibase `intelligence/` changesets live in the service, which is the only service that runs Liquibase against the tenant schema: `extraction_jobs`, `item_metadata_events`, `item_vectors`, `ai_cost_tracking`. Included via `tenant/master.xml`. Full rollback blocks.
- [ ] `[docintelligence-catalog-service]` Seed migration `system/20260901000001-seed-master-venues.xml`: 50 curated venues as `<insert>` changesets, `source = platform_seed`, `metadata._schema_version = 1`. Covers top convention centres and hotel venues in 3 target launch cities.

### docintelligence-catalog-service Bootstrap

- [ ] `[docintelligence-catalog-service]` Scaffold Spring Boot service: Maven parent `boot-parent-pom`, deps (`foundation-tenancy`, MyBatis, Spring Security OAuth2 RS, RabbitMQ, MinIO, Micrometer). `SecurityConfig` (RS256, JWKS URI, bare authority strings). `TenantExtractionFilter`. `GlobalExceptionHandler` (RFC 7807). Actuator + Swagger UI. Passes CI.
- [ ] `[docintelligence-catalog-service]` `MyBatisSchemaInterceptor` wired from `foundation-tenancy`. `TenantLiquibaseRunner` applying `system/master.xml` then `tenant/master.xml` on startup. Integration test: startup against Testcontainers Postgres verifies all tables created in correct schemas.

### Master Venue Admin API

- [ ] `[docintelligence-catalog-service]` `MasterCatalogAdminMapper` (MyBatis, `public.` qualified): `insert`, `findById`, `findAll` (paginated), `update`, `delete`, `findAliasesById`, `insertAlias`, `updateAlias`, `deleteAlias`. Mapper XML + `MasterVenueDtos` records. Unit tests.
- [ ] `[docintelligence-catalog-service]` `MasterVenueNormalizer` utility: 6-step normalize function (trim → strip articles → lowercase → strip non-alphanumeric → collapse whitespace → trim). `MasterVenueDeduplicator`: pg_trgm similarity + `ST_Distance` query, returns top-5 candidates with `name_sim`, `geo_distance_m`, `action_recommendation`. Unit tests for normalizer. Integration test for dedup query.
- [ ] `[docintelligence-catalog-service]` `AdminMasterVenueRestResource` (`/api/v1/admin/master-venues`, `PLATFORM_ADMIN`): `GET /`, `POST /`, `GET /{id}`, `PUT /{id}`, `PATCH /{id}`, `DELETE /{id}`. No TenantContext — operates on `public` schema directly. `ProblemDetail` error responses. Controller + service + mapper integration tests.
- [ ] `[docintelligence-catalog-service]` `AdminMasterVenueRestResource` continuation: `POST /{id}/merge` (`ForceMergeRequest`, three strategies), `POST /dedup-check` (`DeduplicationCheckRequest` → `DeduplicationCheckResponse`). Integration tests covering merge strategies and dedup recommendations.
- [ ] `[docintelligence-catalog-service]` `AdminMasterVenueAliasRestResource` (`/api/v1/admin/master-venues/{id}/aliases`, `PLATFORM_ADMIN`): `GET /`, `POST /`, `GET /{aliasId}`, `PUT /{aliasId}`, `DELETE /{aliasId}`. Integration tests. 404 when parent `master_venue` not found.
- [ ] `[docintelligence-catalog-service]` `POST /bulk-import` (`PLATFORM_ADMIN`, status 202): accepts `BulkImportRequest` (S3 key + force_mode), enqueues `admin.master-catalog.import.apply` RabbitMQ event, returns `BulkImportJobResponse`. Integration test verifies event published.

### Master Venue Public Read

- [ ] `[docintelligence-catalog-service]` `MasterVenueQueryMapper`: `findById` returning safe projection. `MasterVenueRestResource` (`/api/v1/master-venues`, `MEMBER`): `GET /{id}` → `MasterVenuePublicResponse` (no admin-only fields). 404 if `status = ARCHIVED`. Integration test.

### Admin UI

- [ ] `[admin-ui]` Master venue list page (`/admin/master-venues`): paginated table — name, city, confidence, source, last verified. Search by name. `PLATFORM_ADMIN` guard.
- [ ] `[admin-ui]` Master venue detail/edit page (`/admin/master-venues/:id`): display all fields including `admin_notes`, `quality_score`. Inline `PUT`/`PATCH`. Delete with confirmation. Alias list with add/remove/edit.
- [ ] `[admin-ui]` Create master venue form + dedup-check flow: on name/address input call `POST /dedup-check`, display candidate list with similarity scores and action recommendation. Admin confirms Insert or Merge. E2E test.

## Out of scope

- Document upload and extraction pipeline (`docintelligence-catalog-processing-worker`) — deferred.
- Semantic search (pgvector embeddings) — deferred.
- Full-text keyword search — deferred.
- MC_INHERIT merge on extraction completion — deferred.
- `docintelligence-master-venue-worker` bulk import processor — deferred.
- Tenant-facing venue creation UI (covered by the tenant venue profiles idea).

## Open questions

- [ ] Which 3 target launch cities does the 50-venue seed set cover?
- [ ] Does the bulk-import processor (`docintelligence-master-venue-worker`) ship in the same increment as the admin API, or as a follow-up?

## Status

**Status:** Promoted to [E1 Master Venue Catalog](../epics/E1-master-venue-catalog.md)

---

**Docs:** [Ideas index](README.md) · [Vision](../vision.md)
