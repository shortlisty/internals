# Shortlisty — System Design Documentation

> Pre-build design documentation for the Shortlisty platform — built on the iQ Key Value open-source foundation.

---

## What is Shortlisty

Shortlisty is a collaborative workspace for event agencies. It turns venue files into a structured library and that library into an interactive client pitch — ending with a timestamped, approved specification.

Two layers:

- **Personal Venue Catalog** — private venue library with AI extraction and search
- **Digital Sales Room** — interactive pitch board for client sign-off

See [docs/README.md](docs/README.md) for the full plain-language overview.

---

## Documentation index

### Product & Business

| Document                                    | Audience       | What it covers                                       |
| ------------------------------------------- | -------------- | ---------------------------------------------------- |
| [What is Shortlisty?](docs/README.md)          | Anyone         | Problem, solution, how it works, pricing             |
| [Market Structure](docs/business/market.md) | Anyone         | Event chain, tool segmentation, the vacant slot      |
| [Vision](docs/roadmap/vision.md)            | Founders, team | Product direction, strategic bets, north star metric |

### Digital Sales Room for Events

| Document                                                                           | Audience       | What it covers                                                 |
| ---------------------------------------------------------------------------------- | -------------- | -------------------------------------------------------------- |
| [Overview](docs/business/digital-sales-room-for-events/README.md)                  | Founders, team | DSR concept, relationship to catalog layer                     |
| [Product Structure](docs/business/digital-sales-room-for-events/product.md)        | Founders, team | Two-layer architecture, capability pillars, UX concept         |
| [Business Proposal](docs/business/digital-sales-room-for-events/proposal.md)       | Founders, team | ICP, feature phases, pricing, GTM, risks                       |
| [Pitch Mechanics](docs/business/digital-sales-room-for-events/pitch-mechanics.md)  | Founders, team | Micro-site structure, collaboration layer, approval snapshot   |
| [Cold Start Strategy](docs/business/digital-sales-room-for-events/cold-start.md)   | Founders, team | Seed catalog, concierge onboarding, city-by-city expansion     |
| [Competitive Landscape](docs/business/digital-sales-room-for-events/comparison.md) | Founders, team | DSR vs. proposal tools, venue discovery, agency CRM, DIY stack |

### Personal Venue Catalog (segment reference)

> [!NOTE]
> These documents describe the catalog subsystem and its original standalone positioning. They remain valid as the data-layer reference but are not the primary product direction.

| Document                                                                    | What it covers                             |
| --------------------------------------------------------------------------- | ------------------------------------------ |
| [Product Structure](docs/business/personal-venue-catalog/product.md)        | Tenant app, capability pillars, UI concept |
| [Business Proposal](docs/business/personal-venue-catalog/proposal.md)       | ICP, monetisation, GTM, risks              |
| [Competitive Landscape](docs/business/personal-venue-catalog/comparison.md) | Competitor analysis and gap matrix         |
| [Cold Start Strategy](docs/business/personal-venue-catalog/cold-start.md)   | Seeding the library before launch          |
| [Sales materials](docs/business/personal-venue-catalog/sales/)              | Pitch, battlecards, objections, messaging  |

### Platform

| Document                                            | Audience              | What it covers                                           |
| --------------------------------------------------- | --------------------- | -------------------------------------------------------- |
| [Architecture Reference](docs/platform/README.md)   | Engineers, architects | Domain model, services, schema, API, event contracts     |
| [Intelligence Layer](docs/platform/intelligence.md) | Engineers, architects | ETL pipeline, extraction, AI layer, technology decisions |

---

## Contributing

Read [AGENTS.md](AGENTS.md) before adding or editing any document. It defines repository structure, audience tagging, document types, writing standards, and constraints.

---

## Platform context

Shortlisty is built on top of the iQ Key Value open-source foundation. New services introduced:

- **`mi-data-intelligence`** — domain-agnostic shared library (ETL contracts, provenance, vectors, cost tracking).
- **`mi-venue-service`** — venue profiles, assets, metadata, search, plan enforcement
- **`mi-venue-processing-worker`** — async sidecar: document ETL, AI extraction, embeddings
- **`mi-venue-model`** — shared library: domain entities, event contracts, Liquibase migrations
- **`mc-ingest-tagvenue-scraper`** and **`mi-mc-loader`** — master catalog ingestion

**Stage:** v0.1 MVP in progress — Group A Platform Foundation complete, Master Venue Seeding Infrastructure in development, tenant venue features next.

---

## License

Copyright © 2026 iQ Key Value. All rights reserved.

This software and its documentation are proprietary and confidential. The source code is made available to authorized licensees only. You may not use, copy, modify, distribute, or sublicense this software except as expressly permitted under a written agreement with iQ Key Value.

The underlying iQ Key Value platform is built on open-source components, each governed by their respective licenses.
