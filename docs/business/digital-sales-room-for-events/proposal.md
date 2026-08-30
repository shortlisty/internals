# Business proposal — Digital Sales Room for Events

> **Audience:** Founders, team.
> **Purpose:** Full business case — ICP, feature phases, monetisation, go-to-market, risks, and open questions.

---

## The problem

An event agency receives a client brief. The planner knows which venues would work — they have visited them, filed the documents, built the relationships. But turning that knowledge into a confirmed client decision takes far longer than it should.

The planner spends an hour assembling a PDF. The client receives it and responds with questions scattered across three channels. Nobody has a single place where the spec is written down and agreed. Revisions happen informally. When something goes wrong on the day, there is no authoritative record of what was actually confirmed.

The downstream cost is real: lost deals to faster competitors, margin erosion from untracked scope changes, and post-event disputes that damage client relationships.

**The specific pain points:**

- Assembling a client-ready venue proposal takes 2–4 hours per brief
- Client feedback arrives across email, WhatsApp, and phone — never consolidated
- No agreed record of the final spec — disputes arise about what was confirmed
- Senior planners carry relationships and venue knowledge in their heads — when they leave, it goes with them
- The same venue research is repeated for every new event cycle

---

## The solution

Shortlisty closes the brief-to-approval loop in two steps.

**Step 1 — Build the library.** The agency connects their existing files (Drive, Notion, email) and uploads venue documents. Shortlisty extracts the structured data — capacity, catering policy, AV specs, restrictions, contacts — and builds a searchable venue catalog. A human verification step keeps the data trustworthy.

**Step 2 — Generate and close the pitch.** For a client brief, the planner selects venues from the catalog and generates an interactive pitch board — a private web page the client opens on any device. The client browses, asks questions, adjusts the spec, and approves. On approval, the system locks a timestamped snapshot: an immutable record of everything agreed, with source citations.

**The aha moment:** a client brief arrives, the planner generates a pitch board in under five minutes, the client approves the same day. No PDF. No scattered threads. One link, one approval, one record.

Product structure: [product.md](product.md).

---

## Who it is for

### Primary — event planning agencies (5–50 people)

Teams managing weddings, corporate events, conferences, galas, team-building, and private parties. They have an established venue portfolio — PDF decks from venue coordinators, floor plans, notes from site visits — and they pitch venues to clients repeatedly.

- Buying trigger: a senior planner leaves and takes their knowledge, or a client deal falls through due to slow response time, or the team grows past the point where informal knowledge sharing works
- Decision maker: agency owner or managing director
- Decision speed: fast — no procurement committee, credit card purchase
- Willingness to pay: high — one approved deal pays for months of the subscription

### Secondary — corporate event teams (in-house)

EA, HR, or events leads at mid-to-large companies running quarterly offsites, annual conferences, and executive dinners. They repeat the same venue research every cycle because no structured record was kept.

- Buying trigger: a new person joins the team and has to start from scratch, or leadership asks why venue selection takes so long
- Decision maker: head of events, EA, operations manager
- Decision speed: medium — may need one level of approval
- Willingness to pay: medium-high — budget exists, ROI is demonstrable

### End users

**Senior event manager** — uses Shortlisty on every brief. Searches the catalog, generates the pitch, monitors client interaction. The tool replaces their inbox and their spreadsheet.

**Junior coordinator** — relies on the shared catalog to answer client questions independently without asking senior colleagues. Retention driver: the catalog becomes how they learn the portfolio.

**Client (external, no account)** — opens a link, reviews the pitch, approves. No training. No login. The experience is a premium proposal, not a SaaS tool.

### ICP matrix

| Segment               | Size   | Buyer               | ACV target      | Priority        |
| --------------------- | ------ | ------------------- | --------------- | --------------- |
| Small event agency    | 5–20   | Owner / MD          | $1,800/yr       | **Tier 1**      |
| Mid event agency      | 20–50  | Director / MD       | $2,400–4,800/yr | **Tier 1**      |
| Corporate events team | 50–500 | Head of Events / EA | $3,600–6,000/yr | Tier 2          |
| Solo event manager    | 1      | Self                | $1,800/yr       | Tier 3 (volume) |
| Large agency / AMC    | 50+    | VP Events / COO     | $6,000+/yr      | Tier 2          |

**MVP focus:** small to mid agencies (5–50). Fastest decision cycle, clearest pain, highest word-of-mouth density in tight professional communities.

---

## Core value propositions

**Close briefs faster.** A pitch board takes minutes, not hours. Clients approve the same day instead of the same week. Agencies win business that slower competitors lose.

**One agreed record.** The approved snapshot is the source of truth. No post-event dispute about what was confirmed. No reconstructing a chain of WhatsApp messages to find out what was agreed.

**Knowledge that stays.** When a planner leaves, the venue knowledge stays in the catalog. New hires get up to speed in days, not months.

**Margin protection.** Every field in the pitch is sourced from a verified document. Outdated prices and wrong capacity figures — the most common margin-eroding mistakes — are caught before they reach the client.

---

## Features — what we are building

### Phase 1 — MVP

**Venue catalog**

- Ingest via Drive/Notion connect, email forward, or direct upload
- AI extraction: capacity, catering policy, AV specs, restrictions, contacts, pricing
- Human verification: split-screen confirm/correct flow, confidence scores, source citations
- Hybrid search: keyword, semantic, structured filters
- Master venue catalog for major markets (invisible gap-fill at MC_INHERIT priority, provenance tracked per field)

**Pitch board**

- Generate from selected catalog venues in one click
- Private shareable link — no client login required
- Interactive venue cards: photo gallery, floor plan, key metadata, configurable spec
- Client preference indicators: shortlist / consider / decline
- Agency notifications: when client opens the board, views a venue, leaves a question
- Approval → immutable snapshot with timestamp and source citations
- Basic branding: agency name and logo on the pitch page

