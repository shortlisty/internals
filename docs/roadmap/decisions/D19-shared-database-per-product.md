# D19 — Shared database per product: one PostgreSQL instance for all venueintelligence services

> **Audience:** Engineers, architects.
> **Purpose:** Record the decision to use a single shared PostgreSQL instance for all venueintelligence services rather than a separate database per service, and to defer database decomposition to a future milestone.

---

## Context

The canonical microservices pattern prescribes a separate database per service — each service owns its persistence store exclusively and other services never connect to it directly. This eliminates cross-service coupling at the data layer and allows independent scaling, technology choice, and deployment.

For venueintelligence the two primary runtime services — `venueintelligence-catalog-service` and `venueintelligence-catalog-processing-worker` — work on tightly coupled data. The processing worker reads `venue_assets` written by the catalog service, writes extraction results that the catalog service reads back for aggregation, and both services must observe the same tenant schema boundaries. Enforcing database-per-service at this stage would require a cross-service API or an event-carried state pattern for every data access that currently happens as a direct SQL read — adding complexity with no proportional benefit at the current scale.

Additionally, the multi-tenant schema-per-tenant isolation model (D6) is implemented by `foundation-tenancy`'s `MyBatisSchemaInterceptor` routing connections to `t_{tenantKey}` schemas within a single PostgreSQL instance. Splitting that across database instances would require replicating the schema-routing mechanism and tenant provisioning logic across all services.

---

## Options considered

### Option A — Database per service (strict microservices)

Each service gets its own PostgreSQL instance or database. Cross-service data access goes through APIs or events.

**Pros:** Full isolation. Independent schema evolution. Standard microservices pattern.

**Cons:** Requires cross-service API calls or event-carried state for data that is naturally co-located (e.g. processing worker reading `venue_assets`). Duplicates tenant schema provisioning logic. Significant operational overhead for a two-service product at pre-PMF stage. Premature optimisation: the scale that justifies this complexity does not exist yet.

### Option B — Single shared PostgreSQL instance, logical ownership enforced by convention (chosen)

All venueintelligence services share one PostgreSQL instance. Tenant isolation is enforced by schema-per-tenant (D6). Table ownership and cross-boundary access rules are enforced by convention and code review (see [services.md § Table Ownership](services.md)). Database decomposition is deferred until the product reaches a scale or operational constraint that justifies it.

**Pros:** Simple operations — one database to provision, back up, and monitor. No cross-service API overhead for co-located data. Schema-per-tenant isolation provided by `foundation-tenancy` without modification. Fast iteration. Consistent with the foundation platform's existing single-instance model.

**Cons:** Violates the database-per-service principle. A bug or runaway query in one service can affect another. Schema coupling is a migration coordination cost as the product grows.

---

## Decision

**Option B.** Single shared PostgreSQL instance for all venueintelligence services for v0.x through v1.0.

Structure within the shared instance:

- `public` schema — platform-wide master catalog tables (`master_venue`, `master_venue_alias`, `master_venue_external`). Read by all services; written only by `venueintelligence-master-venue-loader`.
- `t_{tenantKey}` schemas — per-tenant tables (venues, assets, extraction jobs, vectors, metadata events). Provisioned by `foundation-tenancy` on tenant creation. Accessed via `MyBatisSchemaInterceptor`.

Future split targets when they become relevant:

- `pitch_db` — Deal Room, pitch boards, approval snapshots (Layer 2, DSR). Natural split point: when the DSR feature set is large enough to benefit from independent deployment and schema evolution.
- `catalog_db` — venue catalog tables separated from pitch tables. Only relevant when read/write patterns diverge significantly.

---

## Rationale

At pre-PMF stage with two services sharing tightly coupled data, the operational simplicity of a shared database outweighs the isolation benefits of database-per-service. The table ownership rules in services.md provide logical isolation without physical separation. When scale or operational pressure makes the shared instance a bottleneck, the decomposition path is clear and the split can be made without changing application logic — only connection configuration and schema migration tooling.

The `venueintelligence-catalog-processing-worker` and `venueintelligence-catalog-service` sharing a database is an intentional, documented decision — not a shortcut. Both services are co-deployed, co-owned, and co-versioned. The processing worker is a sidecar to the catalog service, not an independent domain boundary.

---

## Consequences

- `services.md` documents the shared database explicitly as a deliberate architectural choice, not an exception to a rule.
- Table ownership rules (services.md § Table Ownership) remain the enforcement mechanism for logical isolation within the shared instance.
- `pitch_db` and `catalog_db` splits are documented as named future targets in the milestones backlog, not as immediate concerns.
- Any new service added to the venueintelligence product must explicitly decide whether to share this database or own a separate one, and record that decision here or in a new ADR.

---

## Status

**Status:** Accepted

---

**Docs:** [Decisions index](README.md) · [Services](../../platform/services.md) · [D6 schema-per-tenant isolation](D6-schema-per-tenant-isolation.md) · [Vision](../vision.md)
