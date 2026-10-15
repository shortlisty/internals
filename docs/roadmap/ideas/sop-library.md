# SOP library

> **Audience:** Product, engineering.
> **Status:** Draft

---

## Summary

Event agencies repeat the same operational patterns on every project — how to receive a brief, how to conduct a site visit, how to handle a last-minute change. Today those patterns live in people's heads or in disconnected documents. The SOP library gives the agency a single structured place to write, organise, and reuse its operating procedures, and to attach those procedures as live checklists to Deal Rooms or events at the moment they are needed.

## Why it matters

Agency owners want consistent process across every project. Senior managers want to hand a coordinator a checklist instead of explaining what to do next. Junior coordinators want to tick steps and have progress visible to the team. Without a shared SOP library, process knowledge walks out the door when a senior planner leaves.

## Where it could land

- Likely epic: post-v1.0 epic (formerly E11)
- Likely milestone: Post-v1.0 milestone (TBD)
- Related decision: [D12 — Knowledge base scope expansion](../decisions/D12-knowledge-base-scope-expansion.md)

## In scope

- Create, edit, and delete SOP documents (title, category, description, ordered checklist of steps).
- Category taxonomy: at least four built-in categories (Client intake, Site visit, Deal Room management, Emergency) plus a custom category option.
- Attach an SOP to a Deal Room as a live checklist instance (steps are ticked per Deal Room, not globally).
- Mark individual steps complete within an attached checklist; progress is visible to all team members in the same tenant.
- Keyword search across SOP titles and step text.
- Role gate: only owner and editor roles can create or edit SOPs; viewer role has read-only access.

## Out of scope

- Automated SOP triggering based on event type or Deal Room stage (requires a workflow engine not present in v1.1).
- Version history or change tracking for SOP documents (deferred to a later iteration).
- Client-visible SOP steps (SOPs are internal; clients never see them).
- AI generation of SOP content (may be revisited post-v1.1).
- Integration with external task managers (Asana, Notion, ClickUp).

## Open questions

- [x] Should SOP steps support sub-steps or only a flat numbered list? Decision: flat numbered list for v1.1; nested steps deferred.
- [ ] Should attaching an SOP to a Deal Room copy the steps (snapshot) or link live to the master SOP (updates propagate)? Needs product decision before UI design begins.
- [ ] What is the maximum number of SOPs a tenant can create? Needs plan-tier limit decision (tie to plan enforcement).

## Status

**Status:** Draft — demoted from E11 epic pending v1.0 usage signal.

---

**Docs:** [Ideas index](README.md) · [Vision](../vision.md)
