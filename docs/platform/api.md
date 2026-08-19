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
- [ui-venue-management.md](ui-venue-management.md) — UI integration, asset gallery, annotations
- [ui-deal-workspace.md](ui-deal-workspace.md) — proposal and client board UI integration

---

## Platform Conventions

All endpoints follow platform conventions based on the actual implementation in `foundation-cms-service` and `foundation-iam-service`:

- Base path: `/api/v1` — always versioned
- All endpoints require `Bearer` JWT unless marked **public**
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

| Method   | Path                                   | Authority               | Status | Notes                                                                                               |
| -------- | -------------------------------------- | ----------------------- | ------ | --------------------------------------------------------------------------------------------------- |
| `GET`    | `/`                                    | `MEMBER`                | 200    | Hybrid search + filter. `scope=TENANT_ONLY\|MASTER_CATALOG_ONLY\|BOTH` (default `TENANT_ONLY`)      |
| `POST`   | `/`                                    | `MEMBER`                | 201    | Enforces `max_venues` plan limit before insert                                                      |
| `GET`    | `/{id}`                                | `MEMBER`                | 200    | Full detail with consolidated metadata. 404 if not found or different tenant.                       |
| `PATCH`  | `/{id}`                                | `MEMBER`                | 200    | Partial update — null fields ignored                                                                |
| `DELETE` | `/{id}`                                | `ADMIN`, `TENANT_OWNER` | 204    | Soft delete: sets `status = ARCHIVED`. Hard delete: admin endpoint (Phase 3)                        |
| `POST`   | `/from-master-catalog/{masterVenueId}` | `MEMBER`                | 201    | Explicit promote from Master Catalog. Idempotent: re-promote returns the existing tenant venue row. |

### Search parameters (`GET /`)

| Parameter       | Type              | Description                                                           |
| --------------- | ----------------- | --------------------------------------------------------------------- |
| `search`        | string            | Natural language or keyword query — triggers hybrid mode              |
| `scope`         | enum              | `TENANT_ONLY`, `MASTER_CATALOG_ONLY`, `BOTH` (default: `TENANT_ONLY`) |
| `status`        | enum              | `DRAFT`, `ACTIVE`, `ARCHIVED`                                         |
| `profile_stage` | enum              | `SEEDED`, `ENRICHED`, `CURATED`, `READY`                              |
| `city`          | string            | Exact city filter (btree index)                                       |
| `country_code`  | string (ISO 3166) | Two-letter country filter                                             |
| `capacity`      | integer           | Minimum total capacity                                                |
| `lat`, `lng`    | decimal           | Centre point for geo-spatial search                                   |
| `radius_km`     | decimal           | Radius from lat/lng (requires both to be set)                         |
| `amenities`     | string[]          | Required amenity codes (comma-separated)                              |
| `catering`      | enum              | Catering policy filter                                                |
| `page`          | integer (0-based) | Default: 0                                                            |
| `size`          | integer           | Default: 20, max: 100                                                 |
| `sort`          | string            | e.g. `name,asc` or `relevance` (default when `search` present)        |

### DTOs

```java
CreateVenueRequest
  name (required), address, city, country_code, website_url,
  display_name, description, source

UpdateVenueRequest
  All fields optional — null fields ignored (PATCH semantics).
  display_name, name, address, city, country_code, website_url,
  description, status, primary_photo_asset_id

VenueResponse
  id, name, display_name, address, city, country_code, location,
  website_url, description, primary_photo_url,
  status, profile_stage, source,
  metadata (consolidated), metadata_sources, metadata_aggregated_at,
  master_venue_id (nullable), last_used_in_sales_room_at,
  asset_count, created_by, created_at, updated_at

VenueSummaryView
  id, name, display_name, city, country_code,
  status, profile_stage, source,
  primary_photo_url,                  // resolved server-side from primary_photo_asset_id
  capacity_max_total (int nullable),  // resolved server-side from metadata
  catering_policy (string nullable),  // resolved server-side from metadata
  master_venue_id (nullable),
  last_used_in_sales_room_at,
  created_at, updated_at

VenueSummaryListResponse
  items: List<VenueSummaryView>, totalElements: long
```

