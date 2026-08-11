# AGENTS.md — Documentation Rules for VenueMi System Design

> Rules, structure, and patterns for every document in this repository.
> Any agent, contributor, or tool writing or editing docs here must follow this file.

---

## 1. What this repository is for

This repository is the **single source of truth for VenueMi's pre-build and in-flight system design**. It covers:

- Business rationale and competitive positioning
- Architecture decisions and domain model
- Product roadmap: vision, epics, milestones
- Intelligence layer and ETL pipeline design

It is **not** a code repository, a changelog of Git commits, or a deployment runbook. Do not put operational or infrastructure runbooks here.

---

## 2. Repository structure

```
system-design-documentation/
├── AGENTS.md                          ← This file. Read before writing anything.
├── CHANGELOG.md                       ← Product-level doc changelog (not Git log)
├── README.md                          ← Entry point. Navigation table only.
└── docs/
    ├── README.md                      ← Plain-language product overview (any audience)
    ├── business/
    │   ├── market.md                  ← Event chain, market structure, the vacant slot
    │   ├── Digital_Sales_Room_for_Events/   ← Primary product direction
    │   │   ├── README.md              ← Concept overview and document index
    │   │   ├── product.md             ← Two-layer architecture, capability pillars, UX
    │   │   ├── proposal.md            ← ICP, monetisation, GTM, risks
    │   │   ├── pitch-mechanics.md     ← Micro-site structure, collaboration, approval snapshot
    │   │   ├── cold-start.md          ← Seed catalog, concierge onboarding, city expansion
    │   │   └── comparison.md          ← Competitive landscape and gap matrix
    │   └── Personal_Venue_Catalog/    ← Segment reference — catalog data-layer docs
    │       ├── product.md             ← Tenant app, capability pillars (catalog positioning)
    │       ├── proposal.md            ← ICP, monetisation, GTM (catalog positioning)
    │       ├── comparison.md          ← Competitive analysis (catalog positioning)
    │       ├── cold-start.md          ← Seed strategy (catalog positioning)
    │       └── sales/
    │           ├── pitch.md           ← Demo and intro call narrative flow
    │           ├── battlecards.md     ← Per-competitor positioning for live conversations
    │           ├── objections.md      ← Objection handling: concern, response, follow-up
    │           └── messaging.md       ← Taglines, hero copy, value pillars, naming (copy bank)
    ├── platform/
    │   ├── architecture.md            ← Domain model, services, schema, API, events
    │   └── intelligence.md            ← ETL pipeline, extraction, AI layer
    ├── roadmap/
    │   ├── vision.md                  ← Product direction and strategic bets (not a changelog)
    │   ├── feature-checklist.md       ← Mid-level prioritized product feature checklist (P0–P3)
    │   ├── epics/
    │   │   ├── README.md
    │   │   └── E{N}-{slug}.md
    │   ├── milestones/
    │   │   ├── README.md
    │   │   └── v{X}.{Y}-{slug}.md
    │   └── decisions/
    │       ├── README.md
    │       └── D{N}-{slug}.md
    └── ru/                            ← Russian translations (mirrors docs/ structure - currently on hold)
```

**Rules:**

- Do not add files outside the structure above without updating this file first.
- Do not create nested subdirectories beyond what is shown unless the README.md for that folder is updated to explain the new structure.
- File names are lowercase, hyphen-separated (`my-file-name.md`). No spaces, no underscores, no camelCase. **Exception:** the two `business/` subdirectories use PascalCase with underscores (`Digital_Sales_Room_for_Events/`, `Personal_Venue_Catalog/`) to clearly signal their role as concept containers, not document collections.
- Epic files are prefixed `E{N}-` (e.g. `E1-venue-profiles.md`). Numbers start at 1, increment by 1, never reuse.
- Milestone files are prefixed `v{X}.{Y}-` (e.g. `v0.1-mvp.md`). Align with product version numbering.
- Decision files are prefixed `D{N}-` (e.g. `D1-one-service-vs-two.md`). Numbers start at 1, increment by 1, never reuse.

