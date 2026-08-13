# VenueMi — API Surface

> **Audience:** Engineers.
> **Purpose:** Full REST API for `mi-venue-service` — all endpoints, request/response DTOs, error responses, and platform conventions.

---

## Related Documents

- [architecture-overview.md](architecture-overview.md) — REST controller patterns, DTO pattern, authority strings
- [data-model.md](data-model.md) — domain types and enums referenced in DTOs
- [search.md](search.md) — search modes, `scope` parameter, hybrid RRF details
- [services.md](services.md) — presigned URL upload flow, S3 key structure
- [events.md](events.md) — events published when assets are confirmed / deleted

---

## 7. API Surface (`mi-venue-service`)

All endpoints follow platform conventions based on the actual implementation in `foundation-cms-service` and `foundation-iam-service`:

- Base path: `/api/v1/venues` — always versioned
- All endpoints require `Bearer` JWT — no public routes
- `POST` (create) → `201 Created` with resource body
- `GET` (read) → `200 OK`
- `PUT` (full replace) → `200 OK`
- `PATCH` (partial update) → `200 OK`
- `DELETE` → `204 No Content`
- Paginated responses: custom wrapper records (e.g. `VenueSummaryListResponse(items, totalElements)`) — not Spring's `Page<T>`
- Error responses: RFC 7807 `ProblemDetail`, `type = about:blank`, includes `correlationId` and `requestId` extension properties
- Authority strings are bare — `MEMBER`, `ADMIN`, `TENANT_OWNER` (never `ROLE_` prefixed)
- Tenant context set automatically by `TenantExtractionFilter` from JWT `tenant_id` claim — no tenant path variable on regular tenant-scoped endpoints

---

## Venues

Base: `/api/v1/venues`

| Method   | Path                                   | Authority               | Status | Request / Response                             | Notes                                                                                                                                                                                                                      |
| -------- | -------------------------------------- | ----------------------- | ------ | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET`    | `/`                                    | `MEMBER`                | 200    | Query params → `VenueSummaryListResponse`      | Hybrid search + filter when `search` param present; `scope=TENANT_ONLY\|MASTER_CATALOG_ONLY\|BOTH` (default `TENANT_ONLY`)                                                                                                 |
| `POST`   | `/`                                    | `MEMBER`                | 201    | `CreateVenueRequest` → `VenueResponse`         | Enforces `max_venues` plan limit before insert                                                                                                                                                                             |
| `GET`    | `/{id}`                                | `MEMBER`                | 200    | → `VenueResponse` (with consolidated metadata) | 404 if not found or belongs to different tenant                                                                                                                                                                            |
| `PATCH`  | `/{id}`                                | `MEMBER`                | 200    | `UpdateVenueRequest` → `VenueResponse`         | Partial update — ignores null fields                                                                                                                                                                                       |
| `DELETE` | `/{id}`                                | `ADMIN`, `TENANT_OWNER` | 204    | → empty body                                   | Soft delete: sets `status = ARCHIVED`. Hard delete: separate admin endpoint (Phase 3)                                                                                                                                      |
| `POST`   | `/from-master-catalog/{masterVenueId}` | `MEMBER`                | 201    | → `VenueResponse`                              | Explicit promote from Master Catalog: clones master venue fields into new tenant `venues` row with `metadata_sources[*].source = "MC_INHERIT"` (priority 7). Idempotent: re-promote returns the existing tenant venue row. |

### Search

Search is served via the list endpoint (`GET /`) with query parameters — no separate `POST /search` needed at this scale. If query complexity grows beyond what URL params can express cleanly (Phase 2), introduce `POST /search` with a `VenueSearchRequest` body. `GET` with a body is non-standard and must not be used.

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

### DTOs

```java
CreateVenueRequest  — name (required), address, description, tags
UpdateVenueRequest  — all fields optional; null fields ignored (PATCH semantics)
VenueResponse       — id, name, address, location, status, metadata (consolidated),
                       metadata_aggregated_at, asset_count, master_venue_id (nullable),
                       created_by, created_at, updated_at
VenueSummaryView    — id, name, address, location, summary metadata slice,
                       origin ("TENANT" only — MC origin not surfaced as separate row to UI;
                       MC values merge invisibly into tenant rows via field-level MC_INHERIT)
VenueSummaryListResponse — items: List<VenueSummaryView>, totalElements: long
```

---

## Master Venues (MEMBER-read, ADMIN-write via admin API)

Base: `/api/v1/master-venues`

Master venue GET endpoints are always MEMBER-authenticated (read-only public projection). Write / edit / delete endpoints live under the Platform Admin API scope (`PLATFORM_ADMIN` authority only).

| Method | Path    | Authority | Status | Request / Response                                   | Notes                                                                                                                                                                                                  |
| ------ | ------- | --------- | ------ | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `GET`  | `/{id}` | `MEMBER`  | 200    | → `MasterVenuePublicResponse` (safe projection only) | Read-only public-facing projection of `master_venue` + aliases. Never returns admin-only fields (confidence scores, private notes, source audit). 404 if entry does not exist or is `status=ARCHIVED`. |

```java
MasterVenuePublicResponse — id, name, aliases[], address, location,
                             metadata safe projection,
                             created_at, last_synced_at
