# D16 — Deal Room Trust Model: Immutable History, Bilateral Control, and Confidence-as-Transparency

> **Audience:** Engineers, architects, product.
> **Purpose:** Document the decision that the Digital Sales Room (Deal Room) is not just a pretty link — it is a shared trust space with immutable event history, bilateral data control, provenance-tagged metadata, and transparent confidence scoring designed to build trust between planner and client.

---

## Context

A planner sends a pitch today via one of three paths: (1) a Google Drive folder with 8 PDFs and a Canva deck, (2) a Qwilr page with manually-entered data, or (3) a HoneyBook proposal with an embedded PDF. In all three cases the client's experience is the same: information is scattered, conversations about specific venues happen in WhatsApp or email threads disconnected from the data, and there is no record of what was agreed — only a forward-only email chain or verbal "sounds good."

When a dispute surfaces three weeks later ("I thought catering was included" / "you said the rooftop was available on Friday the 13th"), neither side has a provable record. The planner is exposed. The client resents them. The relationship erodes.

The naive view of the Digital Sales Room is "a prettier link than email." The correct view: the Deal Room is a single shared space that replaces email threads, Drive folders, Canva decks, and WhatsApp chats with one version of events, visible to both sides, with a history that nobody can rewrite. This document defines the trust model — the architectural and product principles that make the Deal Room a credible source of truth for both parties, not a fancy document viewer.

## Options considered

### Option A — General DSR clone (Qwilr-like): document viewer + analytics + comments

Render the venue list as a static web page. Add document analytics (who opened it, how long they spent). Add inline comments. Ship.

Pros:

- Simple to build. Minimal backend state.
- Familiar pattern. DocSend, Papermark, Qwilr users get it immediately.

Cons:

- No structured data underpinning. Comments are on a page, not on a venue or a field. No way to connect a comment to a capacity figure or a catering policy.
- No immutable history. The planner can edit the page between client visits. Client has no way to know what changed. Trust is not built.
- No bilateral control. Client is a passive viewer, not a collaborator. Their preferences (I like venue B, I need AV included) are captured in comments or off-platform, not as structured signals.
- No confidence scoring. The page looks polished whether the underlying data is unverified user data, master-catalog enriched, or manually confirmed. The client cannot tell the difference.

### Option B — Deal Room as shared trust space (chosen)

The Deal Room is a structured collaboration space. Every element carries provenance, every action is written to an append-only log, both sides have bounded control over their own contributions, and the system surfaces data quality as a transparent feature rather than hiding it.

**Six architectural pillars:**

1. **Append-only event log per Deal Room.** Every significant action (venue added/removed, metadata updated, comment posted, preference saved, Approve clicked) is written to an immutable table. No row updates. No deletes. "Edit" is a new entry with a pointer to the previous version.
2. **Bilateral bounded control.** Planner controls venue selection, file visibility, internal-vs-client field scoping, and pitch branding. Client controls preferences, venue rankings, comments, and the final Approve action. Neither side can mutate the other's contributions.
3. **Provenance-tagged pitch metadata.** Every field the client sees carries the same provenance badge the planner sees in the catalog: "From agency's files" / "Enriched by VenueMi" / "Confirmed by venue [date]". Provenance is not hidden behind an admin view.
4. **Transparent confidence scoring.** Each venue card shows a softly-styled completeness/confidence indicator: "Basic data — upload a venue deck for full details" / "Verified from agency documents + master catalog" / "Confirmed with venue July 2026". Three tiers only, no percentages.
5. **Structured preference capture.** Client interactions are structured signals, not free text. Dynamic select boxes (contextual to the venues in this specific pitch) + colored labels (with a planner-editable legend) for rapid preference expression.
6. **Single source of truth view.** A Timeline / History tab visible to both sides that lists every action in chronological order with actor, timestamp, and diff. The history is readable by non-technical users.

Pros:

- Trust is built into the product. Client sees "this is fixed, both sides are on the same page" without being told. The planner is protected from scope disputes.
- Structured preferences feed back into the catalog. Client's "I need AV and kosher" is a filter the planner can reuse for future pitches, not a comment buried in an email thread.
- Competitive differentiation. Qwilr, Dock, Papermark, Notion pages — none of them have venue-specific structured history, bilateral provenance, or confidence-as-transparency.
- The Approve snapshot is not a gimmick. It is the final line in a history of verifiable actions, not an isolated click.

