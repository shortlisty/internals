# Pitch mechanics

> **Audience:** Founders, team.
> **Purpose:** How the pitch board is structured, what it contains, and how both sides interact with it.

---

## What it is technically

The pitch board is a **generated micro-site** — a standalone web page with a unique private URL, produced on demand from the agency's venue catalog and event brief. It lives at a URL like:

```
pitch.shortlisty.com/p/{slug}             — free / Pro tier
pitch.youragency.com/p/{slug}     — Business / Enterprise tier (custom domain)
```

No login required to view. No app to install. The client opens a link.

From the client's perspective it looks and feels like a bespoke agency microsite — agency logo, agency colours, agency name in the header. The only Shortlisty reference is a small footer note: _"Powered by Shortlisty"_ (free/Pro) or removed entirely (Enterprise white-label).

---

## Structure of a pitch board

### Header — event context

Ties the board to a specific brief so the client immediately understands what they are looking at.

- Event name or working title ("Nike Q4 Offsite 2026")
- Date range or target date
- Guest count and format (cocktail / seated dinner / conference / hybrid)
- City / location constraint
- Any key requirements pulled from the brief (kosher catering, outdoor terrace, AV for 500)
- Agency name and logo
- Prepared by (planner name, optional)

### Venue cards

Each selected venue gets a card. Cards are the core of the board — rich, visual, and scannable.

**Visual layer:**

- Full-width photo carousel (photos from the catalog DAM)
- Floor plan preview — tap/click to expand
- 360° or video embed if available

**Info layer — tied to the event brief:**

- Capacity in the relevant configuration for this event (not all configurations — just the ones that match the brief)
- Catering policy relevant to this brief (e.g. "Kosher available: yes")
- Key specs the brief flagged — AV, accessibility, load-in access, parking
- Restrictions that affect this event — noise curfew, no open flame, etc.
- Indicative pricing range (optional, can be hidden per venue)
- Contact name and role

The info layer is not a dump of all extracted metadata. It is filtered and ranked by relevance to the specific event brief. A corporate conference board shows AV specs prominently; a wedding board shows catering and curfew.

**Status indicator:**
Client-side only — shortlisted / considering / declined. Visible to the agency in real time.

### Comparison view (optional)

Side-by-side table of selected venues on key dimensions relevant to the brief. Auto-generated from the same catalog data. Useful when the client needs to compare 3–4 options quickly.

### Event spec panel

A lightweight spec block tied to the board — not a full contract, but the agreed parameters of the event as it currently stands:

- Confirmed date(s)
- Format and guest count
- Setup requirements
- Catering brief
- Budget range (optional, hidden by default)
- Notes / open items

The spec panel is editable by the agency and visible to the client. When the client approves, the spec panel state is included in the snapshot.

---

## Collaboration layer

### Comments and chat

Contextual comments attached to specific venues, photos, or spec items — not a global chat thread.

- Client clicks on a floor plan and asks: "Can we move the stage to the north wall?"
- Agency planner gets a notification, responds in context
- AI assist: planner can ask "what does the catalog say about stage rigging at this venue?" — answer pulled from extraction data

All threads are visible to both sides in the board. No information lives in email.

### Markers

Visual annotations on floor plans and photos — the agency or client can drop a pin or draw an area with a note attached.

- "Stage goes here"
- "This column blocks sightlines for 30% of the room"
- "Emergency exit — keep clear"

Markers are per-venue, stored with the board, and included in the snapshot on approval.

### Activity feed

Lightweight timeline visible to the agency (not the client):

- Client opened the board
- Client spent 4 minutes on venue 2
- Client shortlisted venues 1 and 3
- Client left a question on venue 2

Gives the planner enough signal to follow up at the right moment without being intrusive.

---

## AI assistance layer

The pitch board is not just a display layer for catalog data — it is an active workspace where AI helps the planner produce a stronger, more honest pitch faster.

### AI as a sparring partner

The most useful thing AI can do during pitch assembly is disagree. A planner who has worked with a venue many times may unconsciously present it more favourably than the data supports. The AI assist layer is designed to surface that tension, not smooth it over.

When the planner generates a draft board, the AI reviews the venue cards against the brief and flags:

- Fields in the card that do not match the brief requirements (e.g. capacity stated as 300 but brief asks for 350)
- Missing data that a client is likely to ask about (no catering policy shown, no curfew stated)
- Contradictions between venues on the same board (venue A listed as "exclusive use", venue B not addressed)
- Confidence gaps — fields displayed prominently that carry low extraction confidence and have not been verified

The goal is a pitch the planner is proud to send, not just a fast one.

### AI as a narrative coach

The event spec panel and the intro text on a pitch board carry the planner's voice. AI can tighten that narrative — not by writing it, but by checking it:

- Does the intro text actually reference the client's brief requirements?
- Is the spec panel internally consistent (date range, format, and guest count agree)?
- Are all the key requirements from the brief represented somewhere on the board?

The planner remains the author. AI surfaces gaps so the human can decide whether to fill them.

### AI-assisted client responses

When a client leaves a question on a venue card — "can we extend setup time to four hours?" — the planner can ask the AI: "what does the catalog say about setup hours at this venue?" The answer is pulled from source documents, cited by page, and presented to the planner before they respond. The planner sends the answer; the AI found it.

This pattern keeps the human in the loop on every client-facing message while dramatically reducing the time spent re-reading PDFs to answer questions the catalog already knows.

---

## Approval and snapshot

When the client clicks **Approve**:

1. Board status locks — no further edits without explicit revision
2. System generates an immutable snapshot:
   - Selected venue(s) with full metadata at that version
   - Event spec panel state
   - All comments and markers
   - Client identity (name / email, collected at approval time if not already known)
   - Timestamp (UTC)
   - Source citations for every extracted field
3. Both sides receive a confirmation — email with a link to the locked board and a PDF export option
4. Agency dashboard shows the board as Approved

The snapshot is an operational record — what was agreed, when, by whom, sourced from what documents. It is not a legally binding contract, but it reduces disputes significantly and provides a clear paper trail.

---

## Tiers and white-labelling

| Feature                        | Pro ($150/mo) | Business ($300/mo) | Enterprise |
| ------------------------------ | ------------- | ------------------ | ---------- |
| Shortlisty subdomain           | ✅            | ✅                 | —          |
| Custom domain                  | —             | ✅                 | ✅         |
| Agency logo + colours          | ✅            | ✅                 | ✅         |
| "Powered by Shortlisty" footer | visible       | visible            | removed    |
| Password-protected board       | —             | ✅                 | ✅         |
| Custom email sender            | —             | ✅                 | ✅         |

From the client's perspective, Pro already looks like an agency-branded microsite. The "Powered by Shortlisty" footer is small — comparable to "Sent via Mailchimp" on a newsletter. Business removes it for agencies where white-label matters.

---

## Storage and lifecycle

- Pitch board assets (photos, floor plans referenced in the board) are stored for the duration of the active brief plus 30 days after the event date
- After expiry, the board becomes a static text-only record (metadata and snapshot preserved, binary assets purged)
- Agency can extend retention or export a full archive before expiry

---

**Docs:** [What is Shortlisty?](../../README.md) · [Product Structure](product.md) · [Business Proposal](proposal.md) · [Vision](../../roadmap/vision.md)
