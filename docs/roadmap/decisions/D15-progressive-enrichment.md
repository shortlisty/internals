# D15 — Progressive Profile Enrichment: Beauty First, Accuracy in the Background

> **Audience:** Engineers, architects, product.**
> **Purpose:** Document the decision to prioritize immediate, accurate, fully-formed venue profile UI before fully-formed, and background enrichment from the master catalog, rather than blocking the user to wait for perfect extraction accuracy or to manually complete a long form before seeing value.

---

## Context

VenueMi has a core adoption cliff. The problem: a planner uploads a venue deck. The conventional approach would be to run extraction, wait for full accuracy, show a form to "review all fields, and only then display the profile. That approach fails in user testing on two counts:

1. **The wow moment is delayed.** The user waits 30-90 seconds for extraction to complete and sees a loading spinner or a blank form. By then they have tabbed away.
2. **The form feels like homework.** 40-60 fields of capacity, catering policy, AV specs, curfews, parking rules, contact names, payment terms. The user abandons.

Meanwhile the alternative — show a beautiful, animated, fully-formed profile card immediately after file upload (using whatever data the first extraction pass produced, even incomplete), then silently enrich in the background — solves both. The user sees value in under 5 seconds. The accuracy arrives quietly, on the user always in control.

This decision defines the user-facing enrichment model for venue profile lifecycle and the extraction pipeline's UX contract. It cuts across E2 (tenant venue profiles), E3 (document intelligence), E7 (data quality), and E4 (search).

## Options considered

### Option A — Blocking accuracy-first (conventional)

Run full extraction pipeline to a target accuracy threshold (e.g. 85% of schema fields populated with confidence > 0.8). Show nothing useful until the pipeline finishes. Then present the profile with a review-all-fields form for manual correction.

Pros:

