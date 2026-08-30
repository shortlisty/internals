# D7 — JSONB metadata schema versioning with online migration

> **Audience:** Engineers, architects.
> **Purpose:** Record why `venues.metadata` JSONB carries an explicit `_schema_version` integer with an incremental chained-migration pipeline that runs on every read and write (online incremental convergence), rather than offline batch backfill jobs or no versioning at all.

---

## Context

The `venues.metadata` JSONB column stores the consolidated venue profile: capacity configurations, catering policy, AV specs, accessibility flags, restrictions, contacts, pricing. The canonical field set (the schema that this JSONB is expected to conform to) evolves over time:

- v1 ships with the MVP canonical field set.
- v2 adds `catering.vegan_available` and renames `capacity.configurations.banquet` to `capacity.configurations.round_banquet`.
- v3 adds `av_tech.microphone_count` and splits `logistics.parking_spaces` into `{ onsite, valet, street }`.
- v4 adds `contacts[].linkedin_url`, and so on.

Without a versioning strategy, a venue written against v1 that is read by v4 backend code deserialises incompletely: null pointer fields, type coercion failures, silently dropped keys, UI showing "Unknown" for fields the user previously confirmed as values.

---

## Options considered

### Option A — No versioning. "Schema is whatever is in the JSONB."

The backend reads whatever JSONB is in the row and attempts to deserialise it into the latest `VenueMetadata` POJO. Missing keys → `null`. Extra keys → silently dropped (Jackson `FAIL_ON_UNKNOWN_PROPERTIES = false`).

**Pros:**

- Zero infrastructure code. No migrator, no version stamps, no migration classes.
- No migration bugs — there is nothing to go wrong.

**Cons:**

- **Silent data corruption on every schema bump.** A v1 `capacity.configurations.banquet = 300` value renamed to `round_banquet` in v2 becomes `null` in v2+ code. The UI shows "Capacity not set" and the user must re-enter a value that was confirmed 18 months earlier. For event-planner data that represents hours of human verification, this is product-breaking: the user's previously approved values disappear without explanation on a routine software update.
- **Unbounded type-coercion risk.** When `restrictions` changes from `string` (v1, single value) to `string[]` (v2, array), Jackson's silent coercion either: (a) drops the single string, (b) wraps it in a 1-element array unpredictably depending on ObjectMapper config, or (c) throws a runtime JsonMappingException that returns a 500 to the user reading that venue. No configuration of Jackson handles all coercion cases uniformly across all nested types.
- **Cannot distinguish "user never set this field" from "field was present in an old version."** Both are `null` in the POJO. A future feature that defaults a field to `false` only when the user never touched it cannot distinguish the two cases, leading to incorrect defaults on old venues.
- **Schema evolution becomes a political process.** Engineering avoids field renames and type changes because every one silently breaks old data. Over 3+ years the canonical field set accumulates technical-debt cruft: fields kept under wrong names forever, type changes avoided even when they are correct. This is the classic "JSONB without versioning death spiral."

### Option B — Offline batch backfill migration job

When a schema bump from vN to vN+1 ships, a one-shot admin job runs:

```sql
SELECT id, metadata FROM venues WHERE metadata->_schema_version < N
```

For each row, apply the vN→vN+1 migration in Java and UPDATE. Mark the job complete. Continue with vN+1→vN+2, etc., until all rows are stamped at CURRENT.

**Pros:**

- Read path is clean. Every reader assumes all rows are at CURRENT version; no per-read migration overhead.
- Batch job has monitoring, progress reporting, and dry-run capability standard in admin tooling.

**Cons:**

- **Zero-day window between deploy and job completion = broken reads.** The v2 deploy ships at t=0. User opens a v1 venue at t=10 s before the batch job reaches that row. They see nulls/500. For a product whose primary value is "the data you confirmed is always here," this outage window is unacceptable even if it lasts only minutes.
- **Hot-row write amplification on a live system.** A 100,000-venue tenant's batch UPDATE runs as 100,000 individual UPDATE statements (or a few large batched ones). Each one produces WAL, triggers autovacuum, and competes with live-user writes for the same row locks. Large tenants see visible search/write latency degradation during the backfill window. With schema-per-tenant (D6), this is 1,000 tenants × one backfill run = 1,000 background UPDATE storms.
- **Stale documents never touched = never migrated.** A dormant venue written in v1 that nobody reads for 2 years is fine… until a user opens it in v7 and the backfill job ran in v2. The row never saw v3→v7 migrations because it was dormant during the job run. Result: same silent corruption as Option A. You can re-run the backfill job for "any row below CURRENT" every release, but then you're essentially re-implementing Option C's logic as a batch job — plus you still have the zero-day window on each deploy.
- **Cannot run inside transaction for large tenants.** A backfill spanning 100K rows holds a transaction open for minutes, creates replication lag, risks statement-timeout termination that rolls back the entire run, and on crash leaves a partially-migrated state with no resume marker.

