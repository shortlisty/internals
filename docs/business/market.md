# Market structure

> **Audience:** Anyone.
> **Purpose:** Explain how the event industry software market is structured — the roles, the tools, who builds for whom, and where the gap is. A mental model document, not a competitive analysis. For detailed per-tool breakdowns see [Digital_Sales_Room_for_Events/comparison.md](Digital_Sales_Room_for_Events/comparison.md).

---

## The event chain

Every event involves five distinct roles. Understanding them is the key to understanding why the software market is fragmented the way it is.

**Venue owner** — the entity that owns the physical space: a school district, a hotel group, a performing arts centre, a city parks department. They care about utilisation, revenue from rentals, maintenance, and compliance.

**Venue operator** — the team that manages the space day-to-day: scheduling what goes in and out, handling bookings, managing contracts, coordinating setup and cleanup. Often the same organisation as the venue owner, sometimes outsourced.

**Event producer / production team** — the people who execute the event on the ground: stage managers, technical directors, production managers, crew coordinators. They care about timelines, equipment, labour, and budgets for a specific event.

**Event manager / agency** — the professional who finds the right venue for a client, briefs it, and coordinates the high-level plan. They work across many venues and many clients. Their job is to know the landscape of available venues and match briefs to the right space. This is the role OiQb Intelligence is built for.

**End client** — the person or company commissioning the event: a bride and groom, a corporate marketing team, a conference organiser. They do not manage venues or production — they receive the output of the event manager's work.

---

## How the market is segmented

Every software tool in this market is built for exactly one of these roles. Nobody crosses the boundary deliberately — the business models do not allow it.

### Venue-side tools

Built for venue owners and operators. Their job is to make the venue run more efficiently and earn more rental revenue.

- **Facilitron** — scheduling, rental management, work orders, and compliance for schools, districts, and municipalities. Revenue model: service fee on rental income, no cost to the venue.
- **Tripleseat** — sales and catering software for restaurants, hotels, and hospitality venues. Booking, contracts, invoicing, F&B management.
- **Momentus Technologies** — enterprise operations for convention centres, stadiums, and performing arts venues. End-to-end: booking through finance through analytics.
- **VenueArc** — booking and settlement software for performing arts centres and theatres.

All four serve the venue. None of them help an event manager find or brief the venue.

### Production tools

Built for the production team executing a specific event.

- **Propared** — scheduling, crew booking, inventory tracking, and budgeting for theatres, festivals, and event production companies. The production manager's operating system for a show.

Propared serves the people on the ground making the event happen. It has no interest in the venue selection process that happened before they arrived.

### Discovery platforms

Built to connect event managers with venues they have not worked with yet. Revenue model: the venue pays, not the planner.

- **Cvent** — the largest venue marketplace. 340K+ self-submitted venue listings, RFP automation, enterprise contracts.
- **VenueScanner** — UK-focused marketplace. 19,000+ venues, free for organisers, commission from venues.
- **VenueFindAI** — AI-assisted venue matching with a human concierge layer. Free to planners, venue-side revenue.

All three monetise the venue, not the planner. Their value proposition to venues is: "we send you leads." Their value proposition to planners is: "we show you venues you don't know yet." The moment a planner already knows which venues they trust, these platforms have nothing left to offer.

### Content and workflow tools

Built to help event professionals work faster — writing, briefing, reporting.

- **Spark (GEVME/PCMA)** — 150+ AI task templates for the event lifecycle: agenda drafting, RFP copy, speaker bios, post-event reports. Helps planners write faster. Does not help them know their venues better.

### Generic asset management

Built for marketing and brand teams to store, tag, and distribute digital files.

- **Bynder** — enterprise DAM. Centralised asset library with AI auto-tagging and brand governance workflows.
- **Brandfolder** — mid-market DAM. Smart content classification, natural language search, Smartsheet integration.

Both can store venue PDFs. Neither understands what is inside them. They tag a floor plan as "document" and stop there.

---

## The monetization pattern

Every tool above either charges the venue (Facilitron, Tripleseat, Momentus, VenueArc, Cvent, VenueScanner, VenueFindAI) or charges a generic professional buyer (Propared, Bynder, Brandfolder, Spark).

