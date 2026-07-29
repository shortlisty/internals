# iQ BENE — System Design Documentation

> Pre-build design documentation for the Venue Intelligence Platform — built on the IQKV open-source foundation.

---

## Documents

| Document                                                                | Audience                   | What it covers                                                   |
| ----------------------------------------------------------------------- | -------------------------- | ---------------------------------------------------------------- |
| [What is iQ BENE?](docs/overview.md)                                    | Anyone                     | Plain-language overview: the problem, the solution, who it's for |
| [Business Overview](docs/business-overview.md)                          | Founders, team             | ICP, feature roadmap, monetization, GTM, risks                   |
| [Competitive Landscape](docs/intelligence-and-competitive-landscape.md) | Engineering, founding team | Competitor analysis, ETL pipeline, intelligence layer            |
| [Architecture Reference](docs/architecture.md)                          | Engineers, architects      | Domain model, services, schema, API, event contracts             |

Russian translations: [`docs/ru/`](docs/ru/)

---

## Platform Context

iQ BENE is a new product built on top of the IQKV open-source foundation. It introduces two services and one shared library:

- **`iqbene-venue-service`** — core domain: venue profiles, assets, metadata, search, plan enforcement
- **`iqbene-venue-ingestion-worker`** — async sidecar: document ETL, AI extraction, embeddings, scheduled jobs
- **`iqbene-venue-model`** — shared library: domain entities, event contracts, Liquibase migrations

**Stage:** pre-launch MVP — design complete, implementation not yet started.
Open decisions: [Architecture §15](docs/architecture.md#15-open-decisions-resolve-before-sprint-1).

---

## License

Copyright © 2026 IQKV. All rights reserved.

This software and its documentation are proprietary and confidential. The source code is made available to authorized licensees only. You may not use, copy, modify, distribute, or sublicense this software except as expressly permitted under a written agreement with IQKV.

The underlying IQKV platform is built on open-source components, each governed by their respective licenses.
