# Shortlisty — Event Contracts & Plan Entitlements

> **Audience:** Engineers.
> **Purpose:** All RabbitMQ event contracts, queue topology, consumer configuration, and plan entitlement feature codes with enforcement points.

---

## Related Documents

- [services.md](services.md) — which service publishes and consumes each event
- [aggregation.md](aggregation.md) — `MetadataAggregationConsumer` listener container configuration (FIFO, MANUAL ack)
- [etl-pipeline.md](etl-pipeline.md) — `AssetExtractionConsumer` triggered by `asset.uploaded`
- [master-catalog.md](master-catalog.md) — `admin.master-catalog.import.*` events for scraper dry-run / apply
- [api.md](api.md) — `asset.uploaded` published at `POST /api/v1/venues/{venueId}/assets/{id}/confirm`
- [observability.md](observability.md) — audit trail via passive `foundation-audit-service` binding

---

## 8. Event Contracts (RabbitMQ)

Exchange: `iqkv.events` (Topic) — same exchange used by all foundation services.

### Published by `mi-venue-service`

| Routing key                | Payload fields                                                                                   | Description                             |
| -------------------------- | ------------------------------------------------------------------------------------------------ | --------------------------------------- |
| `venue.created`            | venue_id, tenant_id, created_by                                                                  | New venue profile created               |
| `venue.updated`            | venue_id, tenant_id, changed_fields                                                              | Venue fields updated                    |
| `venue.stage_changed`      | venue_id, tenant_id, previous_stage, new_stage                                                   | profile_stage recalculated (planned)    |
| `asset.uploaded`           | asset_id, venue_id, tenant_id, asset_type, photo_category, s3_key, content_type                  | Asset confirmed, ready for extraction   |
| `asset.deleted`            | asset_id, venue_id, tenant_id                                                                    | Asset removed                           |
| `proposal.shared`          | proposal_id, tenant_id, owner_id, share_token                                                    | Proposal link sent to client            |
| `proposal.approved`        | proposal_id, tenant_id, owner_id, approved_by_client_name, snapshot_id                           | Client clicked Approve, snapshot locked |
| `proposal.client_activity` | proposal_id, tenant_id, event_type (CLIENT_OPENED \| CLIENT_PREFERENCE_SET \| CLIENT_NOTE_ADDED) | Client interacted with the board        |

### Published by `mi-venue-processing-worker`

| Routing key            | Payload fields                                | Description           |
| ---------------------- | --------------------------------------------- | --------------------- |
| `extraction.started`   | job_id, asset_id, venue_id, tenant_id         | Processing began      |
| `extraction.completed` | job_id, asset_id, venue_id, tenant_id         | Extraction succeeded  |
| `extraction.failed`    | job_id, asset_id, venue_id, tenant_id, reason | All retries exhausted |

> For the scalable topology (Variant A2, see [aggregation.md](aggregation.md)), the publisher appends a hash-slot suffix to the `extraction.completed` routing key: `extraction.completed.{slot}` where `slot = Math.abs(venueId.hashCode() % SLOT_COUNT)`. Same `venue_id` always produces the same slot.

### Consumed by `mi-venue-processing-worker`

| Routing key      | Queue                                      | Action                                |
| ---------------- | ------------------------------------------ | ------------------------------------- |
| `asset.uploaded` | `shortlisty.extraction.priority` (Enterprise) | Trigger ETL pipeline immediately      |
| `asset.uploaded` | `shortlisty.extraction.standard` (Free/Pro)   | Trigger ETL pipeline (standard queue) |

### Consumed by `mi-venue-service`

| Routing key            | Queue (MVP A1)                                | Queue (Scalable A2)                                                                                                | Action                                  |
| ---------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | --------------------------------------- |
| `extraction.completed` | `shortlisty.metadata.aggregation` (single queue) | `shortlisty.metadata.aggregation.0` … `shortlisty.metadata.aggregation.15` (16 slots, slot-bound via routing key suffix) | Run metadata aggregation for venue      |
| `extraction.failed`    | `shortlisty.extraction.dlq`                      | `shortlisty.extraction.dlq`                                                                                           | Mark asset `extraction_status = FAILED` |

### Consumed by `foundation-audit-service` (passive, no changes)

| Routing key                                        | Notes                                                      |
| -------------------------------------------------- | ---------------------------------------------------------- |
| `venue.#`, `asset.#`, `extraction.#`, `proposal.#` | Automatically captured by audit service's wildcard binding |

---

## Metadata Aggregation Queue — Consumer Configuration

The `MetadataAggregationConsumer` listener container in `mi-venue-service` must be configured to serialise processing per queue so that per-venue events never execute concurrently:

| Configuration    | Value (A1)                                          | Value (A2)  | Notes                                                                                           |
| ---------------- | --------------------------------------------------- | ----------- | ----------------------------------------------------------------------------------------------- |
| `concurrency`    | `1`                                                 | `16`        | A2: one thread per slot queue. Total threads = `SLOT_COUNT`.                                    |
| `prefetchCount`  | `1`                                                 | `1`         | Critical. A worker must never prefetch a batch; one in-flight message per consumer thread.      |
| Acknowledge mode | `MANUAL`                                            | `MANUAL`    | Ack sent only after the wrapping DB transaction commits (see [aggregation.md](aggregation.md)). |
| Error handler    | Reject + requeue on transient (≤3x) → DLQ on fatal. | Same as A1. | Aggregation is idempotent via `metadata_aggregated_at` debounce → safe requeues.                |

The consumer listens either on one queue (A1) or on all 16 slot queues (A2) via a queue array or wildcard in `@RabbitListener`. The consumer handler code is **identical** across variants — only the listener container configuration differs.

---

## 9. Plan Entitlement Mapping

Feature codes used in `foundation-billing-service` plan config:

| Feature code             | Free | Pro | Enterprise | Enforcement point                          |
| ------------------------ | ---- | --- | ---------- | ------------------------------------------ |
| `max_venues`             | 10   | 500 | unlimited  | `mi-venue-service`: before create          |
| `max_assets_per_venue`   | 20   | 100 | unlimited  | `mi-venue-service`: before upload          |
| `max_proposals`          | 3    | 50  | unlimited  | `mi-venue-service`: before proposal create |
| `basic_extraction`       | ✅   | ✅  | ✅         | AI processing: PDF text only               |
| `advanced_extraction`    | ⛔   | ✅  | ✅         | AI processing: all asset types             |
| `video_support`          | ⛔   | ✅  | ✅         | `mi-venue-service`: reject VIDEO upload    |
| `cad_support`            | ⛔   | ✅  | ✅         | `mi-venue-service`: reject DWG/DXF upload  |
| `semantic_search`        | ⛔   | ✅  | ✅         | `mi-venue-service`: search endpoint        |
| `priority_ai_processing` | ⛔   | ⛔  | ✅         | RabbitMQ: route to priority queue          |
| `api_access`             | ⛔   | ✅  | ✅         | gateway: API key route                     |
| `white_label`            | ⛔   | ⛔  | ✅         | `foundation-ui-app`: branding config       |

Enforcement via `PlanFeatureGuard` (same pattern as IAM service's existing implementation).

Error response for plan-gated features: `403 Forbidden` with `featureCode` extension property in `ProblemDetail`. See [api.md § Error Responses](api.md).

---

**Docs:** [Architecture Index](README.md) · [Services](services.md) · [Aggregation](aggregation.md) · [ETL Pipeline](etl-pipeline.md) · [API](api.md) · [Master Catalog](master-catalog.md) · [Data Model](data-model.md)