### Option C — Online incremental convergence: `_schema_version` + chained migrator on every read and write

Every `venues.metadata` JSONB document carries an integer `_schema_version` at the top level. A `VenueMetadataMigrator` (pure Java, zero Spring deps, shared-library singleton) implements:

- On **every read** via MyBatis `TypeHandler`:

  ```
  raw_jsonb → extract _schema_version (absent = 0)
    → run sequential migrations: v0→v1, v1→v2, v2→v3 ... until CURRENT
    → return JsonNode at CURRENT (safe to deserialize into VenueMetadata POJO)
  ```

  DB row remains unchanged; migration happens in-memory only.

- On **every write** (aggregation, manual PATCH, bulk import, MC_INHERIT merge):
  ```
  candidate_json → VenueMetadataMigrator.ensureCurrent(node)
    → run pending migrations
    → stamp _schema_version = CURRENT
    → validate against canonical schema
    → persist stamped node
  ```

Rules:

- Migrations are pure functions: same input → same output. No I/O, no randomness.
- Unknown keys are preserved (forward-compatibility for A/B testing).
- Migration list is append-only. Classes are never deleted or reordered.

**Pros:**

- **Zero outage window on every deploy.** v2 code deployed at t=0. User reads a v1 venue at t=10 s → in-memory migration runs in microseconds, returns correct v2 POJO. Zero broken reads. Zero dashboards. Zero user-visible field loss.
- **Backfill is incremental and lazy, not a job.** First read of a stale venue → migrated in-memory. Next write to that venue → stamped to CURRENT permanently. "Dormant" venues (not read or written for 2 years) are not migrated until someone actually opens them — no write amplification, no WAL, no user impact. Active venues converge to CURRENT within their first access after deploy. Warm cache effect: venues used by tenants reach CURRENT quickly; nobody notices the ones nobody touches.
- **Pure function migrations are trivially unit-testable.** Each migration class is one Java file with `fromVersion()`, `toVersion()`, and `apply(JsonNode)`. Unit test: input JSON at fromVersion → assert output JSON at toVersion. No Spring context, no database, no integration harness. 100% coverage of every migration with a 3-line test per class.
- **Read path overhead is measurable and bounded.** For a 5-version-stale document (worst case), 5 chained pure-function transformations on a 2 KB JSONB node runs in <50 µs. In benchmarks, this overhead is invisible inside the network + PostgreSQL query latency. MyBatis `TypeHandler` runs synchronously inside the result-set construction; 50 µs per row on a 20-row search result adds 1 ms total to a 50–150 ms query. Undetectable.
- **Schema bumps become boring, routine, zero-incident operations.** A v2 bump is: bump `CURRENT_SCHEMA_VERSION` constant, add `MetadataMigrationV1ToV2.java`, add 3-line unit test, merge, deploy. No admin job, no runbook, no customer communication, no maintenance window. Schema evolution changes from a quarterly event with an SOP to a 30-minute task that ships in any regular release.

**Cons:**

- **Read path code complexity budget consumed.** The `TypeHandler` runner + migrator chain + migration registry is ~500 lines of infrastructure code that must be correct and well-tested. A bug in the migrator (e.g., migration V1→V2 drops unknown keys) corrupts every read of a V1 venue on any tenant. This code path demands a higher code-review bar and static-analysis contract enforcement.
- **Stale document version distribution observability requires custom metrics.** Without a batch backfill, you do not know how many V1 venues remain until someone reads them. The migrator records Micrometer `shortlisty_metadata_migration_versions_total{from_version=N}` on every call; over time, the distribution of `from_version` counts shows when V1→V2 migration count drops to near-zero and the V1→V2 class can be considered for eventual removal (after 24 months with zero calls).

