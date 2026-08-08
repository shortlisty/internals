# Feature checklist

> **Audience:** Product, engineering.
> **Purpose:** Mid-level prioritised checkbox list of every user-visible feature, cutting across epics and milestones so the next thing to build is visible in under 60 seconds.

---

## What this document is

This is the flattened, prioritised view of the roadmap. It lives between:

- **Epics** (`epics/README.md`) — large user-facing capability clusters (too big for daily planning).
- **Milestones** (`milestones/README.md`) — versioned shippable increments, organised by product layer.

Every feature line here maps to exactly one epic and one milestone. It is a view, not a third source of truth. If a feature's scope or priority changes, the epic and milestone files are updated in the same commit as this document.

---

## Priority tiers

Priority is version-bound — it is not a free-ranked "1–5" or "high/medium/low."

| Tier | Label                | Meaning                                                                                                          |
| ---- | -------------------- | ---------------------------------------------------------------------------------------------------------------- |
| P0   | Highly prioritized   | Cannot ship **v0.1 MVP** without these. Implementation begins day one.                                           |
| P1   | Well prioritized     | Cannot ship **v1.0 commercial launch** without these. Implementation begins after P0 or when blocked on P0 work. |
| P2   | Mid priority         | Nice-to-have for v1.0. Can be cut to v1.1 if scope pressure demands.                                             |
| P3   | Post-v1.0 candidates | Committed idea, not committed to v1.0 timeline. Promoted only when v1.0 reaches `In progress`.                   |

---

## How to read a feature line

Each line follows this exact format:

```
- [ ] Feature description in user-visible language · Epic: E{N}-slug · Milestone: v{X}.{Y}-slug · Priority: P{N}
```

- **Epic / Milestone identifiers:** While the target `E{N}-{slug}.md` or `v{X}.{Y}-{slug}.md` file has not been created yet, the identifier is written as plain text (no markdown link), matching the convention used in the epics and milestones index tables. When the real file is created, convert the plain-text identifier to a relative markdown link in the same commit.
- **Tick `- [x]` only when:** The feature is fully demoable end-to-end to a real user in its target milestone staging environment. "Works on my machine" or a unit-test pass is not enough.
- **Never write:** Dates, engineer assignees, or percentage markers. Checkboxes only.

---

## P0 — Highly prioritized (cannot ship v0.1 MVP without)

### Group A — Platform Foundation

- [ ] Agency owner can sign up and create a tenant account (email + password; tenant isolation bootstrap) · Epic: E5-plan-enforcement · Milestone: v0.1-mvp · Priority: P0
- [ ] Tenant isolation at login: users from one agency cannot see venues or documents from another agency · Epic: E5-plan-enforcement · Milestone: v0.1-mvp · Priority: P0
- [ ] Account settings page shows tenant name, user role (owner), and plan tier · Epic: E5-plan-enforcement · Milestone: v0.1-mvp · Priority: P0

### Group B — Personal Venue Catalog (Data Layer)

- [ ] Event planner can create a venue with core fields: name, address, city, capacity, venue category, and one-paragraph visible summary · Epic: E1-venue-profiles · Milestone: v0.1-mvp · Priority: P0
- [ ] Event planner can upload documents (PDF, DOCX, images) to a venue and see them listed on the venue profile page · Epic: E1-venue-profiles · Milestone: v0.1-mvp · Priority: P0
- [ ] Event planner can view a venue profile page with all core fields rendered + a list of attached documents (basic profile view) · Epic: E1-venue-profiles · Milestone: v0.1-mvp · Priority: P0
- [ ] Event planner can edit the core fields of a venue they own (override or correct the typed data) · Epic: E1-venue-profiles · Milestone: v0.1-mvp · Priority: P0
- [ ] Event planner can delete a venue they own and confirm it no longer appears in any list · Epic: E1-venue-profiles · Milestone: v0.1-mvp · Priority: P0

---

## P1 — Well prioritized (cannot ship v1.0 commercial launch without)

### Group A — Platform Foundation

- [ ] Plan tier limits enforced at API: tenant venue count and uploaded-GB cap per Free / Pro / Business tier, enforced at venue-create and upload endpoints · Epic: E5-plan-enforcement · Milestone: v0.2-intelligence-layer · Priority: P1
- [ ] Agency owner can invite a team member (editor or viewer role) via email; invitee joins the same tenant catalogue on acceptance · Epic: E4-team-collaboration · Milestone: v0.4-export-collaboration · Priority: P1
- [ ] Role-based access control: viewer role is read-only (no create/edit/delete); editor role cannot invite or remove members · Epic: E4-team-collaboration · Milestone: v0.4-export-collaboration · Priority: P1
- [ ] Shared venue catalogue: every member of a tenant sees the same set of venues, documents, and edits by default · Epic: E4-team-collaboration · Milestone: v0.4-export-collaboration · Priority: P1

