# D15 — Master catalog population in a dedicated worker (two workers, four components)

> **Audience:** Engineers, architects.
> **Purpose:** Record which component populates the master catalog, and why that role stays in its own worker even though it shares one data model with `docintelligence-catalog-processing-worker`.

---

## Context

The master catalog pipeline is three steps: a per-provider scraper writes master-catalog-ready JSON to S3, a loader writes it into the `public.master_venue*` tables, and the tenant ETL pipeline resolves each venue against those tables at extraction time. The four components and their roles:

| Component                                        | Role                                                                                                                                                                     |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `docintelligence-mc-ingest-<source>-scraper`     | Node CLI, nightly cron. Scrapes provider sites and writes master-catalog-ready JSONL to S3. No LLM, no database access.                                                   |
| `docintelligence-master-venue-worker`            | Reads that JSONL from S3 and writes `public.master_venue*`. Deterministic dedup and normalisation only — no LLM, no tenant schema access.                                 |
| `docintelligence-catalog-processing-worker`      | Tenant ETL: parse, chunk, extract, embed, aggregate; writes tenant-schema tables and maps tenant venues to master catalog matches (copy-on-match MC_INHERIT).              |
| `docintelligence-catalog-service`                | Control plane and APIs: venue CRUD, search, asset upload flow, admin master catalog management, and the only service that runs Liquibase.                                 |

This decomposition is already the documented one ([D1](D1-one-service-vs-two.md), [services.md § Table ownership](../../platform/services.md), component inventory in [README.md](../../../README.md)). What was never recorded is *why* the population role is not folded into the processing worker, given that the two workers share everything below the code:

- One PostgreSQL instance for the whole product ([D13](D13-shared-database-per-product.md)).
- The master catalog tables are created once in `public` by `docintelligence-catalog-service`'s `system/master.xml` changelog, not per tenant ([data-model.md](../../platform/data-model.md)).
- The tenant↔master link is an app-level FK (`venues.master_venue_id`) with no DB-level constraint ([data-model.md](../../platform/data-model.md)).
- Import and ETL both honour the same `_schema_version` contract and must call `VenueMetadataMigrator.ensureCurrent()` before writing metadata ([data-model.md](../../platform/data-model.md)).
- The `MasterVenue` entity and its MyBatis mapper already live in the processing worker's codebase, which is explicitly allowed to name the `public` schema.

So the data model does not separate these roles. The open question is operational: does master catalog population need its own deployable?

---

## Options considered

### Option A — Population role inside `docintelligence-catalog-processing-worker`

The loader runs as a scheduled job inside the processing worker, reusing the entity, mapper, S3 client, and RabbitMQ wiring that are already there.

**Pros:**

- One fewer deployable: no third Java service, no extra CI/CD pipeline, chart values, dashboards, or alert rules
- All required infrastructure already exists in the worker (MyBatis over the shared instance, MinIO client, AMQP, Actuator)
- Scheduled maintenance jobs are already part of the worker's documented responsibilities ("scheduled maintenance jobs (stale re-aggregation, cost reporting)")
- No second copy of the `MasterVenue` model and mapper to keep in agreement

**Cons:**

- The worker is scaled on RabbitMQ queue depth, so a scheduler fires on every replica; correctness needs a leader lock (ShedLock or a Postgres advisory lock) that the worker does not carry today
- Bulk import plus the trigram GIN REINDEX it triggers competes for CPU, memory, and JVM lifecycle with tenant extraction — [D4](D4-cross-source-search-merge.md) already treats REINDEX during scraper import as a search-degradation event
- A malformed scraper file or a failed import batch becomes an extraction-worker incident, marrying two unrelated blast radii
- Scraper-facing concerns (imports prefix, dry-run report artifacts, approved-batch bookkeeping) leak into the ETL codebase
- The ownership rule ("exactly one writer per table") gets harder to reason about when one deployable owns both streaming ETL tables and batch master catalog tables

### Option B — Population stays in a dedicated `docintelligence-master-venue-worker`

The loader remains its own Java service, as [D1](D1-one-service-vs-two.md) and [D13](D13-shared-database-per-product.md) already state.

**Pros:**

- Deterministic, LLM-free batch work gets its own deploy, scale profile, resource quota, and release cadence, independent of tenant extraction throughput
- Blast radius stays narrow in both directions: a bad import batch or a REINDEX storm cannot degrade extraction, and an extraction backlog cannot delay a catalog refresh
- Each role holds its own service account against the shared instance
- Exactly one writer per master catalog table, which is what the ownership contract requires — today that contract has no row for `master_venue*` at all
- Matches the accepted decisions and the existing component inventory; no reversal of D1 or D13 wording

