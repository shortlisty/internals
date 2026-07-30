# Product structure

> **Audience:** Founders, team.
> **Purpose:** Explain what iQ BENE is made of in business terms — the tenant app, the four capability pillars, the UI concept, and the positioning logic. Read this before the architecture reference.

---

## What the product is

iQ BENE is a knowledge base and workflow tool for event managers and event agencies. Its job is to help them organise venues faster, find the right one sooner, and run better events — without the manual work that currently slows everything down.

The product has one user: the event professional. Everything is designed around their daily workflow — receiving venue documents, building a library of trusted venues, answering client briefs, and preparing proposals. The end client (the person who commissions the event) is a recipient of output from that workflow, not a user of the tool.

---

## The tenant app

The product is a single working environment: the tenant app. Each event agency or event manager operates as a tenant with their own isolated library, team, and settings.

The tenant app is utilitarian and fast. It is built for daily use by professionals who need to get things done, not for occasional visitors who need hand-holding. Interface density is a feature — the right amount of information visible at once, without modal-heavy, wizard-driven flows that slow down experienced users.

UI concept: the interaction patterns of tools like Akeneo PIM (structured data editing at speed, grid-first navigation), OroCommerce (dense but scannable layouts, inline editing), and pics.io (media asset browsing without friction, spatial previews). Not enterprise complexity — the right density for a working professional who lives in the tool every day.

Everything in the tenant app is optimised for two moments:

- **Input:** upload a document, enter a detail, correct an extraction result, resolve a conflict — as fast as possible.
- **Retrieval:** find any venue detail, across the entire library, in a natural-language query or a filtered grid, in seconds.

---

## Four capability pillars

The tenant app is built on four capability clusters. Each is a distinct functional area with its own data model and UX surface. Together they form a complete venue intelligence workflow with no external dependencies.

### ETL and self-ingestion

How venue data gets into the system. Event managers import from external sources — spreadsheets, supplier exports, other databases — or enter data manually, without involving IT. AI extraction from uploaded documents is the primary path. Manual entry and bulk import are the fallbacks.

This is the entry point. Without good ingestion, the library stays empty and the product has no value.

Corresponds to epic: [E2 — Document intelligence](../roadmap/epics/E2-document-intelligence.md).

### PIM — Product Information Management

The structured knowledge store. All textual and structured information about a venue lives here: room names and descriptions, capacity configurations, catering policy, pricing, available options, contact details, restrictions.

"Product" in PIM maps to "venue" in iQ BENE. Each venue is a structured record with a schema. The PIM layer enforces that schema, tracks provenance (which source each field came from), manages confidence scores, and resolves conflicts when multiple documents disagree.

This is the core of the platform. Search, sharing, and every downstream workflow depend on the quality of data here.

Corresponds to epics: [E1 — Venue profiles](../roadmap/epics/E1-venue-profiles.md), [E6 — Data quality](../roadmap/epics/E6-data-quality.md).

### DAM — Digital Asset Management

The media store. Floor plans, photos, video walkthroughs, 3D tours, CAD files, branded decks — all binary assets associated with a venue.

DAM in iQ BENE is not generic asset storage. Assets are attached to venues and to specific rooms or spaces within a venue. A floor plan is not just a file — it is a spatial asset linked to a room with known dimensions. That linkage is what makes the asset searchable and useful, not just stored.

Corresponds to epics: [E1 — Venue profiles](../roadmap/epics/E1-venue-profiles.md) (asset sub-model), [E2 — Document intelligence](../roadmap/epics/E2-document-intelligence.md) (floor plan processing).

### Search

The retrieval layer across the entire PIM and DAM. Natural-language queries, keyword search, structured filters (capacity, location, catering type, equipment), and geo-spatial search — all operating against the same structured data store.

Search is the north star interaction. A manager should be able to describe a client brief in plain language and get a ranked, sourced shortlist without opening a single file.

Corresponds to epic: [E3 — Search](../roadmap/epics/E3-search.md).

---

## Positioning summary

iQ BENE occupies a gap that existing tools do not fill.

Generic file storage (Drive, Dropbox) holds venue documents but cannot read them. Enterprise DAM platforms (Bynder, Brandfolder) manage assets generically with no venue schema and no extraction. Venue marketplaces (Cvent, VenueScanner) know publicly listed venues but not the ones in a planner's own files. Generic AI tools (ChatGPT) can answer a one-off question about a PDF but have no memory, no schema, and no team sharing.

iQ BENE combines ETL, PIM, DAM, and search in a single tenant app focused specifically on the venue management workflow. It is not a general-purpose tool adapted to events — it is built from the ground up for event professionals, at the interface density that working professionals need.

---

**Docs:** [What is iQ BENE?](../README.md) · [Business Proposal](proposal.md) · [Competitive Landscape](comparison.md) · [Architecture](../platform/architecture.md) · [Vision](../roadmap/vision.md)
