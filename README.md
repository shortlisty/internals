<div align="center">

# Shortlisty

**Venue files in. Client sign-off out.**

A collaborative workspace for event agencies: scattered venue documents become a structured private library, and that library becomes an interactive client pitch that ends with a timestamped, approved specification.

<sub>Built on the iQ Key Value open-source foundation · Proprietary and confidential · © 2026 iQ Key Value</sub>

</div>

---

> **Audience:** Anyone.
> **Purpose:** Landing page and navigation index for the Shortlisty system-design documentation. Read the summary, check the status, then follow the index into the layer you need.

> [!TIP]
> New here? Start with [What is Shortlisty?](docs/README.md) for the plain-language overview, then [Vision](docs/roadmap/vision.md) for where the product is going.

## Status

| Field       | Value                                |
| ----------- | ------------------------------------ |
| Stage       | v0.1 MVP in progress                 |
| Shipped     | Group A Platform Foundation complete |
| In progress | Master Venue Seeding Infrastructure  |
| Next        | Tenant venue features                |

> [!NOTE]
> This repository holds design documentation, not source code. The platform itself is built in separate source repositories.

## Two layers

Two products that only work together. The catalog feeds the pitch; the pitch produces the approval.

| Layer                      | What it is                                                                                 | Documented in                                                               |
| -------------------------- | ------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------- |
| **Personal Venue Catalog** | The agency's private venue library. Documents in, structured profiles out. The data layer. | [Product Structure](docs/business/personal-venue-catalog/product.md)        |
| **Digital Sales Room**     | The client-facing pitch board that ends in approval. The output layer.                     | [Product Structure](docs/business/digital-sales-room-for-events/product.md) |

## Documentation index

### Product, business and strategy

| Document                                                     | Audience       | What it covers                                               |
| ------------------------------------------------------------ | -------------- | ------------------------------------------------------------ |
| [What is Shortlisty?](docs/README.md)                        | Anyone         | Problem, solution, how it works, pricing                     |
| [Market Structure](docs/business/market.md)                  | Anyone         | Event chain, tool segmentation, the vacant slot              |
| [Competitive Landscape](docs/business/comparison.md)         | Founders, team | Both product angles: category tables and capability matrices |
| [AlphaVenue Spike](docs/business/spikes/alphavenue-spike.md) | Founders, team | Reference deep dive on a venue-side AI operations platform   |

### Digital Sales Room for Events — the output layer

| Document                                                                          | Audience       | What it covers                                               |
| --------------------------------------------------------------------------------- | -------------- | ------------------------------------------------------------ |
| [Overview](docs/business/digital-sales-room-for-events/README.md)                 | Founders, team | DSR concept, relationship to the catalog layer               |
| [Product Structure](docs/business/digital-sales-room-for-events/product.md)       | Founders, team | Two-layer architecture, capability pillars, UX concept       |
| [Business Proposal](docs/business/digital-sales-room-for-events/proposal.md)      | Founders, team | ICP, pricing, monetisation, go-to-market, risks              |
| [Pitch Mechanics](docs/business/digital-sales-room-for-events/pitch-mechanics.md) | Founders, team | Micro-site structure, collaboration layer, approval snapshot |
| [Cold Start Strategy](docs/business/digital-sales-room-for-events/cold-start.md)  | Founders, team | Seed catalog, concierge onboarding, city expansion           |

### Personal Venue Catalog — the data layer

> [!NOTE]
> These documents describe the catalog subsystem and its original standalone positioning. They remain valid as the data-layer reference but are not the primary product direction.

| Document                                                                                       | Audience       | What it covers                                               |
| ---------------------------------------------------------------------------------------------- | -------------- | ------------------------------------------------------------ |
| [Product Structure](docs/business/personal-venue-catalog/product.md)                           | Founders, team | Tenant app, capability pillars, UI concept                   |
| [Business Proposal](docs/business/personal-venue-catalog/proposal.md)                          | Founders, team | ICP, monetisation, go-to-market, risks                       |
| [Cold Start Strategy](docs/business/personal-venue-catalog/cold-start.md)                      | Founders, team | Seeding the library before launch                            |
| [Superseded sales materials](docs/business/personal-venue-catalog/superseded-sales-materials/) | Founders, team | Pitch, battlecards, objections, messaging (kept for history) |

