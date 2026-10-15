# Tenant venue profiles

> **Audience:** Product, engineering.
> **Status:** Promoted to [E2 Tenant Venue Profiles](../epics/E2-tenant-venue-profiles.md)

---

## Summary

Tenant venue profiles let an event planner create, edit, view, and delete the venues in their agency's private catalog, with core fields and attached documents. Profile creation draws on the master venue catalog for one-click suggestions and deduplication, and plan-tier limits cap how many venues a tenant can create. This is the first tenant-facing capability layered on top of the platform foundation.

## Why it matters

The master catalog is invisible to the planner. The value they feel is "I can add the venues I work with, keep their documents, and find them again." Without tenant venue profiles there is no catalog, no pitch board input, and no product. Plan enforcement at venue creation is what makes tiered pricing real — a Free tenant cannot silently exceed their venue cap.

## Where it could land

- Likely epic: TBD (was E2 in the removed v0.1 milestone; plan enforcement was E6 partial)
- Likely milestone: TBD
- Related decision: [D6 — Schema-per-tenant isolation](../decisions/D6-schema-per-tenant-isolation.md)

## In scope

Scope markers: `[shared-lib]` `[venueintelligence-catalog-service]` `[admin-ui]` `[tenant-ui]` `[infra]`

Tasks are ordered by dependency. Each is sized for 2–3 h including tests and CI/CD to UAT.

### Tenant Venue Profiles

- [ ] `[venueintelligence-catalog-service]` `VenueMapper` (MyBatis): `insert`, `findById`, `findAll` (paginated + filters), `update`, `archive`. `VenueDtos` records. `VenueMetadataTypeHandler` wired in ResultMap. Integration tests against Testcontainers.
- [ ] `[venueintelligence-catalog-service]` `VenueRestResource` (`/api/v1/venues`, `MEMBER`): `POST /` (201, enforces `max_venues` plan limit → 402), `GET /` (200, `VenueSummaryListResponse`), `GET /{id}` (200), `PATCH /{id}` (200), `DELETE /{id}` (204, soft-archive, `TENANT_OWNER`/`ADMIN` only). Integration tests per endpoint.
- [ ] `[venueintelligence-catalog-service]` `POST /venues/from-master-catalog/{masterVenueId}` (`MEMBER`, 201): clone master venue fields into new tenant `venues` row with `MC_INHERIT` provenance in `metadata_sources`. Idempotent — re-promote returns existing row. Integration test.
- [ ] `[tenant-ui]` Venue list page (`/venues`): card grid — name, city, status badge, asset count. Empty state. `MEMBER` access.
- [ ] `[tenant-ui]` Create venue form (`/venues/new`): name, address, city, capacity, category, description. On name input call master venue suggest (debounced `GET /api/v1/master-venues?q=`). One-click promote from suggestion. Submit → `POST /venues`.
- [ ] `[tenant-ui]` Venue detail page (`/venues/:id`): render all core fields + metadata panel. Document list (assets). Edit inline via `PATCH`. Archive via `DELETE`. Status badge.

### Plan Enforcement (partial)

- [ ] `[venueintelligence-catalog-service]` `PlanResolver` (non-reactive, `RestTemplate`, 10-min cache): fetches entitlements from `foundation-billing-service /internal/plans`. `PlanFeatureGuard` checks `max_venues` at `POST /venues`. Returns 402 `ProblemDetail` with `featureCode = max_venues` when limit reached. Integration test with mocked billing response.

## Out of scope

- Document upload and extraction pipeline — deferred to a later increment.
- Semantic and keyword search over venues — deferred.
- Team collaboration (roles other than owner, member invites) — deferred.
- Data quality and provenance UI — deferred.

## Open questions

- [ ] Are soft-archived venues restorable, or is archive permanent in this increment?
- [ ] Does the create-venue form validate against the master catalog before allowing a free-text venue, or is free-text always allowed?

## Status

**Status:** Promoted to [E2 Tenant Venue Profiles](../epics/E2-tenant-venue-profiles.md)

---

**Docs:** [Ideas index](README.md) · [Vision](../vision.md)
