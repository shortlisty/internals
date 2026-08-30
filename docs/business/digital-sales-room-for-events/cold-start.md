# Cold start strategy — Digital Sales Room for Events

> **Audience:** Founders, team.
> **Purpose:** How to get from zero to a working demo and first paying agency — without waiting for organic signups.

---

## The two cold start problems

A DSR product has two distinct empty-state problems that compound each other.

**Problem 1 — empty catalog.** An agency that signs up and sees an empty venue library cannot generate a pitch. The product is useless until they have at least 10–20 venues ingested. Most agencies will not do that work unprompted.

**Problem 2 — no demo story.** To sell to an agency, you need to show them a live pitch board with venues they recognise. A demo with placeholder data or generic US venues means nothing to an agency in Naples. A demo with Villa D'Angelo, Palazzo Petrucci, and Castel dell'Ovo — venues they know and pitch regularly — is an immediate aha moment.

The cold start strategy solves both problems at once: build the regional venue catalog yourself before the first agency signs up. By the time the first conversation happens, the demo is already real and the library is already seeded.

---

## Geographic focus — why one city first

Launching in one city before expanding gives three advantages over a broad launch:

1. **Credibility.** A catalog of 30 well-known venues in Naples is more convincing to a Naples agency than 300 venues spread across Europe they have never heard of.
2. **Word of mouth.** The event agency community in a mid-size city is tight. One agency owner telling two colleagues is a meaningful referral network. In a large city, the same referral is noise.
3. **Tractable scope.** 20–40 venues can be seeded in a week. 2,000 cannot.

### Why Naples (or equivalent tier-2 city)

Tier-1 cities (Milan, Rome, London, NYC) have more venues but also more competition, more sophisticated buyers, and worse signal-to-noise ratio for early learning. A concentrated, active event market in a city where agencies know each other produces faster feedback loops.

Naples specifically: active corporate and social event market, high density of independent venues (villas, palazzi, rooftop terraces), strong hospitality culture, and a professional event agency community that is not yet saturated with SaaS tools.

The same logic applies to equivalent cities in other regions: Austin or Nashville (US), Porto or Valencia (EU), Bristol or Edinburgh (UK).

---

## Why focused geography makes everything easier

**Data is easier to get.** The top 30 venues in a mid-size city can be mapped in an evening — Google Maps, local wedding guides, one or two direct calls. Venue coordinators are almost always willing to share their decks when asked directly: "I'm building a tool for event agencies in Naples, can I use your materials?" The answer is usually yes. They want to be found and recommended.

**The demo lands immediately.** Showing a Naples agency a pitch board with Villa D'Angelo and Palazzo Petrucci needs no explanation. They recognise the venues, they understand the value instantly. A generic catalog of 500,000 venues doesn't produce that reaction.

**Agencies are reachable and open.** In a city of 20–40 event agencies, everyone knows everyone — same industry groups, same WhatsApp chats, same local conferences. One happy customer refers three colleagues by name. Local agencies are also less saturated with SaaS pitches than agencies in major capitals, and more willing to talk to someone who already knows their market.

The opener writes itself: "I've already built a catalog of the top venues in Naples — want to see a pitch board with your regular spots?"

---

## Building the seed catalog

### Target

20–40 venues in the launch city. Enough to generate a credible pitch board for any brief in the 50–300 guest range.

### Venue selection criteria

A good seed venue is:

- Independently operated or a small regional group — not a chain hotel with centralised PR
- Publicly marketed with a website, photos, and downloadable specs or PDF decks
- Actively hosting events — reviews on Google Maps, visible social presence
- Capacity range 30–400 guests
- Multiple document types available — deck, floor plan, photos, or a detailed website

### Source channels

**Venue websites (primary)** — most SMB event venues publish spec sheets, floor plans, and photo galleries. Downloading them for extraction testing is the intended use.

**Google Maps / Places API** — confirms address, coordinates, category tags, and provides a starting photo set. Use the Places API within terms of service — do not scrape HTML.

**Instagram and Facebook** — public venue pages carry recent photos and capacity/booking info in their bio. Useful for photo collection and confirming active status.

**Direct outreach** — email the venue directly and ask for their event pack. Most venues send it immediately; they want to be found.

### Process per venue

1. Find the venue (Maps search, local event guides, wedding directories)
2. Download all publicly available documents
3. Create a venue record in Shortlisty — name, address, source URL
4. Upload the documents — this is a live extraction pipeline test
5. Review the AI output, correct key fields manually
6. Mark as verified — ready to appear in pitch boards

