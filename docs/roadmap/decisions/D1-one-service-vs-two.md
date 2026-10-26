# D1 — One service vs. two

> **Audience:** Engineers, architects.
> **Purpose:** Record why the venue domain is split into a synchronous request/response service and an asynchronous ingestion sidecar, rather than a single monolithic service.

---

## Context

The Shortlisty venue domain has two distinct workloads:

1. **Synchronous user traffic** — venue CRUD, search, metadata reads, asset upload flow. These are user-facing HTTP requests that must respond quickly and fail independently.
2. **Asynchronous document processing** — PDF parsing, text extraction, GPT-4o structured extraction calls, embedding generation, metadata aggregation, master catalog matching. These are CPU/IO-bound, external-API-dependent, and can take seconds to minutes per asset.

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

- `docintelligence-catalog-service` — synchronous HTTP only. Owns `venues` and `venue_assets` writes.
- `docintelligence-catalog-processing-worker` — async sidecar only. No inbound HTTP. Consumes RabbitMQ events, runs ETL pipeline, writes `extraction_jobs`, `venue_metadata_events`, `item_vectors`, `ai_cost_tracking`.

Scraping and master catalog population are further decomposed into docintelligence-mc-ingest-<source>-scraper (Node.js per provider) and docintelligence-master-venue-worker (Spring Boot) — see architecture §4.

Both services connect to the same PostgreSQL instance and share the tenant schema (`t_{tenantKey}`). Neither declares a compile dependency on the other, so neither can break the other's build or release. Domain types that are not a contract between the two are shared through a library rather than copied — see [D16](D16-shared-artifacts.md).

**Pros:**

- Independent scaling: scale ingestion workers on queue depth, scale docintelligence-catalog-service on request rate
- Failure isolation: an LLM API outage or a malformed-PDF OOM kills only the ingestion worker; user search and CRUD continue
- Resource isolation: dedicate CPU/memory profiles per service (ingestion gets larger pods with more memory for in-process document parsing; docintelligence-catalog-service gets smaller pods tuned for request concurrency)
- Clean separation of concerns: service team can reason about "reads/API" vs "processing pipeline" independently
- Table ownership model (§4 of README.md) clarifies who writes what, preventing drift

**Cons:**

- Two deployment units, more CI/CD pipeline complexity
- Shared database schema requires coordination on migrations (the tenant-schema Liquibase changelog lives in `docintelligence-catalog-service`, the only service that runs it, mitigating this)
- Cross-boundary reads (processing-worker reads `venue_assets.s3_key`) must be disciplined — enforced by table ownership rules

---

## Decision made

**Option B: Two services sharing one database schema.**

One synchronous service (`docintelligence-catalog-service`) for HTTP and one async sidecar (`docintelligence-catalog-processing-worker`) for the ETL pipeline. Each service owns its event payloads and Liquibase changelogs, and neither declares a compile dependency on the other. Domain types that are not a contract between the two come from a shared library instead of a copy — see [D16](D16-shared-artifacts.md).

---

## Rationale

- **Event-processing workloads are bursty and failure-prone.** Document extraction calls external APIs (GPT-4o, text-embedding-3-small) with unpredictable latency and failure modes. Running this inside the same process as user-facing search guarantees that an upstream API incident degrades user-facing latency.
- **Scaling profiles are opposites.** The ingestion worker is CPU/memory-heavy during parsing and embedding, and largely idle otherwise. The venue service is IO-bound (PostgreSQL queries) with flat steady-state concurrency. Independent pod sizing cuts infrastructure cost at any scale above MVP.
- **Independent build graphs keep the split honest.** Neither service compiles against the other, so neither can break the other's build or release. Payload types stay duplicated on both sides because they are the contract between the two services; row types and persistence helpers that are not a contract are shared as libraries instead of copied ([D16](D16-shared-artifacts.md)). Table shape is kept in agreement by the table-ownership contract (venue-service owns `venues`, `venue_assets`; processing-worker owns the extraction and vector tables) plus the tenant-schema Liquibase changelog, which only `docintelligence-catalog-service` runs. The ownership contract removes write-path ambiguity.
- **Cross-boundary reads are narrow and well-defined.** The only read processing-worker makes across the ownership boundary is `SELECT s3_key FROM venue_assets WHERE id = ?` — a foreign-key lookup delivered inside the RabbitMQ event payload. No business logic crosses.
- **Pattern already validated in platform.** The iQ Key Value foundation already separates synchronous gateway/API services from async workers (billing event processors, audit log consumers). The ops team has tooling and runbooks for this topology.

---

## Consequences

- The deployment manifest ships two containers per environment (`docintelligence-catalog-service` + `docintelligence-catalog-processing-worker`). Helm chart has separate replica counts, HPA triggers, and resource quotas per service.
- Table ownership is documented in README.md §4 and enforced at code review: processing-worker never issues `UPDATE venues`; docintelligence-catalog-service never writes to `extraction_jobs` or `item_vectors` directly.
- Rolling deployments of `docintelligence-catalog-processing-worker` must use RabbitMQ `MANUAL` ack mode so in-flight messages are re-queued on consumer shutdown. No message loss is acceptable.
- HPA for `docintelligence-catalog-processing-worker` scales on RabbitMQ queue depth (`docintelligence.extraction.priority` / `docintelligence.extraction.standard` messages ready), not CPU. HPA for `docintelligence-catalog-service` scales on request rate + p95 latency.

---

## Status

**Accepted.** Architecture and services follow this split. No change planned to the topology. Amended by [D16](D16-shared-artifacts.md) on how code is shared between the two services.

---

**Docs:** [Decisions index](README.md) · [Architecture](../../platform/README.md) · [Epics](../epics/README.md)