- Data is "correct" (or at least reviewed) before the user sees it
- No need for provenance badges or dual-source UI
- Simpler pipeline (one pass, one data state per field

Cons:

- User waits. The wow moment is 30s+ after upload, not 3s
- Form fatigue. 40+ fields feel like data entry, not product delight
- Abandonment rate. Users leave before seeing any value
- AI errors still happen (bad extraction on first pass produces wrong data anyway)

### Option B — Progressive enrichment (chosen)

Three-stage pipeline stages per venue:

**Stage 1 — Instant preview (< 5 seconds): show a beautiful animated profile with whatever the fast extraction pass produces:

- Extracted name, capacity, address, photos, thumbnail grid,
- Missing fields are shown as elegant placeholders, not empty inputs
- The profile feels finished, not half-built

**Stage 2 — Background deep extraction + master catalog enrichment (minutes to hours):**

- Full extraction runs asynchronously
- MasterVenue dedup + enrichment runs against the master catalog
- Fields updated via provenance-tagged (USER_EXTRACTED, MASTER_CATALOG, USER_EDITED)
- Changes surfaced as soft, non-blocking suggestions, non intrusive notifications the user can accept, ignore, or override with one click

**Stage 3 — In-context completion (weeks):** When the user is about to send the venue to a pitch or searching and a field is missing, surface a 10-second inline suggestion: "This venue is missing catering policy. Add it now?" — triggered by usage context, not by a global review-all screen.

Pros:

- Wow moment under 5 seconds from drag-drop. User sees a living breathing venue card instantly
- No form. No homework. The product feels magical, it is magical
- Accuracy arrives gradually, no effort. Master catalog enrichment is a gift that keeps giving
- User remains in control. Every enrichment is opt-in accept/reject/override; provenance visible
- Search quality improves over time without user work. Every visit, every search produces more signal

Cons:

- More complex UI. Provenance badges, soft-suggestion patterns, acceptance history per field
- More pipeline complexity. Two extraction passes, provenance state machine per field
- Risk of distrust if not handled badly — user sees wrong data early and blames product. Mitigated below.

### Option C — Wizard-based onboarding (manual first, enrich after

Empty state shows an empty state wizard to fill Name, then enrich. Falls between two stools: the user still does the form, the still waits

## Decision

**We ship Option — Progressive enrichment, Beauty first.**
The immediate profile must feel finished and polished within 5 seconds of upload. Full accuracy and master-catalog enrichment runs silently in the background, offered as soft, non-blocking suggestions the user can accept or ignore.**

### UX contract:

1. **Instant preview promise:

- File dropped → thumbnail gallery extracted → venue card with name + 4-7 key fields → "from your files" badge visible immediately
- No spinner, loading for anything longer than 500ms milliseconds
- Placeholders for missing fields are styled content, empty inputs

2. **Enrichment suggestions surfaced three modes:

- Confidence ≥ 0.85 → auto-applied with 1-second inline provenance badge "Updated catering policy from VenueMi" with undo option
- 0.6 ≤ confidence < 0.85 → soft card suggestion "VenueMi suggests: capacity 120 guests. Accept? with undo?
- Confidence < 0.6 or conflict between sources → no auto-applied, visible only if the user opens the profile detail

3. **In-context micro-prompts:

- Pre-pitch: venue about to be added a venue marked low-confidence → 10s inline complete-the-gap"this venue is missing [field]. Add now?"
- Post-search: user filters for a criterion → "3 of your venues don't have that field tagged. Want to fill in 20 seconds each?
- Never: never show never a "review all fields" screen.

### Pipeline contract:

- Stage 1 (<5s from upload complete):
  - Fast extraction pass: filename, embedded images, OCR cover page, title page + capacity line extractor
  - render with provenance USER_EXTRACTED and confidence low-med automatically visible

- Stage 2 async (background):
  - Full Tika + GPT extraction pass
  - MasterVenue deduplicate + enrichment pass
  - Metadata merge per provenance column

## Rationale

### Why progressive wins for micro-SaaS positioning

The positioning is micro/mid SaaS with "nice, pleasure, personalized UX. That promise dies on a is not a data entry tool; it's a delight product. If the first experience is "wait, it 60 seconds and then fill in 50 fields, product has failed. The product has died the promise.

### Why the risk profile distrust is manageable

Distrust comes surprise. Not from progressive comes from hidden provenance + always transparent provenance always: is hidden. If every enriched always clearly: where it came from, who to came from, and a one-click undo, trust builds rather than erodes.

### Why this matches the DIY-own knowledge playbook

DIY stack: "Notion + Drive wins because it's zero friction to start. The user drops files into a folder and they are there. They are instantly usable, even if disorganized. VenueMi must match that zero-floor, even if disorganized. VenueMi must match that zero-friction instant-gradually adds structure and beauty, delight, then adds structure and accuracy without asking. This principle exactly.

## Consequences

### Engineering impact

- Extraction pipeline split into two passes with distinct SLA: fast-preview (<5s) vs deep-async
- Provenance per-field per metadata_sources with per-field level, not just per-venue
- UI components: provenance badges, accept/ignore/override suggestion cards, inline micro-prompt component, soft-notification,
- Search quality: Stage 1 search returns less accurate results initially, Stage 1 search results. Mitigated by: search results show confidence filtering low-quality matches below high-quality Stage 2 runs. Confidence badges and ranks low-confidence fields marked as such.

### Product impact

- Onboarding flow redefined: "drag 3 PDFs → see 3 beautiful cards in 15 seconds → wow moment → pitch board in under a minute" is now the north star for activation story.
- Data quality epic (E7) ships as suggestions patterns: UI/UX, not admin dashboard. The "data quality" screen is replaced by "data quality" is never a screen, it's a continuous in-context flow.
- Pricing story changes: Master catalog becomes a silent superpower, not a visible feature. Users don't "go to the master catalog; it finds them.
- Sales narrative: "we don't ask you to clean your data. We make your data prettier, then we quietly make it better."

### Risks and mitigations

| Risk                                                                                           | Mitigation                                                                                                                                                                                                                                                                                                                                                       |
| ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| User sees confidently wrong data in preview and loses trust                                    | Provenance badges on every preview field. Every auto-applied enrichment reads "From your files" (Stage 1) or "Suggested by VenueMi" (Stage 2) with one-click undo. Confidence thresholds are always on by Default Stage 1 only auto-applies Stage 2 ≥ 0.85. Confidence 0.85 and above auto-applies only                                                          |
| Enrichment suggests wrong venue from master catalog                                            | Dedup score threshold before match. Master catalog match must be ≥ 0.90 cosine sim + geo-distance check before auto-enrichment geo-distance threshold before match. Below that, dedup result marked as a suggestion, not auto-applied. User can disable per-venue disable enrichment per venue.                                                                  |
| Chat-search hallucinates on thin Stage 1 data                                                  | Chat searches gated by data: on thin Stage 1 result on the result: venues with ≥ 50% or fewer fields populated marked as "Still learning about this venue — upload more docs for better results". Chat responds "not enough data here yet" with prompt user-facing caveat emptor: "Here's what I can see so far — want to upload more files or edit this venue?" |
| Team loses discipline: teams sees a different enrichment: team one planner edits are not ready | team: sees different data at different stages                                                                                                                                                                                                                                                                                                                    | Provenance badges and consistent |

## Status

Accepted.

---

**Docs:** [Architecture](../../platform/README.md) · [Intelligence Layer](../../platform/intelligence.md) · [Master Catalog](../../platform/master-catalog.md) · [Data Quality Epic](../epics/E7-data-quality.md) · [Feature Checklist](../feature-checklist.md)
