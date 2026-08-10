# D1 — One service vs. two

> **Audience:** Engineers, architects.
> **Purpose:** Record why the venue domain is split into a synchronous request/response service and an asynchronous ingestion sidecar, rather than a single monolithic service.

---

## Context

The OiQb venue domain has two distinct workloads:

1. **Synchronous user traffic** — venue CRUD, search, metadata reads, asset upload flow. These are user-facing HTTP requests that must respond quickly and fail independently.
2. **Asynchronous document processing** — PDF parsing, text extraction, GPT-4o structured extraction calls, embedding generation, metadata aggregation, registry matching. These are CPU/IO-bound, external-API-dependent, and can take seconds to minutes per asset.

The question is whether to combine both in one service or split them.

---

## Options considered

### Option A — Single monolithic service

One Spring Boot application handles both HTTP endpoints and async ETL processing. Extraction jobs run on in-process thread pools or a RabbitMQ listener inside the same process.

**Pros:**

- Simpler deployment topology — one container instead of two
- No shared-database concerns (one writer, no ambiguity)
- Fewer Maven modules, less build complexity

**Cons:**

- Resource contention: ETL workers consume CPU/memory under load, increasing p99 latency for search and CRUD endpoints
- Independent scaling impossible: if queue depth spikes, you scale the entire service including idle HTTP threads; if user traffic spikes, you scale ETL workers unnecessarily
- Failure coupling: an ETL crash (OOM from a malformed PDF, LLM API outage that exhausts retry threads) can bring down the HTTP surface
- Queue processing and HTTP serving share one process lifecycle — rolling deployments interrupt in-flight extraction jobs

### Option B — Two services sharing one database schema

Split into:

- `oiqb-venue-service` — synchronous HTTP only. Owns `venues` and `venue_assets` writes.
- `oiqb-venue-ingestion-worker` — async sidecar only. No inbound HTTP. Consumes RabbitMQ events, runs ETL pipeline, writes `extraction_jobs`, `venue_metadata_events`, `item_vectors`, `ai_cost_tracking`.

Both services connect to the same PostgreSQL instance and share the tenant schema (`t_{tenantKey}`). They share the domain model via a common library (`oiqb-venue-model`) so table definitions agree.

**Pros:**

- Independent scaling: scale ingestion workers on queue depth, scale venue-service on request rate
- Failure isolation: an LLM API outage or a malformed-PDF OOM kills only the ingestion worker; user search and CRUD continue
- Resource isolation: dedicate CPU/memory profiles per service (ingestion gets larger pods with more memory for Tika Pipes forked JVMs; venue-service gets smaller pods tuned for request concurrency)
- Clean separation of concerns: service team can reason about "reads/API" vs "processing pipeline" independently
- Table ownership model (§4 of architecture.md) clarifies who writes what, preventing drift

**Cons:**

- Two deployment units, more CI/CD pipeline complexity
- Shared database schema requires coordination on migrations (Liquibase changelogs live in the shared model library, mitigating this)
- Cross-boundary reads (ingestion-worker reads `venue_assets.s3_key`) must be disciplined — enforced by table ownership rules

---

## Decision made

**Option B: Two services sharing one database schema.**

One synchronous service (`oiqb-venue-service`) for HTTP and one async sidecar (`oiqb-venue-ingestion-worker`) for the ETL pipeline. Both share `oiqb-venue-model` (domain model + Liquibase changelogs) as a compile dependency.

---

## Rationale

- **Event-processing workloads are bursty and failure-prone.** Document extraction calls external APIs (GPT-4o, text-embedding-3-small) with unpredictable latency and failure modes. Running this inside the same process as user-facing search guarantees that an upstream API incident degrades user-facing latency.
- **Scaling profiles are opposites.** The ingestion worker is CPU/memory-heavy during parsing and embedding, and largely idle otherwise. The venue service is IO-bound (PostgreSQL queries) with flat steady-state concurrency. Independent pod sizing cuts infrastructure cost at any scale above MVP.
- **The shared-model library removes the biggest cost of a split.** Because migrations, POJOs, metadata migrations, and the canonical field set live in `oiqb-venue-model`, there is zero risk of the two services drifting apart on schema shape. The table-ownership contract (venue-service owns `venues`, `venue_assets`; ingestion-worker owns extraction/vector tables) removes write-path ambiguity.
- **Cross-boundary reads are narrow and well-defined.** The only read ingestion-worker makes across the ownership boundary is `SELECT s3_key FROM venue_assets WHERE id = ?` — a foreign-key lookup delivered inside the RabbitMQ event payload. No business logic crosses.
- **Pattern already validated in platform.** The iQ Key Value foundation already separates synchronous gateway/API services from async workers (billing event processors, audit log consumers). The ops team has tooling and runbooks for this topology.

---

## Consequences

- The deployment manifest ships two containers per environment (`venue-service` + `venue-ingestion-worker`). Helm chart has separate replica counts, HPA triggers, and resource quotas per service.
- Table ownership is documented in architecture.md §4 and enforced at code review: ingestion-worker never issues `UPDATE venues`; venue-service never writes to `extraction_jobs` or `item_vectors` directly.
- Rolling deployments of `venue-ingestion-worker` must use RabbitMQ `MANUAL` ack mode so in-flight messages are re-queued on consumer shutdown. No message loss is acceptable.
- HPA for `venue-ingestion-worker` scales on RabbitMQ queue depth (`oiqb.asset.uploaded` messages ready), not CPU. HPA for `venue-service` scales on request rate + p95 latency.

---

## Status

**Accepted.** Architecture and services follow this split. No change planned.

---

**Docs:** [Vision](../vision.md) · [Architecture](../../platform/architecture.md) · [Epics](../epics/README.md) · [Milestones](../milestones/README.md)