Cons:

- More backend complexity. Append-only event log, materialized current-state views, diff rendering, permission model per actor type.
- More UI surface. Confidence badges, provenance tags, Timeline tab, colored label legend builder for the planner, dynamic select-box generation per pitch.
- Risk of information overload if executed badly. The history tab is not a Git log for developers. It must read like a plain-English activity feed: "Planner added Villa Medici — 10:42 AM" / "Client marked Villa Medici as Preferred — 11:05 AM" / "Planner updated catering policy for Villa Medici from 'venue-only' to 'external allowed' — 11:18 AM".

### Option C — Full legal portal: e-signature, document versioning, audit trail export to PDF

Go further than B and turn the Deal Room into a legal artifact. Add DocuSign-style e-signature, PDF export of the full audit trail, admin lock on history, ISO 27001 badge everywhere.

Pros:

- Enterprise buyers love audit-trail exports.
- Strong defensive positioning for large accounts.

Cons:

- Overkill for the target ICP (solo planners and 2-8 person agencies). The first 50 customers do not need ISO 27001 or e-signature; they need something better than a Drive folder + WhatsApp.
- The product stops feeling pleasant and starts feeling legal and heavy. The UX "nice, pleasure, personalized" contract is broken.
- Scope bloat for v1.0. E-sign and audit-export are post-v1.0 features, not launch blockers.

## Decision

**We ship Option B — Deal Room as shared trust space. Option C elements (e-signature, PDF audit export) are deferred to post-v1.0.**

The pitch link is not a document. It is a relationship. The architectural product pillars above are mandatory from v0.4 (first DSR milestone) onward — not nice-to-haves to add later.

### Concrete requirements for the first implementation

#### 1. Append-only event log

Table `deal_room_event` (append-only):

| Field        | Type        | Purpose                                                                                                                                 |
| ------------ | ----------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| id           | bigint PK   | Surrogate                                                                                                                               |
| deal_room_id | FK          | Room this event belongs to                                                                                                              |
| actor_type   | enum        | PLANNER / CLIENT / SYSTEM                                                                                                               |
| actor_id     | uuid        | Planner user id or client session id                                                                                                    |
| event_type   | enum        | VENUE_ADDED, VENUE_REMOVED, METADATA_UPDATED, FILE_ADDED, COMMENT_POSTED, PREFERENCE_SET, APPROVE_CLICKED, REJECT_CLICKED, PITCH_OPENED |
| payload      | jsonb       | Diff: what changed, old value, new value, venue id if applicable                                                                        |
| created_at   | timestamptz | Immutable timestamp                                                                                                                     |

No `UPDATE` or `DELETE` permissions on this table. Ever. An "edit" inserts a new row with payload `{previous_event_id, diff}`.

#### 2. Bilateral bounded control

- **Planner can:** add/remove venues, scope which metadata fields are client-visible (internal notes are private by default), set colored label legend, configure dynamic select options, brand the room, archive the room.
- **Client can:** view scoped fields, download visible files, post comments, set preferences via structured controls (labels, selects, likes), Approve/Reject the pitch or individual venues.
- **Neither can:** edit the other's comments, preferences, or Approve state. No "takebacks" on the Approve action — a retracted approval is a new `APPROVE_RETRACTED` event in the log, not a deletion.

#### 3. Provenance and confidence visible to both sides

Every venue card in the client-facing view carries two small, unobtrusive visual elements:

- **Confidence tier dot:** Grey = basic data only, Blue = agency + master catalog, Green = confirmed with venue. Hover tooltip explains the tier.
- **Provenance badge (on-hover per field):** Hover "Capacity 120" → "From agency files (uploaded 2026-07-12)". Hover "Catering: external allowed" → "Enriched by VenueMi from master catalog".

#### 4. Structured preference capture (colored labels + dynamic selects)

Planner configures before sending:

- **Colored label legend:** Green = Planner's recommendation / Blue = Good fit / Orange = Caveats / Grey = Backup / Purple = New. Legend is editable per pitch, always visible to the client.
- **Dynamic select boxes:** Per-pitch contextual questions: "Which date works for you?" → options pulled from pitch event context. "Preferred format?" → Banquet / Cocktail / Conference. "AV included required?" → Yes / No / Don't know.