Each venue takes 15–30 minutes end to end. 40 venues is one focused week.

---

## Using the seed catalog as a demo

Once the catalog has 20+ verified venues, the demo writes itself.

**The demo flow:**

1. Open the catalog — show a searchable, filterable grid of local venues the prospect recognises
2. Run a search: "rooftop terrace for 80 guests, cocktail setup" — results appear in seconds
3. Select three venues, click Generate pitch — the pitch board appears in 10 seconds
4. Share the link — open it on a phone to show the client view
5. Click Approve on one venue — show the snapshot

The prospect has just watched their own workflow compressed from 2 hours to 5 minutes, using venues from their own city. No explanation needed.

---

## First agency onboarding — concierge model

The first 5–10 agencies should not self-serve. They should receive concierge onboarding:

**The offer:** "Sign up, share your Drive folder with me, and I will import your first 20 venues for free before your first call."

What this means in practice:

- Agency shares their Drive folder containing venue PDFs, floor plans, and decks
- Founder runs the import pipeline manually, reviews the output, corrects obvious errors
- By the time the first onboarding call happens, the agency already has a populated catalog
- The first call is a demo of their own data, not a generic demo

This approach eliminates the cold start for the agency completely. The activation barrier drops to near zero. The agency's first experience is a pitch board built from their own venues.

The concierge model does not scale — but it does not need to. The goal is 10 paying agencies, not 1,000. At 10 agencies, the extraction pipeline is validated on real data, the first case studies exist, and word-of-mouth begins.

---

## Validation milestones before opening to self-serve

Before removing the concierge requirement and opening to self-serve signups:

- [ ] Seed catalog: 30+ venues verified in the launch city
- [ ] Extraction benchmark: 50 real venue documents processed, field-level accuracy documented
- [ ] Live pitches: at least 3 agencies have sent a Shortlisty pitch board to a real client
- [ ] First approval: at least 1 client has approved via the board (not just viewed it)
- [ ] Onboarding time: average time from signup to first pitch board under 30 minutes
- [ ] Feedback loop: at least 10 structured agency interviews captured

When these are met, the demo is credible, the pipeline is proven, and the product is ready for unassisted adoption.

---

## Expanding beyond the launch city

Once the launch city has 5+ paying agencies:

1. Pick the next city using the same criteria — active market, mid-size, tight community
2. Build the seed catalog for that city (1 week of work)
3. Identify 3–5 agencies in that city via LinkedIn or MPI/PCMA chapter directories
4. Run the same concierge playbook

Each city adds a self-contained growth pocket. Cross-city word-of-mouth is rare at this stage — treat each city as its own cold start until organic inbound begins.

---

## Concrete outreach channels by audience segment

To turn the concierge model into a repeatable pipeline, outreach focuses on five segments, each with distinct channels and a tailored opener. Each segment maps to a specific toolstack (see below) because the opener references tools the prospect already uses.

### 1. Solo wedding planners (15–40 events/year)

These planners run on HoneyBook or Aisle Planner + Instagram + WhatsApp. They value beauty and speed over enterprise features.

**Channels:**

- **Instagram / TikTok:** Search hashtags `#WeddingPlanner[City]` + `#[City]Weddings`. Engage with recent venue posts (genuine comment, not a pitch), then DM referencing the specific venue they posted.
- **The Knot / WeddingWire / Wezoree / Zola directories:** Filter by city, export agency names, visit their websites, cold email the founder directly.
- **Local wedding association chapters (WIPA, ABC):** Attend chapter meetups as a supplier/guest.

**Opener pattern:** "I saw you posted Villa Medici last month — I built a tool that pulls your venue PDFs into a searchable catalog and turns 3 venues into a branded client link in 60 seconds. Want to see it with venues you've actually worked with?"

### 2. Small event agencies (2–8 people)

These teams live in Notion + Drive + Asana or Monday.com + Canva. They run mixed corporate and social work.

**Channels:**

- **LinkedIn advanced search:** Boolean `("event agency" OR "event management") AND "[City]" AND ("founder" OR "director" OR "owner")`. Engage with a recent post (genuine comment) before sending a personalized connection request referencing a specific event they ran.
- **MPI (Meeting Professionals International) / PCMA (Professional Convention Management Association) chapter directories:** Most chapters publish member lists or host local networking events. Join as a supplier partner if the tier is under $500/year.
- **EventPlanning.com / FindEventProfs.com / Event-Directory.com directories:** Filter by city and agency size.

