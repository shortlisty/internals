# Shortlisty — Architecture Reference

> **Audience:** Engineers, architects.
> **Purpose:** Index and navigation hub for the full architecture documentation. Each section below links to its dedicated document.

---

## Document Map

| Document                                             | Sections covered                                                                                             | Key topics                                                                                                                                               |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [architecture-overview.md](architecture-overview.md) | §1 Platform Context · §14 Tech Decisions · §16 Implementation Patterns                                       | Foundation reuse, stack, multi-tenancy, security config, REST patterns, DTO/mapper conventions                                                           |
| [data-model.md](data-model.md)                       | §2 Domain Model · §2a Schema Versioning · §10 Database Schema                                                | Bounded contexts, canonical field set, JSONB `_schema_version`, `VenueMetadataMigrator`, Liquibase migrations, index strategy, cross-schema access rules |
| [aggregation.md](aggregation.md)                     | §3 Metadata Aggregation                                                                                      | Conflict resolution priority, FIFO race-condition prevention, RabbitMQ Variants A1/A2, consumer transaction boundary                                     |
| [services.md](services.md)                           | §4 Service Architecture · §4a `shortlisty-venue-model` · §4b S3 Storage · §4c `shortlisty-data-intelligence` | Service decomposition, table ownership, shared library internals, S3 key layout, lifecycle rules, GDPR deletion cascade                                  |
| [intelligence.md](intelligence.md)                   | §1 ETL Pipeline · §2 Intelligence Layer · §3 Extension Model · §4 Scalability · §5 Tech Decisions            | Spring AI processing pipeline, venue extraction schema, multi-source aggregation, confidence-sourced metadata, generic-to-domain strategy pattern        |
| [etl-pipeline.md](etl-pipeline.md)                   | §5 ETL Pipeline                                                                                              | Spring AI stages (parse → transform → load), asset type routing, processing SLAs, worker wiring                                                          |
| [search.md](search.md)                               | §6 Search Architecture                                                                                       | Search modes, cross-source orchestration (`VenueSearchOrchestrator`), `scope` parameter, RRF merge, failure isolation                                    |
| [api.md](api.md)                                     | §7 API Surface                                                                                               | All REST endpoints, DTOs, error responses, plan gates                                                                                                    |
| [events.md](events.md)                               | §8 Event Contracts · §9 Plan Entitlements                                                                    | RabbitMQ routing keys, queue topology, consumer configuration, feature codes                                                                             |
| [master-catalog.md](master-catalog.md)               | §4b (master catalog) · cold-start · dedup · MC_INHERIT                                                       | Cold-start channels, alias normalisation, `MasterVenueMatcher` algorithm, threshold calibration                                                          |
| [observability.md](observability.md)                 | §12 Observability · §13 Security                                                                             | Prometheus metrics, Grafana dashboard, tenant isolation, GDPR, asset access, authority summary                                                           |
| [roadmap-decisions.md](roadmap-decisions.md)         | §15 Open Decisions · §17 Next Steps                                                                          | Resolved/pending ADRs, pre-Sprint 1 tasks, Phase 2/3 backlog, strategic bets                                                                             |

---

## New Services Introduced by Shortlisty Intelligence

### Shared Libraries (compile-time JARs, no runtime process)

| Library                        | Language              | Role                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------------ | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `shortlisty-data-intelligence` | Java (no Spring Boot) | Domain-agnostic platform layer — `ExtractionJob`, `ExtractionStatus`, `ExtractorType`, `AssetType`, `MetadataMigration` interface, `MetadataMigrator` chain runner, `MetadataTypeHandler` abstract base, event POJOs (`AssetUploadedEvent`, `ExtractionCompletedEvent`, `ExtractionFailedEvent`), infrastructure Liquibase changelogs (`extraction_jobs`, `item_metadata_events`, `item_vectors`, `ai_cost_tracking`). No venue-specific fields. Reusable across verticals. See [services.md § 4c](services.md). |
| `shortlisty-venue-model`       | Java (no Spring Boot) | Venue-domain layer — `Venue`, `VenueStatus`, `VenueAsset`, `VenueMetadata` (canonical field set), `VenueCapacity`, `VenueMetadataMigrator`, `VenueMetadataTypeHandler`, `VenueMetadataSchemaVersion.CURRENT_SCHEMA_VERSION`, concrete migration classes, `MasterVenue`, `MasterVenueAlias`, `MasterVenueExternal` POJOs, venue and master catalog Liquibase changelogs. Depends on `shortlisty-data-intelligence`. See [services.md § 4a](services.md).                                                          |

### Runtime Services