Client responses are structured events in the log, not comments. They feed back into the planner's catalog as reusable signal.

#### 5. History Timeline (client + planner visible)

Plain-English chronological feed. Not a JSON dump, not a Git diff view:

- 10:42 AM — Planner added Villa Medici, Palazzo Petrucci, Castel dell'Ovo
- 11:05 AM — Client marked Villa Medici as "Preferred"
- 11:18 AM — Planner updated catering policy for Palazzo Petrucci: "venue-only catering" → "external catering allowed"
- 11:47 AM — Client: "Can we see floor plans for the rooftop terrace at Castel dell'Ovo?"
- 12:02 PM — Planner uploaded floor-plan-rooftop-castel-dell-ovo.pdf
- 12:19 PM — Client approved Villa Medici

### Deferred to post-v1.0

- DocuSign-style cryptographic e-signature on Approve
- PDF export of full audit trail with chain-of-custody
- ISO 27001 / SOC 2 type II marketing badges
- Multi-approver weighted voting per stakeholder
- Client-side identity verification (email magic link for Approve — optional, not mandatory)

## Rationale

### Why this is the vacant slot

Qwilr, Papermark, DocSend, Dock, Notion pages — all solve "share a pretty link." None of them solve the event-specific problem of "both sides need to know exactly what was agreed about specific venues with specific metadata versions, and have a human-readable history." The DIY stack (Drive + Canva + WhatsApp) explicitly does not solve this. Cvent solves it for enterprise RFPs at 100x the price.

This is the gap VenueMi occupies. The approval snapshot by itself is not enough. The snapshot sits at the end of a trust-building timeline — without that timeline, it is a marketing gimmick. With it, it is a credible record.

### Why trust model is the real moat, not AI extraction

Competitors can copy AI extraction. They can copy a pretty pitch renderer. They cannot easily copy a product-wide design philosophy where provenance, bilateral control, and immutable history are first-class citizens from the database schema up to the pixel. That philosophy lives in every feature decision; it cannot be added as a late bolt-on.

### Why micro-SaaS positioning demands this

The target user (solo planner, 2-8 person agency) has no legal team, no procurement, no enterprise contract armor. They are exposed every time a client disputes "what was agreed." They need protection more than an enterprise does — but they will never pay for or adopt a heavy legal-portal product. The balance is: feels light, pleasant, simple, like a beautifully designed link. Under the hood, the planner is protected by an audit trail they never have to think about — until they need it.

## Consequences

### Engineering impact

- Event-sourcing style append-only log for the Deal Room domain. Materialized views for current pitch state (current venue list, current client preferences).
- Per-field permission model across the pitch: planner-scope vs client-scope per metadata key.
- Provenance data must be carried through from catalog ingestion all the way to pitch rendering. No provenance stripping at the pitch export boundary.
- Confidence tier calculation is a shared function between PVC (per venue) and DSR (per pitch — aggregated across venues in the room).

### Product impact

- E8 (Export and sharing) is now substantially redefined. It is not an "export" feature. It is a Deal Room feature. The epic scope and user stories need updating.
- Sales pitch: "When you click Approve, that's not the end of a process. It's the last line in a story both sides wrote together, and both sides can read."
- Objection handling for "my clients will just email me anyway": the Deal Room means their email preferences get captured as structured, searchable, reusable data in their catalog. Next time that client briefs them, the planner already knows their format preferences and venue biases.

### UX impact

- The confidence tier and provenance badges cannot feel like data-quality warnings. They must feel like honesty badges. Transparency is the aesthetic. Grey tier = "we started with the basics, and as you work together it only gets better." Green tier = "this is locked in."
- The History Timeline tab must never feel like a forensics tool. It reads like a shared notebook of what happened. No database ids, no raw JSON. Sentences. Plain English.

## Status

Accepted.

---

**Docs:** [Pitch Mechanics](../../business/digital-sales-room-for-events/pitch-mechanics.md) · [Product Structure](../../business/digital-sales-room-for-events/product.md) · [Architecture](../../platform/README.md) · [Export & Sharing Epic](../epics/E8-export-sharing.md) · [Feature Checklist](../feature-checklist.md)
