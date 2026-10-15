# E2 — Tenant Venue Profiles

> **Audience:** Product, engineering.
> **Purpose:** The core planner-facing capability of v0.1 — creating and managing venues in a tenant's private catalog, with document upload and a basic profile view.

---

## Summary

Tenant venue profiles give planners a private catalog of venues they own. A planner can create a venue from scratch or promote one from the master catalog, view and edit its profile, upload documents and assets, and delete it. This is the main user-facing outcome of v0.1 — without it, nothing else the planner does has a place to live.

---

## User stories

- As a planner, I want to create a venue with core fields (name, address, city, capacity, category, summary) so that I have a record for it.
- As a planner, I want autocomplete suggestions from the master catalog when creating a venue so that I can promote an existing venue in one click.
- As a planner, I want to view a venue profile page with all core fields rendered so that I can review what I have stored.
- As a planner, I want to edit the core fields of a venue I own so that I can keep the record up to date.
- As a planner, I want to upload documents (PDF, DOCX, images) to a venue and see them listed so that supporting materials live with the venue.
- As a planner, I want to delete a venue I own so that it no longer appears in my catalog.

---

## In scope

- Tenant venue CRUD through the tenant app UI: create, view, edit, delete
- Core venue fields: name, address, city, capacity, venue category, one-paragraph summary
- Master catalog autocomplete + one-click promotion into the tenant catalog
- Document and asset upload to a venue (PDF, DOCX, images)
- Uploaded documents listed on the venue profile
- Tenant isolation: one tenant's venues and documents are not visible to any other tenant

---

## Out of scope

- Extracted fields surfaced on the profile (lives in E3)
- Custom tags on venues (v0.2+)
- Soft-delete and recycle bin (v1.0+)
- Plan enforcement (venue-count and upload-GB caps per tier) — deferred
- Team collaboration, invites, roles (v0.4+)
- Search and filters (v0.2+)

---

## Milestone references

- [v0.1 — Foundation + Catalog Core](../milestones/v0.1-foundation-catalog-core.md)

---

## Open questions

- [x] No open questions at this time.

---

## Status

**Status:** Not started

---

**Docs:** [Epics index](README.md) · [E1 Master Venue Catalog](E1-master-venue-catalog.md) · [E3 Document Intelligence (Basic)](E3-document-intelligence.md) · [v0.1 milestone](../milestones/v0.1-foundation-catalog-core.md)
