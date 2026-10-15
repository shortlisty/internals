# E3 — Document Intelligence (Basic)

> **Audience:** Product, engineering.
> **Purpose:** Turns uploaded venue documents into structured venue profile data — the first intelligence-layer capability in v0.1.

---

## Summary

Document Intelligence (Basic) runs uploaded documents through an extraction pipeline that produces raw text and a small set of core fields (capacity, address, category, contact). The extracted fields and a per-field confidence score are surfaced on the venue profile. After this epic ships, a planner who uploads a venue brochure sees capacity, address, and contact details auto-populated instead of typing them by hand.

---

## User stories

- As a planner, I want uploaded PDFs to be parsed and their text stored on the venue record so that the content is searchable later.
- As a planner, I want DOCX and plain-text documents extracted alongside PDFs so that all my source material is covered.
- As a planner, I want core fields (capacity, address, category, contact) auto-populated from uploaded documents so that I do not have to type them.
- As a planner, I want to see a confidence score next to each extracted field so that I know whether to trust it.

---

## In scope

- Extraction pipeline: document upload → parse → text extraction → field extraction
- Supported formats: PDF, DOCX, plain text
- Core field extraction: capacity, full address, venue category, contact name, contact email
- Per-field confidence score stored and visible on the venue profile
- Extracted text stored on the venue record for later search

---

## Out of scope

- Semantic search and vector embeddings over extracted text (v0.2+)
- Field-level provenance badges showing the source document and page (v0.2+)
- Conflict resolution UI when two documents disagree on a field (v0.3+)
- Image and floor-plan layout-aware extraction (v0.2+)
- CAD/DWG processing, video transcription (post-v1.0)
- Docling high-fidelity table and layout parsing — Tika only in v0.1

---

## Milestone references

- [v0.1 — Foundation + Catalog Core](../milestones/v0.1-foundation-catalog-core.md)

---

## Open questions

- [x] Extraction engine for v0.1: Tika. Docling is deferred to a later milestone per D2.
- [ ] What is the documented extraction SLA for v0.1? (Needed for the v0.1 acceptance criteria.)

---

## Status

**Status:** Not started

---

**Docs:** [Epics index](README.md) · [E2 Tenant Venue Profiles](E2-tenant-venue-profiles.md) · [v0.1 milestone](../milestones/v0.1-foundation-catalog-core.md) · [Intelligence layer](../../platform/intelligence.md)