**Opener pattern:** "I looked at [their recent corporate event case study] — most agencies we talk to lose 2 hours per brief hunting through Drive folders and Notion tables for the right venues. Shortlisty turns that into 5 minutes. Want to see a demo with 5 Naples venues you'd recognise?"

### 3. In-house corporate event leads

These people sit inside mid-to-large companies, report to Marketing or HR, and use Cvent or Bizzabo for conferences + Asana for project management. They are harder to reach but have budget and recurring venue problems.

**Channels:**

- **LinkedIn Sales Navigator:** Titles: "Head of Events," "Director of Corporate Events," "Events Marketing Manager," "Senior Event Planner." Filter by company size (200–5,000 employees) + industry (tech, finance, professional services) + city.
- **Via agency introductions:** Corporate leads trust their agency's vendor recommendations far more than cold outreach. Ask every agency that goes live: "Who are the three best corporate event leads you work with? I'll give you a free month for a warm intro."

**Opener pattern:** "I work with [agency they know] here in Naples. They cut their venue sourcing time per brief from 2 hours to 10 minutes using Shortlisty. Do you have 15 minutes next week to see a 3-venue demo with local spots?"

### 4. Venue managers / venue sales teams

Venue managers are a secondary audience — they are not the primary buyer, but they are connectors, and a version of the product that helps them send pitches to planners is a natural upsell.

**Channels:**

- **IAVM (International Association of Venue Managers) chapters and member directory**
- **Direct outreach via venue website contact forms** — the sales or events manager usually responds if you reference their venue's specific materials: "I was reviewing your rooftop deck PDF — I noticed you send custom proposals to planners one-off. Shortlisty can turn your deck into a branded interactive link in 30 seconds."
- **Venue open days and supplier networking nights** — every mid-size city has these. In-person handshakes beat cold email for venue people.

### 5. DMCs (Destination Management Companies)

DMCs manage venues across entire regions for corporate clients. They manage more venues per brief than any other segment and are the highest-LTV early adopter candidate.

**Channels:**

- **ADMEI (Association of Destination Management Executives International) directory**
- **MPI conferences** — DMCs are heavily represented at MPI WEC and European MPI events
- **Referral via local agencies** — agencies work with DMCs on out-of-town events and make warm intros

**Opener pattern:** "Most DMCs I talk to manage 200+ venues across 3–5 cities but still run on spreadsheets and 10-year-old Access databases. Shortlisty turns every venue deck you already have into a searchable catalog with one-click client pitch links. Want to see it with 10 of your venues imported free?"

### Outreach cadence: quick start plan for month one

1. LinkedIn Sales Navigator: 3 saved searches (solo planners, small agencies, corporate leads) in the launch city. 10 saved leads per search.
2. Join 3 associations/groups: MPI local chapter, WIPA local chapter, one LinkedIn group (Event Planning & Event Management or BizBash).
3. Browse 3 directories: The Knot (weddings), EventPlanning.com (agencies), one local city business directory. Export/save 20–30 profiles.
4. First 10 conversations: aim for "send me your Drive folder and I'll import 5 venues for free" over "sign up for a demo." The concierge onboarding is the pitch.

---

## Toolstack landscape and integration priority roadmap

Early adopters do not live in Shortlisty. They live in a 5–8 tool stack that already works most days. Shortlisty must slot in next to it, not replace it. Integration priority maps to how often each tool appears in the ICP segments above.

### Tier 1 — Must have before v1.0 commercial launch

| Tool / platform                      | Why it matters                                                                                                                                                                         | Integration target                                                                                                                       |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **Google Drive / Dropbox**           | 95% of agencies store venue source files here. Concierge onboarding today is "share the Drive folder" — import should be a one-click OAuth connect, not a manual download-then-upload. | One-click import of selected folder(s) → Shortlisty triggers extraction pipeline. File browser picker UI inside the app.                 |
| **Google Workspace / Microsoft 365** | Client pitches are sent via Gmail or Outlook. Deal Room links are composed in email, Approve confirmations come back into email.                                                       | Send pitch link directly from app via connected Gmail/Outlook account. Inbound Approve email receipts auto-linked to Deal Room timeline. |

### Tier 2 — Ship in v1.1, before aggressive self-serve

