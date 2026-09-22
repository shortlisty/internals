# AlphaVenue.ai — Deep-Dive Spike

> **Audience:** Founders, team.
> **Purpose:** Detailed capability analysis of AlphaVenue.ai across the full platform — not just the AI sales layer. Assess where it sits relative to Shortlisty and whether any of its capabilities represent genuine architectural overlap or competitive risk.

---

## TL;DR

AlphaVenue is the most fully featured of the venue-side AI platforms. Where VenueX, Mikla, and VenueAI are focused narrowly on lead-response automation, AlphaVenue extends that into AI-powered wedding planning, floor plan creation (2D + 3D), guest seating, meal selection, team training, and an internal AI help desk. It is built by a practicing venue operator (3 venues, 200+ events/year), not a software company — and that shows in the depth of the operational scope.

It does not compete with Shortlisty. But it is the most sophisticated venue-side platform currently in the market, and it is moving toward the same informational infrastructure (a centralized venue knowledge base) from the opposite direction.

**Side:** Venue operator. Not planner.
**Threat level to Shortlisty:** Low — different side of the market, different user, different data flow.
**Interesting for Shortlisty:** The Venue Intelligence module is architecturally similar to Shortlisty's catalog layer, just pointed inward (venue trains its own AI) rather than outward (planner extracts intel from venue documents).

---

## 1. Product overview

AlphaVenue is an all-in-one AI operations platform for wedding and event venues. Its central character is **Ella** — a named, venue-branded AI agent that the venue owner configures with their own pricing, packages, policies, floor plans, FAQs, availability, and brand voice. Once configured, Ella runs autonomously across sales, planning, and client support.

The platform is positioned explicitly against:

- Generic chatbots (FAQ widgets)
- Lead generation marketplaces (WeddingWire, The Knot)
- Marketing agencies
- One-size-fits-all CRM software

**Built by:** Adam J. Etrheim — owner/operator of 3 award-winning venues, 200+ events/year since 2012. The "built by venue owners, not outsiders" positioning is central to the product identity.

**Pricing:** Quote-based, custom per venue. Published ROI math: ~$33,588/year for the fully autonomous tier (white-glove onboarding included). Modeled average return: $170,000/year ($65K saved + $105K additional sales). 90-day performance guarantee — if the system hasn't automated lead flow and delivered meaningful operational improvement, three additional months at no cost.

---

## 2. Capability map

### 2.1 AI Sales Engine

The core, and where most of the marketing emphasis sits.

**What it does:**

- Responds to new leads in seconds across **text, email, website chat, voice, and video** — all channels simultaneously
- Handles inbound from **WeddingWire, The Knot, website forms, Google, Instagram, Facebook**
- Sends personalised replies with available dates, venue info, sneak-peek video links, catering details
- Qualifies prospects and books tours directly onto the venue's calendar
- Builds and sends proposals
- Supports contract workflows
- Runs follow-up sequences (lead nurturing between inquiry and tour, tour reminder, post-tour nudge)
- Schedules planning meetings and consultations

**Guardrail model:** Ella does not operate freely. The venue owner defines upfront what Ella can answer, what she can send, when she follows up, what pricing/packages she uses, and when she escalates to a human. She runs the approved playbook automatically — no per-message approval required.

**Draft mode:** An approval layer exists for messages the venue wants to review before they go out. The UI shows a draft with Approve / Edit / Reject controls.

**Channel depth:** The addition of **voice and video** distinguishes AlphaVenue from most competitors. VenueAI also has voice; Mikla has voice cloning on the Pro tier. But video is unique to AlphaVenue in this set — Ella can host planning meetings over video, not just respond to text/email.

---

### 2.2 Venue Intelligence (Knowledge Base)

The architectural foundation behind Ella. This is the most strategically interesting module from a Shortlisty perspective.

**What it is:** A centralised venue knowledge base — a single source of truth that Ella draws from across every channel and interaction.

**What goes into it:**

- Pricing and packages
- Menus, amenities, venue details
- Policies, contracts, FAQs
- Photos and floor plans
- Availability and booked dates
- Contact information
- Vendor information and service guidelines
- Brand voice and communication style

**How it's populated:** The website describes "PDFs, inboxes, spreadsheets, calendars, contracts, and your team's heads" as the sources — suggesting document ingestion from existing venue files. The exact ETL mechanism (manual upload vs. automated extraction vs. web crawl) is not publicly detailed, but the framing ("one centralized source of venue intelligence... instead of relying on scattered documents, team memory") mirrors Shortlisty's catalog architecture almost exactly — just pointed inward.

