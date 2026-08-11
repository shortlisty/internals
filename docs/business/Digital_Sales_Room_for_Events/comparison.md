# Competitive landscape — Digital Sales Room for Events

> **Audience:** Founders, team.
> **Purpose:** Where VenueMi sits relative to adjacent tools, and why no existing product covers the same ground.

---

## The gap in one sentence

Every tool either helps agencies **find** venues they don't know yet, or helps venues **manage** their own operations, or helps agencies **create** polished documents — but none combines a planner-owned venue knowledge layer with a client-facing approval workflow in a single product.

---

## Tool categories

### 1. Digital Sales Room / Proposal tools

The closest overlap on the output side. These tools produce beautiful client-facing web pages or interactive documents. They have no understanding of venues, no extraction layer, no structured catalog feeding the output.

| Tool                 | What it does                                       | Gap vs. VenueMi                                                    |
| -------------------- | -------------------------------------------------- | ------------------------------------------------------------------ |
| **Qwilr**            | Interactive web-based proposals, e-sign, analytics | No venue knowledge layer; planner manually enters all content      |
| **Dock.us**          | Client portal with embedded content, tasks, links  | General-purpose; no venue schema, no extraction, no event context  |
| **Trumpet**          | Digital sales rooms for B2B sales teams            | Sales-cycle focused; no event or venue concept                     |
| **Pandadoc**         | Proposals, contracts, e-sign                       | Document-centric; no interactive venue browsing or structured data |
| **Proposify**        | Proposal builder with templates                    | Template-based manual entry; no catalog or AI extraction           |
| **Better Proposals** | Fast proposal creation, e-sign                     | Same limitations as Proposify                                      |

**Summary:** These tools are good at _presentation_. VenueMi is good at _knowledge + presentation_. Using Qwilr for venue proposals means the planner manually copies capacity figures and catering policy from PDFs into a template — every time. VenueMi's catalog eliminates that step.

---

### 2. Venue discovery and sourcing

These tools help planners find venues they haven't worked with yet. They operate on publicly submitted venue data, not on the agency's own documents. The moment a planner already knows which venues they trust, these platforms have little left to offer.

| Tool             | What it does                                          | Gap vs. VenueMi                                                     |
| ---------------- | ----------------------------------------------------- | ------------------------------------------------------------------- |
| **Cvent**        | Largest venue marketplace, RFP automation, enterprise | Discovery only; no planner-owned knowledge base; enterprise pricing |
| **VenueScanner** | UK marketplace, free for planners, AI ranking         | Discovery only; no document intelligence; no client pitch output    |
| **Hopskip**      | Hotel/venue RFP, 150K+ properties                     | RFP sourcing; no persistent catalog; no client collaboration        |
| **Tagvenue**     | Self-serve marketplace, 80K+ venues                   | Discovery only; no planner-side library                             |
| **VenueFindAI**  | AI + human concierge matching                         | Ephemeral recommendations; no stored portfolio; no pitch output     |
| **Ventur3**      | RFP builder, response tracking                        | Data comes from venue self-submission, not agency files             |

**Summary:** Complementary to VenueMi, not competing. A planner discovers a venue through Cvent or VenueScanner, then ingests that venue's documents into VenueMi to build a permanent, searchable profile and generate pitches from it.

---

### 3. Event agency CRM and ops tools

These tools manage the agency's client relationships and project workflows. Some generate proposals or contracts, but they are business-management tools — not venue intelligence tools.

| Tool             | What it does                                                              | Gap vs. VenueMi                                                 |
| ---------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------- |
| **HoneyBook**    | Proposals, contracts, invoicing, payments for creative/event solopreneurs | No venue library or extraction; proposals are manual templates  |
| **Dubsado**      | CRM, forms, contracts, workflows                                          | Same as HoneyBook; no venue concept                             |
| **Planning Pod** | Venue-operator tool (BEO, booking, billing)                               | Built for venue operators, not planners; no agency-side catalog |
| **Tripleseat**   | Sales and catering software for restaurants/hotels                        | Venue-side; data is transactional, not document-extracted       |

**Summary:** These tools cover what happens _after_ the venue is confirmed — contracts, invoicing, project management. VenueMi covers what happens _before_: finding, evaluating, pitching, and getting client sign-off. The two workflows are adjacent and can coexist.

---

### 4. Venue management platforms

