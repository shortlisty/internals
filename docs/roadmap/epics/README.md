# Epics

> **Audience:** Product, engineering.
> **Purpose:** One file per user-facing capability cluster. Epics describe what we are building and for whom — not how and not when.

---

## What is an epic

An epic is a cluster of related user-facing capabilities that can be described from a user's perspective. It is not a task, not a sprint, and not a service boundary. One epic may span multiple milestones and multiple services.

The unit of value is the user outcome. If a set of capabilities cannot be summarised as "As a [role], I can now [outcome]", it is not one epic — split it.

Each epic file follows the template in [AGENTS.md § 4.4](../../../AGENTS.md#44-epics-docsroadmapepics).

---

## Epic lifecycle

Every epic has one status. Status transitions are forward-only — an epic marked `Done` never moves back.

| Status      | Meaning                                                                                                  | Entry condition                                                                                                                                          |
| ----------- | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Not started | The epic is defined in the index with a target milestone. No implementation work has begun.              | Epic file created with complete sections, dependencies mapped to earlier epics, target milestone assigned.                                               |
| In progress | Implementation is active. Code is being written or merged for the epic's user stories.                   | At least one user story from the epic has a corresponding PR. The target milestone is `In progress`.                                                     |
| Done        | All user stories are shipped, acceptance criteria met, and the containing milestone reached `Completed`. | Containing milestone document marks the epic as included and all feature checkboxes ticked. Epic user stories in the epic file are individually `- [x]`. |

**Status transitions:** `Not started` → `In progress` → `Done`.

---

## How epics are organised

### Product layer groups

Epics are grouped by the product layer they belong to, matching the two-layer architecture defined in [Vision](../vision.md#product-structure):

| Group                                      | Description                                                                                                                                                                            |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **A. Platform Foundation**                 | Infrastructure that must exist before any user-facing capability runs. Tenancy, shared libraries, schema contracts, registry. No end-user value on its own — everything depends on it. |
| **B. Personal Venue Catalog (Data Layer)** | The agency's private library: venue profiles, document ingestion, extraction, metadata, search, data quality. Layer 1 per the vision statement.                                        |
| **C. Digital Sales Room (Output Layer)**   | Client-facing pitch boards, the approval loop, snapshots. Layer 2 per the vision statement. The buying trigger — agencies pay because this layer exists.                               |
| **D. Post-v1.0 Enhancements**              | Scaling features, deeper collaboration, integrations, marketplace, vertical extension. Not part of the v1.0 launch scope.                                                              |

### Numbering and ordering

- Epics are numbered `E1, E2, E3, …` sequentially. Numbers are never reused and never reordered.
- Within each group, epics appear in intended implementation order (dependencies first).
- An epic's position in the index implies a soft dependency on epics above it within the same group. Hard cross-epic dependencies are listed explicitly in the `Dependencies` column.

### File naming

Each epic file is named `E{N}-{slug}.md` — for example, `E1-venue-profiles.md`. The slug is lowercase, hyphen-separated, and matches the epic's title in plain language.

---

## Epic anatomy (required sections)

Every epic file must contain all of the following sections, in this order. The template is formally defined in [AGENTS.md § 4.4](../../../AGENTS.md#44-epics-docsroadmapepics).

| #   | Section              | What goes in it                                                                                                                  |
| --- | -------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Title + audience     | `# E{N} — Title` on line 1. Audience blockquote immediately below (always `Product, engineering` for epics).                     |
| 2   | Summary              | 2–3 sentence plain-language description of the user outcome. What can users do after this epic ships that they cannot do before? |
| 3   | User stories         | Bulleted list, each line: `As a [role], I want [action] so that [outcome].` Minimum 3 stories per epic.                          |
| 4   | In scope             | Concrete list of capabilities included. Bullet points, not narrative.                                                            |
| 5   | Out of scope         | What this epic explicitly does **not** include. Prevents scope creep. Even if the list feels obvious, write it.                  |
| 6   | Milestone references | Which milestones include this epic (usually ≥1). Links to the relevant [milestone files](../milestones/README.md).               |
| 7   | Open questions       | `- [ ]` checkboxes for unresolved questions. Mark `- [x]` with resolution inline when answered.                                  |
| 8   | Status               | Single line: `**Status:** Not started` / `In progress` / `Done`.                                                                 |
| 9   | Navigation footer    | Standard `**Docs:**` footer (see [AGENTS.md § Links](../../../AGENTS.md#links)).                                                 |

Before creating a new epic file, verify all 9 sections against this checklist.

---

## Epics index

### Group A — Platform Foundation

Infrastructure prerequisites. No direct end-user value. All later epics depend on these.

| ID  | File                                                       | Title                 | Target milestone(s)     | Dependencies | Status      |
| --- | ---------------------------------------------------------- | --------------------- | ----------------------- | ------------ | ----------- |
| E1  | [E1-venue-profiles.md](E1-venue-profiles.md)               | Venue profiles        | v0.1-mvp                | —            | Not started |
| E2  | [E2-document-intelligence.md](E2-document-intelligence.md) | Document intelligence | v0.2-intelligence-layer | E1           | Not started |
| E3  | [E3-search.md](E3-search.md)                               | Search                | v0.2-intelligence-layer | E1, E2       | Not started |

### Group B — Personal Venue Catalog (Data Layer)

The agency's private knowledge base. Layer 1 of the product architecture.

| ID  | File                                                 | Title              | Target milestone(s)               | Dependencies | Status      |
| --- | ---------------------------------------------------- | ------------------ | --------------------------------- | ------------ | ----------- |
| E4  | [E4-team-collaboration.md](E4-team-collaboration.md) | Team collaboration | v0.4-export-collaboration         | E1           | Not started |
| E5  | [E5-plan-enforcement.md](E5-plan-enforcement.md)     | Plan enforcement   | v0.1-mvp, v0.2-intelligence-layer | E1, E2       | Not started |
| E6  | [E6-data-quality.md](E6-data-quality.md)             | Data quality       | v0.3-data-quality                 | E2, E3       | Not started |

### Group C — Digital Sales Room (Output Layer)

Client-facing pitch boards and the approval loop. Layer 2 of the product architecture — the buying trigger.

| ID  | File                                         | Title              | Target milestone(s)       | Dependencies | Status      |
| --- | -------------------------------------------- | ------------------ | ------------------------- | ------------ | ----------- |
| E7  | [E7-export-sharing.md](E7-export-sharing.md) | Export and sharing | v0.4-export-collaboration | E1, E3, E6   | Not started |

### Group D — Post-v1.0 Enhancements

Beyond the v1.0 platform launch. Scaling and deeper capabilities.

| ID  | File                                     | Title        | Target milestone(s) | Dependencies | Status      |
| --- | ---------------------------------------- | ------------ | ------------------- | ------------ | ----------- |
| E8  | [E8-integrations.md](E8-integrations.md) | Integrations | — (post-v1.0)       | E1, E7       | Not started |
| E9  | [E9-marketplace.md](E9-marketplace.md)   | Marketplace  | — (post-v1.0)       | E1, E3, E7   | Not started |

> **Note on the current index:** This table is the **pre-implementation standardisation snapshot**. Before any epic file is created, review whether the grouping, target milestone assignments, and dependency chains above are final. The act of writing E1 should validate whether the Group A / Group B boundary is correct — if not, update this index first, then create the file.

---

## Creating a new epic

Follow this procedure in order. Do not create the file until step 4 is complete.

1. **Confirm it is a real epic, not a task, not a milestone.**
   - Can you summarise it as "As a [role], I can now [outcome]"? If not, it is too small (a task) or too large (split it).
   - Does it span more than one milestone? If yes, it is correctly scoped as an epic. Milestones reference epics, not the reverse.

2. **Map dependencies and assign a group.**
   - Which product layer group does it belong to? (A / B / C / D above.)
   - Which epics must be `Done` before implementation can begin? List them explicitly.
   - If no existing epic is a prerequisite, it probably belongs in Group A.

3. **Assign target milestone(s) and the next available E{N} number.**
   - Pick the milestone(s) from the [milestones index](../milestones/README.md). An epic can span multiple milestones.
   - Use the **next unused integer**. Do not skip numbers. Do not reuse numbers from deleted epics.
   - Update the `Epics index` table in **this file** first (add the row, mark status `Not started`), save, and commit the index update before creating the epic file.

4. **Write the epic file against the anatomy checklist above.**
   - Filename: `E{N}-{slug}.md` in this directory.
   - Populate all 9 required sections (Epic anatomy above). For user stories, write at least 3.
   - For open questions: if everything is resolved, write `- [x] No open questions at this time.` rather than leaving the section empty.
   - Mark `Status: Not started` in the file (never start with `In progress`).

5. **Cross-reference the milestone.**
   - Open the relevant milestone file(s) and add the epic to the `Epics included` section with a link.
   - If the milestone file does not exist yet, create it following the [milestones README procedure](../milestones/README.md#creating-a-new-milestone) **before** linking to it.

6. **Final check before commit.**
   - Epic file has all 9 anatomy sections? → Yes.
   - Index row added with correct group, dependencies, status `Not started`? → Yes.
   - At least one milestone file references this epic? → Yes.
   - AGENTS.md rules: audience block correct, no `##` title case issues, relative links only, navigation footer present? → Yes.

---

## Backlog candidates

Ideas that have not been formalised into epics yet. Capture them here so they are not lost. Promote a candidate to a real epic by: (1) moving its row out of this table, (2) creating the `E{N}` file following the procedure above, (3) updating the epics index.

| Candidate name                | Proposed group | Why it matters                                                                                                               | Blocked by                                      |
| ----------------------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| Metadata schema versioning UI | B (PVC)        | Users viewing and managing `_schema_version` state for their own venues; forced-convergence admin actions.                   | E2, E6 written first                            |
| Vertical extension tooling    | D (Post-v1.0)  | Scaffolding generator for new domain libraries (medical, agro) — strategy pattern implementation aids.                       | E1–E7 all at least In progress                  |
| CAD file visual processing    | B (PVC)        | DWG/DXF → PNG conversion → GPT-4o vision for floor plan layout understanding. Currently "metadata only" per architecture §5. | E2 at Done, Docling D2 Phase 2 decision         |
| Video walkthrough extraction  | B (PVC)        | Keyframe extraction + Whisper transcription + scene-level amenity detection. Out of scope Phase 1.                           | E2 at Done, E6 In progress                      |
| Multi-stakeholder voting      | C (DSR)        | Client-side multi-approver workflow with weighted voting (e.g., brand team + events manager + finance).                      | E7 at Done, approval snapshot epic exists first |

Do not let this table grow beyond 10 rows. Review it monthly: promote candidates to real epics, or delete them with a one-line reason recorded in [CHANGELOG.md](../../../CHANGELOG.md).

---

**Docs:** [Vision](../vision.md) · [Milestones](../milestones/README.md) · [Decisions](../decisions/README.md) · [Architecture](../../platform/architecture.md) · [Intelligence Layer](../../platform/intelligence.md)