---

## 3. Audience tagging

Every document must declare its audience in a blockquote immediately after the title:

```markdown
> **Audience:** Engineers, architects.
> **Purpose:** Single source of truth for all technical decisions before development starts.
```

| Audience value          | Who it means                                     |
| ----------------------- | ------------------------------------------------ |
| `Anyone`                | No technical background assumed                  |
| `Founders, team`        | Internal stakeholders, not necessarily technical |
| `Engineering`           | Software engineers building the platform         |
| `Engineers, architects` | Same as above, plus system-level decision makers |
| `Product, engineering`  | Cross-functional audience sharing a decision     |

Do not invent new audience tags. If a document genuinely serves two separate audiences at different depths, split it into two files.

---

## 4. Document types and their rules

### 4.1 Overview / explainer (`docs/README.md`, `docs/business/proposal.md`)

- Written in plain prose. No code blocks unless illustrating a concept with a short example.
- No bullet point walls. Use bullet points only for lists of 3+ items that are genuinely parallel.
- Sections must flow as a narrative that a reader can read top to bottom without jumping around.
- No internal jargon without first defining it in plain language.

### 4.2 Architecture reference (`docs/platform/architecture.md`, `docs/platform/intelligence.md`)

- Audience is always engineers or architects.
- Every major section must have a one-sentence summary before any tables or diagrams.
- Tables are preferred over bullet lists for structured comparative data (columns: field, type, notes).
- Diagrams use ASCII art inline. No external image links unless the image is committed to the repository.
- Code blocks must specify the language: ` ```java `, ` ```sql `, ` ```yaml `, etc.
- Open decisions are tracked with `- [ ]` checkboxes. Resolved decisions are `- [x]` with the resolution inline.

### 4.3 Vision (`docs/roadmap/vision.md`)

- Describes **direction**, not completed work. If something is done, it belongs in CHANGELOG.md or a milestone file, not here.
- Written in present or future tense. No past tense except in the "Where we are now" section.
- No version numbers, sprint numbers, or dates. Vision is timeless; milestones carry dates.
- Maximum 1,500 words. If it exceeds that, the scope is too broad — split into vision + strategy.

### 4.4 Epics (`docs/roadmap/epics/`)

- One epic = one user-facing capability cluster. Not a task, not a sprint, not a service.
- The unit of value is the user outcome. If a set of capabilities cannot be summarised as "As a [role], I can now [outcome]", it is not one epic — split it.
- **Anatomy (required sections, in order):** 1) Title + audience blockquote, 2) Summary (2–3 sentences), 3) User stories (minimum 3), 4) In scope, 5) Out of scope, 6) Milestone references (links), 7) Open questions (checkboxes, resolved `- [x]` inline), 8) Status, 9) Navigation footer.
- User stories follow the format: `As a [role], I want [action] so that [outcome].`
- **Status field is one of:** `Not started` | `In progress` | `Done`. Nothing else. Status transitions are forward-only.
  - `Not started`: Epic file exists with complete anatomy; dependencies mapped to earlier epics; target milestone assigned and milestone file links back.
  - `In progress`: At least one user story has a corresponding PR; the containing milestone is `In progress`.
  - `Done`: All user stories `- [x]` ticked; containing milestone reached `Completed` with all feature checkboxes ticked; no unresolved open `-[ ]` questions remain.
- **Index-first rule:** Before creating a new `E{N}-{slug}.md` file, add its row to the index table in [epics/README.md](docs/roadmap/epics/README.md), commit that index update, then create the file. Never create an epic file first and add it to the index later.
- **Cross-reference rule:** Every epic referenced by a milestone file must, in its own `Milestone references` section, link back to that milestone file. Bidirectional links are mandatory — a one-way link is a drift bug.
- Do not describe implementation details here. Implementation lives in architecture.md.

### 4.5 Milestones (`docs/roadmap/milestones/`)

- One milestone = one shippable increment with a clear goal sentence.
- The unit of value is the shipped increment. If a set of features cannot be demonstrated end-to-end to a real user in one session, it does not belong in one milestone — split it.
- **Goal sentence rule:** The `## Goal` section must contain exactly one sentence. If you need two or more sentences, the milestone scope is wrong — either scope it down or split it.
- **Anatomy (required sections, in order):** 1) Title + audience blockquote, 2) Goal (one sentence), 3) Epics included (links), 4) Feature checklist (`- [ ]`), 5) Acceptance criteria (measurable, minimum 3), 6) Deferred items, 7) Status, 8) Navigation footer.
- Checkboxes `- [ ]` / `- [x]` are the only tracking mechanism. No external issue links.
- **Status field is one of:** `Planned` | `In progress` | `Completed`. Nothing else. Status transitions are forward-only.
  - `Planned`: Milestone file exists with complete anatomy; all referenced epics have index rows at `Not started` minimum; no dates written.
  - `In progress`: At least one included epic is `In progress`; at least one feature checkbox has work active; **if a date is needed, write it only when transitioning here.**
  - `Completed`: All acceptance criteria met; all feature checkboxes `- [x]`; all deferred items have been explicitly moved to the next milestone's `Deferred items` section or to a new milestone; a dated one-paragraph summary has been added to CHANGELOG.md.
