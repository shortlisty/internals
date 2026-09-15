# E14 — Vendor registry

> **Audience:** Product, engineering.
> **Purpose:** Define the scope and user outcomes for the preferred vendor registry — the agency's curated list of trusted suppliers (caterers, AV, décor, photographers, transport) with performance notes drawn from past events.

---

## Summary

Event agencies work with the same trusted suppliers across dozens of events, yet that supplier knowledge lives in personal contacts, spreadsheets, and memory. The vendor registry gives the agency a shared, searchable list of preferred vendors — categorised by service type, annotated with performance notes from past events, and linked to the retrospectives where they appeared. When a planner assembles a new brief or Deal Room, the right vendor recommendation is one search away.

---

## User stories

- As an agency owner, I want to add a vendor with their name, service category, contact details, and a performance rating so that my team has a shared, up-to-date supplier list.
- As a senior event manager, I want to write a performance note on a vendor after an event and link it to the relevant retrospective so that future planners know what to expect before they book.
- As a planner assembling a brief, I want to search the vendor registry by service category and filter by rating so that I can recommend trusted suppliers to my client in under a minute.
- As a coordinator, I want to see which vendors have been used at a specific venue so that I can recommend suppliers who already know the space.
- As an agency owner, I want to mark a vendor as preferred, on-hold, or inactive so that the team knows immediately which suppliers are currently recommended and which should be avoided.

---

## In scope

- Vendor record: name, service category (Catering, AV, Décor, Photography, Videography, Transport, Entertainment, Other), contact name, email, phone, website, overall rating (1–5), status (Preferred / On hold / Inactive).
- Performance notes per vendor: free text, date, author, optional link to a retrospective record (E12).
- Link vendor records to venue records: "vendors used at this venue" association, visible on the venue profile.
- Registry view: list with category filter, status filter, rating sort, keyword search.
- Role gate: owner and editor can create, edit, or change vendor status; viewer role is read-only.
- Vendor count is subject to plan-tier limits (tie to E6 plan enforcement — specific caps to be defined).

---

## Out of scope

- Public vendor marketplace or cross-tenant vendor sharing (entirely separate product direction — see E10 marketplace).
- Automated vendor suggestions based on brief type or event parameters (AI recommendation layer — post-v1.1).
- Contract or invoice management for vendors (out of Shortlisty's scope per vision.md).
- Calendar availability or booking integration for vendors.
- Client-visible vendor recommendations (vendor registry is internal).

---

## Milestone references

- [v1.1 — Agency knowledge base](../milestones/v1.1-agency-knowledge-base.md)

---

## Open questions

- [ ] Should vendor records surface inside Deal Rooms as an optional "recommended suppliers" section visible to the client? This would cross the internal/external boundary — needs explicit product decision.
- [x] Is vendor data scoped per tenant or shared at platform level? Decision: per tenant (private to each agency) for v1.1. Platform-level vendor catalog is post-v1.0 scope per D17.
- [ ] Should the vendor-venue link be many-to-many (one vendor can be linked to many venues, one venue can have many vendors)? Assumption is yes — confirm before schema design.

---

## Status

**Status:** Not started

---

**Docs:** [Epics index](README.md) · [v1.1 milestone](../milestones/v1.1-agency-knowledge-base.md) · [Vision](../vision.md) · [D17 decision](../decisions/D17-knowledge-base-scope-expansion.md)
