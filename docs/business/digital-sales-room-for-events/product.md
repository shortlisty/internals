# Product structure — Digital Sales Room for Events

> **Audience:** Founders, team.
> **Purpose:** Explain what VenueMi is made of in product terms — the two-layer architecture, capability pillars, UX concept, and positioning logic.

---

## What the product is

VenueMi is a client collaboration workspace for event agencies. Its job is to close the gap between "we know which venues work" and "the client has confirmed and we can proceed" — replacing scattered PDFs, email threads, and WhatsApp back-and-forth with a single interactive loop that ends in an approved, traceable specification.

The product has two sides: the **agency** (planner and team) who builds and manages the venue knowledge and assembles the pitch, and the **client** (event buyer) who reviews, provides input, and approves. The agency pays and works in the product daily. The client opens a link — no account, no setup, no training required.

---

## Two-layer architecture

### Layer 1 — Personal Venue Catalog (data layer)

The agency's private library of venues. Documents come in from any source — Drive, Notion, email forwards, direct uploads. AI extracts the details that matter for event planning: capacity in every configuration, catering policy, AV specs, restrictions, contacts, indicative pricing. A human-in-the-loop verification step lets the planner confirm or correct key fields in seconds. The library is structured and searchable from day one.

This layer is the foundation. Without reliable venue data, the pitch has nothing to draw from.

Key capabilities:

- Zero-friction ingestion from Drive, Notion, email (forward to ingest address), direct upload
- AI extraction with per-field confidence scores and source citations
- One-click human verification — split-screen: source document left, extracted fields right
- Flexible metadata schema (JSONB) — no rigid field set, adapts to any venue type
- Hybrid search: keyword, semantic (natural language), structured filters, geo-spatial
- Platform master venue catalog for major markets — invisible gap-fill when the agency's files are sparse (provenance MC_INHERIT, lowest non-scrape priority)

### Layer 2 — Digital Sales Room (output layer)

The client-facing pitch board. The planner selects venues from the catalog, the system assembles an interactive web page, and shares a private link. The client reviews on any device — photos, floor plans, configurable spec, indicative pricing. They can indicate preferences, leave questions, and ultimately approve.

On approval, the system locks an **immutable snapshot**: a timestamped record of every field, file reference, and decision, attributed to its source. That snapshot is the Single Source of Truth for the event — the record both sides can point to if anything is disputed later.

Key capabilities:

- One-click pitch board generation from selected catalog venues
- Private shareable link — no client account required
- Interactive spec: configuration toggles, notes, preference indicators
- Two-sided collaboration: client comments, agency responses, AI-assisted answers from catalog data
- Approval trigger → immutable snapshot with timestamp and source citations
- Retention policy: pitch assets stored through event date + buffer, then auto-purged

---

## Capability pillars

### 1. Ingestion

How venue data enters the library without manual re-entry.

- Connect Drive or Notion workspace — background sync picks up new files automatically
- Forward venue emails to a dedicated ingest address — attachments processed on receipt
- Direct upload for PDFs, photos, floor plans, spec sheets
- AI extraction pipeline: Tika (parsing) → GPT-4o (structured extraction) → confidence scores per field
- Human verification screen: planner reviews AI output against source, corrects in one click
- Master catalog match: if the venue is in the master catalog, gap-fill at MC_INHERIT priority (invisible backdrop merge)

### 2. Venue catalog

The structured knowledge store.

- Each venue is a record with flexible metadata: capacity configurations, catering policy, AV specs, accessibility, restrictions, contacts, pricing, custom notes
- Assets (photos, floor plans, PDFs, CAD files) attached at venue level, not just stored as files
- Provenance per field: which source document, which page, what confidence level, what alternatives exist
- Conflict resolution: when two sources disagree, the planner sees both values and resolves once
- Version history: every field change is logged with actor and timestamp

### 3. Search

Retrieval across the entire library.

- Natural language: "rooftop venue for 80 guests, own alcohol allowed, no curfew"
- Structured filters: capacity, catering policy, amenities, location, venue type
- Semantic similarity: "find venues like this one"
- Geo-spatial: venues within X km of a given address or city centre
- Hybrid mode (default): keyword + semantic + structured filters in one query
- Master catalog values merge invisibly into tenant results; provenance (MC_INHERIT) tracked internally per field, not surfaced as a separate result bucket

### 4. Pitch board

The client-facing output.

- Planner selects 2–6 venues, clicks Generate — system builds the pitch page
- Private link, no client login required, works on any device
- Each venue card: photo gallery, floor plan preview, key metadata, configurable spec
- Agency branding: logo, colours, custom domain (Enterprise)
- Client interaction: preference indicators (shortlist / consider / decline), inline questions
- Agency view: real-time notifications when client opens the board, views a venue, asks a question
- AI-assisted responses: planner asks "what does the catalog say about loading access at this venue?" — answer pulled from source documents

### 5. Approval and snapshot

The closing step.

- Client clicks Approve on a venue or configuration
- System generates an immutable snapshot: venue metadata at that version, selected configuration, client identity, timestamp, source citations for every field
- Both sides receive a copy — PDF export and persistent web view
- Board status changes to Approved — no further edits without explicit revision workflow
- Snapshot stored for event date + 30 days, then purged (tenant can extend)

---

## UX concept

**Agency-side:** information-dense, fast, built for daily use. Grid-first venue library (Akeneo PIM pattern), side-panel preview on row click (pics.io pattern), inline editing without modal flows. The planner should be able to answer a client question, add a new venue, and generate a pitch in under five minutes without touching a mouse more than necessary.

**Client-side:** minimal, self-evident, mobile-first. One link opens a clean pitch board. No navigation, no menus, no settings. The client sees venues, browses details, and approves. The experience should feel like receiving a premium proposal from a technologically sophisticated agency — not like logging into another SaaS tool.

---

## Positioning

VenueMi occupies a gap between two categories that currently do not overlap.

**Venue discovery platforms** (Cvent, VenueScanner) — know venues that have self-submitted publicly. They do not know what is in the agency's own files, and they do not help close a deal with a specific client.

**Generic proposal tools** (Qwilr, Pandadoc, Dock.us) — produce beautiful client-facing documents. They do not understand venue data, cannot extract from PDFs, and have no structured knowledge layer feeding the output.

VenueMi combines the knowledge layer with the client-facing output in a single product built specifically for the event planning workflow. The catalog feeds the pitch. The pitch generates the approval. Neither exists without the other, and no existing tool provides both.

---

**Docs:** [What is VenueMi?](../../README.md) · [Business Proposal](proposal.md) · [Architecture](../../platform/README.md) · [Intelligence Layer](../../platform/intelligence.md) · [Vision](../../roadmap/vision.md)
