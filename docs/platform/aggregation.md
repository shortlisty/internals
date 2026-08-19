# VenueMi — Metadata Aggregation

> **Audience:** Engineers.
> **Purpose:** How conflicting field values from multiple extraction sources are resolved into the single consolidated `venues.metadata` JSONB column, and how concurrent aggregation is serialised without distributed locks.

---

## Related Documents

- [data-model.md](data-model.md) — `venue_metadata_events` table, canonical field set, provenance structure
- [services.md](services.md) — which service owns the aggregation consumer
- [events.md](events.md) — RabbitMQ event contracts and queue configuration
- [etl-pipeline.md](etl-pipeline.md) — how extraction jobs produce the events that trigger aggregation
- [master-catalog.md](master-catalog.md) — MC_INHERIT priority source (priority 7)

---

## Overview

Multiple assets per venue produce multiple extraction events, potentially with conflicting values. The aggregation service resolves conflicts and maintains the consolidated `metadata` column.

### Conflict Resolution Priority

```
MANUAL_OVERRIDE  (10)  → always wins
VERIFIED          (9)  → verified extraction / human confirmed
HIGH_CONF_AI      (8)  → confidence ≥ 0.9
MC_INHERIT        (7)  → master catalog inherited
MEDIUM_CONF_AI    (6)  → confidence 0.7–0.9
LOW_CONF_AI       (5)  → confidence < 0.7
SCRAPE_PROVIDER   (4)  → raw provider scrape value (lowest)
```

### Array Fields (`amenities`, `restrictions`)

Set-union across all sources. An entry is included if at least one source reports it with confidence ≥ 0.6.

### Trigger Points

Aggregation runs (async, via RabbitMQ) when:

- An `asset.uploaded` event triggers extraction → extraction completes → `extraction.completed` triggers aggregation
- A user submits a manual override → immediate re-aggregation
- A scheduled job catches stale venues (24h without re-aggregation)

Aggregation is debounced (5s) to batch rapid successive events. The `venuemi_metadata_schema_version_seen_total` Micrometer metric (see [observability.md](observability.md)) records the pre-migration version on every aggregation pass — use it to monitor stale-document distribution and scheduled job catch-up rate.

---

## Profile Stage Recalculation

`profile_stage` is recomputed inside every aggregation consumer transaction — immediately after the metadata merge, before the `UPDATE` commits. It is never set by application code outside `MetadataAggregationConsumer`.

Transition rules evaluated in order:

| Stage      | Condition                                                                                  |
| ---------- | ------------------------------------------------------------------------------------------ |
| `SEEDED`   | Default. Name present + at least one asset exists in `venue_assets`.                       |
| `ENRICHED` | `capacity.max_total` non-null + `venue_type` non-empty + `catering.policy` non-null.       |
| `CURATED`  | All `ENRICHED` conditions met + `COUNT(*) FROM venue_annotations WHERE venue_id = ?` ≥ 1.  |
| `READY`    | All `CURATED` conditions met + `primary_photo_asset_id` non-null + `website_url` non-null. |

Regression is allowed: if a key field is removed (e.g. manual override clears `catering.policy`), stage drops back to `SEEDED` or `ENRICHED` accordingly.

The `venue_annotations` COUNT and the `venue_assets` existence check are the only two reads outside `venues` that happen inside the aggregation transaction. Both are indexed (`idx_annotations_venue`, `idx_assets_venue`) and add negligible overhead.

The full step sequence with `profile_stage` included:

Metadata aggregation is a read-modify-write operation: `SELECT venues.metadata` → merge extracted fields → `UPDATE venues.metadata`. If three extraction jobs for the same venue complete in parallel, three workers can simultaneously read stale metadata, each merge one PDF's fields, and each write back — two of the three writes are lost (Lost Update anomaly).

The chosen approach eliminates the race at the messaging layer using RabbitMQ's built-in capabilities. No distributed locks, no optimistic locking retry loops, no row-level `SELECT … FOR UPDATE` contention.

### Why RabbitMQ FIFO Routing Per `venue_id`

| Constraint                | How FIFO routing delivers it                                                                                                                                          |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Zero new dependencies     | Uses existing RabbitMQ (already in platform stack). No Redis, no distributed lock library.                                                                            |
| Minimal code footprint    | One-line routing key computation on publish; consumer configuration only. No retry-loop code, no deadlock corner cases.                                               |
| Extraction stays parallel | Three PDFs extract concurrently (CPU/IO-bound — fast). Only the final metadata merge step (microseconds of in-memory merge + 1 SQL `UPDATE`) is serialised per venue. |
| No infrastructure cost    | Same RabbitMQ cluster, same number of consumer threads. No extra services or sidecars.                                                                                |
| Horizontal scalability    | As venue count grows, increase the hash slot count and consumer pool size. Different venues always process in parallel.                                               |