---

## Master Venues (MEMBER read)

Base: `/api/v1/master-venues`

Read-only public projection for tenant members. Write/edit/delete lives under Platform Admin API.

| Method | Path    | Authority | Status | Notes                                                                                      |
| ------ | ------- | --------- | ------ | ------------------------------------------------------------------------------------------ |
| `GET`  | `/{id}` | `MEMBER`  | 200    | Safe projection only — no confidence scores or admin fields. 404 if not found or ARCHIVED. |

```java
MasterVenuePublicResponse
  id, name, aliases[], address, city, country_code, location,
  metadata (safe projection), created_at, last_synced_at
```

---

## Assets

Base: `/api/v1/venues/{venueId}/assets`

Upload uses the two-phase presigned URL pattern (no multipart to the service). Full S3 key layout in [services.md § 4b](services.md).

| Method   | Path            | Authority               | Status | Notes                                                                     |
| -------- | --------------- | ----------------------- | ------ | ------------------------------------------------------------------------- |
| `GET`    | `/`             | `MEMBER`                | 200    | Query: `type=`, `category=`. Returns `VenueAssetSummary[]` — no tableData |
| `GET`    | `/{id}`         | `MEMBER`                | 200    | Full asset including `tableData` (if present)                             |
| `POST`   | `/initiate`     | `MEMBER`                | 201    | Returns `asset_id` + presigned S3 PUT URL (15 min TTL)                    |
| `POST`   | `/{id}/confirm` | `MEMBER`                | 200    | Marks asset ready, publishes `asset.uploaded`                             |
| `PATCH`  | `/{id}`         | `MEMBER`                | 200    | Update `label`, `photo_category`, `display_order`                         |
| `DELETE` | `/{id}`         | `ADMIN`, `TENANT_OWNER` | 204    | Deletes asset record + S3 object + associated vectors                     |

Enforces `max_assets_per_venue` plan limit at `POST /initiate`.
`video_support` feature checked when `asset_type = VIDEO`.
`cad_support` feature checked when `asset_type = CAD_FILE`.

### Asset query parameters (`GET /`)

| Parameter  | Type | Description                                                     |
| ---------- | ---- | --------------------------------------------------------------- |
| `type`     | enum | Filter by `AssetType` — `PHOTO`, `FLOOR_PLAN`, `PDF_DECK`, etc. |
| `category` | enum | Filter by `PhotoCategory` — only meaningful when `type=PHOTO`   |

### DTOs

```java
InitiateUploadRequest
  file_name (required), content_type (required), size_bytes (required),
  asset_type (required): PHOTO | FLOOR_PLAN | PDF_DECK | SPEC_SHEET |
                          MENU | PRICE_LIST | DATA_TABLE | VIDEO | CAD_FILE | MISC
  photo_category (required when asset_type = PHOTO):
    EXTERIOR | INTERIOR | SETUP | DETAIL | CATERING | OUTDOOR | TEAM | OTHER
  label (optional): human-readable label

InitiateUploadResponse
  asset_id (UUID), upload_url (presigned S3 PUT, 15 min), expires_at

VenueAssetSummary             // list / gallery view — fast, no tableData
  id, venue_id, asset_type, photo_category, display_order, label,
  file_name, size_bytes, cdn_url, thumbnail_cdn_url,
  extraction_status, uploaded_at

VenueAsset                    // full detail — includes tableData
  ...all VenueAssetSummary fields...
  content_type, table_data (AssetTableData | null), uploaded_by

AssetTableData
  headers: string[], rows: string[][], source_sheet: string | null,
  row_count: int, parsed_at: timestamp

UpdateAssetRequest
  label (optional), photo_category (optional), display_order (optional)
```

---

## Annotations

Base: `/api/v1/venues/{venueId}/annotations`

Personal context added by agent members — notes, tags, ratings, bookmarks.

| Method   | Path    | Authority | Status | Notes                                     |
| -------- | ------- | --------- | ------ | ----------------------------------------- |
| `GET`    | `/`     | `MEMBER`  | 200    | Returns annotations visible to caller     |
| `POST`   | `/`     | `MEMBER`  | 201    | Create annotation                         |
| `PATCH`  | `/{id}` | `MEMBER`  | 200    | Update text/color/is_private — owner only |
| `DELETE` | `/{id}` | `MEMBER`  | 204    | Soft delete — owner only                  |

