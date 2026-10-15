# Vendor registry

> **Audience:** Product, engineering.
> **Status:** Draft

---

## Summary

Event agencies work with the same trusted suppliers across dozens of events, yet that supplier knowledge lives in personal contacts, spreadsheets, and memory. The vendor registry gives the agency a shared, searchable list of preferred vendors — categorised by service type, annotated with performance notes from past events, and linked to the retrospectives where they appeared. When a planner assembles a new brief or Deal Room, the right vendor recommendation is one search away.

## Why it matters

Agency owners want a shared, up-to-date supplier list with performance ratings. Senior managers want to write performance notes linked to retrospectives so future planners know what to expect. Planners assembling a brief want to find trusted suppliers by category and rating in under a minute. Coordinators want to see which vendors already know a specific venue. Without a registry, supplier knowledge is siloed in personal contacts.

## Where it could land

- Likely epic: post-v1.0 epic (formerly E14)
- Likely milestone: Post-v1.0 milestone (TBD)
- Related decision: [D11 — Knowledge base scope expansion](../decisions/D11-knowledge-base-scope-expansion.md)

## In scope

- Vendor record: name, service category (Catering, AV, Décor, Photography, Videography, Transport, Entertainment, Other), contact name, email, phone, website, overall rating (1–5), status (Preferred / On hold / Inactive).
- Performance notes per vendor: free text, date, author, optional link to a retrospective record.
- Link vendor records to venue records: "vendors used at this venue" association, visible on the venue profile.
- Registry view: list with category filter, status filter, rating sort, keyword search.
- Role gate: owner and editor can create, edit, or change vendor status; viewer role is read-only.
- Vendor count is subject to plan-tier limits (specific caps to be defined).

## Out of scope

- Cross-tenant vendor sharing (entirely separate product direction).
- Automated vendor suggestions based on brief type or event parameters (AI recommendation layer — post-v1.1).
- Contract or invoice management for vendors (out of Shortlisty's scope per vision.md).
- Calendar availability or booking integration for vendors.
- Client-visible vendor recommendations (vendor registry is internal).

## Open questions

- [ ] Should vendor records surface inside Deal Rooms as an optional "recommended suppliers" section visible to the client? This would cross the internal/external boundary — needs explicit product decision.
- [x] Is vendor data scoped per tenant or shared at platform level? Decision: per tenant (private to each agency) for v1.1. Platform-level vendor catalog is post-v1.0 scope per D11.
- [ ] Should the vendor-venue link be many-to-many (one vendor can be linked to many venues, one venue can have many vendors)? Assumption is yes — confirm before schema design.

## Status

**Status:** Draft — demoted from E14 epic pending v1.0 usage signal.

---

**Docs:** [Ideas index](README.md) · [Vision](../vision.md)
