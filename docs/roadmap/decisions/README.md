# Decisions

> **Audience:** Engineers, architects.
> **Purpose:** One file per architectural or strategic decision. Records the context, alternatives, and rationale so the same ground is never re-covered.

---

## What is a decision record

A decision record (DR) captures a choice that was made, the alternatives that were considered, and why the chosen option was selected. Once a decision is `Accepted`, it is not re-debated in other documents — those documents link here instead.

Each decision file follows the template in [AGENTS.md § 4.6](../../../AGENTS.md#46-decisions-docsroadmapdecisions).

---

## Decisions index

| ID  | File                                                                                                 | Title                                                                                                                 | Domain                     | Status   | Superseded by |
| --- | ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- | -------------------------- | -------- | ------------- |
| D19 | [D19-shared-database-per-product.md](D19-shared-database-per-product.md)                             | Shared database per product: one PostgreSQL instance for all venueintelligence services                               | Infrastructure / data      | Accepted | —             |
| D18 | [D18-self-hosted-inference-stack.md](D18-self-hosted-inference-stack.md)                             | Self-hosted inference stack: open-source models (Phi-4, Qwen) on own infrastructure as primary AI layer               | Infrastructure / AI        | Accepted | —             |
| D17 | [D17-knowledge-base-scope-expansion.md](D17-knowledge-base-scope-expansion.md)                       | Agency Knowledge Base: SOPs, retrospectives, templates, and vendor registry as post-v1.0 scope                        | Product scope              | Accepted | —             |
| D16 | [D16-deal-room-trust-model.md](D16-deal-room-trust-model.md)                                         | Deal Room trust model: immutable history, bilateral control, confidence-as-transparency                               | Product / architecture     | Accepted | —             |
| D15 | [D15-progressive-enrichment.md](D15-progressive-enrichment.md)                                       | Progressive profile enrichment: beauty first, accuracy in background                                                  | ETL / UX                   | Accepted | —             |
| D14 | [D14-master-venue-seeding-infrastructure-first.md](D14-master-venue-seeding-infrastructure-first.md) | Master venue seeding infrastructure first                                                                             | Data / infrastructure      | Accepted | —             |
| D8  | [D8-vertical-extension-strategy.md](D8-vertical-extension-strategy.md)                               | Strategy pattern: generic core + domain library swap                                                                  | Architecture               | Accepted | —             |
| D7  | [D7-jsonb-schema-versioning.md](D7-jsonb-schema-versioning.md)                                       | `_schema_version` + online incremental JSONB migration                                                                | Data / schema              | Accepted | —             |
| D6  | [D6-schema-per-tenant-isolation.md](D6-schema-per-tenant-isolation.md)                               | Schema-per-tenant isolation vs. tenant_id column                                                                      | Architecture / tenancy     | Accepted | —             |
| D5  | [D5-metadata-aggregation-fifo.md](D5-metadata-aggregation-fifo.md)                                   | RabbitMQ FIFO routing per venue for aggregation concurrency                                                           | Infrastructure / messaging | Accepted | —             |
| D4  | [D4-cross-source-search-merge.md](D4-cross-source-search-merge.md)                                   | Parallel queries + app-level RRF merge for tenant venues + master venue catalog (invisible backdrop merge by default) | Search / architecture      | Accepted | —             |
| D3  | [D3-pgvector-vs-dedicated-store.md](D3-pgvector-vs-dedicated-store.md)                               | pgvector vs. dedicated vector store                                                                                   | Infrastructure / search    | Accepted | —             |
| D2  | [D2-tika-vs-docling-phase1.md](D2-tika-vs-docling-phase1.md)                                         | Tika-only for Phase 1                                                                                                 | ETL / extraction           | Accepted | —             |
| D1  | [D1-one-service-vs-two.md](D1-one-service-vs-two.md)                                                 | One service vs. two                                                                                                   | Architecture               | Accepted | —             |

---

**Docs:** [Vision](../vision.md) · [Epics](../epics/README.md) · [Milestones](../milestones/README.md)