---

## Decision made

**Option C: Online incremental convergence with `_schema_version` + chained migrator on every read and write.**

- `_schema_version` integer key at the top level of every `venues.metadata` JSONB document. Absent → v0.
- `VenueMetadataMigrator` singleton in `shortlisty-venue-model` (shared library, zero Spring deps).
- `VenueMetadataTypeHandler` (extends platform generic `MetadataTypeHandler`) runs `migrateToCurrent()` on every SELECT via MyBatis ResultMap.
- Every write path (aggregation consumer, PATCH metadata handler, bulk import, master catalog backdrop gap-fill, `CreateVenueRequest` default builder) calls `ensureCurrent()` before INSERT/UPDATE.
- Migration classes are pure-Java, unit-tested with 3-line fixtures, append-only list, never reordered or deleted.

---

## Rationale

- **Zero broken reads on deploy is a hard product requirement.** Event-planner agencies rely on Shortlisty for their venue pitch data. A deploy that silently shows "capacity not set" on 30% of venues until a batch job completes, or that returns 500s on type-coercion failures, would cause users to permanently distrust the product's data reliability guarantee. Option C provides perfect correctness on every deploy with zero coordination — that alone is decisive.
- **Schema bumps happen far more frequently than intuition suggests.** The canonical field set started with ~15 leaf fields. After 12 months of real agency usage, it has been expanded to ~40 leaf fields across 6 nested objects, with two renames and one type widening. Without Option C, every one of those 12 changes would have required an admin batch backfill job with runbook, maintenance window, customer communication, and user-visible outage risk. With Option C, all 12 shipped in regular releases with zero operational overhead.
- **JSONB pure-function migrations are orders of magnitude simpler than SQL schema migrations.** A Liquibase `ALTER TABLE` migration on a live 100K-row table requires `CREATE INDEX CONCURRENTLY`, transaction-scope planning, replication-lag monitoring, and rollback procedures. A `MetadataMigrationV1ToV2` Java class is 20 lines of Jackson `ObjectNode` manipulation with a 3-line unit test. The engineering effort for each schema evolution drops from 1–3 days to 30 minutes. This qualitatively changes the team's willingness to correct bad field-shape decisions early rather than accumulating cruft.
- **The migrator is the same singleton on read and write, so drift is impossible.** Because `VenueMetadataMigrator` lives in `shortlisty-venue-model` and is imported by both `shortlisty-catalog-service` (reads/writes) and `shortlisty-catalog-processing-worker` (writes), both services use bit-identical migration chain logic. There is no path where a writer stamps V5 via a different migration sequence than a reader's in-memory V5 output produces. Classpath-identical migrator instance guarantees writer→reader agreement regardless of which service touches the row.

---

## Consequences

- `_schema_version` must never be set to a future number, never manually edited via SQL, and never removed. An ArchUnit/static-analysis rule validates that no code path writes a `_schema_version` value other than via `VenueMetadataMigrator.ensureCurrent()`.
- Migration classes are added, never removed or reordered. After 24 months of zero `shortlisty_metadata_migration_versions_total{from_version=1}` readings, a follow-up decision record evaluates whether `MetadataMigrationV0ToV1` can be deleted (requires guaranteeing zero v0 documents remain in any tenant schema, verified via a one-shot admin SELECT across all schemas). Until then, it stays.
- `shortlisty_metadata_migration_versions_total` is a core release-readiness metric. Before each deploy, the dashboards are reviewed; if from_version counts for the "next oldest" migration are still non-trivial, the release proceeds but a ticket is opened to investigate why those venues are stale.
- A one-shot admin endpoint exists for force-convergence: `POST /api/v1/admin/tenants/{id}/venues/converge-metadata` issues `UPDATE venues SET metadata = metadata WHERE id IN (...)` per venue to trigger the write-path stamping for a specific tenant. Used only before heavy schema jumps where customer has asked for proactive backfill, not as a routine release step.

---

## Status

**Accepted.** `_schema_version` + chained migrator pattern implemented in shared model library.

---

**Docs:** [Vision](../vision.md) · [Architecture](../../platform/README.md) · [Intelligence Layer](../../platform/intelligence.md) · [Epics](../epics/README.md)