| Service                                 | Language / Runtime | Deployment                                             | Role                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| --------------------------------------- | ------------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `shortlisty-catalog-service`            | Java / Spring Boot | `Deployment` (HTTP, HPA on request rate + p95 latency) | Synchronous REST API — venue CRUD, asset upload flow (presigned URL), metadata read/write, search (all 5 modes), plan entitlement enforcement, master catalog backdrop lookup. Exposes `/api/v1/venues`. See [api.md](api.md).                                                                                                                                                                                                              |
| `shortlisty-catalog-processing-worker`  | Java / Spring Boot | `Deployment` (HPA on RabbitMQ queue depth)             | Async sidecar — document ETL pipeline (Tika parse → chunk → GPT-4o extract → embed), extraction job lifecycle, `MasterVenueMatcher` MC_INHERIT merge, metadata aggregation consumer, scheduled maintenance jobs (stale re-aggregation, cost reporting). No inbound HTTP. Shares PostgreSQL schema with `shortlisty-catalog-service`. See [etl-pipeline.md](etl-pipeline.md).                                                                |
| `shortlisty-mc-ingest-tagvenue-scraper` | Node.js (≥ 22.13)  | `CronJob` (nightly)                                    | Stateless CLI scraper for Tagvenue.com venue and room listings. Produces `MasterVenueRecord`-compatible CSV/JSONL files. No DB connections — uploads output to S3 `shortlisty/master-catalog/imports/{importId}/` for `shortlisty-master-venue-loader` to consume. Supports SSR-only and Playwright-browser modes. See [master-catalog.md](master-catalog.md).                                                                              |
| `shortlisty-master-venue-loader`        | Java / Spring Boot | `Deployment` (event-driven, low replica count)         | Master Catalog population service — listens for `admin.master-catalog.import.dry-run` RabbitMQ event, runs `MasterCatalogImportOrchestrator` (dedup check, CSV audit report upload to S3), then applies reviewed actions on `admin.master-catalog.import.apply`. Writes to `public.master_venue`, `master_venue_alias`, `master_venue_external`. Never makes autonomous insert/merge decisions. See [master-catalog.md](master-catalog.md). |

### Dependency Graph

```
shortlisty-data-intelligence  (platform JAR — no vertical deps)
        │
        ▼
shortlisty-venue-model        (venue-domain JAR — imports shortlisty-data-intelligence)
        │
        ├── shortlisty-catalog-service          (Spring Boot — synchronous API)
        └── shortlisty-catalog-processing-worker (Spring Boot — async ETL sidecar)

shortlisty-mc-ingest-tagvenue-scraper  (Node.js CronJob — no Java deps)
        │ CSV/JSONL → S3
        ▼
shortlisty-master-venue-loader          (Spring Boot — master catalog apply, reads shortlisty-venue-model)
        │ writes
        ▼
public.master_venue / master_venue_alias / master_venue_external
```

---

## New Infrastructure

| Component          | Notes                                        |
| ------------------ | -------------------------------------------- |
| pgvector extension | On existing PostgreSQL — no new service      |
| PostGIS extension  | On existing PostgreSQL — no new service      |
| IBM Docling        | Optional self-hosted container, Phase 2 only |

---

## Quick Cross-Reference

| If you need to know…                                                               | Go to…                                                          |
| ---------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| How `venues.metadata` evolves over time without a migration job                    | [data-model.md § 2a](data-model.md)                             |
| Why three PDFs for the same venue don't produce a Lost Update                      | [aggregation.md](aggregation.md)                                |
| How a master catalog field ends up in a tenant's venue                             | [master-catalog.md](master-catalog.md)                          |
| Why `MasterVenueQueryMapper` is one of only three mappers allowed to use `public.` | [data-model.md § Cross-Schema Access Rules](data-model.md)      |
| How `asset.uploaded` becomes a vector in `item_vectors`                            | [etl-pipeline.md](etl-pipeline.md)                              |
| What `scope=BOTH` does in the search API                                           | [search.md](search.md)                                          |
| Which table `shortlisty-catalog-processing-worker` may not write to                | [services.md § Table Ownership](services.md)                    |
| How a new vertical reuses the extraction pipeline                                  | [services.md § 4c](services.md)                                 |
| Where `CURRENT_SCHEMA_VERSION` is defined                                          | [services.md § 4a](services.md)                                 |
| Why `USER` authority does not exist on this platform                               | [architecture-overview.md § Security](architecture-overview.md) |

---

**Docs:** [What is Shortlisty?](../README.md) · [Architecture Overview](architecture-overview.md) · [Data Model](data-model.md) · [Services](services.md) · [Aggregation](aggregation.md) · [Master Catalog](master-catalog.md) · [Intelligence](intelligence.md) · [ETL Pipeline](etl-pipeline.md) · [Search](search.md) · [API](api.md) · [Events](events.md) · [Observability](observability.md) · [Roadmap & Decisions](roadmap-decisions.md) · [Vision](../roadmap/vision.md)
