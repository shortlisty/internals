# E12 — Event retrospectives

> **Audience:** Product, engineering.
> **Purpose:** Define the scope and user outcomes for structured post-event retrospectives — the mechanism by which agencies capture institutional memory from completed events.

---

## Summary

When an event ends, the knowledge of what worked, what failed, and what the venue was really like in practice disappears into memory and scattered emails. The retrospectives capability gives the agency a structured template to record a post-event debrief, link it to the specific event and venue, and surface those learnings the next time a similar brief arrives. Over time, retrospectives enrich venue profiles with real operational experience — not just official specs.

---

## User stories

- As a senior event manager, I want to fill in a structured post-event retrospective immediately after an event so that the team's experience is captured while details are still fresh.
- As a planner starting a new brief, I want to see all past retrospectives linked to a venue so that I know what actually happened when my colleagues worked there — not just the venue's own marketing copy.
- As an agency owner, I want to filter retrospectives by event type, outcome rating, and venue so that I can identify which venues consistently receive positive feedback and which repeatedly cause problems.
- As a coordinator, I want to attach client feedback (a rating and a short quote) to a retrospective so that the client's perspective is recorded alongside the internal debrief.
- As a team member, I want to search retrospectives by keyword so that I can find past experience relevant to a specific scenario (e.g. "last-minute AV failure" or "outdoor rain contingency").

---

## In scope

- Retrospective record linked to a closed Deal Room or manually created with a venue reference.
- Structured fields: event date, event type, outcome rating (1–5), what worked well (free text), what went wrong (free text), venue-specific notes, client feedback quote and rating, recommendations for future events.
- Link retrospective to one or more venue records — notes surface on the linked venue's profile page under a "Past experience" tab, visible to the agency team only.
- Filter and browse retrospectives by venue, event type, outcome rating, and date range.
- Keyword search across all retrospective fields.
- Export a retrospective as a PDF internal report.
- Role gate: all roles (owner, editor, viewer) can read retrospectives; only owner and editor can create or edit.

---

## Out of scope

- Client-facing retrospective view (retrospectives are internal; clients never see them).
- Automated retrospective creation (a prompt may appear when a Deal Room is closed, but filling it in is always manual).
- AI summarisation of retrospectives (may be revisited post-v1.1).
- Venue rating visible in the master venue catalog (tenant retrospectives are private to the tenant; no cross-tenant aggregation).
- Integration with external survey tools (e.g. Typeform, SurveyMonkey).

---

## Milestone references

- [v1.1 — Agency knowledge base](../milestones/v1.1-agency-knowledge-base.md)

---

## Open questions

- [ ] When a Deal Room closes (approved snapshot), should the system automatically prompt the planner to create a retrospective? Needs UX decision — a prompt is low-friction; auto-creation of an empty record is higher friction.
- [ ] Should venue-level "past experience" notes derived from retrospectives affect search ranking (surface venues with positive retrospectives higher)? Needs product decision before search layer is modified.
- [x] Are retrospectives per-event or per-venue? Decision: per-event (one retrospective per Deal Room / event), linked to venue(s). A venue can have many retrospectives from different events.

---

## Status

**Status:** Not started

---

**Docs:** [Epics index](README.md) · [v1.1 milestone](../milestones/v1.1-agency-knowledge-base.md) · [Vision](../vision.md) · [D17 decision](../decisions/D17-knowledge-base-scope-expansion.md)
