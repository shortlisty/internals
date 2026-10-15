# Template library

> **Audience:** Product, engineering.
> **Status:** Draft

---

## Summary

Every new project in an event agency starts the same way: copy last month's brief format, paste the venue shortlist structure, rewrite the intro email. The template library gives agencies a managed collection of reusable scaffolds — brief questionnaires, Deal Room structures, and client communication drafts — that can be applied in one click at the start of a new project. Applied templates create a ready-to-fill instance; the original template stays intact.

## Why it matters

Agency owners want consistent intake questionnaires and on-brand client communications across the team. Senior managers want juniors to open a new pitch board that already matches the agency's style. Coordinators want to pre-structure a Deal Room and fill in only event-specific details. Without templates, every project starts from scratch and quality drifts between planners.

## Where it could land

- Likely epic: post-v1.0 epic (formerly E13)
- Likely milestone: Post-v1.0 milestone (TBD)
- Related decision: [D12 — Knowledge base scope expansion](../decisions/D12-knowledge-base-scope-expansion.md)

## In scope

- Create, edit, and delete templates with a type tag: Brief, Deal Room, Communication.
- Template content editor: rich text for Deal Room and Communication templates; structured question builder for Brief templates (text, multiple-choice, and date field types).
- Apply a template to a new Deal Room or brief: creates an independent copy (instance); changes to the instance do not affect the master template.
- Template library view: list with type filter, search by keyword, and a preview panel.
- Built-in starter templates shipped with every new tenant account (minimum: one brief template, one Deal Room template, one shortlist delivery email).
- Role gate: only owner and editor can create or edit templates; all roles can apply them.

## Out of scope

- Dynamic / conditional template logic (if field X is Y then show section Z) — deferred to a later iteration.
- Template sharing across tenants — out of scope for v1.1.
- AI-assisted template generation from retrospectives — may be revisited post-v1.1.
- Version history for templates — deferred.
- Integration with external document editors (Google Docs, Word Online).

## Open questions

- [ ] Should applying a Brief template generate a client-shareable intake form link (like a Deal Room link but for brief collection)? This would significantly extend scope — needs explicit product decision before implementation.
- [x] Are templates tenant-specific or shared globally? Decision: tenant-specific for v1.1. Cross-tenant template sharing is post-v1.1.
- [ ] Should starter templates be localised (language variants for EN/ES/FR)? Needs decision before first-tenant onboarding in non-English markets.

## Status

**Status:** Draft — demoted from E13 epic pending v1.0 usage signal.

---

**Docs:** [Ideas index](README.md) · [Vision](../vision.md)
