# iQ BENE — System Design Documentation

> Pre-build design documentation for the Venue Intelligence Platform — built on the IQKV open-source foundation.

---

## Documents

| Document                                                | Audience              | What it covers                                                   |
| ------------------------------------------------------- | --------------------- | ---------------------------------------------------------------- |
| [What is iQ BENE?](docs/README.md)                      | Anyone                | Plain-language overview: the problem, the solution, who it's for |
| [Business Proposal](docs/business/proposal.md)          | Founders, team        | ICP, feature roadmap, monetization, GTM, risks                   |
| [Competitive Landscape](docs/business/comparison.md)    | Founders, team        | Competitor analysis and gap summary matrix                       |
| [Pitch](docs/business/sales/pitch.md)                   | Founders, team        | Intro call and demo narrative flow                               |
| [Battlecards](docs/business/sales/battlecards.md)       | Founders, team        | Per-competitor positioning for live conversations                |
| [Objections](docs/business/sales/objections.md)         | Founders, team        | Objection handling: underlying concern, response, follow-up      |
| [Messaging](docs/business/sales/messaging.md)           | Founders, team        | Taglines, hero copy, value pillars, naming — copy bank           |
| [Architecture Reference](docs/platform/architecture.md) | Engineers, architects | Domain model, services, schema, API, event contracts             |
| [Intelligence Layer](docs/platform/intelligence.md)     | Engineers, architects | ETL pipeline, extraction, AI layer, technology decisions         |
| [Roadmap Vision](docs/roadmap/vision.md)                | Founders, team        | Product direction, strategic bets, north star metric             |

Russian translations: [`docs/ru/`](docs/ru/)

---

## Contributing to this documentation

Read [AGENTS.md](AGENTS.md) before adding or editing any document. It defines the repository structure, audience tagging, document type rules, writing standards, and what agents and contributors must not do.

---

## Platform context

iQ BENE is a new product built on top of the IQKV open-source foundation. It introduces two services and one shared library:

- **`iqbene-venue-service`** — core domain: venue profiles, assets, metadata, search, plan enforcement
- **`iqbene-venue-ingestion-worker`** — async sidecar: document ETL, AI extraction, embeddings, scheduled jobs
- **`iqbene-venue-model`** — shared library: domain entities, event contracts, Liquibase migrations

**Stage:** pre-launch MVP — design complete, implementation not yet started.
Open decisions: [Architecture §15](docs/platform/architecture.md#15-open-decisions-resolve-before-sprint-1).

---

## License

Copyright © 2026 IQKV. All rights reserved.

This software and its documentation are proprietary and confidential. The source code is made available to authorized licensees only. You may not use, copy, modify, distribute, or sublicense this software except as expressly permitted under a written agreement with IQKV.

The underlying IQKV platform is built on open-source components, each governed by their respective licenses.