### Implementation Variants

Two variants share the same conceptual model. Start with A1 for MVP; both use the same consumer code shape.

**Variant A1 — Simple (MVP).** One queue, one serialising consumer.

| Aspect                | Specification                                                                                                                                                               |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Queue                 | Single queue `venuemi.metadata.aggregation`.                                                                                                                                |
| Publisher routing key | `extraction.completed` unchanged. No slot computation.                                                                                                                      |
| Consumer              | `@RabbitListener` with `concurrency = 1`, `prefetchCount = 1`. Exactly one thread processes all aggregation events sequentially across all tenants and all venues.          |
| Backlog envelope      | Aggregation per event is ~1 ms (merge + SQL `UPDATE`). Even 100 events/s sustained yields a 100 ms backlog, invisible to end users and well within the 5 s debounce window. |
| When to choose        | MVP. The product does not expect mass-parallel upload across many accounts simultaneously. Migrate to A2 when queue depth metrics breach threshold.                         |

**Variant A2 — Scalable.** N hash-partitioned queues with per-queue single-threaded consumption.

| Aspect               | Specification                                                                                                                                    |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Queues               | `venuemi.metadata.aggregation.0` through `venuemi.metadata.aggregation.15` (16 slots; configurable via `application.yml`).                       |
| Publisher routing    | Slot = `Math.abs(venueId.hashCode() % SLOT_COUNT)`. Same `venue_id` always maps to the same slot.                                                |
| Consumer pool        | 16 consumer threads. Each thread binds to exactly one slot queue with `prefetchCount = 1`.                                                       |
| Parallelism property | Different venues process in parallel across slots. Same venue always routes to the same slot → strict FIFO ordering per venue.                   |
| When to choose       | If immediate horizontal headroom is desired, or to avoid an A1→A2 queue topology migration later. ~20 extra lines of code on the publisher side. |

The full step sequence with `profile_stage` included:

```
Consumer transaction boundary (single DB transaction):
  1. BEGIN
  2. SELECT venues.metadata, venues.metadata_aggregated_at
     FROM venues WHERE id = ?
  3. If metadata_aggregated_at within 5 s debounce window → no-op, ack message.
  4. Else → merge all unprocessed venue_metadata_events into metadata
     via VenueMetadataMigrator.ensureCurrent() + conflict resolution
  5. SELECT COUNT(*) FROM venue_assets       WHERE venue_id = ?   (idx_assets_venue)
     SELECT COUNT(*) FROM venue_annotations  WHERE venue_id = ?   (idx_annotations_venue)
     → compute new profile_stage from merged metadata + counts above
  6. UPDATE venues SET
       metadata                  = ?,
       metadata_sources          = ?,
       metadata_aggregated_at    = NOW(),
       profile_stage             = ?,
       updated_at                = NOW()
     WHERE id = ?
  7. DELETE / mark-consumed processed venue_metadata_events
  8. COMMIT
  9. RabbitMQ ack — only after successful COMMIT
```

Acknowledgement mode on the listener container must be `MANUAL` (or `AUTO` with `prefetchCount = 1`). One message at a time per queue — never batch.

---

## Concurrency Control and Race Condition Prevention

### Debounce + FIFO Synergy

The 5-second debounce window composes naturally with FIFO ordering:

1. Three PDFs finish extraction almost simultaneously → three `extraction.completed` events published, all routed to the same slot queue for venue X.
2. Event 1 is consumed first. `metadata_aggregated_at` is older than 5 s → aggregation runs, `metadata_aggregated_at` is set to `NOW()`.
3. Event 2 is consumed next. `metadata_aggregated_at` is within the 5 s window → no-op, message acked without SQL `UPDATE`.
4. Event 3 is consumed next. Same no-op path.

Outcome: one SQL `UPDATE` instead of three.

### Secondary Benefits

- **No retry loops or conflict handling.** No conflict exception, no exponential-backoff retry code, no test surface for livelock.
- **Event sourcing friendly.** If `venue_metadata_events` are replayed, per-venue ordering is preserved at the messaging layer.
- **Straightforward integration testing.** Seed three `extraction.completed` events for the same `venue_id`, consume, assert final `venues.metadata` contains merged fields from all three sources. No `CountDownLatch` multi-threaded test harness.
- **Compatible with `_schema_version`** (see [data-model.md](data-model.md) §2a). The `VenueMetadataMigrator` runs inside step 4 of the consumer transaction, before the `UPDATE` commits.

### Queue Consumer Configuration

See [events.md](events.md) for the full RabbitMQ queue and listener container configuration table.

---

**Docs:** [Architecture Index](README.md) · [Data Model](data-model.md) · [Services](services.md) · [ETL Pipeline](etl-pipeline.md) · [Events](events.md) · [Master Catalog](master-catalog.md) · [Observability](observability.md)
