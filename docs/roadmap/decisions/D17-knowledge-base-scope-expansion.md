# D17 — Agency Knowledge Base: SOPs, retrospectives, templates, and vendor registry as post-v1.0 scope

> **Audience:** Engineers, architects.
> **Purpose:** Record the decision to extend Shortlisty beyond venue catalog + pitch board into a broader agency operational knowledge base, and to defer that extension to post-v1.0.

---

## Context

The original v1.0 product scope covers two layers: the Personal Venue Catalog (structured venue data) and the Digital Sales Room (client-facing pitch board with approval snapshot). Together they close the brief-to-approval loop.

A strategic analysis in September 2026 (see `chats/2026.09.15-Grok-SOP_Retrospectives_Templates_Vendor_Briefing.md`) identified a broader opportunity: the same agencies that use the venue catalog and pitch board also carry a large body of operational knowledge — how they run projects, what happened at past events, which suppliers they trust, what email templates they reuse. Today that knowledge lives in people's heads, shared drives, and informal channels. When a senior planner leaves, most of it goes with them.

Capturing that knowledge inside Shortlisty would increase platform stickiness (the more an agency stores, the harder it is to leave), differentiate from pure proposal tools (Qwilr, Pandadoc, Dock.us) which have no knowledge layer, and open a credible expansion path beyond the initial venue-catalog-and-pitch-board positioning.

Four capability areas were identified:

1. **SOP library** — standard operating procedures with step checklists, attached to Deal Rooms.
2. **Event retrospectives** — structured post-event debriefs linked to venues and deal rooms.
3. **Template library** — reusable scaffolds for briefs, Deal Rooms, and client communications.
4. **Vendor registry** — preferred suppliers with ratings and performance notes linked to retrospectives.

---

## Options considered

### Option A — Include knowledge base capabilities in v1.0

Extend the v1.0 scope to include SOPs, retrospectives, templates, and the vendor registry alongside the venue catalog and Deal Room.

**Pros:** Single launch that positions Shortlisty as an operational platform from day one. No "version 2" messaging required.

**Cons:** Significantly increases v1.0 scope and delays first commercial launch. The core value proposition (brief-to-approval loop) is already validated independently — adding four new capability clusters before measuring adoption of the first two increases build risk substantially. The knowledge-base features depend on the Deal Room being trusted and actively used (retrospectives link to closed Deal Rooms; SOPs attach to live ones); without v1.0 usage, there is no feedback signal to validate the feature design.

### Option B — Build knowledge base as a post-v1.0 milestone (chosen)

Ship v1.0 with venue catalog and Deal Room. Add the knowledge-base layer in v1.1 after v1.0 reaches `Completed`.

**Pros:** Maintains focus and launch speed for v1.0. Knowledge-base features are informed by real agency usage patterns from v1.0 (which SOPs matter, which event types generate retrospectives, which vendor categories are most common). Lower launch risk. Each knowledge-base epic (E11–E14) is independently deliverable within a single milestone (v1.1).

**Cons:** Agencies with an immediate need for SOP or template management must wait or use a secondary tool in the interim. Some early-adopter agencies may not renew if they expected a broader platform sooner.

### Option C — Build knowledge base as a separate product

Spin up a distinct product ("Shortlisty Ops" or similar) rather than extending the main platform.

**Pros:** Clean separation of concerns; different ICP possible.

**Cons:** Loses the integration value: retrospectives linked to venue profiles, SOPs attached to Deal Rooms, vendors surfacing from retrospective data. A separate product requires a separate acquisition and onboarding funnel. Not justified given that the primary buyer is the same person (agency owner / senior planner) and the data is tightly coupled to the same domain.

---

## Decision

**Option B.** Build the agency knowledge base as milestone v1.1, shipping after v1.0 reaches `Completed`.

The four capability clusters are formalised as epics E11–E14 in the Group D (Post-v1.0 Enhancements) layer of the epics index.

---

## Rationale

The core v1.0 value proposition is the brief-to-approval loop. That loop must be proven and trusted before adding capabilities that depend on it (retrospectives require closed Deal Rooms; SOPs attach to active ones). Adding all four knowledge-base clusters to v1.0 would delay launch and dilute focus without proportional benefit. Deferring to v1.1 preserves momentum, gives a concrete near-term expansion path to existing customers, and ensures the knowledge-base design is informed by real v1.0 usage rather than speculation.

The combined knowledge-base layer also strengthens Shortlisty's defensibility: once an agency stores its SOPs, retrospectives, templates, and vendor list inside the platform, switching cost rises substantially. This is a deliberate strategy to improve long-term retention, but it requires the base platform to be trusted first.

**These capabilities are secondary by design.** The permanent centre of gravity for Shortlisty is the venue catalog, fast shortlisting, and delivering structured venue knowledge to the client through the pitch board. The knowledge-base layer (E11–E14) exists to make that core better — retrospectives enrich venue profiles with real operational experience, templates accelerate pitch assembly, SOPs raise consistency of the work that surrounds a pitch, and the vendor registry makes recommendations inside a brief faster and more reliable. None of these features change what Shortlisty is about. They should never compete with the catalog or pitch board for engineering priority, and they should never be marketed as the primary reason to buy the product.

---

## Consequences

- Epics E11, E12, E13, E14 are created at `Not started` status and target v1.1.
- No knowledge-base features appear in v0.x or v1.0 milestone files or feature checklist P0–P2 tiers.
- v1.0 milestone and epics E1–E8 are unaffected by this decision.
- v1.1 milestone (`v1.1-agency-knowledge-base.md`) is created at `Planned` status, blocked on v1.0 `Completed`.
- If v1.0 adoption shows that agencies primarily want the knowledge-base features before the full approval flow (unexpected signal), this decision should be revisited and a new decision record created.
- Cross-tenant knowledge sharing, a public template marketplace, and vendor-side marketplace features remain out of scope for v1.1. They are captured as backlog candidates and are subject to separate future decisions.

---

## Status

**Status:** Accepted

---

**Docs:** [Decisions index](README.md) · [v1.1 milestone](../milestones/v1.1-agency-knowledge-base.md) · [E11 SOP library](../epics/E11-sop-library.md) · [E12 Retrospectives](../epics/E12-event-retrospectives.md) · [E13 Template library](../epics/E13-template-library.md) · [E14 Vendor registry](../epics/E14-vendor-registry.md) · [Vision](../vision.md)