### Platform — engineering reference

| Document                                               | Audience              | What it covers                                           |
| ------------------------------------------------------ | --------------------- | -------------------------------------------------------- |
| [Architecture Reference](docs/platform/README.md)      | Engineers, architects | Hub: links every platform document listed here and below |
| [Intelligence Layer](docs/platform/intelligence.md)    | Engineers, architects | ETL pipeline, extraction, AI layer, tech decisions       |
| [Service Architecture](docs/platform/services.md)      | Engineers, architects | Service decomposition, table ownership, S3 layout        |
| [Domain Model and Schema](docs/platform/data-model.md) | Engineers, architects | Bounded contexts, canonical field set, database schema   |
| [Master Catalog](docs/platform/master-catalog.md)      | Engineers, product    | Cold start, alias normalisation, venue matching          |
| [API Surface](docs/platform/api.md)                    | Engineers             | REST endpoints, DTOs, plan gates                         |
| [Event Contracts](docs/platform/events.md)             | Engineers             | RabbitMQ routing keys, queues, plan entitlements         |

The Architecture Reference is the hub for all platform documents, including search, aggregation, observability, and the UI specifications.

### Roadmap

| Document                                               | Audience              | What it covers                                       |
| ------------------------------------------------------ | --------------------- | ---------------------------------------------------- |
| [Vision](docs/roadmap/vision.md)                       | Founders, team        | Product direction, strategic bets, north star metric |
| [Feature Checklist](docs/roadmap/feature-checklist.md) | Product, engineering  | Prioritised feature list (P0–P3)                     |
| [Epics](docs/roadmap/epics/README.md)                  | Product, engineering  | User-facing capability clusters                      |
| [Milestones](docs/roadmap/milestones/README.md)        | Product, engineering  | Versioned shippable increments                       |
| [Decisions](docs/roadmap/decisions/README.md)          | Engineers, architects | Architecture decision records (D1–D13)               |
| [Ideas](docs/roadmap/ideas/README.md)                  | Product, engineering  | Worked pitches not yet promoted to epics             |

## Platform at a glance

Shortlisty builds on the iQ Key Value open-source foundation (`com.iqkv.foundation`). New components live under `com.iqkv.docintelligence`.

| Component                                    | Type         | Role                                                                                     |
| -------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------- |
| `docintelligence-catalog-service`            | Java service | Single REST API for all venue management, tenant and platform admin                      |
| `docintelligence-catalog-processing-worker`  | Java service | Async ETL worker: document processing, self-hosted LLM inference, master catalog mapping |
| `docintelligence-mc-ingest-tagvenue-scraper` | Node CLI     | Nightly cron scraper; writes master catalog JSONL to S3                                  |
| `docintelligence-master-venue-loader`        | Java service | Loads JSONL from S3 into the master catalog tables                                       |

All services share a single PostgreSQL instance: the `public` schema holds the master catalog, and `t_{tenantKey}` schemas hold tenant data. Database decomposition is a named future target once scale justifies it (see [D13](docs/roadmap/decisions/D13-shared-database-per-product.md)).

Full detail: [Architecture Reference](docs/platform/README.md) · [Service Architecture](docs/platform/services.md).

## Contributing

Read [AGENTS.md](AGENTS.md) before adding or editing any document. It defines the repository structure, audience tagging, document types, writing standards, and constraints. Significant documentation changes are recorded in [CHANGELOG.md](CHANGELOG.md).

## License

Copyright © 2026 iQ Key Value. All rights reserved.

This software and its documentation are proprietary and confidential. The source code is made available to authorized licensees only. You may not use, copy, modify, distribute, or sublicense this software except as expressly permitted under a written agreement with iQ Key Value.

The underlying iQ Key Value platform is built on open-source components, each governed by their respective licenses.
