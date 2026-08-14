# Decisions

> **Audience:** Engineers, architects.
> **Purpose:** One file per architectural or strategic decision. Records the context, alternatives, and rationale so the same ground is never re-covered.

---

## What is a decision record

A decision record (DR) captures a choice that was made, the alternatives that were considered, and why the chosen option was selected. Once a decision is `Accepted`, it is not re-debated in other documents — those documents link here instead.

Each decision file follows the template in [AGENTS.md § 4.6](../../../AGENTS.md#46-decisions-docsroadmapdecisions).

---

## Decisions index

| File                                                                                                 | Title                                                                                                                 | Status   |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- | -------- |
| [D1-one-service-vs-two.md](D1-one-service-vs-two.md)                                                 | One service vs. two                                                                                                   | Accepted |
| [D2-tika-vs-docling-phase1.md](D2-tika-vs-docling-phase1.md)                                         | Tika-only for Phase 1                                                                                                 | Accepted |
| [D3-pgvector-vs-dedicated-store.md](D3-pgvector-vs-dedicated-store.md)                               | pgvector vs. dedicated vector store                                                                                   | Accepted |
| [D4-cross-source-search-merge.md](D4-cross-source-search-merge.md)                                   | Parallel queries + app-level RRF merge for tenant venues + master venue catalog (invisible backdrop merge by default) | Accepted |
| [D5-metadata-aggregation-fifo.md](D5-metadata-aggregation-fifo.md)                                   | RabbitMQ FIFO routing per venue for aggregation concurrency                                                           | Accepted |
| [D6-schema-per-tenant-isolation.md](D6-schema-per-tenant-isolation.md)                               | Schema-per-tenant isolation vs. tenant_id column                                                                      | Accepted |
| [D7-jsonb-schema-versioning.md](D7-jsonb-schema-versioning.md)                                       | `_schema_version` + online incremental JSONB migration                                                                | Accepted |
| [D8-vertical-extension-strategy.md](D8-vertical-extension-strategy.md)                               | Strategy pattern: generic core + domain library swap                                                                  | Accepted |
| [D14-master-venue-seeding-infrastructure-first.md](D14-master-venue-seeding-infrastructure-first.md) | Master venue seeding infrastructure first                                                                             | Accepted |
| [D15-progressive-enrichment.md](D15-progressive-enrichment.md)                                       | Progressive profile enrichment: beauty first, accuracy in background                                                  | Accepted |
| [D16-deal-room-trust-model.md](D16-deal-room-trust-model.md)                                         | Deal Room trust model: immutable history, bilateral control, confidence-as-transparency                               | Accepted |

---

**Docs:** [Vision](../vision.md) · [Epics](../epics/README.md) · [Milestones](../milestones/README.md)