Nobody charges the event manager as a knowledge worker.

The reason is historical: event managers were seen as a distribution channel for venues, not as a paying persona in their own right. Discovery platforms gave planners free access because planners were the mechanism for getting venue bookings — the venue paid. That made sense when the product was discovery. It created a blind spot for everything that happens after discovery.

After discovery, the event manager has a venue. They have files — PDFs, floor plans, spec sheets, photo sets — and they need to turn that raw material into actionable knowledge. That workflow has no software. It happens in email, in shared drives, in spreadsheets, and in people's heads.

---

## The vacant slot

The gap is precise: **the event manager as a professional knowledge worker managing a portfolio of known venues**.

Not discovering new venues — they already know the ones they trust.
Not executing an event — that is the production team's job.
Not managing their own space — they do not own any venues.
Not generating content — they need facts before they can write anything.

What they need: a structured, searchable, shared knowledge base built from the documents venues have sent them over years of working relationships. The knowledge exists. It is just buried in files.

No incumbent fills this slot because:

1. Venue-side tools have no incentive to help planners be smarter — they want bookings.
2. Discovery platforms only know what venues self-submit — they cannot read a planner's own documents.
3. Generic DAMs store files with generic tags — they do not understand venue semantics.
4. AI writing tools generate text — they have no memory, no schema, no team sharing.
5. Production tools start after the venue is booked — the knowledge management problem is upstream.

The slot is vacant not because nobody noticed it, but because every incumbent's business model points away from it.

---

## Where OiQb sits

OiQb Intelligence is the tool for the vacant slot: the event manager's venue knowledge base.

It sits downstream of discovery (you already know the venues) and upstream of production (you are still in the briefing and selection phase). It serves one role — the event manager and the agency they work in — and does one thing: turns the venue documents they already have into a structured, searchable, shared library.

The business model follows the gap: charge the planner, not the venue. The event manager is the paying customer. The knowledge base is the product.

For the detailed per-tool competitive analysis: [Digital_Sales_Room_for_Events/comparison.md](Digital_Sales_Room_for_Events/comparison.md).
For the product structure: [Digital_Sales_Room_for_Events/product.md](Digital_Sales_Room_for_Events/product.md).
For the business case: [Digital_Sales_Room_for_Events/proposal.md](Digital_Sales_Room_for_Events/proposal.md).

---

## Market References

```

├── 1. Venue Catalog / Discovery
│   ├── Peerspace
│   ├── Tagvenue
│   ├── Giggster
│   ├── VenueFindAI
│   ├── VenuClaw (AI Venue Finder)
│   ├── Naboo (AI event procurement)
│   ├── BoomPop
│   ├── VenueScanner
│   ├── Lime Venue Portfolio
│   ├── The Vendry
│   └── Spalba / Revel Street (regional)
│
├── 2. PIM (Product Information Management)
│   ├── Akeneo (PIM + DAM demo)
│   ├── Cvent Supplier Network (venue data structure)
│   ├── Ventur3 (RFP + structured venue data)
│   ├── Planner Hero
│   └── Facilitron (facility data model)
│
├── 3. DAM (Digital Asset Management)
│   ├── CELUM — Best DAM systems 2026
│   ├── Pixx.io
│   ├── Pics.io
│   ├── Akeneo DAM Extension
│   └── Siemens DAM references
│
├── 4. Pitch Presentation / Sales Room
│   ├── Envelope (AI event planning platform)
│   ├── SPOVIX (AI event & venue management)
│   ├── Tripleseat (venue marketplace + sales tools)
│   ├── HoneyBook (client-facing proposals)
│   ├── Dubsado (proposals + CRM)
│   └── PlanningPod
│
└── Adjacent / Context
    ├── Open-source: EasyVenue, eventseats, venue
    ├── Lists: Capterra Venue Management, PlanningPod comparison
    └── Reddit: r/EventPlanners, r/techtheatre (venue software)
```

---

**Docs:** [What is oiqb?](../README.md) · [Product Structure](Digital_Sales_Room_for_Events/product.md) · [Business Proposal](Digital_Sales_Room_for_Events/proposal.md) · [Competitive Landscape](Digital_Sales_Room_for_Events/comparison.md) · [Vision](../roadmap/vision.md)