**The consistency argument:** The pitch is that the same question gets the same answer across chat, text, email, voice, and video. The illustrated failure mode is identical to Shortlisty's: a venue coordinator says "I think so", a colleague texts a different answer, the brochure says something else, and the sticky note says "ask the owner."

**Security:** Data encryption, secure servers, access controls mentioned — no detail on certifications or compliance frameworks.

**Shortlisty parallel:** Shortlisty builds a structured venue knowledge base from documents the _planner_ owns (venue decks, PDFs, floor plans sent by the venue). AlphaVenue builds a structured venue knowledge base from documents the _venue_ owns. Same infrastructure problem, opposite direction of the data flow.

---

### 2.3 AI Wedding and Event Planner

A distinct capability layer beyond sales — Ella hosts actual planning meetings.

**What it does:**

- Conducts planning conversations with booked couples around the clock
- Covers: finalising event details, answering planning questions, managing client communication, sending reminders
- "Expert knowledge, human warmth, your venue's unique voice"
- Scales: supports dozens of simultaneous planning conversations without degradation

This is materially different from VenueX/Mikla/NurturePro, which stop at inquiry-to-tour conversion. AlphaVenue continues through the full event lifecycle — post-booking planning, not just pre-booking conversion.

---

### 2.4 Floor Plan Creator

**What it does:** Creates event-ready floor plans in **2D and 3D** directly within the platform.

**Who uses it:** The UI copy suggests couples do the configuration work themselves ("so your couples do the work for you"), with venue-defined parameters and room setups as the base.

**What's notable:**

- 2D and 3D rendering in one tool — this is the most capable floor plan feature in any of the venue AI platforms surveyed
- The context is event layout (tables, head table, rounds, linens) — not architectural extraction
- No mention of CAD/DXF import or PDF floor plan parsing — this appears to be a layout builder, not a document extraction tool
- The example shown: "Grand Ballroom · cap. 200 / 10 rounds and a head table · 90 seats / Linen"

**Shortlisty parallel:** Shortlisty extracts structured venue data _from_ floor plan PDFs (ETL/parsing). AlphaVenue creates floor plan _layouts_ for events in real time. Adjacent capability, different direction. AlphaVenue's floor plan creator is downstream (layout for a confirmed booking); Shortlisty's floor plan intelligence is upstream (extraction for portfolio intelligence before booking).

---

### 2.5 Guest Seating

A self-service tool for couples to manage their own seating chart.

**What it does:**

- Couples manage guest placement and table assignments directly
- Reduces venue team involvement in seating logistics
- Tracks seating progress (example shown: "9 of 14 seated")

---

### 2.6 Guest Meal Choice Selector

**What it does:**

- Collects and organises guest meal selections in one place
- Tracks choices per guest (example shown: Beef 4 / Fish 3 / Garden 2)
- Eliminates the scattered email/spreadsheet approach for meal counts

---

### 2.7 Tour and Meeting Scheduling

**What it does:**

- Availability calendar with real-time visibility for couples ("Available dates: instant visibility so couples can decide faster")
- Tour scheduling with availability rules, confirmations, and reminders — no back-and-forth
- Meeting scheduling for planning consultations — same flow

---

### 2.8 Contracts and Payments

**What it does:**

- Send and e-sign contracts digitally within the platform
- Card processing: deposits, invoices, event payments

---

### 2.9 AI Team Trainer

**What it does:** An internal-facing AI agent — not client-facing — trained on venue operations to give new team members instant answers and accelerate onboarding.

This is unusual in the venue AI space. It treats the venue's knowledge base as a training resource for human staff, not just an answer engine for clients. Strategically it means the knowledge base has dual value: it powers Ella externally and onboards staff internally.

---

### 2.10 AI Help Desk

**What it does:** An always-on internal support tool that "answers questions, shows you the answers, and even does the work for you."

This appears to be a venue-operator-facing assistant — less about client interaction and more about operational support for the venue's own team. Details are sparse on the public site.

---

## 3. Architectural observations

### The knowledge base as the moat

AlphaVenue's most durable competitive advantage is not Ella — it is the **Venue Intelligence knowledge base**. Any competitor can build a chatbot that answers leads fast. The hard part is building a structured, accurate, conflict-free representation of how a specific venue operates. AlphaVenue's pitch is that once a venue has invested in building that knowledge base, Ella becomes a genuine extension of the team rather than a generic bot.

This is structurally identical to Shortlisty's core bet: the catalog (the structured representation of venue knowledge) is the moat, not the UI that sits on top of it.

