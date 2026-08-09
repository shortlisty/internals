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
3. Create a venue record in StashRoom — name, address, source URL
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
- [ ] Live pitches: at least 3 agencies have sent a StashRoom pitch board to a real client
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

**Docs:** [What is StashRoom?](../../README.md) · [Product Structure](product.md) · [Business Proposal](proposal.md) · [Vision](../../roadmap/vision.md)