```

---

## Assets

Base: `/api/v1/venues/{venueId}/assets`

Upload uses the two-phase presigned URL pattern (same as IAM avatar upload — no multipart to the service). Full S3 key layout in [services.md § 4b](services.md).

| Method   | Path            | Authority               | Status | Request / Response                                 | Notes                                                        |
| -------- | --------------- | ----------------------- | ------ | -------------------------------------------------- | ------------------------------------------------------------ |
| `GET`    | `/`             | `MEMBER`                | 200    | → `List<AssetResponse>`                            | All assets for venue; not paginated (reasonable upper bound) |
| `POST`   | `/initiate`     | `MEMBER`                | 201    | `InitiateUploadRequest` → `InitiateUploadResponse` | Returns `asset_id` + presigned S3 PUT URL (15 min TTL)       |
| `POST`   | `/{id}/confirm` | `MEMBER`                | 200    | → `AssetResponse`                                  | Marks asset ready, publishes `asset.uploaded` event          |
| `DELETE` | `/{id}`         | `ADMIN`, `TENANT_OWNER` | 204    | → empty body                                       | Deletes asset record + S3 object + associated vectors        |

Enforces `max_assets_per_venue` plan limit at `POST /initiate`.

### DTOs

```java
InitiateUploadRequest  — file_name (required), content_type (required), size_bytes (required),
                          asset_type (required: PDF_DECK | FLOOR_PLAN | PHOTO | VIDEO |
                                      CAD_FILE | SPEC_SHEET | MISC)
InitiateUploadResponse — asset_id (UUID), upload_url (presigned S3 PUT, 15 min), expires_at
AssetResponse          — id, venue_id, asset_type, file_name, content_type, size_bytes,
                          extraction_status, uploaded_by, uploaded_at
```

Plan gate: `cad_support` feature checked at `POST /initiate` when `asset_type = CAD_FILE`. Returns `403 Forbidden` with `ProblemDetail` (feature not on current plan) if not on qualifying plan.

---

## Metadata

Base: `/api/v1/venues/{venueId}/metadata`

| Method  | Path               | Authority | Status | Request / Response                                  | Notes                                                                      |
| ------- | ------------------ | --------- | ------ | --------------------------------------------------- | -------------------------------------------------------------------------- |
| `GET`   | `/`                | `MEMBER`  | 200    | → `MetadataResponse` (consolidated + provenance)    | Includes confidence scores and source attribution                          |
| `PATCH` | `/{field}`         | `MEMBER`  | 200    | `MetadataOverrideRequest` → `MetadataFieldResponse` | Manual override — sets source = `MANUAL_OVERRIDE`, triggers re-aggregation |
| `GET`   | `/{field}/history` | `MEMBER`  | 200    | → `List<MetadataEventResponse>`                     | Full extraction + override history for a single field                      |

`PATCH /{field}` is correct for manual override (partial update of a single field). `POST` would imply creating a new resource.

### DTOs

```java
MetadataResponse        — fields (map of field → value + confidence + source),
                           aggregated_at, conflict_count (int)
MetadataOverrideRequest — value (required), reason (optional free text)
MetadataFieldResponse   — field, value, confidence, source_type, source_id, updated_at
MetadataEventResponse   — event_type, source_type, source_id, value, confidence,
                           occurred_at, created_by
```

---

## Extraction Jobs

Base: `/api/v1/venues/{venueId}/extractions`

Read-only for API clients. Jobs are created internally when `asset.uploaded` is consumed.

| Method | Path       | Authority | Status | Response                      | Notes                                             |
| ------ | ---------- | --------- | ------ | ----------------------------- | ------------------------------------------------- |
| `GET`  | `/`        | `MEMBER`  | 200    | `List<ExtractionJobResponse>` | All jobs for venue, ordered by `started_at DESC`  |
| `GET`  | `/{jobId}` | `MEMBER`  | 200    | `ExtractionJobResponse`       | Single job status + extracted data (if completed) |

### DTOs

```java
ExtractionJobResponse — id, asset_id, status, extractor_type,
                         confidence_scores (map), started_at,
                         completed_at, error_message