### Full lifecycle coverage

AlphaVenue is the only venue-side AI platform that spans the complete venue-client lifecycle:

```
Inquiry → Lead qualification → Tour booking → Proposal → Contract →
Deposit → Planning meetings → Seating → Meal choices → Event execution
```

VenueX, Mikla, VenueAI, and NurturePro stop at "tour booked." AlphaVenue keeps going. This makes it the closest thing to an operating system for a venue's client-facing operations.

### Vertical depth vs. horizontal breadth

AlphaVenue chose vertical depth (everything a wedding venue needs, end to end) over horizontal breadth (many venue types, many industries). Mikla claims "151 industries" but is clearly optimised for wedding venues. AlphaVenue makes the vertical bet explicitly.

### Pricing model as a signal

Quote-based, ~$33,588/year for the full platform. This is a significant price point — 10–20x higher than Mikla ($149–$499/month) or WedyPro ($25–$35/month). The rationale is that the annual investment covers white-glove setup, customisation, support, and ongoing optimisation — not just software access. It positions AlphaVenue as a premium managed service, not self-serve SaaS. That also means slower distribution and lower volume than the transactional SaaS competitors.

---

## 4. What AlphaVenue does NOT do

| Capability                                         | AlphaVenue |
| -------------------------------------------------- | ---------- |
| Planner-owned venue portfolio management           | ⛔         |
| Document extraction from planner-supplied files    | ⛔         |
| Cross-venue search and comparison for planners     | ⛔         |
| Client-facing pitch/shortlist for planner's client | ⛔         |
| Approval → immutable snapshot (SSOT)               | ⛔         |
| Multi-source aggregation / conflict resolution     | ⛔         |
| Semantic search across extracted venue metadata    | ⛔         |
| Provenance-tagged metadata with confidence tiers   | ⛔         |
| Planner-side audit trail of what was agreed        | ⛔         |

None of these are gaps in AlphaVenue's product — they are simply out of scope because AlphaVenue serves venue operators, not event planners.

---

## 5. Relationship to Shortlisty

### Not a competitor

AlphaVenue and Shortlisty serve opposite ends of the same transaction. AlphaVenue helps a venue respond to the planner's inquiry; Shortlisty helps the planner decide which venue to send the inquiry to.

### Potentially complementary

A venue using AlphaVenue has a well-structured internal knowledge base (pricing, packages, floor plans, policies, availability). That knowledge base is exactly what Shortlisty's ETL pipeline would want to ingest. If AlphaVenue ever exposes a structured data export, Shortlisty could pull from it directly — instead of parsing the venue's raw PDF deck.

The planner workflow: planner sends inquiry → AlphaVenue (Ella) responds with structured proposal → planner ingests that proposal into Shortlisty → Shortlisty extracts structured venue record → venue goes into the catalog.

### Architectural mirror

Both products solve the same underlying problem (scattered, document-bound, person-dependent venue knowledge) for different users. AlphaVenue solves it for the venue; Shortlisty solves it for the planner. The parallel is precise:

| AlphaVenue                                              | Shortlisty                                         |
| ------------------------------------------------------- | -------------------------------------------------- |
| Venue's own pricing/packages/policies → knowledge base  | Venue's PDFs/decks/floor plans → planner's catalog |
| Ella answers planner inquiries from that knowledge base | Planner searches catalog to build client pitches   |
| Same answer across text/email/voice/video               | Same venue data across brief → pitch → approval    |
| Reduces "ask the owner" dependency                      | Reduces "where's that venue deck" dependency       |
| AI team trainer: staff get answers from KB              | Planner team searches shared catalog               |
| Proprietary to the venue                                | Proprietary to the agency                          |

---

## 6. Verdict for comparison.md

AlphaVenue should be added to section 4 (venue management platforms) with a note distinguishing it from the simpler lead-response tools. It warrants more narrative depth than VenueX or Mikla because:

1. It covers the full event lifecycle, not just lead conversion
2. Its Venue Intelligence module is architecturally interesting relative to Shortlisty's catalog
3. Its pricing and positioning ($33K/year, white-glove) represents a different commercial model than the transactional SaaS competitors
4. The floor plan creator, guest seating, meal selection, team trainer, and internal help desk collectively make it a genuine operational platform, not a chatbot with good marketing

---

**Docs:** [Competitive Landscape](../comparison.md) · [Product — DSR](../digital-sales-room-for-events/product.md) · [Platform Intelligence](../../platform/intelligence.md)
