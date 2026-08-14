# Changelog

> Tracks significant changes to this documentation repository — new documents, major rewrites, structural changes.
> This is not a Git commit log. Trivial edits (typos, formatting) are not recorded here.
> Format and rules: [AGENTS.md § 7](AGENTS.md#7-changelogmd-rules).

---

## [Unreleased]

### Added

- `docs/roadmap/decisions/D15-progressive-enrichment.md` — New ADR formalising the progressive-enrichment UX principle: instant beautiful profile preview (<5s from upload) using first-pass extracted data, followed by background deep extraction + master catalog enrichment surfaced as soft non-blocking suggestions, with in-context micro-prompts replacing any "review all fields" screen. Defines provenance UI contract and pipeline SLAs.
- `docs/roadmap/decisions/D16-deal-room-trust-model.md` — New ADR defining the Deal Room as a shared trust space rather than a pretty link. Six architectural pillars: append-only event log per room, bilateral bounded control (planner vs. client permissions), provenance-tagged pitch metadata visible to both sides, transparent 3-tier confidence badges, structured preference capture via colored labels + dynamic context selects, and a plain-English History Timeline view. Deferred: e-sign, PDF audit export, multi-approver weighted voting.

### Changed

- `docs/roadmap/decisions/README.md` — Decisions index updated with two new rows (D15 and D16). Table column spacing re-aligned across all rows.
- `docs/business/digital-sales-room-for-events/cold-start.md` — Added two full new sections: (1) "Concrete outreach channels by audience segment" covering solo wedding planners, small agencies, corporate in-house leads, venue managers, and DMCs — each with specific channels (LinkedIn Sales Navigator boolean search, MPI/PCMA/WIPA/IAVM/ADMEI directories, Instagram/TikTok hashtag search, The Knot/WeddingWire/Wezoree/Zola, EventPlanning.com, etc.), tailored opener patterns per segment, and a month-one quick-start cadence. (2) "Toolstack landscape and integration priority roadmap" defining three integration tiers: Tier 1 (pre-v1.0: Google Drive/Dropbox folder import + Gmail/Outlook), Tier 2 (v1.1: HoneyBook/Aisle Planner/Dubsado OAuth brief-push-pull), Tier 3 (v1.2–v1.3: Asana/Monday/Planning Pod/Cvent/Notion/Canva). Navigation footer expanded with Competitive Landscape link. Integration tiers refined: Tier-2 rows rewritten with concrete pull/push contracts (HoneyBook proposal seed + custom fields, Aisle Planner proposal/timeline anchor/asset-library sync, Dubsado workflow triggers + project custom fields). Tier-3 rows rewritten: Asana/Monday pull brief context; Planning Pod expands to BEO seed + budget line items + bi-directional floor-plan PDF asset sync.
- `docs/roadmap/epics/README.md` — Backlog candidates table restructured and expanded from 5 to 10 rows (max cap). Promoted DSR/trust-model candidates to the top of the table: Provenance & confidence badge pitch renderer, Colored label system + legend builder, Dynamic context-aware select boxes in Deal Room, Deal Room Timeline / immutable history viewer, Tier-1/2/3 integrations. Retained Metadata schema versioning UI, CAD file visual processing, and Multi-stakeholder voting. Dropped Vertical extension tooling and Video walkthrough extraction from the candidates table (recorded here: deferred — revisit only after E1–E8 are at least In progress). Candidates restructure: added three new high-priority rows — Mood-board / vision-card context for chat-search (Planning Pod Vision Boards-inspired), Aisle Planner sync (proposal seed + timeline anchor + assets), Planning Pod sync (BEO seed + floor-plan bi-directional PDF sync). Consolidated former Tier-2 CRM integrations and Tier-3 project integrations into these dedicated rows. Removed Multi-stakeholder voting row (deferred: revisit after DSR multi-approver pattern stabilises) to keep 10-row cap.
- `docs/business/personal-venue-catalog/sales/objections.md` — Added new objection entry to the "On switching cost and adoption" section: "We already have Aisle Planner / Planning Pod / HoneyBook. Why another tool?" with underlying concern on tool-sprawl and data drift, a response built on the adjacent-not-competing framework (upstream venue selection & sign-off in VenueMi, downstream contracts/timelines/BEOs in the existing all-in-one, one-click Approve→snapshot sync with no retyping), and a follow-up that walks their current end-to-end flow to map friction points.
- `docs/business/digital-sales-room-for-events/product.md` — Added new subsection "Positioning analogy: Papermark for venues" under the Positioning section, as a brainstorm reference frame (not copy). Analogous to Papermark winning against DocSend on a narrow frictionless delivery slice (not broader product), VenueMi wins on the narrow venue-selection + approval loop (personal venue knowledge governance + frictionless shared-trust pitch delivery) without competing with Planning Pod / Aisle Planner's all-in-one downstream operations. Handoff via one-click Approve snapshot sync. Closing analogy sentence rephrased to explicitly surface both product halves ("smart personal venue catalog you can actually search" + "one-click approval deal room") and mirror the reader's current tool stack by name ("your existing Planning Pod or Aisle Planner workflow") so the adjacent-not-competing message reads clearly to Planning Pod/Aisle Planner users.
- `docs/business/personal-venue-catalog/sales/messaging.md` — New subsection "DSR positioning — taglines for Planning Pod / Aisle Planner users" added under ## Taglines. Four Candidate taglines, each carrying the full two-product logic (smart catalog + one-click approval deal room) while explicitly positioning VenueMi as a plug-in, not a replacement: (1) "Your smart venue catalog. One-click client approval. The rest stays in Planning Pod." (2) "The two hours before the venue is confirmed? 10 minutes. Everything downstream stays exactly where it is." (3) "Scattered PDFs + WhatsApp → smart venue catalog + one-click approval. Straight back into your Aisle Planner workflow." (4) "Papermark for venues — all the friction out of pitching and sign-off, none of the project management you already have."

---

## 2026-08-08

### Changed

- Standardised audience blockquotes across 4 docs per AGENTS.md §3: added `> **Audience:** X.` / `> **Purpose:** Y.` format to docs/README.md, docs/business/digital-sales-room-for-events/README.md, docs/platform/intelligence.md, and docs/business/personal-venue-catalog/comparison.md.
- Normalised navigation footers per AGENTS.md §5: removed redundant top-of-file `**Docs:**` nav blocks from docs/platform/README.md, docs/platform/intelligence.md, and docs/business/personal-venue-catalog/comparison.md so the footer nav appears exactly once as the last element.
- Removed duplicate metadata pre-footer block (Document type / Stage / Audience lines) from docs/platform/intelligence.md — that information now lives in the standard audience blockquote.
- Updated docs/business/market.md cross-references and footer nav to point to digital-sales-room-for-events docs as the primary product positioning, replacing personal-venue-catalog links that preceded the DSR direction shift.
- Resolved 17 broken index links in roadmap READMEs: docs/roadmap/epics/README.md (E1–E9), docs/roadmap/milestones/README.md (v0.1–v1.0), docs/roadmap/decisions/README.md (D1–D3) now list the planned files as plain text rather than linking to files that have not yet been written.
- Expanded docs/platform/README.md Docs footer nav to include Intelligence Layer and Vision links, matching the other platform docs.

---

## 2026-08-03

### Added

- `docs/business/digital-sales-room-for-events/README.md` — concept overview and document index for the DSR positioning; explains the two-layer architecture (Personal Venue Catalog as data layer, Digital Sales Room as output layer).
- `docs/business/digital-sales-room-for-events/product.md` — product structure for the DSR concept: two-layer architecture, five capability pillars (ingestion, catalog, search, pitch board, approval/snapshot), UX concept for agency and client sides, positioning against discovery platforms and generic proposal tools.
- `docs/business/digital-sales-room-for-events/proposal.md` — business case: ICP matrix with updated pricing ($150–300/mo), feature phases, unit economics, GTM playbook (concierge onboarding + viral loop via pitch board), risks, and open questions.
- `docs/business/digital-sales-room-for-events/pitch-mechanics.md` — pitch board mechanics: micro-site structure, event context header, venue cards filtered by brief, collaboration layer (contextual comments, markers, activity feed), approval → immutable snapshot, white-label tiers.
- `docs/business/digital-sales-room-for-events/cold-start.md` — cold start strategy for DSR: two cold-start problems (empty catalog + no demo story), city-focused launch rationale, why focused geography eases outreach, seed catalog build process, concierge onboarding model, validation milestones before self-serve.
- `docs/business/digital-sales-room-for-events/comparison.md` — competitive landscape for DSR: five category sections (DSR/proposal tools, venue discovery, agency CRM/ops, venue management, DIY stack), per-category tables, unified capability matrix, durable edge analysis.

### Changed

- `docs/roadmap/vision.md` — fully rewritten to reflect DSR positioning: updated one-sentence, two-role audience (agency + client), two-layer product structure, new strategic bets (pitch board as buying trigger, approval snapshot for dispute prevention, client onboarding = open a link), north star metric changed from "weekly active searches" to "pitch boards approved per team per month".
- `docs/README.md` — fully rewritten: problem framing expanded to include scattered feedback and missing agreed spec, two-layer structure section added, pricing updated to $150/$300, nav bar updated with Competitive Landscape link.
- `README.md` — DSR section expanded from one row to a full six-row table; blank line formatting fixed.
- `AGENTS.md` — §2 repository structure tree updated to reflect `digital-sales-room-for-events/` and `personal-venue-catalog/` subdirectories; file naming rule updated with explicit PascalCase exception for concept containers; §4.8 and §4.9 rewritten to reference new file paths and DSR document conventions.

### Restructured

- `docs/business/` — original flat business documents (`product.md`, `proposal.md`, `comparison.md`, `cold-start.md`, `sales/`) moved into `docs/business/personal-venue-catalog/`. File `market.md` remains at `docs/business/market.md` — still relevant across both positioning layers. All internal links updated.
- All files in `docs/business/personal-venue-catalog/` marked with `[!NOTE]` callout identifying them as segment reference documents for the catalog data-layer positioning, not the primary product direction.

---

## 2026-07-29 (continued)

### Added

- `docs/business/product.md` — product structure in business terms: who the product is for (event managers and agencies), two interfaces (backoffice and storefront), four capability pillars (ETL/self-ingestion, PIM, DAM, read-only storefront), and positioning summary.

### Changed

- `docs/roadmap/vision.md` — sharpened "one sentence" and added "Who this is for" section; added "Product structure" summary section; renamed "For a planner" to "For an event manager"; updated out-of-scope list to explicitly include storefront as a later epic; updated nav footer.
- `AGENTS.md` — added `product.md` to repository structure tree; added §4.8 Product structure doc type rules; renumbered former §4.8 Competitive landscape to §4.9.
- `README.md` — added Product Structure row to navigation table.

---

## 2026-07-29 (continued)

### Added

- `docs/business/sales/pitch.md` — demo and intro call narrative: five beats with goal, transition, and "what not to do" section.
- `docs/business/sales/battlecards.md` — 11 competitor cards across five groups (venue discovery, venue operations, file storage/DAM, AI productivity, document intelligence APIs). Each card: what they have, what they do not have, verbatim response example, win condition.
- `docs/business/sales/objections.md` — 14 objections across five categories (AI accuracy, switching cost/adoption, data/security, pricing/value, product/roadmap). Each entry: underlying concern, response, follow-up question.

### Changed

- `AGENTS.md` — added `docs/business/sales/` to the repository structure tree; added §4.7 Sales materials doc type rules; renumbered former §4.7 Competitive landscape to §4.8.

---

## 2026-07-29

### Added

- `AGENTS.md` — documentation rules, structure, audience tagging, writing standards, and enforcement policy for all documents in this repository.
- `CHANGELOG.md` — this file.
- `docs/roadmap/vision.md` — product vision: direction, strategic bets, north star metric, and long-horizon intent.
- `docs/roadmap/` — roadmap directory structure introduced. Sub-directories `epics/`, `milestones/`, `decisions/` are defined in `AGENTS.md` and ready to populate.

### Changed

- `README.md` navigation table — pending update to include `AGENTS.md` and roadmap vision link once those files are reviewed.

---

## 2026-07-01

### Added

- `docs/README.md` — plain-language product overview. Audience: anyone. Covers the problem, what VenueMi does, who it is for, how it works, monetization model, and one-sentence summary.
- `docs/business/proposal.md` — full business proposal: ICP matrix, feature phases (Phase 1–3), monetization tiers, go-to-market strategy, international expansion plan, key risks, and open questions.
- `docs/business/comparison.md` — competitive landscape analysis: Cvent, Tripleseat, Momentus, VenueScanner, VenueFindAI, VenueArc, Spark (GEVME/PCMA), Bynder, Brandfolder, Unstructured.io, Docling, Apache Tika. Includes gap summary matrix.
- `docs/platform/README.md` — architecture reference: platform context, domain model (Venue, VenueAsset, ExtractionJob, MetadataEvent), metadata aggregation engine, service architecture (mi-venue-service, mi-venue-processing-worker, mi-venue-model), ETL pipeline, search architecture, API surface, event contracts, plan entitlement mapping, database schema, UI integration, observability, security, technology decisions, and open decisions.
- `docs/platform/intelligence.md` — intelligence layer reference: Spring AI ETL pipeline, asset-type processing matrix, chunking strategy, Apache Tika rationale, Docling integration, venue-specific extraction schema, confidence-sourced metadata model, multi-source aggregation design, scalability architecture, and technology decisions summary.