Built for venue operators — helping them run their own space, manage bookings, and generate revenue. No relevance to the planner workflow.

| Tool                      | What it does                             | Gap vs. VenueMi                                  |
| ------------------------- | ---------------------------------------- | ------------------------------------------------ |
| **Perfect Venue**         | Lightweight venue ops, bookings, BEO     | Venue-side; no planner portfolio concept         |
| **Event Temple**          | Venue CRM, bookings, contracts           | Same as Perfect Venue                            |
| **Momentus Technologies** | Enterprise venue ops, convention centres | Enterprise; venue-side; no document intelligence |

**Summary:** No overlap with VenueMi's use case.

---

### 5. The DIY stack

The most common "competitor" is not a product — it is the patchwork of general tools agencies already use. This is the real incumbent.

| Component       | Tool used                                | Why it fails                                       |
| --------------- | ---------------------------------------- | -------------------------------------------------- |
| Venue knowledge | Google Drive folders / Notion / Airtable | Files stored, not read; data manual and goes stale |
| Pitch creation  | Canva / PowerPoint / Google Slides       | Hours per pitch; static output; no structured data |
| Client sharing  | Email attachment / PDF                   | No interaction; feedback scattered across channels |
| Client feedback | WhatsApp / email threads                 | Not consolidated; no record of what was agreed     |
| Sign-off        | Verbal / email "sounds good"             | No traceable approval; disputes arise later        |

**Summary:** The DIY stack works until it doesn't — usually when a brief is urgent, a senior planner is unavailable, or a client dispute arises. VenueMi replaces the entire stack with one loop: ingest → catalog → pitch → approve.

---

## Capability matrix

| Capability                                   |      VenueMi      | Qwilr / Dock.us | Cvent / VenueScanner | HoneyBook / Dubsado |     DIY stack      |
| -------------------------------------------- | :---------------: | :-------------: | :------------------: | :-----------------: | :----------------: |
| Planner-owned venue knowledge base           |        ✅         |       ⛔        |          ⛔          |         ⛔          |  Partial (manual)  |
| AI extraction from agency's own docs         |        ✅         |       ⛔        |          ⛔          |         ⛔          |         ⛔         |
| Structured venue catalog (search + filter)   |        ✅         |       ⛔        |          ⛔          |         ⛔          | Partial (Airtable) |
| Event-specific filtering of venue data       |        ✅         |       ⛔        |          ⛔          |         ⛔          |         ⛔         |
| Client-facing interactive pitch (micro-site) |        ✅         |       ✅        |          ⛔          |       Partial       |         ⛔         |
| No client login required                     |        ✅         |       ✅        |          —           |       Partial       |         ✅         |
| Two-sided comments and collaboration         |        ✅         |     Partial     |          ⛔          |         ⛔          |  Partial (email)   |
| Floor plan / photo preview in pitch          |        ✅         |     Partial     |          ⛔          |         ⛔          |         ⛔         |
| Approval → immutable snapshot (SSOT)         |        ✅         |       ⛔        |          ⛔          |  Partial (e-sign)   |         ⛔         |
| Agency white-label (custom domain)           |  ✅ (Business+)   |       ✅        |          —           |         ✅          |        n/a         |
| SMB-friendly pricing                         |        ✅         |       ✅        |          ⛔          |         ✅          |     ✅ (free)      |
| Venue discovery (new venues)                 | Platform registry |       ⛔        |          ✅          |         ⛔          |         ⛔         |

---

## VenueMi's durable edge

Three things that would require a competitor to build from scratch:

**1. The venue knowledge layer.** Qwilr could add a "venue database" feature tomorrow — but it would still be manual entry, not extracted from the agency's existing files. The extraction pipeline (Tika + GPT-4o + confidence scores + human verification) is months of work, not a feature flag.

**2. Event-specific pitch rendering.** The pitch board doesn't show all metadata for every venue — it filters and ranks by what matters for _this_ brief. That requires a structured catalog with per-field data, not a PDF upload.

**3. The approval snapshot with provenance.** The snapshot isn't just "client clicked approve" — it records the exact metadata version, source citations, and configuration at the moment of approval. Rebuilding this on top of a general DSR tool would require the entire catalog layer anyway.

---

**Docs:** [What is VenueMi?](../../README.md) · [Product Structure](product.md) · [Business Proposal](proposal.md) · [Vision](../../roadmap/vision.md)