- **No planned dates rule:** Never write a date in a milestone file while status is `Planned`. Write one only on the `Planned → In progress` transition. A `Completed` milestone carries the date of completion.
- **Index-first rule:** Before creating a new `v{X}.{Y}-{slug}.md` file, add its row to the index table in [milestones/README.md](docs/roadmap/milestones/README.md), commit that index update, then create the file. Never create a milestone file first and add it to the index later.
- **Cross-reference rule:** Every milestone that references epics must be, in return, listed in each referenced epic's `Milestone references` section. Bidirectional links are mandatory.
- When a milestone reaches `Completed`, add a one-paragraph summary of what shipped to CHANGELOG.md.

### 4.6 Decisions (`docs/roadmap/decisions/`)

- One file = one architectural or strategic decision with clear alternatives considered.
- Must contain: context, options considered, decision made, rationale, consequences, status.
- Status field is one of: `Proposed` | `Accepted` | `Superseded`. Nothing else.
- If a decision is superseded, add a link to the replacement decision file. Do not delete the old file.
- Do not re-litigate accepted decisions in other documents. Link to the decision file instead.

### 4.7 Sales materials (`docs/business/sales/`)

- Audience is always `Founders, team`. Sales docs are internal — they are not shown to prospects.
- **pitch.md** — structured as named beats with a goal per beat and a transition. No script prose. Must include a "what not to do" section.
- **battlecards.md** — one `###` section per competitor. Each card must contain exactly: What they have, What they do not have, When a prospect brings them up (with a verbatim example response in a blockquote), Win condition. The win condition is one sentence.
- **objections.md** — one `###` section per objection. Each entry must contain exactly: the stated objection as a heading, the underlying concern in italics, the response, and the follow-up question. Do not write the response as a rebuttal — acknowledge and reframe first.
- **messaging.md** — a copy bank. Every entry must have a `Status` field: `Candidate`, `Approved`, or `Retired`. Never delete `Retired` entries. Only `Approved` copy is used in external-facing materials. Add a one-line note when retiring an entry.
- Do not duplicate content from `comparison.md` into `battlecards.md`. Battlecards reference the competitive analysis; they do not reproduce it.
- Do not include pricing figures that are not confirmed. Mark uncertain numbers with `(TBC)`.

### 4.8 Product structure (`docs/business/Digital_Sales_Room_for_Events/product.md`)