### DTOs

```java
CreateAnnotationRequest
  annotation_type (required): NOTE | TAG | RATING | BOOKMARK | INTERNAL_FLAG
  text_value (required for NOTE, TAG)
  color_hex (optional, for TAG and INTERNAL_FLAG)
  numeric_value (required for RATING, 0.0–5.0)
  is_private (optional, default: true)

UpdateAnnotationRequest
  text_value, color_hex, is_private — all optional, null ignored

AnnotationResponse
  id, venue_id, annotation_type, text_value, color_hex, numeric_value,
  is_private, created_by, created_at, updated_at
```

---

## Metadata

Base: `/api/v1/venues/{venueId}/metadata`

| Method  | Path               | Authority | Status | Notes                                                            |
| ------- | ------------------ | --------- | ------ | ---------------------------------------------------------------- |
| `GET`   | `/`                | `MEMBER`  | 200    | Consolidated metadata + per-field provenance                     |
| `PATCH` | `/{field}`         | `MEMBER`  | 200    | Manual override — sets source = `MANUAL_OVERRIDE`, re-aggregates |
| `GET`   | `/{field}/history` | `MEMBER`  | 200    | Full extraction + override history for a single field            |

### DTOs

```java
MetadataResponse
  fields: map<string, MetadataFieldResponse>, aggregated_at, conflict_count

MetadataOverrideRequest
  value (required), reason (optional)

MetadataFieldResponse
  field, value, confidence, source_type, source_id, updated_at

MetadataEventResponse
  event_type, source_type, source_id, value, confidence,
  occurred_at, created_by
```

---

## Extraction Jobs

Base: `/api/v1/venues/{venueId}/extractions`

Read-only. Jobs are created internally when `asset.uploaded` is consumed.

| Method | Path       | Authority | Status | Notes                                            |
| ------ | ---------- | --------- | ------ | ------------------------------------------------ |
| `GET`  | `/`        | `MEMBER`  | 200    | All jobs for venue, ordered by `started_at DESC` |
| `GET`  | `/{jobId}` | `MEMBER`  | 200    | Single job status + extracted data if completed  |

```java
ExtractionJobResponse
  id, asset_id, status, extractor_type,
  confidence_scores (map<string, double>),
  started_at, completed_at, error_message
```

---

## Proposals

Base: `/api/v1/proposals`

Deal Workspace — planner assembles venue shortlists, sends to client for review and approval.

| Method  | Path              | Authority               | Status | Notes                                                             |
| ------- | ----------------- | ----------------------- | ------ | ----------------------------------------------------------------- |
| `GET`   | `/`               | `MEMBER`                | 200    | Paginated list. Params: `status=`, `owner_id=`, `page=`, `size=`  |
| `POST`  | `/`               | `MEMBER`                | 201    | Create proposal. Enforces `max_proposals` plan limit.             |
| `GET`   | `/{id}`           | `MEMBER`                | 200    | Full proposal + venue list                                        |
| `PATCH` | `/{id}`           | `MEMBER`                | 200    | Update title, clientName, clientEmail, eventDate, brandingEnabled |
| `POST`  | `/{id}/share`     | `MEMBER`                | 200    | Generate / refresh `shareToken` → returns `shareUrl`              |
| `PATCH` | `/{id}/owner`     | `TENANT_OWNER`, `ADMIN` | 200    | Reassign owner. `ownerId` is mandatory — cannot be set to null.   |
| `POST`  | `/{id}/archive`   | `MEMBER`                | 200    | Set `status = ARCHIVED`                                           |
| `GET`   | `/{id}/events`    | `MEMBER`                | 200    | Append-only event log. Params: `page=`, `size=`                   |
| `GET`   | `/{id}/snapshot`  | `MEMBER`                | 200    | Locked snapshot — only available when `status = APPROVED`         |
| `POST`  | `/{id}/ai-assist` | `MEMBER`                | 200    | AI-suggested response for a client question                       |

### Proposal Venues