### Group B — Personal Venue Catalog (Data Layer)

- [ ] PDFs uploaded to a venue run through the extraction pipeline and extracted text is stored on the venue record · Epic: E2-document-intelligence · Milestone: v0.2-intelligence-layer · Priority: P1
- [ ] DOCX and plain-text documents run through extraction alongside PDFs · Epic: E2-document-intelligence · Milestone: v0.2-intelligence-layer · Priority: P1
- [ ] Field-specific extraction: capacity, full address, venue category, contact name, and contact email are populated from uploaded documents · Epic: E2-document-intelligence · Milestone: v0.2-intelligence-layer · Priority: P1
- [ ] Extraction confidence score per field is visible alongside each extracted field value · Epic: E2-document-intelligence · Milestone: v0.2-intelligence-layer · Priority: P1
- [ ] Venue images and floor plan files are stored as assets; venue profile page renders a photo gallery thumbnail strip · Epic: E2-document-intelligence · Milestone: v0.2-intelligence-layer · Priority: P1
- [ ] Keyword search across venue name, core fields, and extracted document text returns matching venues · Epic: E3-search · Milestone: v0.2-intelligence-layer · Priority: P1
- [ ] Semantic search over venue descriptions and extracted document chunks returns conceptually-matching venues via pgvector embeddings · Epic: E3-search · Milestone: v0.2-intelligence-layer · Priority: P1
- [ ] Search filters: planner can filter results by capacity range, venue category, and city · Epic: E3-search · Milestone: v0.2-intelligence-layer · Priority: P1
- [ ] Planner can add or remove custom tags on a venue (e.g. "outdoor", "rooftop", "wedding-ready") for later filtering · Epic: E1-venue-profiles · Milestone: v0.2-intelligence-layer · Priority: P1
- [ ] Low-confidence extracted fields are visually highlighted on the venue profile as "needs human review" · Epic: E6-data-quality · Milestone: v0.3-data-quality · Priority: P1
- [ ] Planner can resolve a field-level conflict (document A says 250 pax, document B says 300) by picking a source or typing an override value · Epic: E6-data-quality · Milestone: v0.3-data-quality · Priority: P1

### Group C — Digital Sales Room (Output Layer)

- [ ] Planner can build a shortlist: add venues to a named saved list, remove venues, and reorder them · Epic: E7-export-sharing · Milestone: v0.4-export-collaboration · Priority: P1
- [ ] Planner can export a shortlist to a branded static PDF proposal with cover header + per-venue cards · Epic: E7-export-sharing · Milestone: v0.4-export-collaboration · Priority: P1
- [ ] Planner can generate a read-only pitch board link and send it to a client; the client opens it without signing up · Epic: E7-export-sharing · Milestone: v0.4-export-collaboration · Priority: P1
- [ ] Client clicks "Approve" on a pitch board; the system writes an immutable approval snapshot with timestamp, approver identity, and the exact venue set frozen at moment of approval · Epic: E7-export-sharing · Milestone: v1.0-platform · Priority: P1
- [ ] Agency side can view the audit list of every approved snapshot (who approved, when, which shortlist) · Epic: E7-export-sharing · Milestone: v1.0-platform · Priority: P1

---

## P2 — Mid priority (nice-to-have for v1.0; cut to v1.1 if scope pressure)

### Group A — Platform Foundation

- [ ] Account settings page shows current plan usage: venues used / cap, GB uploaded / cap · Epic: E5-plan-enforcement · Milestone: v1.0-platform · Priority: P2
- [ ] When a tenant hits their venue-count or upload-GB limit, the UI blocks the action and shows an upgrade CTA with tier pricing · Epic: E5-plan-enforcement · Milestone: v1.0-platform · Priority: P2
- [ ] Soft-deleted venues can be restored within 30 days via a recycle-bin view · Epic: E1-venue-profiles · Milestone: v1.0-platform · Priority: P2

### Group B — Personal Venue Catalog (Data Layer)

