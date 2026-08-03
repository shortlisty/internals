# BENE — System Design Documentation

> Pre-build design documentation for the BENE platform — built on the IQ Key Value open-source foundation.

---

## What is BENE

BENE is a collaborative workspace for event agencies. It turns venue files into a structured library and that library into an interactive client pitch — ending with a timestamped, approved specification.

Two layers:
- **Personal Venue Catalog** — private venue library with AI extraction and search
- **Digital Sales Room** — interactive pitch board for client sign-off

See [docs/README.md](docs/README.md) for the full plain-language overview.

---

## Documentation index

### Product & Business

| Document | Audience | What it covers |
| --- | --- | --- |
| [What is BENE?](docs/README.md) | Anyone | Problem, solution, how it works, pricing |
| [Market Structure](docs/business/market.md) | Anyone | Event chain, tool segmentation, the vacant slot |
| [Vision](docs/roadmap/vision.md) | Founders, team | Product direction, strategic bets, north star metric |
| [Digital Sales Room](docs/business/Digital_Sales_Room_for_Events/README.md) | Founders, team | DSR concept — current primary direction |

### Personal Venue Catalog (segment reference)

> [!NOTE]
> These documents describe the catalog subsystem and its original standalone positioning. They remain valid as the data-layer reference but are not the primary product direction.

| Document | What it covers |
| --- | --- |
| [Product Structure](docs/business/Personal_Venue_Catalog/product.md) | Tenant app, capability pillars, UI concept |
| [Business Proposal](docs/business/Personal_Venue_Catalog/proposal.md) | ICP, monetisation, GTM, risks |
| [Competitive Landscape](docs/business/Personal_Venue_Catalog/comparison.md) | Competitor analysis and gap matrix |
| [Cold Start Strategy](docs/business/Personal_Venue_Catalog/cold-start.md) | Seeding the library before launch |
| [Sales materials](docs/business/Personal_Venue_Catalog/sales/) | Pitch, battlecards, objections, messaging |

### Platform

| Document | Audience | What it covers |
| --- | --- | --- |
| [Architecture Reference](docs/platform/architecture.md) | Engineers, architects | Domain model, services, schema, API, event contracts |
| [Intelligence Layer](docs/platform/intelligence.md) | Engineers, architects | ETL pipeline, extraction, AI layer, technology decisions |

Russian translations: [`docs/ru/`](docs/ru/)

---

## Contributing

Read [AGENTS.md](AGENTS.md) before adding or editing any document. It defines repository structure, audience tagging, document types, writing standards, and constraints.

---

## Platform context

BENE is built on top of the IQ Key Value open-source foundation. New services introduced:

- **`bene-venue-service`** — venue profiles, assets, metadata, search, plan enforcement
- **`bene-venue-ingestion-worker`** — async sidecar: document ETL, AI extraction, embeddings
- **`bene-venue-model`** — shared library: domain entities, event contracts, Liquibase migrations

**Stage:** pre-launch MVP — design complete, implementation not yet started.

---

## License

Copyright © 2026 IQ Key Value. All rights reserved.

This software and its documentation are proprietary and confidential. The source code is made available to authorized licensees only. You may not use, copy, modify, distribute, or sublicense this software except as expressly permitted under a written agreement with IQ Key Value.

The underlying IQ Key Value platform is built on open-source components, each governed by their respective licenses.
