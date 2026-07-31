# AGENTS.md — Documentation Rules for BENE System Design

> Rules, structure, and patterns for every document in this repository.
> Any agent, contributor, or tool writing or editing docs here must follow this file.

---

## 1. What this repository is for

This repository is the **single source of truth for BENE's pre-build and in-flight system design**. It covers:

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
    │   ├── proposal.md                ← ICP, monetization, GTM, risks
    │   ├── comparison.md              ← Competitive landscape
    │   ├── product.md                 ← Product structure in business terms (two interfaces, four pillars)
    │   └── sales/
    │       ├── pitch.md               ← Demo and intro call narrative flow
    │       ├── battlecards.md         ← Per-competitor positioning for live conversations
    │       ├── objections.md          ← Objection handling: concern, response, follow-up
    │       └── messaging.md           ← Taglines, hero copy, value pillars, naming (copy bank)
    ├── platform/
    │   ├── architecture.md            ← Domain model, services, schema, API, events
    │   └── intelligence.md            ← ETL pipeline, extraction, AI layer
    ├── roadmap/
    │   ├── vision.md                  ← Product direction and strategic bets (not a changelog)
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
- File names are lowercase, hyphen-separated (`my-file-name.md`). No spaces, no underscores, no camelCase.
- Epic files are prefixed `E{N}-` (e.g. `E1-venue-profiles.md`). Numbers start at 1, increment by 1, never reuse.
- Milestone files are prefixed `v{X}.{Y}-` (e.g. `v0.1-mvp.md`). Align with product version numbering.
- Decision files are prefixed `D{N}-` (e.g. `D1-one-service-vs-two.md`). Numbers start at 1, increment by 1, never reuse.
- `ru/` translations are optional and mirror the English structure exactly. If a translation is missing, link to the English source.

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
- Must contain: summary, user stories, in/out of scope, milestone references, open questions, status.
- User stories follow the format: `As a [role], I want [action] so that [outcome].`
- Status field is one of: `Not started` | `In progress` | `Done`. Nothing else.
- Do not describe implementation details here. Implementation lives in architecture.md.

### 4.5 Milestones (`docs/roadmap/milestones/`)

- One milestone = one shippable increment with a clear goal sentence.
- Must contain: goal, epics/features included (with links), feature checklist, acceptance criteria, deferred items, status.
- Checkboxes `- [ ]` / `- [x]` are the only tracking mechanism. No external issue links.
- Status field is one of: `Planned` | `In progress` | `Completed`. Nothing else.
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

### 4.8 Product structure (`docs/business/product.md`)

- Audience is `Founders, team`.
- Explains the product in business terms: who it is for, the two interfaces (backoffice and storefront), and the four capability pillars (ETL, PIM, DAM, storefront).
- Do not describe implementation details here. For domain model, schemas, and service boundaries, see `architecture.md`.
- Do not duplicate the competitive gap analysis from `comparison.md`. The positioning summary here is a one-paragraph conclusion, not a re-analysis.
- Link to epic files for each pillar rather than re-listing their scope.

### 4.9 Competitive landscape (`docs/business/comparison.md`)

- Structured as: per-competitor sections followed by a gap summary matrix.
- Each competitor section has exactly: What it is, Strengths, Gaps relevant to BENE, Verdict.
- The verdict is one sentence only.
- The gap matrix is a Markdown table. Add a column for every competitor analysed.

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

**Docs:** [What is BENE?](../README.md) · [link](path) · [link](path)
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

**Docs:** [What is BENE?](docs/README.md) · [Architecture](docs/platform/architecture.md) · [Roadmap Vision](docs/roadmap/vision.md)
