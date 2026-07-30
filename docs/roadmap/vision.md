# Vision

> **Audience:** Founders, team.
> **Purpose:** Describe where iQ BENE is going and why — not what has been built. For completed work, see [CHANGELOG.md](../../CHANGELOG.md) and the milestone files.

---

## One sentence

iQ BENE is the knowledge base for event professionals — turning the venue files, floor plans, and spec sheets a team has collected over years into a structured, searchable library so anyone can find any detail in seconds and organise better events, faster.

---

## Where we are now

The platform is designed and ready to build. The architecture is settled, the domain model is defined, and the core extraction pipeline is chosen. No code has shipped yet. The first milestone targets the smallest version of the product that delivers the aha moment: upload a PDF, watch it become a structured venue profile, run a search that returns it.

Product structure in business terms: [product.md](../business/product.md).

---

## Who this is for

iQ BENE is built for event managers and event agencies — the professionals whose job is to find, evaluate, and brief venues on behalf of clients. They manage weddings, corporate events, conferences, workshops, and private parties. They visit dozens of venues a year and receive hundreds of documents: PDFs, floor plans, spec sheets, photo packs, branded decks.

The end client — the person commissioning the event — is not a user of the tool. They are the recipient of better, faster answers from the manager who uses it.

The tenant app is built for the event professional's daily workflow: utilitarian, fast, information-dense. See [product.md](../business/product.md) for the full UI concept.

---

## The problem we are solving

Event professionals carry most of their venue knowledge in their heads, in personal inboxes, and in scattered folder structures. The knowledge exists — it just cannot be found when it needs to be. A client calls with a specific brief, and the answer is buried in a 40-page PDF that someone emailed three years ago.

The cost is not only time. It is client confidence, missed opportunities, and institutional knowledge that walks out when a senior planner leaves. New hires take months to get up to speed. The same venues get researched from scratch for every new event.

---

## What success looks like

### For an event manager

The manager no longer digs through folders or asks colleagues where the venue file is. They type a natural-language query — "kosher catering, 150 guests, freight elevator, downtown" — and get a sourced answer in under ten seconds. They upload a document during a site visit and it is in the team's shared library by the time they get back to the office. They organise the same class of event faster with every iteration because the knowledge compounds.

### For an agency

Nothing is lost when a manager leaves. New hires can answer client questions independently within their first week. The team has one shared, authoritative source for every venue they have ever worked with — enriched from every document they have ever received from that venue. Client proposals are faster to build and more accurate because the underlying data is structured and current.

### For the platform

iQ BENE becomes the intelligence layer that sits beneath every stage of the event planning workflow. Venues are no longer a pile of files — they are structured, versioned, queryable knowledge assets. The platform earns trust by being accurate, sourced, and transparent about what it knows and what it is uncertain about.

---

## Product structure

The product is a single tenant app. The full description is in [product.md](../business/product.md). The summary:

- **Tenant app** — the working environment for event managers. ETL ingestion, PIM (structured venue data), DAM (floor plans, photos, media), search, team collaboration. One app, one user type, built for daily professional use.
- **ETL and self-ingestion** — how data gets in: AI extraction from uploaded documents, bulk import, manual entry.
- **PIM** — the structured knowledge store: venue schema, confidence scores, conflict resolution, provenance.
- **DAM** — media and spatial assets attached to venues and rooms, not just stored as files.
- **Search** — natural-language and structured retrieval across the entire library. The primary interaction.

---

## Strategic bets

These are the assumptions the product depends on. If any of them turn out to be wrong, the roadmap must change.

**Extraction quality is good enough to trust.**
The core value proposition only holds if event managers trust the extracted data. Real-world venue documents vary enormously — text-based, scanned, design-heavy, multi-column. We need a benchmark on 50 real venue documents before making accuracy claims to customers. Confidence scores and one-click manual overrides are not a fallback — they are first-class features that exist precisely because extraction will sometimes be wrong.

**Search is the primary workflow, not filing.**
The product is not a better folder structure. It is an answer engine. If managers use it primarily to store and browse files rather than to search and get answers, the semantic search investment was wrong and the UX needs to change. Weekly active searches — not venues uploaded — is the metric that proves the product is working.

**Team knowledge retention is the buying trigger.**
Individual managers may find the product useful, but the buying decision happens when an agency owner recognises that institutional knowledge is at risk. The shared library is the moat. A solo user switching tools is low friction; a team of fifteen who have built their venue library together is not.

**The gap is unoccupied.**
No existing tool combines planner-owned document ingestion, AI extraction with a venue-specific schema, multi-source conflict resolution, and semantic search. This assessment is documented in [competitive landscape](../business/comparison.md). If a direct competitor emerges, the roadmap must be re-evaluated against their actual capabilities, not their marketing.

---

## Out of scope now

The following are deliberate exclusions from the initial product. They may be revisited as the platform matures.

- **Venue discovery and marketplace.** We are not building a public venue database. We help managers work with venues they already know. Discovery is a Phase 3 direction.
- **Booking and operations.** Contracts, invoicing, calendar management, and RFP sending are venue operator problems. iQ BENE serves event professionals, not operators.
- **CRM and contact management.** iQ BENE extracts venue contacts from documents and stores them as structured data on the venue profile. It does not replace a CRM or manage deal pipelines.
- **Event planning workflow tools.** Content generation, agenda drafting, and speaker management are out of scope. iQ BENE provides the venue intelligence that makes those workflows accurate — it does not own the workflows themselves.
- **Video walkthroughs.** Keyframe extraction and vision-based walkthrough analysis are deferred to Phase 2. The Phase 1 pipeline handles PDFs, images, DOCX, and CAD files.

---

## North star metric

**Weekly active searches per team.**

A team that searches weekly is using iQ BENE as a knowledge tool, not a filing tool. This metric proves that the intelligence layer is trusted and embedded in daily work. Venue uploads and extraction jobs are inputs. Searches are the outcome.

---

## Long horizon

The tenant app is the foundation. Once the venue knowledge base is proven — accurate, trusted, embedded in daily agency workflows — the long-horizon direction is a two-sided marketplace: event managers manage their portfolio, venues compete to be visible to managers actively searching. That is years away. Nothing in the near-term roadmap is designed to serve venues. A client-facing storefront is part of that same long horizon, not a near-term feature. Everything now is designed to serve the managers who work with them.

---

**Docs:** [What is iQ BENE?](../README.md) · [Product Structure](../business/product.md) · [Business Proposal](../business/proposal.md) · [Competitive Landscape](../business/comparison.md) · [Architecture](../platform/architecture.md)
