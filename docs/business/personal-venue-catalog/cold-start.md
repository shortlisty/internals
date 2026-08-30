# Cold start strategy — Venue Master Catalog seed

> [!NOTE]
> This document was written for the **Personal Venue Catalog** segment — an internal knowledge-base tool for event agencies. It is a reference example for that positioning, not the primary product direction. The current product focus is the [Digital Sales Room for Events](../../digital-sales-room-for-events/README.md).

> **Audience:** Founders, team.
> **Purpose:** How to seed the **internal platform Venue Master Catalog (MC)** before the first paying customer — collecting real venue data during development to power three hidden backdrop capabilities for all future tenants, test the ETL/importer/dedup pipeline on real documents, and build demo content that is credible.

---

## The problem

An event manager who signs up and sees a blank Create-Venue form does not get the aha moment — they get the manual-entry problem. The product clicks only when 20+ fields are auto-populated the moment they type a venue name they already know.

This creates a bootstrapping challenge: the **Master Catalog backdrop layer** needs real reference data to silently auto-populate tenant venues, but tenants have not yet uploaded any documents to draw from.

The solution is to collect and curate a high-quality Master Catalog seed dataset ourselves during development. This is not a workaround — it is a platform foundation requirement. Collecting and ingesting 50–200 real venues before launch achieves four things simultaneously:

1. **Master Catalog importer & dedup pipeline testing** — real provider records (from web scrapers such as Tagvenue, or manual admin CSV imports) are the only honest test of the fuzzy name+geo matching algorithm, the `master_venue_external` UPSERT logic, and the field-level provenance conflict resolution. Clean, invented test fixtures do not reveal the merge anomalies or ambiguity cases that matter.
2. **ETL & extraction pipeline testing** — accompanying venue documents (PDF decks, floor plans, spec sheets downloaded from the venues' own websites) are the only honest benchmark of AI extraction accuracy on real-world files.
3. **Demo & concierge credibility** — a live onboarding session with real, recognisable venues in a specific city (20–40 venues each in Austin, Nashville, Miami, Naples) that arrive in the planner's form 90% pre-filled is orders of magnitude more convincing than placeholder data or a blank empty state.
4. **Silent backdrop value for every future tenant** — every new agency (free or paid) that creates a venue whose name+city+geo matches the seed Master Catalog immediately receives the gap-fill and form auto-populate behaviour, without any team configuration. The platform "just knows" common venues the moment a planner types their name.

This is also the 50-document extraction accuracy benchmark called out in [vision.md](../../roadmap/vision.md) under strategic bets. Collecting the seed dataset and running the accuracy benchmark are the same activity.

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

Three US cities + one EU launch candidate, 20–40 venues each, chosen for active event markets, concentration of SMB agencies, and good public venue data availability. All four cities feed directly into the shared platform **Venue Master Catalog (`public.master_venue`)** — every future tenant (US or EU-based small agency) that works with venues in these cities gets silent auto-populate immediately on signup, regardless of tier.

**Austin, TX**
Strong corporate event market (SXSW ecosystem, tech company offsites), growing agency scene, active social events (weddings, galas). Venue culture leans toward unique spaces — converted warehouses, rooftop terraces, hotel ballrooms. Strong public web presence for local venues.

**Nashville, TN**
One of the fastest-growing US event markets. Corporate events, conferences, bachelorette/bachelor industry, music-adjacent unique venues. High density of boutique hotel event spaces and non-traditional venues. Venues are web-savvy and typically publish spec sheets and floor plans.

**Miami, FL**
Active corporate event market, international flavour, strong hospitality sector. Art Deco hotel ballrooms, rooftop venues, waterfront spaces. High concentration of independent event spaces with public-facing marketing material.

**Naples, IT (EU launch candidate, primary for DSR concierge onboarding pilot)**
Smaller, tighter event market with strong luxury/wedding/conference agencies and high venue density per square km. Ideal cold-start test for the DSR primary product direction because the local planner community is tight-knit — 25–30 curated local venues in the Master Catalog will cover ~80% of brief requirements for a typical boutique agency. Good public web presence for boutique hotels, villas, and historic palazzo event spaces.

**Reserve cities** (expand to if the primary four go well): Denver, Charlotte, Portland, San Antonio, Milan, Berlin, Marbella.

### What to avoid

- Tier-1 cities (NYC, LA, Chicago, Las Vegas) — save for the demo pitch deck, not the Master Catalog seed dataset
- Stadium, arena, and convention centre venues — too large, operator-side tools already cover them, not Shortlisty's ICP
- Chain hotel ballrooms (Marriott, Hilton, Hyatt branded) — their venue data is centralised and controlled, not publicly accessible per-property, and fuzzy dedup against chain names produces too many false-positives
- Venues with no web presence or no downloadable spec materials — they are not useful for ETL testing, and the Master Catalog seed rows benefit most from having source documents to benchmark extraction against

---

## Venue types to target

### Selection criteria

A good seed venue is:

- SMB-operated: independently owned or a small regional group (not a national chain)
- Publicly marketed: has a website with event/venue information, ideally a downloadable spec sheet or floor plan
- Actively hosting events: listed on Google Maps as an event venue, has reviews, has photos
- Capacity range: 30–300 guests — the core Shortlisty ICP brief range
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

120–150 Master Catalog entries total across four cities before MVP launch:

- Austin, TX: 35 venues
- Nashville, TN: 35 venues
- Miami, FL: 30 venues
- Naples, IT: 25–30 venues

### Process per venue

1. Find the venue (Google Maps search, local event guide websites, wedding/corporate event directories)
2. Confirm it meets the selection criteria above
3. Download all publicly available documents from the venue's website (PDF decks, floor plans, menus)
4. Save photo URLs or download photos from the venue's website and Google Maps
5. Record the raw structured data as a **`MasterVenueRecord` JSON** entry — same field shape that `mc-ingest-tagvenue-scraper` emits, so these manual entries are interchangeable with scraper entries and feed directly into the same `mi-mc-loader` pipeline later.
6. Submit the JSON through either the **admin MasterVenue CRUD API** or the `mi-mc-loader --file` CLI so it is inserted into `public.master_venue` with an associated `master_venue_external` row (`external_source='platform_seed'`). This is a live test of the importer UPSERT dedup pipeline.
7. Run the same venue documents through the asset-ETL pipeline for a linked test tenant venue to test extraction quality end-to-end.
8. Review extraction output, note failures, log confidence scores per field. If a seed entry has any fields that extraction missed but we verified manually on the venue website, update the `master_venue.metadata` row via admin edit as `MANUAL_OVERRIDE` (priority 10/10) — this tests the provenance priority chain because future scraper re-runs will not overwrite hand-verified seed data.

This process is simultaneously: (a) the 50-document extraction accuracy benchmark from the vision strategic bets, (b) the `mi-mc-loader` fuzzy dedup & merge integration test, (c) the concierge onboarding demo dataset for every future concierge pilot.

### Tracking

Maintain a simple spreadsheet during collection, plus the resulting `master_venue_import_log` audit rows from `public.master_venue_import_log`:

| City | Venue name | Category | Documents collected | Seed JSON written? | Inserted via importer? | Extraction quality (1–5) | Impoter dedup: new row or merged? | Notes |
| ---- | ---------- | -------- | ------------------- | ------------------ | ---------------------- | ------------------------ | --------------------------------- | ----- |

Score extraction quality 1–5 per venue after reviewing the output. Track which field types fail most often (capacity tables, catering policy, curfew, contacts). Track importer dedup outcomes separately: did this venue create a new master row, or did it correctly merge with an existing Tagvenue scraper record for the same physical venue? This is the only honest way to validate the 0.75 / 0.90 fuzzy-match thresholds are calibrated correctly.

---

## Exit criteria

The **Venue Master Catalog seed dataset** is good enough to enable backdrop functionality for every future tenant when:

- [ ] 120–150 `public.master_venue` rows inserted across four seed cities via the importer pipeline
- [ ] At least 80% of seed rows have an associated `master_venue_external` record with `external_source='platform_seed'`; any remaining 20% come from a parallel Tagvenue scraper run for the same cities to validate dedup merges.
- [ ] Cross-source dedup test passes: for 20 known venues in Austin/Naples, a manually created CSV seed entry + a Tagvenue scraper entry for the same physical venue produce **one merged `master_venue` row** with two `master_venue_external` children (no duplicate master rows). This proves the fuzzy 0.75/0.90 thresholds are production-ready.
- [ ] At least 60% of seed venues have `MANUAL_OVERRIDE` or `VERIFIED` confidence of 4 or above on the four core fields (capacity, catering policy, venue type, contacts) via admin edits.
- [ ] Silent form auto-populate (backdrop pattern 2) verified end-to-end: for 10 seed venues in a test tenant, typing the venue name into CreateVenue returns a correctly pre-filled form at 90%+ field coverage, with provenance `MC_INHERIT` stored on each filled field.
- [ ] Silent gap-fill (backdrop pattern 1) verified end-to-end: for 10 test tenant venues where extraction leaves 5+ fields null, the gap-fill stage copies the correct values from the linked matched master row.
- [ ] The 50-document extraction accuracy benchmark is documented: field-level pass/fail rates across 50+ real venue PDFs, with low-confidence fields clearly flagged for the ETL roadmap.

When these criteria are met, the platform is ready for the first concierge onboarding pilot and first customer conversation. The Master Catalog stays in production as the hidden backdrop layer — tenants never interact with it directly, but every tenant benefits from its 20–30 field auto-populate on every CreateVenue and every asset upload.

---

**Docs:** [What is Shortlisty?](../../README.md) · [Business Proposal](proposal.md) · [Market Structure](../market.md) · [Vision](../../roadmap/vision.md)