- Audience is `Founders, team`.
- Explains the product in business terms: the two-layer architecture (Personal Venue Catalog as data layer, Digital Sales Room as output layer), capability pillars, and UX concept for both the agency side and the client side.
- Do not describe implementation details here. For domain model, schemas, and service boundaries, see `architecture.md`.
- Do not duplicate the competitive gap analysis from `comparison.md`. The positioning summary is a one-paragraph conclusion, not a re-analysis.
- Link to epic files for each pillar rather than re-listing their scope.
- The `Personal_Venue_Catalog/product.md` is a segment reference document — it describes the catalog subsystem in its original standalone positioning. It carries a `[!NOTE]` callout and must not be treated as the primary product definition.

### 4.9 Competitive landscape (`docs/business/Digital_Sales_Room_for_Events/comparison.md`)

- Structured as: category sections (one per competitor group), each with a per-tool table, followed by a unified capability matrix.
- Each per-tool table row has exactly: Tool, What it does, Gap vs. VenueMi.
- The capability matrix is a Markdown table with VenueMi in the first column and competitor groups as subsequent columns.
- A "VenueMi's durable edge" section explains why key capabilities cannot be quickly replicated.
- The `Personal_Venue_Catalog/comparison.md` is a segment reference document with a `[!NOTE]` callout. New competitive analysis belongs in the DSR comparison file.

### 4.10 Roadmap index README files (`docs/roadmap/{epics,milestones,decisions}/README.md`)

These READMEs are the **canonical indexes** for their document collections. They carry stronger rules than a typical README because they are the entry point for all roadmap planning — they must be updated _before_ the documents they index are created.

**Shared rules for all three roadmap READMEs:**

- Audience is always `Product, engineering`. Audience blockquote is required (per §3).
- Every real document (`E{N}-*.md`, `v{X}.{Y}-*.md`, `D{N}-*.md`) must have a row in its corresponding README index. Documents without index rows do not exist for planning purposes.
- **Index-first rule (§4.4/§4.5/§4.6):** Update the index README's table(s), save, and commit **before** creating the `E{N}`, `v{X}.{Y}`, or `D{N}` file it describes. Never create the leaf document first and add it to the index later.
- **Backlog candidates section (required for epics/milestones, optional for decisions):**
  - Format: a Markdown table with columns for the concept name, its target placement (group / version / etc.), why it matters, and what blocks it.
  - Purpose: capture concrete but unrefined ideas so they are not lost. Do not let the section become a dumping ground for half-thoughts.
  - Size caps: epics README max 10 rows; milestones README max 8 rows; decisions README no cap because ADR candidates are naturally bounded.
  - Review cadence: on every milestone transition to `Completed`, review the candidates table. Promote one or two to real documents or delete them with a one-line entry in CHANGELOG.md explaining why.
- **Pre-implementation snapshot note:** While no real `E{N}` or `v{X}.{Y}` files have been written yet, the index READMEs must carry a note explicitly stating the tables are the standardisation snapshot and should be validated as correct before the first leaf document is created. This note is removed when the first leaf document is created (it no longer applies).

**Epics README (`docs/roadmap/epics/README.md`) additional rules:**

- Epics are grouped by product layer matching the two-layer architecture defined in vision.md:
  - **Group A — Platform Foundation:** infrastructure only. No direct end-user value.
  - **Group B — Personal Venue Catalog (Data Layer):** agency's private library (Layer 1).
  - **Group C — Digital Sales Room (Output Layer):** client-facing pitch boards + approval (Layer 2, the buying trigger).
  - **Group D — Post-v1.0 Enhancements:** scaling and deeper capabilities. Not part of v1.0 scope.
- Index table columns, in order: `ID | File | Title | Target milestone(s) | Dependencies | Status`.
- Within each group, epics appear in intended implementation order (dependencies first). Dependencies across groups are listed explicitly in the `Dependencies` column.
- `Creating a new epic` section (required): step-by-step procedure mirroring the rules in §4.4.

**Milestones README (`docs/roadmap/milestones/README.md`) additional rules:**