**Cons:**

- Third Java deployable: pipelines, chart values, quotas, observability, on-call surface
- Shared-schema coordination burden (mitigated: migrations stay in `docintelligence-catalog-service`, and neither worker runs them)
- A second copy of the `MasterVenue` model and mapper across repositories, per the self-contained-service rule

---

## Decision made

**Option B: master catalog population stays in a dedicated `docintelligence-master-venue-worker`.** Two workers, four components, non-overlapping responsibilities:

- `docintelligence-mc-ingest-<source>-scraper` — scrape providers and emit master-catalog-ready JSON to S3. No database access.
- `docintelligence-master-venue-worker` — read that JSON, run deterministic dedup and normalisation, write `public.master_venue*`. No LLM, no tenant schema.
- `docintelligence-catalog-processing-worker` — tenant ETL, plus master catalog match and MC_INHERIT merge. Read-only across the master catalog boundary.
- `docintelligence-catalog-service` — control plane, APIs, admin master catalog management, migrations.

This record is **provisional**: the two-worker split is the working assumption, and the import trigger design (admin dry-run/apply events, `--file` CLI, human review before apply) continues to live in [master-catalog.md](../../platform/master-catalog.md) unchanged.

---

## Rationale

- **A shared data model is not a service boundary.** Both workers would connect to the same instance with the same contract whether they are one deployable or two. Since the model is shared anyway, the split costs nothing structurally — it only removes the operational coupling described below. Accepted for the same reason as [D13](D13-shared-database-per-product.md): sharing an instance is a deliberate decision, not an accident to be consolidated away.
- **The two roles have opposite runtime shapes.** Tenant extraction is streaming, LLM-bound, and scaled on queue depth. Master catalog population is batch, deterministic, LLM-free, and driven by a scraper's output file. Merging them means one scaling policy, one resource quota, and one failure mode for both.
- **Import work is allowed to be disruptive to search.** [D4](D4-cross-source-search-merge.md) explicitly accepts search degradation while the master catalog trigram index is rebuilt during a bulk import. That concession is tolerable when it is confined to a batch worker and hazardous when the same process is also draining the extraction queue.
- **Ownership discipline requires exactly one writer.** The ownership contract in [services.md](../../platform/services.md) says cross-boundary writes are not allowed, and it currently has no rows for `master_venue`, `master_venue_alias`, or `master_venue_external`. Naming a single owning service per table is what makes that rule checkable at code review.
- **The scraper boundary is a file, not a call.** The hand-off between scraping and population is an object in S3, so the loader can be scaled, restarted, or re-run over a historical import prefix without touching the scraper or the catalog service. That property is worth keeping in a separate deployable that has no inbound HTTP surface of its own.

---

## Consequences

- Four deployables per environment: one Node CLI scraper, `docintelligence-master-venue-worker`, `docintelligence-catalog-processing-worker`, `docintelligence-catalog-service`.
- The ownership table in [services.md](../../platform/services.md) gains rows for `master_venue`, `master_venue_alias`, and `master_venue_external`, owned by `docintelligence-master-venue-worker`; DDL for those tables stays with `docintelligence-catalog-service`'s `system/master.xml`.
- `MasterVenueMapper` in the processing worker currently exposes write methods. Under this decision, writes belong to `docintelligence-master-venue-worker` and the mapper in the processing worker is read-only for the match step. This is a code follow-up, tracked outside this record.
- Neither worker runs migrations; neither worker exposes inbound HTTP beyond health, readiness, and metrics.
- The cold-start seeding path described in [D8](D8-master-venue-seeding-infrastructure-first.md) and [cold-start.md](../../business/personal-venue-catalog/cold-start.md) (admin CRUD API or importer CLI) keeps both entry points, one per component.
- Sequencing — which of the two workers is built first — is deliberately not fixed here.

---

## Status

**Proposed.** Recorded as a working assumption: the topology is settled (two workers, four components), the trigger and deployment details of the importer are not. If the assumption is dropped in favour of a single worker, this record is superseded by a new decision rather than edited.

---

**Docs:** [Decisions index](README.md) · [Architecture](../../platform/README.md) · [Master catalog](../../platform/master-catalog.md)
