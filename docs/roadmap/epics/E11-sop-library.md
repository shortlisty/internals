# E11 — SOP library

> **Audience:** Product, engineering.
> **Purpose:** Define the scope and user outcomes for the agency Standard Operating Procedures (SOP) capability inside Shortlisty.

---

## Summary

Event agencies repeat the same operational patterns on every project — how to receive a brief, how to conduct a site visit, how to handle a last-minute change. Today those patterns live in people's heads or in disconnected documents. The SOP library gives the agency a single structured place to write, organise, and reuse its operating procedures, and to attach those procedures as live checklists to Deal Rooms or events at the moment they are needed.

---

## User stories

- As an agency owner, I want to write an SOP with a title, a description, and a numbered checklist of steps so that my team follows a consistent process on every project.
- As a senior event manager, I want to attach an SOP checklist to an open Deal Room so that the assigned coordinator can work through the steps without asking me what to do next.
- As a junior coordinator, I want to open an SOP and tick steps as I complete them so that my progress is visible to the rest of the team.
- As an agency owner, I want to organise SOPs into categories (Client intake, Site visit, Deal Room, Emergency) so that the right procedure is easy to find at the right moment.
- As a team member, I want to search the SOP library by keyword so that I can find the relevant procedure in under 30 seconds.

---

## In scope

- Create, edit, and delete SOP documents (title, category, description, ordered checklist of steps).
- Category taxonomy: at least four built-in categories (Client intake, Site visit, Deal Room management, Emergency) plus a custom category option.
- Attach an SOP to a Deal Room as a live checklist instance (steps are ticked per Deal Room, not globally).
- Mark individual steps complete within an attached checklist; progress is visible to all team members in the same tenant.
- Keyword search across SOP titles and step text.
- Role gate: only owner and editor roles can create or edit SOPs; viewer role has read-only access.

---

## Out of scope

- Automated SOP triggering based on event type or Deal Room stage (backlog candidate — requires workflow engine not present in v1.1).
- Version history or change tracking for SOP documents (deferred to a later iteration).
- Client-visible SOP steps (SOPs are internal; clients never see them).
- AI generation of SOP content (may be revisited post-v1.1).
- Integration with external task managers (Asana, Notion, ClickUp).

---

## Milestone references

- [v1.1 — Agency knowledge base](../milestones/v1.1-agency-knowledge-base.md)

---

## Open questions

- [x] Should SOP steps support sub-steps or only a flat numbered list? Decision: flat numbered list for v1.1; nested steps deferred.
- [ ] Should attaching an SOP to a Deal Room copy the steps (snapshot) or link live to the master SOP (updates propagate)? Needs product decision before UI design begins.
- [ ] What is the maximum number of SOPs a tenant can create? Needs plan-tier limit decision (tie to E6 plan enforcement).

---

## Status

**Status:** Not started

---

**Docs:** [Epics index](README.md) · [v1.1 milestone](../milestones/v1.1-agency-knowledge-base.md) · [Vision](../vision.md) · [D17 decision](../decisions/D17-knowledge-base-scope-expansion.md)
