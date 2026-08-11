# Cold start strategy

> [!NOTE]
> This document was written for the **Personal Venue Catalog** segment — an internal knowledge-base tool for event agencies. It is a reference example for that positioning, not the primary product direction. The current product focus is the [Digital Sales Room for Events](../../Digital_Sales_Room_for_Events/README.md).

> **Audience:** Founders, team.
> **Purpose:** How to seed the venue library before the first paying customer — collecting real venue data during development to solve the empty-library problem, test the ETL pipeline on real documents, and build demo content that is credible.

---

## The problem

A venue library with one venue in it is not useful. An event manager who signs up and sees an empty screen does not get the aha moment — they get the onboarding problem. The product only clicks when the library already has substance.

This creates a bootstrapping challenge: the product needs data to demonstrate value, but customers provide the data only after they see value.

The solution is to collect the seed data ourselves during development. This is not a workaround — it is a development requirement. Collecting and ingesting 50–100 real venues before launch achieves three things simultaneously:

1. **ETL pipeline testing** — real venue documents are the only honest test of extraction quality. Clean, invented test fixtures do not reveal the parsing failures that matter.
2. **Demo content** — a live demo with real, recognisable venues in a specific city is orders of magnitude more convincing than placeholder data.
3. **Free-tier seed library** — new users on the free tier start with a non-empty library. Their first interaction is a search that returns results, not an empty state.

This is also the 50-document benchmark called out in [vision.md](../../roadmap/vision.md) under strategic bets. Collecting seed data and running the accuracy benchmark are the same activity.

---

## Geographic focus

### Why tier-2 US cities

Tier-1 cities (NYC, LA, Chicago) are obvious but have three problems for a cold-start strategy:

- Venue count is overwhelming — there are thousands of options, which makes curation harder
- Competition for early customers is higher — event agencies in NYC are already well-served by existing tools and harder to convert
- Noise-to-signal ratio in public venue data is higher — more venues means more low-quality listings

Tier-2 cities have active, professional event markets with a concentrated set of well-known venues. A planner in Austin knows 40–60 venues by name. Seeding those 40–60 gives the product immediate local credibility. When a local planner signs up and searches "rooftop venue for 80 guests" and the right answer comes back, the aha moment is instant.

Additionally, word-of-mouth travels faster in smaller professional communities. One agency owner in Nashville telling two colleagues is worth more than one agency owner in Manhattan where the same referral gets lost in noise.

### Primary seed cities

Three cities, 30–40 venues each, chosen for active event markets, concentration of SMB agencies, and good public venue data availability.

**Austin, TX**
Strong corporate event market (SXSW ecosystem, tech company offsites), growing agency scene, active social events (weddings, galas). Venue culture leans toward unique spaces — converted warehouses, rooftop terraces, hotel ballrooms. Strong public web presence for local venues.

**Nashville, TN**
One of the fastest-growing US event markets. Corporate events, conferences, bachelorette/bachelor industry, music-adjacent unique venues. High density of boutique hotel event spaces and non-traditional venues. Venues are web-savvy and typically publish spec sheets and floor plans.

**Miami, FL**
Active corporate event market, international flavour, strong hospitality sector. Art Deco hotel ballrooms, rooftop venues, waterfront spaces. High concentration of independent event spaces with public-facing marketing material.

**Reserve cities** (expand to if the primary three go well): Denver, Charlotte, Nashville, Portland, San Antonio.

### What to avoid

- Tier-1 cities (NYC, LA, Chicago, Las Vegas) — save for the demo pitch deck, not the seed library
- Stadium, arena, and convention centre venues — too large, operator-side tools already cover them, not VenueMi's ICP
- Chain hotel ballrooms (Marriott, Hilton, Hyatt branded) — their venue data is centralised and controlled, not publicly accessible per-property
- Venues with no web presence or no downloadable spec materials — they are not useful for ETL testing

---

## Venue types to target

### Selection criteria

A good seed venue is:

- SMB-operated: independently owned or a small regional group (not a national chain)
- Publicly marketed: has a website with event/venue information, ideally a downloadable spec sheet or floor plan
- Actively hosting events: listed on Google Maps as an event venue, has reviews, has photos
- Capacity range: 30–300 guests — the core VenueMi ICP brief range
- Data richness: multiple document types available (PDF deck, photos, floor plan, or at minimum a detailed website)

### Categories

**Event lofts and creative spaces**
Converted industrial or commercial spaces repurposed as event venues. Typically have detailed floor plans and capacity charts. Strong web presence with downloadable PDFs. Examples: art studios, warehouse lofts, converted factories.

**Boutique hotel event spaces**
Independent and boutique hotels (not branded chains) with ballrooms or event floors. Often publish venue packages, floor plans, and catering menus as downloadable PDFs. Good for testing multi-document ingestion (package PDF + floor plan + photo gallery).

**Restaurants with private dining**
Private rooms and buyout options at notable local restaurants. Typically publish PDFs or web pages with capacity, catering policy, and setup options. Good for testing extraction of catering constraints, capacity per configuration, and restrictions.