- Milestones are staged by version range, enforcing the layer-first progression: Foundation → PVC (data layer) → DSR (output layer) → v1.0 → Post-v1.0.
- Index table columns, in order: `Version | File | Title | Goal sentence | Included epic groups | Included epics | Status`.
- The `Goal sentence` column must contain exactly one sentence — the same one-sentence content as the milestone file's `## Goal` section. If the file's goal sentence changes, update the index row in the same commit.
- `Creating a new milestone` section (required): step-by-step procedure mirroring the rules in §4.5.

**Decisions README (`docs/roadmap/decisions/README.md`) additional rules:**

- Index table columns, in order: `ID | File | Title | Domain | Status | Superseded by`.
- `Superseded by` column is populated only when a decision reaches `Superseded` status; it links to the replacing `D{N}` file. An `Accepted` decision that is later superseded is never deleted from the index — it stays, its status changes, and the new decision appears as a new row above it (newest first in the index table, while `D{N}` numbers always increment).
- `Creating a new decision` section (recommended): short procedure referencing §4.6, including when to write a decision (at least two genuinely distinct options considered; the choice has consequences beyond one sprint).

### 4.11 Feature checklist (`docs/roadmap/feature-checklist.md`)

The feature checklist is the **mid-level planning document** that sits between epics (large user capability clusters, narrative) and milestones (versioned shippable increments). It flattens the epics×milestones matrix into a single prioritised checkbox list so that anyone — founder, PM, engineer — can scan "what do we build next, what's the priority, what does it ship in" in under 60 seconds.

**Why this document exists:** Epics are too big for day-to-day planning (an epic can span 3 milestones). Milestones are version containers and their checkboxes live inside each `v{X}.{Y}` file. The feature checklist is the only place where priority ranking cuts across epics and milestones — you can see every P0 feature regardless of which epic or milestone it belongs to.

**Rules:**

- Audience is `Product, engineering`.
- **Priority tiers (exactly these 4, version-bound):**
  - **P0 — Highly prioritized.** Cannot ship v0.1 MVP without these. Implementation begins day one.
  - **P1 — Well prioritized.** Cannot ship v1.0 commercial launch without these. Implementation begins after P0 is `Done`, or in parallel when an engineer is blocked on P0.
  - **P2 — Mid priority.** Nice to have for v1.0; can cut to v1.1 if scope pressure demands. Implementation begins only when all P0 + P1 above it are `Done`.
  - **P3 — Post-v1.0 candidates.** Committed idea, not committed to v1.0 timeline. Promoted to P2 only when v1.0 reaches `In progress` and a P2 is cut.
- **One feature = one checkbox line.** If a feature takes more than about 3 engineer-days, split it into more granular checkboxes. If it takes less than 0.5 days, consider merging it with another.
- **Anti-drift rule (hard):** Every single `- [ ]` feature checkbox line MUST include all of the following metadata at the end of the line, in this exact format:
  ```
  · Epic: [E{N}-slug](relative-link) · Milestone: [v{X}.{Y}-slug](relative-link) · Priority: P{N}
  ```
  A feature line without all three links/tags is not committed scope and must be either completed or removed. The checklist is a flattened VIEW over epics + milestones — never a third independent source of truth. If the scope or priority of a feature changes, update the epic/milestone file AND this document in the same commit.
- **Grouping:** Features are grouped by priority tier first (P0 at top, P3 at bottom). Within a tier, group by the product layer from §4.10 (Group A / B / C / D) so related features stay together. No alphabetical ordering.
- **Feature descriptions write in user-visible terms, not implementation terms.** "Event planner can upload a PDF floor plan" (good), not "Implement Tika PDF parser with spatial OCR" (implementation — that belongs in architecture.md).
- **No dates, no assignees, no percentages.** Checkboxes only. No `40%` markers. Tick `- [x]` only when the feature is fully demoable end-to-end to a real user in its target milestone environment. An engineer's "works on my machine" does not qualify.
- **No additional sections beyond:** 1) Title + audience, 2) What this document is, 3) Priority tier definitions, 4) How to read each feature line (legend for the metadata format), 5) P0 block, 6) P1 block, 7) P2 block, 8) P3 block, 9) Navigation footer. No other sections.

