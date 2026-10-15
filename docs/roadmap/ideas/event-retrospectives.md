# Event retrospectives

> **Audience:** Product, engineering.
> **Status:** Draft

---

## Summary

When an event ends, the knowledge of what worked, what failed, and what the venue was really like in practice disappears into memory and scattered emails. The retrospectives capability gives the agency a structured template to record a post-event debrief, link it to the specific event and venue, and surface those learnings the next time a similar brief arrives. Over time, retrospectives enrich venue profiles with real operational experience — not just official specs.

## Why it matters

Senior managers want to capture experience while details are fresh. Planners starting a new brief want to see what actually happened at a venue, not just its marketing copy. Agency owners want to identify which venues consistently perform well and which repeatedly cause problems. Without retrospectives, institutional memory is lost between events.

## Where it could land

- Likely epic: post-v1.0 epic (formerly E12)
- Likely milestone: Post-v1.0 milestone (TBD)
- Related decision: [D12 — Knowledge base scope expansion](../decisions/D12-knowledge-base-scope-expansion.md)

## In scope

- Retrospective record linked to a closed Deal Room or manually created with a venue reference.
- Structured fields: event date, event type, outcome rating (1–5), what worked well (free text), what went wrong (free text), venue-specific notes, client feedback quote and rating, recommendations for future events.
- Link retrospective to one or more venue records — notes surface on the linked venue's profile page under a "Past experience" tab, visible to the agency team only.
- Filter and browse retrospectives by venue, event type, outcome rating, and date range.
- Keyword search across all retrospective fields.
- Export a retrospective as a PDF internal report.
- Role gate: all roles (owner, editor, viewer) can read retrospectives; only owner and editor can create or edit.

## Out of scope

- Client-facing retrospective view (retrospectives are internal; clients never see them).
- Automated retrospective creation (a prompt may appear when a Deal Room is closed, but filling it in is always manual).
- AI summarisation of retrospectives (may be revisited post-v1.1).
- Venue rating visible in the master venue catalog (tenant retrospectives are private to the tenant; no cross-tenant aggregation).
- Integration with external survey tools (e.g. Typeform, SurveyMonkey).

## Open questions

- [ ] When a Deal Room closes (approved snapshot), should the system automatically prompt the planner to create a retrospective? Needs UX decision — a prompt is low-friction; auto-creation of an empty record is higher friction.
- [ ] Should venue-level "past experience" notes derived from retrospectives affect search ranking (surface venues with positive retrospectives higher)? Needs product decision before search layer is modified.
- [x] Are retrospectives per-event or per-venue? Decision: per-event (one retrospective per Deal Room / event), linked to venue(s). A venue can have many retrospectives from different events.

## Status

**Status:** Draft — demoted from E12 epic pending v1.0 usage signal.

---

**Docs:** [Ideas index](README.md) · [Vision](../vision.md)