**Rooftop terraces and outdoor spaces**
Rooftop venues, garden spaces, and outdoor event areas at hotels or standalone venues. Rich photo content. Often have published specifications for standing vs. seated capacity, weather restrictions, and curfews.

**Community and cultural venues**
Local arts centres, galleries, cultural centres, and community event spaces. Often publicly funded or non-profit, which means their venue data is openly published. Good diversity of document quality — some are polished PDFs, some are basic web pages.

**Historic and character venues**
Mansions, estates, historic buildings, and unique architectural spaces. Strong photo presence. Typically publish detailed spec sheets because their uniqueness requires explanation. Good for testing extraction of unusual constraints (preservation rules, access restrictions, load-in limitations).

---

## Free and open data sources

### Venue websites (primary)

Most SMB event venues publish spec sheets, floor plans, and photo galleries on their own websites. These are publicly accessible and intended to be shared — that is their purpose. Downloading them for extraction testing and seed data is legal and appropriate.

What to look for:

- PDF downloads labelled "venue package", "event brochure", "floor plan", "capacity chart", "catering menu"
- Photo galleries (right-click save or linked image files)
- Embedded Google Maps links (confirms address and pin)

### Google Maps / Places

Google Maps public venue pages include photos uploaded by the venue, capacity information from the venue's own listing, and user-uploaded photos. The Google Places API free tier provides structured data (address, phone, website, category, hours, reviews) for up to $200/month of free usage — sufficient for seeding 100–200 venues.

Specifically useful for:

- Confirming venue existence and address
- Pulling category tags (event venue, ballroom, rooftop bar, etc.)
- Getting a starting set of photos via the Places Photos API

**Note:** Do not scrape Google Maps pages with a bot. Use the official Places API within its terms of service.

### OpenStreetMap

OSM tagging includes `amenity=event_venue`, `leisure=` tags, and free-form venue attributes contributed by the community. Useful for geo-coordinates and basic categorisation. Data quality varies — treat as a discovery mechanism, not a primary data source.

### Social media (Instagram, Facebook)

Public Instagram accounts and Facebook pages for event venues are rich in photos and often include capacity and booking information in their bio or posts. Useful for photo collection. Content is publicly viewable; for seed data purposes, saving publicly posted photos of a venue you are creating a profile for is reasonable fair use for internal testing.

**Note:** Do not scrape at scale. Manual collection per venue is appropriate for 100 seed venues.

### Yelp and TripAdvisor

Public venue pages include photos, reviews, and sometimes structured attributes (private room available, capacity, event hosting). Useful for photo collection and cross-referencing venue details. Do not use their data commercially or at scale — for 100 seed venues, manual reference is appropriate.

### What is legally off-limits

- Scraping data from platforms that explicitly prohibit it in their ToS at scale (Google Maps HTML scraping, Yelp bulk scraping)
- Downloading copyrighted floor plan PDFs and republishing them publicly — keep seed data internal to the platform
- Using venue photos in any external-facing marketing material without permission — seed photos are for internal ETL testing and demo only

---

## Collection process

### Target

100 venues total across three cities before MVP launch:

- Austin: 35 venues
- Nashville: 35 venues
- Miami: 30 venues

### Process per venue

1. Find the venue (Google Maps search, local event guide websites, wedding/corporate event directories)
2. Confirm it meets the selection criteria
3. Download all publicly available documents from the venue's website (PDF decks, floor plans, menus)
4. Save photo URLs or download photos from the venue's website and Google Maps
5. Create a venue card in VenueMi with name, address, and source URL
6. Upload all documents — this is a live ETL pipeline test run
7. Review extraction output, note failures, log confidence scores per field
8. Fix extraction errors manually — this populates the override history and tests the correction UX

This process is the 50-document accuracy benchmark and the seed library build at the same time. Every venue ingested is both a test case and a library asset.

### Tracking

Maintain a simple spreadsheet during collection:

| City | Venue name | Category | Documents collected | Extraction quality (1–5) | Notes |
| ---- | ---------- | -------- | ------------------- | ------------------------ | ----- |

Score extraction quality 1–5 per venue after reviewing the output. Track which field types fail most often (capacity tables, catering policy, curfew, contacts). This feeds directly into ETL pipeline improvements before launch.

---

## Exit criteria

The seed library is good enough to demo and launch when:

- [ ] 100 venues ingested across three cities
- [ ] At least 60% of venues have a confidence score of 4 or above on core fields (capacity, catering policy, venue type, contacts)
- [ ] At least five natural-language search queries return accurate, sourced results from the library
- [ ] The demo flow (upload PDF → see profile → run search) works end-to-end with a real, messy venue document
- [ ] The accuracy benchmark is documented: field-level pass/fail rates across 50+ documents

When these criteria are met, the product is ready for the first customer conversation. The seed library stays in the platform as free-tier content for early users.

---

**Docs:** [What is VenueMi?](../../README.md) · [Business Proposal](proposal.md) · [Market Structure](../market.md) · [Vision](../../roadmap/vision.md)