Base: `/api/v1/proposals/{proposalId}/venues`

| Method   | Path      | Authority | Status | Notes                                            |
| -------- | --------- | --------- | ------ | ------------------------------------------------ |
| `POST`   | `/`       | `MEMBER`  | 201    | Add venue to proposal                            |
| `PATCH`  | `/{pvId}` | `MEMBER`  | 200    | Update `order`, `exposed_fields`, `planner_note` |
| `DELETE` | `/{pvId}` | `MEMBER`  | 204    | Remove venue from proposal                       |

### Proposal Labels

Base: `/api/v1/proposals/{proposalId}/labels`

| Method   | Path                                 | Authority | Status | Notes                                |
| -------- | ------------------------------------ | --------- | ------ | ------------------------------------ |
| `GET`    | `/`                                  | `MEMBER`  | 200    | All labels for proposal              |
| `POST`   | `/`                                  | `MEMBER`  | 201    | Create colored label                 |
| `PATCH`  | `/{labelId}`                         | `MEMBER`  | 200    | Update label text or color           |
| `DELETE` | `/{labelId}`                         | `MEMBER`  | 204    | Remove label (unassigns from venues) |
| `POST`   | `/../venues/{pvId}/labels/{labelId}` | `MEMBER`  | 201    | Assign label to proposal venue       |
| `DELETE` | `/../venues/{pvId}/labels/{labelId}` | `MEMBER`  | 204    | Remove label from proposal venue     |

### DTOs

```java
CreateProposalRequest
  title (required), client_name (required), client_email,
  event_date, branding_enabled (default: false)

UpdateProposalRequest
  title, client_name, client_email, event_date, branding_enabled — all optional

ProposalResponse
  id, tenant_id, title, client_name, client_email, status,
  branding_enabled, share_url,
  owner_id, owner_name, owner_email,
  event_date, approved_at, approved_by_client_name,
  snapshot_id (nullable), created_by, created_at, updated_at

ProposalSummary
  id, title, client_name, status, share_url, event_date,
  owner_id, owner_name, approved_at, venue_count,
  last_client_activity_at, created_at, updated_at

ProposalVenueRequest
  venue_id (required), order (required), exposed_fields (string[]), planner_note

ProposalVenueResponse
  id, proposal_id, venue_id, order, exposed_fields,
  planner_note, client_preference, client_note,
  venue_snapshot (ProposalVenueSnapshot), created_at, updated_at

ProposalLabelRequest
  key (required), display_text (required), color_hex (required)

ProposalLabelResponse
  id, proposal_id, key, display_text, color_hex

AIAssistRequest
  proposal_venue_id (required), question (required)

AIAssistResponse
  suggested_text, source_citations (list<{asset_id, asset_name, excerpt}>), confidence

ProposalSnapshotResponse
  id, proposal_id, venues (list), client_name, approved_at,
  owner_name, owner_email, tenant_name, event_log (list<ProposalEventResponse>)

ProposalEventResponse
  id, event_type, actor_type, actor_name, payload, occurred_at
```

---

## Client Board (Public — no auth)

Base: `/api/v1/share/{token}`

No JWT required. Server resolves tenant from the share token. Exposes only `exposed_fields` per venue — server-enforced, not UI-only.

| Method  | Path                        | Status | Notes                                                          |
| ------- | --------------------------- | ------ | -------------------------------------------------------------- |
| `GET`   | `/`                         | 200    | Full board with venues (exposed fields only)                   |
| `PATCH` | `/venues/{pvId}/preference` | 200    | Client sets `SHORTLISTED                                       | CONSIDERING | DECLINED` |
| `POST`  | `/venues/{pvId}/notes`      | 201    | Client adds a question or comment                              |
| `POST`  | `/approve`                  | 200    | Client approves — triggers `PROPOSAL_APPROVED` + snapshot lock |

```java
ClientBoardResponse
  proposal_id, title, owner_name, owner_email,
  branding_enabled, agency_logo_url (nullable),
  venues (list<ClientVenueView>)

ClientVenueView
  proposal_venue_id, order,
  name, city, primary_photo_url,
  exposed_metadata (only fields in exposed_fields),
  labels (list<ProposalLabelResponse>),
  client_preference, client_note

ClientPreferenceRequest
  preference (required): SHORTLISTED | CONSIDERING | DECLINED

ClientNoteRequest
  text (required)

ClientApproveRequest
  client_name (required)   // captured for snapshot
```

