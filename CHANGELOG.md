# Changelog

> Tracks significant changes to this documentation repository — new documents, major rewrites, structural changes.
> This is not a Git commit log. Trivial edits (typos, formatting) are not recorded here.
> Format and rules: [AGENTS.md § 7](AGENTS.md#7-changelogmd-rules).

---

## [Unreleased]

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

- `docs/README.md` — plain-language product overview. Audience: anyone. Covers the problem, what iQ BENE does, who it is for, how it works, monetization model, and one-sentence summary.
- `docs/business/proposal.md` — full business proposal: ICP matrix, feature phases (Phase 1–3), monetization tiers, go-to-market strategy, international expansion plan, key risks, and open questions.
- `docs/business/comparison.md` — competitive landscape analysis: Cvent, Tripleseat, Momentus, VenueScanner, VenueFindAI, VenueArc, Spark (GEVME/PCMA), Bynder, Brandfolder, Unstructured.io, Docling, Apache Tika. Includes gap summary matrix.
- `docs/platform/architecture.md` — architecture reference: platform context, domain model (Venue, VenueAsset, ExtractionJob, MetadataEvent), metadata aggregation engine, service architecture (iqbene-venue-service, iqbene-venue-ingestion-worker, iqbene-venue-model), ETL pipeline, search architecture, API surface, event contracts, plan entitlement mapping, database schema, UI integration, observability, security, technology decisions, and open decisions.
- `docs/platform/intelligence.md` — intelligence layer reference: Spring AI ETL pipeline, asset-type processing matrix, chunking strategy, Apache Tika rationale, Docling integration, venue-specific extraction schema, confidence-sourced metadata model, multi-source aggregation design, scalability architecture, and technology decisions summary.
- `docs/ru/` — Russian translation directory (mirrors `docs/` structure).
