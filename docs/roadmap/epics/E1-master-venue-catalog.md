# E1 — Master Venue Catalog

> **Audience:** Product, engineering.
> **Purpose:** Platform foundation epic — the shared master venue catalog that all tenant catalogs build on. No direct end-user value on its own; everything else depends on it.

---

## Summary

The master venue catalog is the platform-wide reference library of venues. It provides the schema, seed data, and admin API that tenant venue creation depends on for autocomplete suggestions, gap-fill, and deduplication. After this epic ships, the platform can store and manage a canonical set of venues that tenant catalogs promote from.

---

## User stories

- As a platform admin, I want to create, edit, and delete master venue entries so that the reference library stays accurate.
- As a platform admin, I want to seed the catalog with a curated venue set so that tenant autocomplete has real data from day one.
- As a planner creating a tenant venue, I want autocomplete suggestions from the master catalog so that I can promote an existing venue instead of re-typing it.

---

## In scope

- Shared platform libraries: common types, venue schema contracts, shared database access
- Master venue database schema (core fields: name, address, city, capacity, category, summary)
- Seed data: a curated set of venues loaded into the master catalog
- Admin API: create, read, update, delete master venue entries
- Platform admin UI for master venue entry management (list, create, view, edit, delete)
- S3 bucket/folder structure for venue document and asset storage
- Deduplication infrastructure: data model and detection primitives supporting tenant venue gap-fill and duplicate detection (no admin-UI merge workflow in v0.1)

---

## Out of scope

- Platform admin UI for merging duplicate master venues or managing aliases (deferred)
- Tenant-facing venue creation UI (lives in E2)
- Document upload and extraction (lives in E3)
- Semantic search and vector embeddings (v0.2+)
- Multi-tenant data isolation enforcement (covered by platform tenancy, not this epic)

---

## Milestone references

- [v0.1 — Foundation + Catalog Core](../milestones/v0.1-foundation-catalog-core.md)

---

## Open questions

- [ ] Does v0.1 ship with dedup infrastructure only (no merge UI), or is the merge workflow also in scope? Current assumption: infrastructure only; merge UI deferred.
- [x] What is the seed venue source? Curated manual seed set for v0.1; concierge onboarding expands it later.

---

## Status

**Status:** Not started

---

**Docs:** [Epics index](README.md) · [v0.1 milestone](../milestones/v0.1-foundation-catalog-core.md) · [Platform architecture](../../platform/README.md) · [Decisions](../decisions/README.md)