---

## Platform Admin API

Base: `/api/v1/admin/master-venues`

**Authority required:** `PLATFORM_ADMIN` only.

| Method   | Path           | Status | Notes                                                          |
| -------- | -------------- | ------ | -------------------------------------------------------------- |
| `GET`    | `/`            | 200    | Paginated list — all fields including admin-only data          |
| `POST`   | `/`            | 201    | Create master venue                                            |
| `GET`    | `/{id}`        | 200    | Full admin view including confidence scores, source audit      |
| `PUT`    | `/{id}`        | 200    | Full replace — can override metadata and confidence scores     |
| `PATCH`  | `/{id}`        | 200    | Partial update — any subset of fields                          |
| `DELETE` | `/{id}`        | 204    | Permanent deletion — removes venue, aliases, external records  |
| `POST`   | `/{id}/merge`  | 200    | Force merge with target venue, bypassing similarity thresholds |
| `POST`   | `/bulk-import` | 202    | Async bulk import — returns job tracking response              |
| `POST`   | `/dedup-check` | 200    | Check for duplicates without saving                            |

### Aliases

Base: `/api/v1/admin/master-venues/{id}/aliases`

| Method   | Path         | Status | Notes        |
| -------- | ------------ | ------ | ------------ |
| `GET`    | `/`          | 200    | All aliases  |
| `POST`   | `/`          | 201    | Add alias    |
| `PUT`    | `/{aliasId}` | 200    | Update alias |
| `DELETE` | `/{aliasId}` | 204    | Remove alias |

### Admin DTOs

```java
MasterVenueResponse
  id, name, aliases[], address, city, country_code, location,
  metadata (full), confidence, source,
  admin_notes, quality_score, last_verified_at,
  created_at, updated_at

CreateMasterVenueRequest
  name (required), address, city, country_code, location, metadata,
  confidence (optional, default: 1.0), source, admin_notes

ForceMergeRequest
  target_venue_id (required),
  merge_strategy: KEEP_TARGET | KEEP_SOURCE | MERGE_METADATA

BulkImportRequest
  import_source: CSV | JSONL
  s3_key (required)
  force_mode (boolean, default: false) — bypasses dedup checks

DeduplicationCheckRequest
  name (required), address, location

DeduplicationCheckResponse
  candidates (list<MasterVenueSummary with similarity_score>),
  action_recommendation: INSERT | MERGE | REVIEW
```

---

## Error Responses

All errors use Spring's `ProblemDetail` (RFC 7807). `type = about:blank`. Every response includes `correlationId` and `requestId` extension properties.

| Scenario                       | Status | Title                   |
| ------------------------------ | ------ | ----------------------- |
| Venue not found                | 404    | `Not Found`             |
| Asset not found                | 404    | `Not Found`             |
| Proposal not found             | 404    | `Not Found`             |
| Share token invalid or expired | 404    | `Not Found`             |
| Venue quota reached            | 402    | `Plan upgrade required` |
| Asset quota reached            | 402    | `Plan upgrade required` |
| Proposal quota reached         | 402    | `Plan upgrade required` |
| Feature not on plan            | 403    | `Plan upgrade required` |
| Validation failure             | 400    | `Validation Failed`     |
| Tenant context missing         | 400    | `Bad Request`           |
| Access denied                  | 403    | `Forbidden`             |
| Token revoked / expired        | 401    | `Unauthorized`          |

Plan-gated features (`PlanFeatureNotAvailableException`): status `403` + `featureCode` extension property (e.g. `cad_support`, `video_support`, `max_proposals`).
Quota limits: status `402`.

---

**Docs:** [Architecture Index](README.md) · [Architecture Overview](architecture-overview.md) · [Search](search.md) · [Services](services.md) · [Events](events.md) · [Data Model](data-model.md) · [UI: Venue Management](ui-venue-management.md) · [UI: Deal Workspace](ui-deal-workspace.md)