**Collaboration**

- Invite agency team members with role-based access
- Client-side: inline questions on any venue or spec item
- Agency response visible to client in the same board

**Infrastructure**

- Storage retention: catalog sources 30 days, pitch assets through event date + 30 days
- Mobile-responsive on both sides

### Phase 2 — depth

- CAD/DWG file support and floor plan geometry extraction
- Deeper photo analysis: amenity detection via vision AI
- Multi-language extraction (Spanish, French)
- Geo-spatial search and map view
- Saved searches with alerts
- Multi-stakeholder voting on the client side (multiple approvers)
- Revision workflow on approved pitches
- White-label: custom domain and brand colours

### Phase 3 — ecosystem

- Integration hooks: CRM, calendar, booking systems
- Export: PDF spec sheet, Excel, structured JSON
- Master venue catalog surfacing: verified master venue profiles agencies can promote from backdrop into pitches directly
- Analytics: brief-to-approval conversion rate, time-to-approval, most-pitched venues

---

## Monetisation

### Pro — $150/month per agency

- Full catalog (unlimited venues)
- Pitch board generation
- Unlimited team members
- Unlimited clients (no account required)
- 30-day catalog source retention, event-date pitch retention
- Target: small to mid agencies

### Business — $300/month per agency

- Everything in Pro
- White-label pitch boards (agency branding, custom domain)
- Priority AI processing queue
- API access
- Extended retention
- Target: established agencies, corporate event teams

### Enterprise — custom

- Unlimited everything
- SSO / SAML
- SLA-backed uptime
- Dedicated onboarding
- Target: large agencies, AMCs, corporate events at scale

### Future revenue

- **Master venue listings:** venues pay for a verified profile in the master catalog that agencies can surface into pitches directly, reducing ingestion time
- **Lead signal products:** anonymised aggregate data on which venue types and configurations appear most in approved pitches — sold to hospitality operators for market intelligence

### Unit economics target

At 100 Pro agencies: $15,000 MRR. AI processing and storage costs at that scale: ~$1,500–2,000/month. Gross margin: ~85%. The numbers work at 50 agencies; they become comfortable at 150.

---

## Go-to-market

### Phase 1 — first 10 paying agencies

Direct and concierge. No paid acquisition.

- Identify 20–30 agencies in one target city (Naples, Austin, or Nashville — see cold-start strategy)
- Reach out directly: "I built a tool that generates a client pitch board from your venue files in 5 minutes — want to try it on your next brief?"
- Concierge onboarding: agency shares their Drive folder, team imports their first 20 venues for free
- The demo is the pitch board itself — no explainer needed, show the client link

### Phase 2 — 10 to 100 agencies

- Product Hunt launch after first 10 paying customers provide testimonials
- Industry press: BizBash, Skift Meetings, EventMB
- Content: "How event agencies are closing briefs 3x faster" — case study format with real numbers
- Community seeding: MPI, PCMA, ILEA, EventProfs Slack, LinkedIn groups
- Referral: one free month per paying referral

### Retention

- Onboarding sequence: day 1 (first venue), day 3 (first pitch board), day 7 (first client approval)
- Monthly changelog — build the habit of checking what is new
- Upgrade prompts at natural limits (team size, storage, branding)

### Viral loop

Every pitch board sent to a client is a passive product demo. The "Powered by Shortlisty" footer on the free tier reaches event buyers who then ask their own agencies "what is this tool you used?" One paying agency can introduce the product to ten potential clients per month.

---

## Key risks

**Pitch board adoption requires client buy-in on the agency side.**
The agency has to trust that their client will open a link rather than expect a PDF attachment. Mitigation: make the board so visually compelling that the agency is proud to send it. The first time a client says "this is the best proposal I've ever received" — the agency never goes back to PDF.

**AI extraction accuracy on real documents.**
The catalog's value depends on extracted data being reliable. Real venue documents vary enormously in format and quality. Mitigation: benchmark 50 real venue documents before launch, publish accuracy results internally, never promise field-level accuracy that hasn't been measured. Human verification is a first-class feature, not a backup.

**Cold start — empty catalog on signup.**
An empty library produces an empty pitch. Mitigation: concierge onboarding (team imports the first 20 venues for free), plus the master venue catalog gap-fills invisibly for major markets so the first search always returns something useful.

**Client-side experience must be zero-friction.**
If the client needs to create an account, download an app, or ask the agency how to use the board — the product has failed. Mitigation: test the board link with non-technical users before launch. If anyone asks "how do I open this?" — fix the board before shipping.

**Snapshot legal weight.**
Agencies will rely on the approval snapshot to resolve disputes. Mitigation: be explicit that the snapshot is an operational record, not a legally binding contract. It reduces disputes but does not replace a signed contract. Do not overstate this in marketing.

---

## Open questions

- What is the minimum catalog size before the pitch board feels useful? Hypothesis: 10 venues in the relevant category is enough for a first pitch.
- Do clients prefer to approve inside the board, or do they want to export a PDF and sign it separately? Test with first 10 agencies.
- Which city to launch in first? Naples (EU, low competition), Austin (US, active market), or Nashville (US, fastest-growing event market)?
- Is $150/month the right entry price, or does starting at $99 accelerate early adoption enough to offset lower ACV?
- How much of the concierge onboarding can be automated before the first 10 customers, and how much needs to be manual?

---

**Docs:** [What is Shortlisty?](../../README.md) · [Product Structure](product.md) · [Market Structure](../market.md) · [Architecture](../../platform/README.md) · [Vision](../../roadmap/vision.md)
