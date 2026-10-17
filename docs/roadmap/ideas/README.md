# Ideas

> **Audience:** Product, engineering.
> **Purpose:** Index of speculative product and architecture pitches that outgrew the backlog-candidates table but are not yet formalised as epics, milestones, or decisions.

---

## What this folder is

An idea is a worked-through proposal — summary, why it matters, where it could land, open questions. It is not a commitment. Ideas sit between the one-line backlog-candidates table (max 10 rows) and the heavier epic / milestone / decision anatomy.

An idea becomes an epic, milestone, or decision when it is promoted. The original idea file stays in place with a `Promoted` status and a link to what it became.

## Lifecycle

```
backlog candidates (1-line, ≤10)
        │
        ▼
ideas/{slug}.md  (worked pitch, status: Draft)
        │
        ├─► Promoted to E{N} / v{X}.{Y} / D{N}
        │       (file stays, status updated, link added)
        │
        └─► Retired (one-line note on why)
```

## Index

| File                                                       | Title                 | Likely landing                                                                                                           | Status         | Notes                                                                                         |
| ---------------------------------------------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------ | -------------- | --------------------------------------------------------------------------------------------- |
| [00-master-venue-catalog.md](00-master-venue-catalog.md)   | Master venue catalog  | [E1 Master Venue Catalog](../epics/E1-master-venue-catalog.md) / [v0.1](../milestones/v0.1-foundation-catalog-core.md)   | Promoted to E1 | Carried from the removed v0.1-mvp milestone. Platform-wide reference dataset + admin tooling. |
| [00-tenant-venue-profiles.md](00-tenant-venue-profiles.md) | Tenant venue profiles | [E2 Tenant Venue Profiles](../epics/E2-tenant-venue-profiles.md) / [v0.1](../milestones/v0.1-foundation-catalog-core.md) | Promoted to E2 | Carried from the removed v0.1-mvp milestone. Tenant-facing venue CRUD + document upload.      |
| [sop-library.md](sop-library.md)                           | SOP library           | Post-v1.0 epic (was E11) / v1.1                                                                                          | Draft          | Demoted from E11 epic to idea pending v1.0 usage signal.                                      |
| [event-retrospectives.md](event-retrospectives.md)         | Event retrospectives  | Post-v1.0 epic (was E12) / v1.1                                                                                          | Draft          | Demoted from E12 epic to idea pending v1.0 usage signal.                                      |
| [template-library.md](template-library.md)                 | Template library      | Post-v1.0 epic (was E13) / v1.1                                                                                          | Draft          | Demoted from E13 epic to idea pending v1.0 usage signal.                                      |
| [vendor-registry.md](vendor-registry.md)                   | Vendor registry       | Post-v1.0 epic (was E14) / v1.1                                                                                          | Draft          | Demoted from E14 epic to idea pending v1.0 usage signal.                                      |

---

**Docs:** [Roadmap vision](../vision.md) · [Epics](../epics/README.md) · [Milestones](../milestones/README.md) · [Decisions](../decisions/README.md) · [Feature checklist](../feature-checklist.md)