| Tool / platform   | Why it matters                                                                                                                                                                                                                   | Integration target                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **HoneyBook**     | Dominant CRM/proposal tool for solo wedding planners and small boutique agencies. Shortlisty must not feel like "yet another CRM" — it feeds pitch-ready venue selections back into HoneyBook proposals.                         | OAuth connect → **Pull**: read existing HoneyBook client briefs (headcount, dates, style tags, event type) → auto-populate pitch context and seed Shortlisty chat-search filters. **Push**: on Approve, write the approved venue snapshot back as (1) a HoneyBook proposal seed with venue contact, capacity, catering policy, and floor-plan assets pre-filled; (2) brief custom fields reflecting the client's Deal Room preferences.                                                                                                       |
| **Aisle Planner** | Wedding planner specific. Shares a user base with HoneyBook, but Aisle Planner has deeper wedding-specific workflow (timelines, seating, design boards, BEOs). Planners here live inside the design board and timeline features. | OAuth connect → **Pull**: pull Aisle Planner client brief questionnaire answers, guest count ranges, and wedding-style mood-board tags → seed context for Shortlisty search and pitch. **Push**: on Approve, write the approved venue as (1) proposal line-item seed in the Aisle Planner proposal builder; (2) timeline anchor milestone (venue date, curfew, guest cap, catering rules) in the timeline template system; (3) venue assets (floor plans, photos, PDFs) into the Aisle Planner design-board asset library to avoid re-upload. |
| **Dubsado**       | Automation-heavy solopreneur CRM. Heavy user base among planners who want everything wired together. Dubsado's killer feature is the automated workflow chain.                                                                   | OAuth connect → **Pull**: Dubsado form and questionnaire answers → brief context in Shortlisty. **Push**: Deal Room Approve event automatically triggers Dubsado workflow stages: "Approve received" → advance pipeline, auto-fire contract template, kick off invoice, ping the planner. Client Deal Room preferences (format, catering rules, AV flags) → written to Dubsado project custom fields for downstream tasks.                                                                                                                    |

### Tier 3 — v1.2–v1.3, enterprise and volume segments

| Tool / platform                  | Why it matters                                                                                                                                                                        | Integration target                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Asana / Monday.com / ClickUp** | Corporate event leads and larger agencies run multi-stakeholder projects here. Shortlisty's Deal Room activities should write back to the project rather than exist in a silo.        | Approve event → auto-complete the matching Asana/Monday task with a comment linking the Deal Room history snapshot. Client preferences captured in Shortlisty → written as subtasks or custom fields. Brief context (guests, date, style tags) pulled from Asana project description + custom fields → seed Shortlisty pitch context.                                                                                                                                                                                                                     |
| **Planning Pod**                 | Mixed venue-corporate teams use Planning Pod for budgets, BEOs, and floor plans. Planning Pod has a venue module — Shortlisty complements it with a richer knowledge and pitch layer. | **Pull**: client questionnaires, event briefs, custom budget envelopes from Planning Pod → seed Shortlisty search context and pitch brief. **Push**: on Approve, sync approved venue + metadata as (1) a **BEO seed** in the Planning Pod BEO builder (capacity, catering policy, F&B rules, contact, AV defaults pre-filled); (2) budget line items reflecting venue fee and catering estimate; (3) bi-directional floor-plan asset sync so Planner updates to a PDF floor plan in Shortlisty show up in Planning Pod floor-plan layouts and vice versa. |
| **Cvent Vendor Marketplace**     | Discovery complement. Planners find a new venue via Cvent → one-click import into Shortlisty personal catalog for extraction and permanent storage.                                   | Cvent vendor listing deep link → import venue's public materials (deck, floor plans, photos) into Shortlisty with a single bookmarklet or extension click.                                                                                                                                                                                                                                                                                                                                                                                                |
| **Notion**                       | The default DIY knowledge base. Agencies who have invested in a Notion venue database want a gradual migration path, not a cold switch.                                               | One-way read of a Notion venue database into Shortlisty as a seed import. Optional two-way sync for text notes and tags after v2.0.                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **Canva**                        | Planners today recreate custom pitch visuals in Canva for every brief. Shortlisty's output should be brand-equivalent so they don't feel the need.                                    | Export Deal Room venue card grid as a Canva-compatible template (PNG assets + metadata CSV) for the 10% of cases where they need bespoke design output Canva still does better.                                                                                                                                                                                                                                                                                                                                                                           |

### How integrations reinforce the positioning

The pattern across all tiers is consistent: Shortlisty is a venue knowledge + pitch approval specialist that plugs into the tools the agency already runs their business on. The value proposition is not "replace your stack" — it is "make the venue part of your stack 10x faster and 10x more pleasant, without making you leave anything behind."

---

**Docs:** [What is Shortlisty?](../../README.md) · [Product Structure](product.md) · [Business Proposal](proposal.md) · [Vision](../../roadmap/vision.md) · [Competitive Landscape](comparison.md)