- [ ] Each extracted field value shows its source provenance badge ("From: floorplan_v3.pdf p.2") when clicked · Epic: E2-document-intelligence · Milestone: v0.3-data-quality · Priority: P2
- [ ] Keyword + semantic results are blended with Reciprocal Rank Fusion (RRF) and deduplicated · Epic: E3-search · Milestone: v0.3-data-quality · Priority: P2
- [ ] Search results display a "matched in snippet" preview highlight for the top matching fragment per venue · Epic: E3-search · Milestone: v0.3-data-quality · Priority: P2
- [ ] Data completeness score per venue (% of recommended fields populated with non-null, non-low-confidence values) · Epic: E6-data-quality · Milestone: v0.3-data-quality · Priority: P2
- [ ] Duplicate venue detection: venues with identical addresses surface a merge prompt for the planner · Epic: E6-data-quality · Milestone: v1.0-platform · Priority: P2
- [ ] Team members can leave inline comments on a venue profile (visible to the agency team only, not to clients) · Epic: E4-team-collaboration · Milestone: v1.0-platform · Priority: P2

### Group C — Digital Sales Room (Output Layer)

- [ ] Client can leave per-venue or per-board comments on a pitch board (feedback visible to the agency) · Epic: E7-export-sharing · Milestone: v1.0-platform · Priority: P2
- [ ] Pitch board white-label branding: agency can upload their logo and pick header colours for exported PDFs and shared links · Epic: E7-export-sharing · Milestone: v1.0-platform · Priority: P2
- [ ] Approval snapshot opens a dedicated audit-log page showing per-field state at moment of approval · Epic: E7-export-sharing · Milestone: v1.0-platform · Priority: P2

---

## P3 — Post-v1.0 candidates (committed idea, not committed to v1.0 timeline)

### Group B — Personal Venue Catalog (Data Layer)

- [ ] CAD/DWG floor plan file → PNG conversion → layout-aware amenity extraction via vision model · Epic: E2-document-intelligence · Milestone: — (post-v1.0) · Priority: P3
- [ ] Venue walkthrough videos: keyframe extraction + Whisper transcription → scene-level amenity tagging · Epic: E2-document-intelligence · Milestone: — (post-v1.0) · Priority: P3
- [ ] Saved search presets: planner saves a "wedding 150 pax + outdoor" query as a reusable filter · Epic: E3-search · Milestone: — (post-v1.0) · Priority: P3
- [ ] New-venue alerting: weekly email when a new venue matching a saved search preset appears in the catalogue or registry · Epic: E3-search · Milestone: — (post-v1.0) · Priority: P3
- [ ] Tenant-wide metadata-schema versioning UI: view `_schema_version` state and run forced-convergence admin actions per venue · Epic: E6-data-quality · Milestone: — (post-v1.0) · Priority: P3
- [ ] Vertical-extension library scaffolding generator: add medical or agro venue sub-types using the strategy-pattern extension hooks · Epic: E1-venue-profiles · Milestone: — (post-v1.0) · Priority: P3

### Group C — Digital Sales Room (Output Layer)

- [ ] Multi-stakeholder client approval: weighted voting (brand team + events manager + finance) with per-role approval thresholds · Epic: E7-export-sharing · Milestone: — (post-v1.0) · Priority: P3

### Group D — Post-v1.0 Enhancements (E8 / E9)

- [ ] Export venue shortlist to Google Sheets (one row per venue, one column per extracted field) · Epic: E8-integrations · Milestone: — (post-v1.0) · Priority: P3
- [ ] CRM sync (HubSpot / Salesforce): push shortlist + client contact to a CRM deal record on board approval · Epic: E8-integrations · Milestone: — (post-v1.0) · Priority: P3
- [ ] When a pitch board is approved, auto-attach a calendar booking link (Cal.com / Google Calendar) for the next call · Epic: E8-integrations · Milestone: — (post-v1.0) · Priority: P3
- [ ] Venue-owner verified profile in a public marketplace (two-sided listing, distinct from agency private catalogues) · Epic: E9-marketplace · Milestone: — (post-v1.0) · Priority: P3
- [ ] Agency planner can send a one-click introduction message to a venue owner from the marketplace venue card · Epic: E9-marketplace · Milestone: — (post-v1.0) · Priority: P3
- [ ] Marketplace venues can be searched across the public registry before being copy-on-imported into the tenant's private library · Epic: E9-marketplace · Milestone: — (post-v1.0) · Priority: P3

---

**Docs:** [Vision](vision.md) · [Epics](epics/README.md) · [Milestones](milestones/README.md) · [Decisions](decisions/README.md) · [Architecture](../platform/architecture.md) · [Intelligence Layer](../platform/intelligence.md)