---

## 5. Writing rules

### Language

- Write in **English**. Use simple, direct sentences. Prefer active voice.
- Use "we" for decisions and direction. Use "the platform" or the service name for technical descriptions.
- Avoid: "very", "quite", "really", "basically", "simply", "just". Cut them without replacement.
- Avoid: em dashes as a clause separator. Use a period or rewrite.
- Define acronyms on first use in every document: `ETL (Extract, Transform, Load)`.
- Spell out numbers below 10 in prose. Use numerals for 10 and above.

### Headings

- Use `#` for the document title only. One per document.
- Use `##` for top-level sections. Use `###` for subsections. Do not go deeper than `####`.
- Headings are sentence case: `## Domain model`, not `## Domain Model`.
- No punctuation at the end of headings.

### Links

- All internal links are relative: `../platform/architecture.md`, not absolute paths.
- Every document that is part of the `docs/` tree must include a navigation footer:

```markdown
---

**Docs:** [What is VenueMi?](../README.md) · [link](path) · [link](path)
```

- The navigation footer must be the last element in the file.
- Link text is the document title, not "click here" or "see this".

### Tables

- Every table must have a header row.
- Align columns with spaces for readability in the raw Markdown source (pipe-aligned tables).
- If a table has more than eight columns, split it into two tables or use a list.

### Checkboxes

- `- [ ]` means not done.
- `- [x]` means done.
- Never use any other marker (`~~`, `?`, `-/-`) for tracking state.

---

## 6. What agents and tools must not do

- **Do not summarise content from one doc into another.** Instead, link. Duplication creates drift.
- **Do not add implementation code to design docs.** Short illustrative sketches are acceptable in architecture.md and intelligence.md only. Full implementations belong in source repositories.
- **Do not delete open decision checkboxes.** Mark them `- [x]` with the resolution inline, then leave them.
- **Do not change the audience of an existing document** without considering whether it should be split.
- **Do not create a new document** without first checking whether the content belongs in an existing file.
- **Do not use `# H1` headings inside a document** except for the document title at line one.
- **Do not write in past tense in vision.md.** Vision is forward-looking.
- **Do not write dates in milestone files** unless the milestone is `Completed`. Planned dates create false commitments.
- **Do not put roadmap content in architecture.md** and architectural content in roadmap files. They serve different readers.

---

## 7. CHANGELOG.md rules

CHANGELOG.md tracks significant changes to the **documentation itself** — new documents added, major rewrites, structural changes. It is not a Git commit log.

Format:

```markdown
## [Unreleased]

## YYYY-MM-DD

### Added

- Brief description of what was added.

### Changed

- Brief description of what was changed and why.

### Removed

- Brief description of what was removed and why.
```

Rules:

- Use ISO date format: `YYYY-MM-DD`.
- Entries under `[Unreleased]` are moved to a dated section when the documentation ships with a milestone.
- Keep entries brief — one line per change. Link to the file if context is needed.
- Do not log trivial edits (typo fixes, formatting). Only log additions, structural changes, and major rewrites.

---

## 8. Enforcement

These rules are enforced by:

1. **Pre-commit hook** — Prettier formats all Markdown files on commit. Config: `.prettierrc`.
2. **PR review** — All PRs that touch `docs/` must be reviewed against this file before merge.
3. **Agent instructions** — Any AI agent generating or modifying documentation in this repository must load this file first and treat it as the highest-priority constraint. If a user instruction conflicts with a rule in this file, surface the conflict rather than silently violating the rule.

If a rule in this file is wrong or needs updating, change this file first, then apply the change. Do not establish ad-hoc exceptions.

---

**Docs:** [What is VenueMi?](docs/README.md) · [Architecture](docs/platform/architecture.md) · [Roadmap Vision](docs/roadmap/vision.md)
