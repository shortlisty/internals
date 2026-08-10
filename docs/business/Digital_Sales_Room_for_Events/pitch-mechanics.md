# Pitch mechanics

> **Audience:** Founders, team.
> **Purpose:** How the pitch board is structured, what it contains, and how both sides interact with it.

---

## What it is technically

The pitch board is a **generated micro-site** — a standalone web page with a unique private URL, produced on demand from the agency's venue catalog and event brief. It lives at a URL like:

```
pitch.oiqb.com/p/{slug}             — free / Pro tier
pitch.youragency.com/p/{slug}     — Business / Enterprise tier (custom domain)
```

No login required to view. No app to install. The client opens a link.

From the client's perspective it looks and feels like a bespoke agency microsite — agency logo, agency colours, agency name in the header. The only OiQb reference is a small footer note: _"Powered by OiQb"_ (free/Pro) or removed entirely (Enterprise white-label).

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

| Feature                       | Pro ($150/mo) | Business ($300/mo) | Enterprise |
| ----------------------------- | ------------- | ------------------ | ---------- |
| OiQb subdomain           | ✅            | ✅                 | —          |
| Custom domain                 | —             | ✅                 | ✅         |
| Agency logo + colours         | ✅            | ✅                 | ✅         |
| "Powered by OiQb" footer | visible       | visible            | removed    |
| Password-protected board      | —             | ✅                 | ✅         |
| Custom email sender           | —             | ✅                 | ✅         |

From the client's perspective, Pro already looks like an agency-branded microsite. The "Powered by OiQb" footer is small — comparable to "Sent via Mailchimp" on a newsletter. Business removes it for agencies where white-label matters.

---

## Storage and lifecycle

- Pitch board assets (photos, floor plans referenced in the board) are stored for the duration of the active brief plus 30 days after the event date
- After expiry, the board becomes a static text-only record (metadata and snapshot preserved, binary assets purged)
- Agency can extend retention or export a full archive before expiry

---

**Docs:** [What is OiQb?](../../README.md) · [Product Structure](product.md) · [Business Proposal](proposal.md) · [Vision](../../roadmap/vision.md)
