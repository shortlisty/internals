# Changelog

> Tracks significant changes to this documentation repository — new documents, major rewrites, structural changes.
> This is not a Git commit log. Trivial edits (typos, formatting) are not recorded here.
> Format and rules: [AGENTS.md § 7](AGENTS.md#7-changelogmd-rules).

---

## [Unreleased]

### Added

- `AGENTS.md` §4.10 — new "Roadmap index README files" document type covering `docs/roadmap/{epics,milestones,decisions}/README.md` as canonical planning indexes; defines shared rules (index-first convention, backlog candidates size caps, pre-implementation snapshot note) plus per-README rules for column sets, grouping (epics by product layer, milestones by layer-first progression), and cross-reference integrity.
- `AGENTS.md` §4.11 — new "Feature checklist" document type covering `docs/roadmap/feature-checklist.md` as the mid-level planning document between epics and milestones; defines P0–P3 version-bound priority tiers, 0.5–3 day granularity per checkbox, anti-drift mandatory metadata format (Epic · Milestone · Priority per line), grouping by tier then product layer, user-visible language rule, and checkbox-only tracking (no dates, no assignees, no percentages).
- `docs/roadmap/feature-checklist.md` — new mid-level prioritised product feature checklist with 54 feature checkboxes across 4 tiers (P0=Highly prioritized, P1=Well prioritized, P2=Mid priority, P3=Post-v1.0 candidates). Each line carries its Epic, Milestone, and Priority tag for cross-reference integrity with epics and milestones READMEs. P0 scope locked to v0.1 MVP demoability (account creation, venue CRUD, document upload, profile view); P1 covers v0.2–v1.0 end-to-end commercial loop (ETL, search, extraction, plans, team, shortlist, export, approval snapshot).

### Changed

- `AGENTS.md` §2 repository structure tree — added `docs/roadmap/feature-checklist.md` entry between `vision.md` and `epics/`, per the AGENTS.md rule that new files must be added to the structure tree before being created on disk.
- `AGENTS.md` §4.4 Epics — expanded from 5 bullet rules to a complete standard: added user-outcome unit-of-value criterion, 9-section anatomy checklist with explicit ordering, forward-only status transitions with concrete entry conditions per status, index-first file creation rule, and bidirectional epic↔milestone cross-reference rule.
- `AGENTS.md` §4.5 Milestones — expanded from 5 bullet rules to a complete standard: added shipped-increment unit-of-value criterion, one-sentence `## Goal` rule (multi-sentence = split it), 8-section anatomy checklist with explicit ordering, forward-only status transitions with concrete entry conditions per status (including the `In progress`-only date-writing window), strengthened no-planned-dates rule, index-first file creation rule, and bidirectional milestone↔epic cross-reference rule.
- `docs/roadmap/epics/README.md` — full restructure from a flat file list to the pre-implementation standardisation index: added epic lifecycle, product-layer grouping convention (A: Foundation, B: PVC data layer, C: DSR output layer, D: Post-v1.0), 9-section epic anatomy checklist, grouped index tables with 6 columns including explicit dependencies and target-milestone cross-references, 6-step "Creating a new epic" procedure (index first, document second), and a 5-row backlog candidates table with size cap.
- `docs/roadmap/milestones/README.md` — full restructure from a flat file list to the pre-implementation standardisation index: added milestone lifecycle, layer-first progression convention (Foundation → PVC → DSR → v1.0 → Post-v1.0) with version numbering ranges, 8-section milestone anatomy checklist (including the one-sentence goal rule), staged index tables with 7 columns including a goal-sentence column that mirrors the milestone file's `## Goal`, 6-step "Creating a new milestone" procedure (index first, document second), and a 5-row backlog candidates table with size cap.

---

## 2026-08-08

### Changed

- Standardised audience blockquotes across 4 docs per AGENTS.md §3: added `> **Audience:** X.` / `> **Purpose:** Y.` format to docs/README.md, docs/business/Digital_Sales_Room_for_Events/README.md, docs/platform/intelligence.md, and docs/business/Personal_Venue_Catalog/comparison.md.
- Normalised navigation footers per AGENTS.md §5: removed redundant top-of-file `**Docs:**` nav blocks from docs/platform/architecture.md, docs/platform/intelligence.md, and docs/business/Personal_Venue_Catalog/comparison.md so the footer nav appears exactly once as the last element.
- Removed duplicate metadata pre-footer block (Document type / Stage / Audience lines) from docs/platform/intelligence.md — that information now lives in the standard audience blockquote.
- Updated docs/business/market.md cross-references and footer nav to point to Digital_Sales_Room_for_Events docs as the primary product positioning, replacing Personal_Venue_Catalog links that preceded the DSR direction shift.
- Resolved 17 broken index links in roadmap READMEs: docs/roadmap/epics/README.md (E1–E9), docs/roadmap/milestones/README.md (v0.1–v1.0), docs/roadmap/decisions/README.md (D1–D3) now list the planned files as plain text rather than linking to files that have not yet been written.
- Expanded docs/platform/architecture.md Docs footer nav to include Intelligence Layer and Vision links, matching the other platform docs.

---

## 2026-08-03

### Added

- `docs/business/Digital_Sales_Room_for_Events/README.md` — concept overview and document index for the DSR positioning; explains the two-layer architecture (Personal Venue Catalog as data layer, Digital Sales Room as output layer).
- `docs/business/Digital_Sales_Room_for_Events/product.md` — product structure for the DSR concept: two-layer architecture, five capability pillars (ingestion, catalog, search, pitch board, approval/snapshot), UX concept for agency and client sides, positioning against discovery platforms and generic proposal tools.
- `docs/business/Digital_Sales_Room_for_Events/proposal.md` — business case: ICP matrix with updated pricing ($150–300/mo), feature phases, unit economics, GTM playbook (concierge onboarding + viral loop via pitch board), risks, and open questions.
- `docs/business/Digital_Sales_Room_for_Events/pitch-mechanics.md` — pitch board mechanics: micro-site structure, event context header, venue cards filtered by brief, collaboration layer (contextual comments, markers, activity feed), approval → immutable snapshot, white-label tiers.
- `docs/business/Digital_Sales_Room_for_Events/cold-start.md` — cold start strategy for DSR: two cold-start problems (empty catalog + no demo story), city-focused launch rationale, why focused geography eases outreach, seed catalog build process, concierge onboarding model, validation milestones before self-serve.
- `docs/business/Digital_Sales_Room_for_Events/comparison.md` — competitive landscape for DSR: five category sections (DSR/proposal tools, venue discovery, agency CRM/ops, venue management, DIY stack), per-category tables, unified capability matrix, durable edge analysis.

### Changed

- `docs/roadmap/vision.md` — fully rewritten to reflect DSR positioning: updated one-sentence, two-role audience (agency + client), two-layer product structure, new strategic bets (pitch board as buying trigger, approval snapshot for dispute prevention, client onboarding = open a link), north star metric changed from "weekly active searches" to "pitch boards approved per team per month".
- `docs/README.md` — fully rewritten: problem framing expanded to include scattered feedback and missing agreed spec, two-layer structure section added, pricing updated to $150/$300, nav bar updated with Competitive Landscape link.
- `README.md` — DSR section expanded from one row to a full six-row table; blank line formatting fixed.
- `AGENTS.md` — §2 repository structure tree updated to reflect `Digital_Sales_Room_for_Events/` and `Personal_Venue_Catalog/` subdirectories; file naming rule updated with explicit PascalCase exception for concept containers; §4.8 and §4.9 rewritten to reference new file paths and DSR document conventions.

### Restructured

- `docs/business/` — original flat business documents (`product.md`, `proposal.md`, `comparison.md`, `cold-start.md`, `sales/`) moved into `docs/business/Personal_Venue_Catalog/`. File `market.md` remains at `docs/business/market.md` — still relevant across both positioning layers. All internal links updated.
- All files in `docs/business/Personal_Venue_Catalog/` marked with `[!NOTE]` callout identifying them as segment reference documents for the catalog data-layer positioning, not the primary product direction.

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

- `docs/README.md` — plain-language product overview. Audience: anyone. Covers the problem, what OiQb does, who it is for, how it works, monetization model, and one-sentence summary.
- `docs/business/proposal.md` — full business proposal: ICP matrix, feature phases (Phase 1–3), monetization tiers, go-to-market strategy, international expansion plan, key risks, and open questions.
- `docs/business/comparison.md` — competitive landscape analysis: Cvent, Tripleseat, Momentus, VenueScanner, VenueFindAI, VenueArc, Spark (GEVME/PCMA), Bynder, Brandfolder, Unstructured.io, Docling, Apache Tika. Includes gap summary matrix.
- `docs/platform/architecture.md` — architecture reference: platform context, domain model (Venue, VenueAsset, ExtractionJob, MetadataEvent), metadata aggregation engine, service architecture (oiqb-venue-service, oiqb-venue-ingestion-worker, oiqb-venue-model), ETL pipeline, search architecture, API surface, event contracts, plan entitlement mapping, database schema, UI integration, observability, security, technology decisions, and open decisions.
- `docs/platform/intelligence.md` — intelligence layer reference: Spring AI ETL pipeline, asset-type processing matrix, chunking strategy, Apache Tika rationale, Docling integration, venue-specific extraction schema, confidence-sourced metadata model, multi-source aggregation design, scalability architecture, and technology decisions summary.
