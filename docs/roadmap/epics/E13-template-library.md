# E13 — Template library

> **Audience:** Product, engineering.
> **Purpose:** Define the scope and user outcomes for the reusable template capability — pre-built scaffolds for briefs, Deal Rooms, and client communications that eliminate starting from scratch on every project.

---

## Summary

Every new project in an event agency starts the same way: copy last month's brief format, paste the venue shortlist structure, rewrite the intro email. The template library gives agencies a managed collection of reusable scaffolds — brief questionnaires, Deal Room structures, and client communication drafts — that can be applied in one click at the start of a new project. Applied templates create a ready-to-fill instance; the original template stays intact.

---

## User stories

- As an agency owner, I want to create a brief template with a set of structured questions so that any planner on the team can hand a new client a consistent intake questionnaire without writing it from scratch.
- As a senior event manager, I want to create a Deal Room template (pre-set venue card layout, intro text, and spec configuration) so that my juniors open a new pitch board that already matches our agency's style.
- As a coordinator, I want to apply a template when starting a new Deal Room so that the board is pre-structured and I only need to fill in the event-specific details.
- As an agency owner, I want to maintain a library of standard client communication drafts (shortlist delivery email, venue approval confirmation, last-minute change notice) so that anyone on the team can send a consistent, on-brand message.
- As a team member, I want to browse templates by category (Brief, Deal Room, Communication) and preview their content before applying them so that I can pick the right scaffold for the situation.

---

## In scope

- Create, edit, and delete templates with a type tag: Brief, Deal Room, Communication.
- Template content editor: rich text for Deal Room and Communication templates; structured question builder for Brief templates (text, multiple-choice, and date field types).
- Apply a template to a new Deal Room or brief: creates an independent copy (instance); changes to the instance do not affect the master template.
- Template library view: list with type filter, search by keyword, and a preview panel.
- Built-in starter templates shipped with every new tenant account (minimum: one brief template, one Deal Room template, one shortlist delivery email).
- Role gate: only owner and editor can create or edit templates; all roles can apply them.

---

## Out of scope

- Dynamic / conditional template logic (if field X is Y then show section Z) — deferred to a later iteration.
- Template sharing across tenants or a public template marketplace — out of scope for v1.1.
- AI-assisted template generation from retrospectives — may be revisited post-v1.1.
- Version history for templates — deferred.
- Integration with external document editors (Google Docs, Word Online).

---

## Milestone references

- [v1.1 — Agency knowledge base](../milestones/v1.1-agency-knowledge-base.md)

---

## Open questions

- [ ] Should applying a Brief template generate a client-shareable intake form link (like a Deal Room link but for brief collection)? This would significantly extend scope — needs explicit product decision before implementation.
- [x] Are templates tenant-specific or shared globally? Decision: tenant-specific for v1.1. Cross-tenant template sharing is post-v1.1.
- [ ] Should starter templates be localised (language variants for EN/ES/FR)? Needs decision before first-tenant onboarding in non-English markets.

---

## Status

**Status:** Not started

---

**Docs:** [Epics index](README.md) · [v1.1 milestone](../milestones/v1.1-agency-knowledge-base.md) · [Vision](../vision.md) · [D17 decision](../decisions/D17-knowledge-base-scope-expansion.md)