```

---

## Error Responses

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

**Docs:** [Architecture Index](README.md) · [Architecture Overview](architecture-overview.md) · [Search](search.md) · [Services](services.md) · [Events](events.md) · [Data Model](data-model.md)
---

## Platform Admin API - Master Catalog Management

Base: `/api/v1/admin/master-venues`

**Authority Required:** `PLATFORM_ADMIN` only. Full unrestricted CRUD access to master venue catalog.

### Master Venues

| Method   | Path           | Authority        | Status | Request / Response                                         | Notes                                                                              |
| -------- | -------------- | ---------------- | ------ | ---------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `GET`    | `/`            | `PLATFORM_ADMIN` | 200    | Query params → `MasterVenueListResponse`                   | Paginated list with all fields including admin-only data                           |
| `POST`   | `/`            | `PLATFORM_ADMIN` | 201    | `CreateMasterVenueRequest` → `MasterVenueResponse`         | **No validation constraints** — any data accepted                                  |
| `GET`    | `/{id}`        | `PLATFORM_ADMIN` | 200    | → `MasterVenueResponse` (full admin view)                  | Complete venue data including confidence scores, source audit, private notes       |
| `PUT`    | `/{id}`        | `PLATFORM_ADMIN` | 200    | `UpdateMasterVenueRequest` → `MasterVenueResponse`         | **Full replace** — can override any field including metadata and confidence scores |
| `PATCH`  | `/{id}`        | `PLATFORM_ADMIN` | 200    | `PatchMasterVenueRequest` → `MasterVenueResponse`          | **Partial update** — can modify any subset of fields                               |
| `DELETE` | `/{id}`        | `PLATFORM_ADMIN` | 204    | → empty body                                               | **Permanent deletion** — removes venue and all aliases/external records            |
| `POST`   | `/{id}/merge`  | `PLATFORM_ADMIN` | 200    | `MergeVenueRequest` → `MasterVenueResponse`                | Force merge with target venue, bypassing similarity thresholds                     |
| `POST`   | `/bulk-import` | `PLATFORM_ADMIN` | 202    | `BulkImportRequest` → `BulkImportJobResponse`              | Async bulk import with job tracking                                                |
| `POST`   | `/dedup-check` | `PLATFORM_ADMIN` | 200    | `DeduplicationCheckRequest` → `DeduplicationCheckResponse` | Check for duplicates without saving — validation endpoint                          |

### Master Venue Aliases

| Method   | Path                      | Authority        | Status | Request / Response                     | Notes                 |
| -------- | ------------------------- | ---------------- | ------ | -------------------------------------- | --------------------- |
| `GET`    | `/{id}/aliases`           | `PLATFORM_ADMIN` | 200    | → `List<AliasResponse>`                | List all aliases      |
| `POST`   | `/{id}/aliases`           | `PLATFORM_ADMIN` | 201    | `CreateAliasRequest` → `AliasResponse` | Add alternative name  |
| `GET`    | `/{id}/aliases/{aliasId}` | `PLATFORM_ADMIN` | 200    | → `AliasResponse`                      | Get specific alias    |
| `PUT`    | `/{id}/aliases/{aliasId}` | `PLATFORM_ADMIN` | 200    | `UpdateAliasRequest` → `AliasResponse` | Update specific alias |
| `DELETE` | `/{id}/aliases/{aliasId}` | `PLATFORM_ADMIN` | 204    | → empty body                           | Remove alias          |

### Admin Override Capabilities

- **Force operations** that bypass all deduplication warnings
- **Manual confidence scores** — set any value 0.0-1.0
- **Metadata schema management** — force-migrate venues to current version
- **Bulk operations** — import, export, cleanup with admin override
- **Emergency fixes** — modify any field for operational issues
- **Bypass validation** — create venues that fail normal business rules

### Admin DTOs

```java
MasterVenueResponse          — id, name, aliases[], address, location, metadata (full),
                               confidence, source, created_at, updated_at,
                               admin_notes, quality_score, last_verified_at

CreateMasterVenueRequest     — name (required), address, location, metadata,
                               confidence (optional), source (optional), admin_notes

UpdateMasterVenueRequest     — all fields including confidence, source, metadata_schema_version

PatchMasterVenueRequest      — any subset of fields, null = no change

ForceMergeRequest           — target_venue_id (UUID, required), merge_strategy (enum, required),
                               "KEEP_TARGET" = target metadata wins on conflict,
                               "KEEP_SOURCE" = source metadata wins on conflict,
                               "MERGE_METADATA" = manual merge with field-level picks

BulkImportRequest           — import_source (enum: "CSV" | "JSONL"), s3_key (string, required),
                               force_mode (boolean, default: false) bypasses dedup checks

DeduplicationCheckRequest   — name (string, required), address (string), location (geography)
DeduplicationCheckResponse  — candidates (list of MasterVenueSummary with similarity scores),
                               action_recommendation (enum: "INSERT" | "MERGE" | "REVIEW"),
                               confidence_scores (map of candidate_id → score)
```

**No Restrictions:** Platform admins can create, modify, or delete any master venue record without validation constraints. Any possible change is accepted.
