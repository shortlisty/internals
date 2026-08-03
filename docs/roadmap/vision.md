# Vision

> **Audience:** Founders, team.
> **Purpose:** Describe where BENE Intelligence is going and why — not what has been built. For completed work, see [CHANGELOG.md](../../CHANGELOG.md) and the milestone files.

---

## One sentence

BENE is the collaborative workspace where event agencies turn their venue knowledge into a signed-off proposal — so a client brief becomes a confirmed venue spec in hours, not days.

---

## Where we are now

The product concept has evolved. The original design — a private venue catalog with AI extraction and semantic search — remains the correct data layer. The position has sharpened: the primary output is not an internal search engine, but a client-facing interactive pitch board that ends with a mutually approved specification.

Architecture and domain model are settled and documented. No code has shipped yet. New product documents are being written under [Digital Sales Room for Events](../business/Digital_Sales_Room_for_Events/).

---

## Who this is for

Event agencies and independent event managers — the professionals whose job is to find the right venue, present it to a client, and get sign-off before the event date locks in. They manage weddings, corporate events, conferences, team-building sessions, and private parties in the 50–1000 guest range.

Two roles matter on every project:

**The agency (planner side)** — builds and maintains the venue library, assembles a shortlist for a client brief, generates the pitch board, answers questions, and drives toward approval.

**The client (buyer side)** — reviews the shortlist, asks questions, indicates preferences, and gives the final sign-off. They do not need an account or any training. They open a link.

---

## The problem we are solving

An agency receives a client brief. The planner knows the right venues are somewhere in their files — a PDF from two years ago, a floor plan on someone's laptop, a WhatsApp exchange with the venue coordinator. Finding and assembling that into something presentable takes hours. The client gets a static PDF. Feedback arrives scattered across email and WhatsApp. Nobody has a clear record of what was agreed.

The cost is not just time. It is lost deals when a competitor responds faster, margin erosion when the planner sends the wrong price, and post-event disputes when the agreed spec is nowhere to be found.

---

## What success looks like

### For an agency

A client brief arrives. The planner opens BENE, filters the venue library in under a minute, picks three options, and sends the client a link. The client opens an interactive board, browses floor plans and photos, adjusts the configuration, and clicks Approve. The planner receives a notification. Both sides have a timestamped, immutable record of exactly what was agreed. The whole cycle takes hours, not days.

### For a client

No inbox back-and-forth. No PDF attachments. One link, open on any device. The venues are laid out clearly. The spec is adjustable in real time. Approving takes one click. The result feels like working with a premium agency — not because the planner is more experienced, but because the tool makes their work visible and trustworthy.

### For the platform

BENE becomes the layer that closes the gap between "we know which venues work for this client" and "the client has confirmed and we can proceed." The venue catalog feeds the pitch. The pitch generates the approval. The approval locks the spec. Each step is tracked, sourced, and auditable.

---

## Product structure

BENE has two interlocking layers.

**Layer 1 — Personal Venue Catalog** (data layer)
The agency's private library of venues, structured and searchable. Documents come in from Drive, Notion, email, and direct uploads. AI extracts metadata — capacity, catering policy, AV specs, restrictions, contacts — and maps it against a flexible schema. The planner verifies key fields with one click. The catalog is the foundation: without accurate venue data, the pitch has nothing to draw from.

Full design: [Personal Venue Catalog](../business/Personal_Venue_Catalog/product.md).

**Layer 2 — Digital Sales Room** (output layer)
The client-facing pitch board. The planner selects venues from the catalog, the system assembles an interactive web page — photos, floor plans, metadata, a configurable spec — and shares a private link. The client browses, comments, and approves. When they click Approve, the system freezes a snapshot: an immutable record of every field, file, and decision, timestamped and attributed. That snapshot is the Single Source of Truth for the event.

Full design: [Digital Sales Room for Events](../business/Digital_Sales_Room_for_Events/).

```
Personal Venue Catalog  →  data layer  (ingestion, ETL, metadata, search)
Digital Sales Room      →  output layer (pitch board, client portal, snapshot)
```

---

## Strategic bets

**The pitch board is the buying trigger, not the catalog.**
Agencies will pay for a tool that directly accelerates client sign-off and makes their proposals look premium. They will not pay for a better file organiser. The catalog is a prerequisite, but the value event — the moment that justifies the subscription — is when the client opens the board and approves in the same session.

**Speed of response is a competitive differentiator for agencies.**
An agency that can respond to a brief with a polished, interactive shortlist in under an hour wins deals that a slower competitor loses. BENE's value is measured in hours saved and deals won, not in database entries.

**The approval snapshot prevents post-event disputes.**
Agencies and clients regularly disagree after the fact about what was agreed. An immutable, timestamped record of the approved spec — with source citations for every field — eliminates that dispute. This is a legal and financial protection story, not just a UX convenience. It is the reason agencies at higher price points will pay $150–300/month without hesitation.

**Extraction quality needs to be verified before accuracy promises are made.**
The catalog's value depends on extracted data being trustworthy. Real venue documents vary enormously in format and quality. Before any accuracy claims reach customers, 50 real venue documents must be processed and measured. Human-in-the-loop verification is a first-class feature, not a workaround.

**The client side cannot require onboarding.**
The client opens a link. That is the entire onboarding. If the experience requires an account, an app download, or any explanation — it will fail. The pitch board must be self-evident on first open, on any device.

---

## Out of scope now

- **Booking and payments.** Contracts, invoicing, deposit collection, and calendar holds are venue operator problems. BENE ends at sign-off on the spec, not at the financial transaction.
- **Venue discovery and marketplace.** BENE works with venues the agency already knows. Finding new venues is a separate workflow — and a Phase 3 direction at earliest.
- **CRM and deal pipeline.** BENE is not a sales CRM. It does not manage leads, stages, or follow-up sequences.
- **Event execution tools.** Timelines, crew management, run-of-show, and on-site logistics are out of scope. BENE covers the venue selection and sign-off phase only.
- **CAD and video processing in Phase 1.** Floor plans and photos are handled. DWG/DXF parsing and video walkthrough extraction are Phase 2.

---

## North star metric

**Pitch boards approved per team per month.**

An approved board means the product completed its job: the agency got sign-off, the client has a confirmed spec, and BENE was the instrument that made it happen. Venues ingested and searches run are inputs. Approvals are the outcome.

---

## Long horizon

Once the pitch-to-approval loop is proven and trusted, the platform can grow in two directions.

**Upward:** deeper collaboration features — inline comments, version history, multi-stakeholder voting on the client side, integration with calendar and booking tools to bridge the gap between approval and execution.

**Outward:** a two-sided layer where venues pay to maintain a verified public profile that agencies can pull into pitches directly, without manual ingestion. That turns BENE into a marketplace — but only after the agency-side workflow is embedded and trusted. The venue side is years away. Nothing in near-term planning is designed to serve venues.

---

**Docs:** [What is BENE?](../README.md) · [Personal Venue Catalog](../business/Personal_Venue_Catalog/product.md) · [Digital Sales Room](../business/Digital_Sales_Room_for_Events/README.md) · [Market Structure](../business/market.md) · [Architecture](../platform/architecture.md)
