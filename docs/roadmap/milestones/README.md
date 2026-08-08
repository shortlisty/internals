# Milestones

> **Audience:** Product, engineering.
> **Purpose:** One file per shippable increment. Milestones describe what ships together, when it is done, and what was deferred.

---

## What is a milestone

A milestone is a releasable unit of work with a clear goal sentence. It references the epics it advances, carries a feature checklist, and defines how we know it is done. Milestones do not carry planned dates until they reach `In progress` status — see [AGENTS.md § 4.5](../../../AGENTS.md#45-milestones-docsroadmapmilestones).

The unit of value is the **shipped increment**. If a set of features cannot be demonstrated end-to-end to a real user in one session, it does not belong in one milestone — split it.

Each milestone file follows the template in [AGENTS.md § 4.5](../../../AGENTS.md#45-milestones-docsroadmapmilestones).

---

## Milestone lifecycle

Every milestone has one status. Status transitions are forward-only — a milestone marked `Completed` never moves back.

| Status      | Meaning                                                                                                                              | Entry condition                                                                                                                                                                                                         |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Planned     | The milestone is defined in the index with a goal sentence, included epics, and deferred items. No implementation work is committed. | Milestone file created with all required sections (see anatomy below). All referenced epics exist in the [epics index](../epics/README.md) with status at least `Not started`. No dates written.                        |
| In progress | Implementation is active. Work is merged against the milestone's feature checklist.                                                  | At least one included epic is `In progress`. At least one feature checkbox `- [ ]` is being worked on (first PR merged or in review). If the milestone needs a date, write it **only** when transitioning here.         |
| Completed   | All acceptance criteria met, all feature checkboxes ticked, all deferred items moved out to the next milestone.                      | Full end-to-end demo runs against the acceptance criteria. CHANGELOG.md has the one-paragraph "what shipped" entry for this milestone. All included epics are either `Done` or explicitly split into a later milestone. |

**Status transitions:** `Planned` → `In progress` → `Completed`.

---

## How milestones are sequenced

### Layer-first progression

Milestones follow the product layer architecture from [Vision](../vision.md#product-structure). The progression is always:

```
Foundation  →  Personal Venue Catalog (data layer)  →  Digital Sales Room (output layer)  →  Platform (v1.0)  →  Post-v1.0 scaling
```

You cannot ship the Digital Sales Room (output layer) before the Personal Venue Catalog (data layer) has at least the core ingestion + extraction + search working. The catalog feeds the pitch board — no catalog, no board.

### Version numbering convention

| Version range | Meaning                                                                                                                     |
| ------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `v0.1 – v0.5` | Pre-product-market-fit increments. Each adds one major capability block. Not yet commercially viable.                       |
| `v1.0`        | First commercially viable release. All three layers (Foundation / PVC / DSR) work end-to-end. Pitch boards can be approved. |
| `v1.1, v1.2…` | Post-launch enhancements. Integrations, deeper collaboration, performance.                                                  |
| `v2.0`        | Major architectural shift (e.g., marketplace launch, multi-vertical).                                                       |

### File naming

Each milestone file is named `v{X}.{Y}-{slug}.md` — for example, `v0.1-mvp.md`. The slug matches the milestone's title in plain language, lowercase, hyphen-separated.

---

## Milestone anatomy (required sections)

Every milestone file must contain all of the following sections, in this order. The template is formally defined in [AGENTS.md § 4.5](../../../AGENTS.md#45-milestones-docsroadmapmilestones).

| #   | Section             | What goes in it                                                                                                                                                                |
| --- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Title + audience    | `# v{X}.{Y} — Title` on line 1. Audience blockquote immediately below (always `Product, engineering` for milestones).                                                          |
| 2   | Goal                | **One sentence.** What does the user get after this milestone ships that they did not have before? No run-on sentences.                                                        |
| 3   | Epics included      | Bulleted list of links to the [epics](../epics/README.md) this milestone advances. Not all user stories in each epic need to ship — that is captured in the feature checklist. |
| 4   | Feature checklist   | `- [ ]` checkboxes. One checkbox per user-visible behaviour. This is the daily tracking mechanism.                                                                             |
| 5   | Acceptance criteria | Measurable, testable statements that define "done." Not subjective. If it needs a human judgement call, rewrite it.                                                            |
| 6   | Deferred items      | Bulleted list of what was explicitly cut from this milestone and moved to the next one. Prevents scope creep.                                                                  |
| 7   | Status              | Single line: `**Status:** Planned` / `In progress` / `Completed`. If `Completed`, also write the date here.                                                                    |
| 8   | Navigation footer   | Standard `**Docs:**` footer (see [AGENTS.md § Links](../../../AGENTS.md#links)).                                                                                               |

Before creating a new milestone file, verify all 8 sections against this checklist.

---

## Milestones index

### v0.x — Pre-product-market-fit (building the layers)

| Version | File                                                         | Title                    | Goal sentence                                                                                                         | Included epic groups                   | Included epics    | Status  |
| ------- | ------------------------------------------------------------ | ------------------------ | --------------------------------------------------------------------------------------------------------------------- | -------------------------------------- | ----------------- | ------- |
| v0.1    | [v0.1-mvp.md](v0.1-mvp.md)                                   | MVP foundations          | An event planner can create a venue, upload documents, and see extracted metadata in a basic profile view.            | A (Foundation), B1 (PVC core profiles) | E1, E5 (partial)  | Planned |
| v0.2    | [v0.2-intelligence-layer.md](v0.2-intelligence-layer.md)     | Intelligence layer       | Uploaded documents run through the full ETL pipeline and extracted fields power semantic + structured search.         | A (Foundation), B2 (PVC intelligence)  | E2, E3, E5 (full) | Planned |
| v0.3    | [v0.3-data-quality.md](v0.3-data-quality.md)                 | Data quality             | A planner can verify, override, and see provenance for every extracted field; conflicts surface for human resolution. | B3 (PVC quality)                       | E6                | Planned |
| v0.4    | [v0.4-export-collaboration.md](v0.4-export-collaboration.md) | Export and collaboration | A planner can share a venue shortlist with teammates and export a static proposal document for a client.              | B4 (PVC team), C1 (DSR basic sharing)  | E4, E7            | Planned |

### v1.0 — Commercially viable (complete loop)

| Version | File                                 | Title           | Goal sentence                                                                                                                                      | Included epic groups           | Included epics     | Status  |
| ------- | ------------------------------------ | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ | ------------------ | ------- |
| v1.0    | [v1.0-platform.md](v1.0-platform.md) | Platform launch | An agency can build a venue library, assemble a pitch board, send it to a client, and receive an approved snapshot with a timestamped audit trail. | A + B full + C2 (DSR approval) | E1–E7 all included | Planned |

### Post-v1.0 — Scaling and deeper capabilities

No milestones committed here yet. Add v1.1+ when v1.0 reaches `In progress`. Candidates are in the backlog candidates table at the bottom of this file.

> **Note on the current index:** This table is the **pre-implementation standardisation snapshot**. Before creating any milestone file, review whether the goal sentences above, epic-to-milestone assignments, and layer progression boundaries are correct. The act of writing `v0.1-mvp.md` should validate that the v0.1 scope is genuinely demoable end-to-end — if not, update this index first, then create the file.

---

## Creating a new milestone

Follow this procedure in order. Do not create the file until step 4 is complete.

1. **Confirm it is a real milestone, not an epic, not a task.**
   - Can you write a **single-sentence goal** that describes what users gain? If not, it is too small (an epic/task) or too large (split it).
   - Can you run an end-to-end demo of the goal in one user session (real or staging environment)? If not, the scope is wrong — shrink it. Milestones are **shippable increments.**

2. **Assign version number and confirm layer position.**
   - Which stage is it in? (v0.x pre-PMF / v1.0 launch / v1.x post-launch / v2.0 major shift.)
   - Use the **next available minor version** within the current stage. Do not skip numbers. If the last v0.x is v0.4, the next is v0.5.
   - Confirm the progression: you cannot create a C-group (DSR) milestone before all B-group prerequisites are at least in a previous milestone.

3. **Map included epics, update the index, then create the file.**
   - Pick epics from the [epics index](../epics/README.md). Every included epic must already exist as a row in that index (status at least `Not started`).
   - Define which epics are **fully included** vs. **partially advanced** (partial is fine — the feature checklist in the milestone file captures exactly which user stories ship).
   - List the **deferred items**: what capabilities from the included epics are explicitly NOT shipping in this milestone. Write them down now; they prevent scope creep later.
   - Update the `Milestones index` table in **this file** first (add the row, mark status `Planned`), save, and commit the index update before creating the milestone file.

4. **Write the milestone file against the anatomy checklist above.**
   - Filename: `v{X}.{Y}-{slug}.md` in this directory.
   - Populate all 8 required sections. The **Goal sentence must be exactly one sentence.**
   - Feature checklist: at least 3 checkboxes, each representing one user-visible behaviour.
   - Acceptance criteria: at least 3 testable statements. No "the UX is good" or "performance is acceptable" — write concrete, measurables.
   - Mark `Status: Planned` in the file (never start with `In progress`). **Do not write dates until status moves to `In progress`.**

5. **Cross-reference all included epics.**
   - Open each included epic file. Ensure its `Milestone references` section links back to this milestone.
   - If an epic does not have a file yet, create it following the [epics README procedure](../epics/README.md#creating-a-new-epic) **before** finalising this milestone.

6. **Final check before commit.**
   - Milestone file has all 8 anatomy sections? → Yes.
   - Goal is exactly one sentence? → Yes.
   - Index row added with correct version, goal sentence, epic groups, status `Planned`? → Yes.
   - Every included epic has a row in the epics index and a file that links back to this milestone? → Yes.
   - AGENTS.md rules: no planned dates written (status is still `Planned`), audience block correct, relative links only, navigation footer present? → Yes.

---

## Backlog candidates

Milestone-level ideas that have not been formalised yet. Capture them so they are not lost. Promote a candidate to a real milestone by: (1) moving its row out of this table, (2) creating the `v{X}.{Y}` file following the procedure above, (3) updating the milestones index.

| Candidate milestone        | Proposed version | Why it matters                                                                                                                | Blocked by                                                        |
| -------------------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Marketplace foundations    | v1.1             | Venue-side verified profiles, two-sided listing maintenance. Explicitly out of near-term scope per the vision.                | v1.0 must be `Completed` first                                    |
| Advanced search & filters  | v1.1             | Saved searches, filter presets, alerting on new venues matching a saved query.                                                | v0.2 at least `In progress`                                       |
| Multi-tenant admin tooling | v1.2             | Platform admin dashboards, per-tenant usage metrics, support tooling for "find this venue across all tenants" queries.        | v0.1 `Completed`, billing integration epic at least `Not started` |
| Vertical extension launch  | v2.0             | First non-venue domain (medical or agro) using the strategy pattern platform. Validates D8 at production scale.               | v1.0 `Completed` and all v1.x enhancements at least `In progress` |
| Docling full rollout       | v1.0 or v1.1     | High-fidelity table + layout PDF parsing replacing Tika for floor plans and spec sheets. Decision D2 anchors this at Phase 2. | v0.2 `Completed` with Tika baseline accuracy measured             |

Do not let this table grow beyond 8 rows. Review it when any milestone transitions to `Completed`: promote one or two candidates to real next-milestone entries, or delete them with a one-line reason recorded in [CHANGELOG.md](../../../CHANGELOG.md).

---

**Docs:** [Vision](../vision.md) · [Epics](../epics/README.md) · [Decisions](../decisions/README.md) · [Architecture](../../platform/architecture.md) · [Intelligence Layer](../../platform/intelligence.md)
