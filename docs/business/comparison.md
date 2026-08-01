# Venue Intelligence Platform — Competitive Landscape

> Strategic reference for the competitive positioning of BENE.

**Docs:** [What is BENE?](../README.md) · [Business Proposal](proposal.md) · [Competitive Landscape](comparison.md) · [Intelligence Layer](../platform/intelligence.md) · [Architecture](../platform/architecture.md)

---

## 1. Competitive Landscape

### 1.1 Venue & Event Management Platforms

These are the established players. None of them are _document intelligence_ platforms — they are operational booking/CRM systems that have started bolting AI on top of structured data they already own.

---

#### Cvent

**What it is:** The largest enterprise event management platform. 340K+ venues in its supplier network.

**Strengths:**

- Massive venue database (supply-side moat)
- RFP automation (send a brief, receive structured bids)
- AI-powered venue sourcing and recommendation
- Full event lifecycle: sourcing → registration → onsite → reporting
- Global footprint, strong enterprise contracts

**Gaps relevant to BENE Intelligence:**

- Cvent is a _discovery and booking_ platform — it doesn't help you manage your own venue library
- Venues in Cvent are self-submitted by venue owners, not extracted from your own documents
- No document intelligence (no PDF/floor plan/CAD parsing)
- No team-owned venue knowledge base
- Enterprise pricing puts it out of reach for SMB agencies

**Verdict:** Not a direct competitor. Cvent is a venue marketplace. BENE Intelligence is an intelligence layer for your own venue portfolio. They could be _complementary_ (import discovered venues from Cvent into BENE).

---

#### Tripleseat

**What it is:** Sales and catering software for restaurants, hotels, and unique venues. 20,000+ venue clients.

**Strengths:**

- Strong operational workflows (booking, contracts, invoicing)
- Just launched "Tripleseat Intelligence" — AI suite built on their dataset of millions of events
- AI for: demand forecasting, F&B inventory recommendations, conversational analytics, peer benchmarking
- Deeply embedded in hospitality operations

**Gaps relevant to BENE Intelligence:**

- Tripleseat is built _for venues_ to manage their events — not _for planners_ to manage their venue portfolio
- Their AI runs on their own transactional data (bookings), not on unstructured documents
- No document parsing, no cross-venue search for planners
- No support for planner's own uploaded assets

**Verdict:** Different side of the market. Tripleseat serves venues; BENE serves planners. The intelligence architectures are fundamentally different: Tripleseat mines structured operational data; BENE mines unstructured documents.

---

#### Momentus Technologies (formerly Ungerboeck / VenueOps)

**What it is:** Enterprise-grade venue and event management. Serves convention centers, performing arts, stadiums, universities. 700+ performing arts centers, 50+ countries.

**Strengths:**

- End-to-end: booking → operations → finance → analytics
- AI-powered platform enhancements (Feb 2026): operational insights, space optimization
- 20+ years of venue and event intelligence baked into their models
- WeTrack product for safety/sustainability/risk management

**Gaps relevant to BENE Intelligence:**

- Heavy enterprise product, not accessible to SMB agencies
- Focused on venue operators managing their own space, not planners curating a portfolio
- No document intelligence or ETL pipeline
- Implementation takes months, not minutes

**Verdict:** Enterprise venue ops software. No overlap with BENE's document intelligence core.

---

#### Facilitron

**What it is:** Fully managed facility scheduling, rental, and operations platform targeting schools, school districts, municipalities, and public organisations. Self-described as "the world's largest public spaces rental marketplace." Funded entirely through service fees on external rental revenue — no licensing cost to facility owners.

**Strengths:**

- Unified scheduling: one calendar for internal events and external community rental requests
- Public-facing rental storefronts with facility photos (including drone and 360-degree), pricing, real-time availability, and e-commerce-style checkout
- Managed payments and insurance: handles insurance confirmation, payment processing, and refunds on behalf of the facility owner
- Work order management: schedule utilities, security, custodial, maintenance, and asset tracking in the same platform
- Facilitron FIT: mobile facility inspection tool for California Williams Act compliance, from inspection to SAB form submission
- Strong analytics and KPI dashboards: bookings, revenue, utilisation, cost recovery
- No direct cost to facility owners — funded by a service fee on external rental amounts
- 24/7 customer support for community requesters by phone, email, and live chat
- Fully managed onboarding: account configuration, photo ingestion, schedule migration, and training at no cost
- Cloud-based, mobile-optimised

**Gaps relevant to BENE Intelligence:**

- Serves _facility operators_ (schools, districts, cities) — not event managers or event planning agencies
- No document intelligence: no PDF ingestion, no floor plan parsing, no spec sheet extraction
- No planner-side venue library or portfolio management — the platform manages one organisation's own facilities, not a curated collection of third-party venues
- Venue data is entered and managed by the facility owner, not extracted from documents
- Narrow vertical: public sector institutions (K–12 schools, colleges, municipalities) — not corporate or social event agencies
- No semantic search across venue attributes extracted from documents
- No multi-source aggregation or conflict resolution
- No team knowledge base for event professionals

**Verdict:** Facilitron is a public-sector facility operations and community rental platform. It serves the venue operator side — helping schools and cities manage their own spaces and earn rental revenue from the community. It has no overlap with BENE's core use case. A school district using Facilitron to manage gym rentals is not an BENE customer; an event agency trying to find and brief that school's gym is.

---

#### Propared

**What it is:** Cloud-based production planning and scheduling software built for arts and events organisations — theatres, opera and dance companies, festivals, performing arts centres, and corporate events producers. Covers scheduling, crew management, inventory tracking, and budgeting in one platform. Explicitly no AI, no data repurposing.

**Strengths:**

- Unified production timeline: rehearsals, performances, load-ins, crew calls, deadlines, run-of-show — all in one filterable schedule
- Auto-updating, shareable webpages for cast, crew, and clients — no login required for viewers
- Labour management: crew search by position and availability, pay rates, overtime rules, shift scheduling, personal itineraries sent directly to crew
- Inventory and resource tracking: multiple warehouses, QR code scanning, mobile app, allocation across concurrent projects, over-allocation alerts
- Live budgeting: estimates by vendor, project, and department; pull lists and rental quote requests generated from the same data
- Production reports and task management: notes by department, to-dos with status tracking, shareable across projects
- Project templates and cloning: build a season calendar or recurring event from a past project in hours, not weeks
- Pricing model based on editing users — viewers are always free, enabling wide sharing without per-seat costs
- $45/month per editor, 30-day free trial, no setup fees, free onboarding
- Explicitly privacy-first: no AI, no third-party data repurposing

**Gaps relevant to BENE Intelligence:**

- Production management tool for event _producers and producers' teams_ — not for event planning agencies managing a venue portfolio
- No venue knowledge base: no venue profiles, no document ingestion, no extraction from venue-supplied PDFs or floor plans
- No document intelligence of any kind — attachments are links to external files, not parsed and structured
- Inventory is the producer's own equipment, not venue assets; "spaces" are scheduling entries, not structured venue data records
- Search is limited to filtering scheduled events and inventory — not semantic search across extracted venue attributes
- No multi-source data aggregation or conflict resolution
- Narrow vertical: performing arts, live events, corporate production teams — not general event planning agencies managing third-party venues

**Verdict:** Propared is production operations software for the teams _producing_ an event — stage managers, production managers, technical directors. BENE Intelligence is knowledge management software for event managers _finding and briefing_ venues. The workflows are adjacent but the users and the data are entirely different. A production manager at a theatre company uses Propared to schedule the load-in; the event agency that booked that theatre used BENE to find it.

---

#### Perfect Venue / Event Temple

**What it is:** Lightweight venue management tools targeting independent venues, small hotels, wineries.

**Strengths:** Affordable, easy to set up, covers booking basics.

**Gaps:** No AI, no document intelligence, no team venue library concept. More CRM than intelligence platform.

**Verdict:** Irrelevant to BENE's positioning. Different price/feature tier entirely. Planning Pod (see above) is the more capable tool in this tier and has its own entry.

---

#### VenueScanner

**What it is:** UK-based venue marketplace and concierge service. 19,000+ venues across the UK and internationally. Described by Forbes as "the Airbnb of venue hire." 1M+ event organisers use the platform annually.

**Strengths:**

- Large self-serve search with filters for location, capacity, price, and amenities
- Free for event organisers — revenue model is commission from venues
- VenueScanner for Business: concierge team handles corporate briefs, negotiates rates, shortlists within 24–48 hours
- AI-ranking algorithm boosts venues that respond quickly to enquiries
- Expanding beyond UK into international markets

**Gaps relevant to BENE Intelligence:**

- Pure _discovery and booking_ marketplace — venues are listed by venue owners, not extracted from planner-owned documents
- No planner-side venue library or knowledge base
- No document intelligence — planners can't upload their own PDFs, floor plans, or spec sheets
- AI is limited to ranking and response-time scoring, not semantic extraction
- No cross-venue comparison against a planner's own portfolio

**Verdict:** VenueScanner is a consumer-grade venue search engine, not a planner intelligence tool. A planner who already knows their preferred venues gets nothing from VenueScanner — it only helps with first-pass discovery of venues they haven't worked with yet. Complementary to BENE, not competitive.

---

#### VenueFindAI

**What it is:** AI-powered venue discovery and concierge service (venuefindai.com). Positions itself as an intelligent assistant that delivers personalised venue recommendations for corporate events and special celebrations, backed by human experts available on demand. Free to use for organisers — no fees, no commission.

**Model:** AI-first matchmaking with a human-in-the-loop concierge layer. The AI surfaces recommendations; human specialists are available "at the touch of a button" for complex or high-value briefs. Revenue model appears to be venue-side (similar to VenueScanner, Hire Space — venues pay for leads/placement rather than planners paying for the service).

**Strengths:**

- Zero cost to planners — lowers the barrier to trial significantly
- AI + human hybrid model hedges against the accuracy limits of pure AI sourcing
- Broad scope: corporate events and celebrations, suggesting a wide venue inventory or intent to build one
- Straightforward positioning that's easy for non-technical buyers to understand

**Gaps relevant to BENE Intelligence:**

- Discovery platform: works from a database of venues that _have listed themselves_ — not from documents a planner already owns
- No planner-side knowledge base — recommendations are ephemeral, not stored as a team asset
- No document intelligence, no PDF/floor plan ingestion, no structured extraction
- Human concierge layer adds latency and doesn't scale to a planner's full portfolio of 50–100 known venues
- AI recommendations are only as good as what venues have self-submitted — the moment a planner needs intelligence from their own files (a venue deck sent by email, a floor plan from 2019), VenueFindAI has nothing

**Verdict:** Same quadrant as VenueScanner and Cvent — a venue marketplace/sourcing tool for _discovering_ new venues. BENE solves the adjacent and complementary problem: once you've found and worked with venues, how do you manage, extract intelligence from, and search across everything you already know about them. A planner could use VenueFindAI to discover a venue, then use BENE to ingest that venue's documents and build a permanent, searchable profile.

---

#### VenueArc

**What it is:** SaaS venue booking and event management platform built specifically for performing arts centers, theatres, and live event venues. Covers booking calendars, contracts, settlements, CRM, and financial reporting.

**Strengths:**

- Specialised for performing arts / live event venue operators
- Booking calendar with conflict prevention, event holds, dynamic scheduling
- Contract generation, settlement portal, financials, and reporting
- Ticketmaster / Nexus integration for live event data
- CRM with client data visibility and lead capture
- Cloud-based, mid-market pricing

**Gaps relevant to BENE Intelligence:**

- Serves _venue operators_, not event planners — the same side-of-market distinction as Momentus and Tripleseat
- No document intelligence, no PDF/floor plan ingestion, no ETL pipeline
- No planner-facing portfolio management
- Narrow vertical: performing arts centres and theatres, not the broader corporate/social events market

**Verdict:** Niche venue operations software for a specific vertical (performing arts). No overlap with BENE's core use case. Closer to a lighter-weight Momentus than anything in BENE's competitive set.

---

#### Planning Pod

**What it is:** Venue management software built specifically for venue operators — event and wedding venues, restaurants with private dining programs, country clubs, wineries, breweries, museums, and hotels. 40+ modular tools covering booking calendars, proposals, contracts, BEOs, floor plans, billing, integrated payments (credit card + ACH), and CRM. Self-described as venue-first: event planners without their own venue can use it, but the product, support, and roadmap are explicitly built for venue operators. Pricing starts around $59/month; scales with venue size and tool set.

**Strengths:**

- Full venue sales cycle in one record: lead capture → proposal → e-signature contract → BEO → invoice → payment — no re-typing between steps
- Booking calendar with soft/hard holds, blackout dates, multi-space scheduling, and tour scheduling — all shared across the team
- BEO builder with saved menu/equipment libraries and package assembly; shared in real time with kitchen and clients
- Integrated payments (Stripe; credit card + ACH) tied directly to the booking record — auto-payment schedules, automated invoice sending, reconciliation without manual export
- Built-in floor plan designer with drag-and-drop to-scale layouts, connected to the same booking record rather than a separate app
- Lead capture webforms integrate directly with pipeline; WeddingWire, The Knot, QuickBooks, Google/Apple/Microsoft calendar sync out of the box
- Ticketed event support: online ticket sales, RSVPs, registration, event websites, on-site check-in — synced to the booking record
- Modular: tools turn on by venue type and size; venues don't pay for features they don't need
- Dedicated onboarding specialist and ongoing product specialist support — not self-serve helpdesk
- Claims venues grow bookings by an average of 64% after switching; users save 62 hours/month

**Gaps relevant to BENE Intelligence:**

- Serves _venue operators_, not event planners — the same side-of-market distinction as Tripleseat, Momentus, and VenueArc
- No document intelligence: no PDF ingestion, no spec sheet extraction, no floor plan parsing from uploaded files
- No planner-facing venue portfolio management — the platform manages one venue's own operations, not a planner's curated portfolio of third-party venues
- Venue data is entered and managed by venue staff, not extracted from documents
- No semantic search across extracted venue attributes; search is limited to booking records and CRM data the venue has manually entered
- No multi-source aggregation or conflict resolution across documents

**Verdict:** Planning Pod is Tripleseat's most direct SMB/mid-market competitor — both serve venue operators, not planners. Planning Pod wins on pricing accessibility and modularity where Tripleseat wins on scale and depth of transactional data. Neither has anything to do with BENE's use case. The floor plan designer is the only surface-level overlap: Planning Pod lets a venue build and reuse floor plan layouts for BEOs; BENE parses floor plan documents uploaded by planners and extracts structured intelligence from them. Entirely different users, entirely different data flows.

---

### 1.2 Digital Asset Management (DAM) Platforms

DAM platforms are the closest adjacent category to BENE Intelligence — they centralize, tag, search, and distribute digital files for marketing and brand teams. Unlike the booking/CRM tools above, DAMs do handle unstructured files (images, PDFs, videos). But their intelligence is built for brand governance, not venue intelligence.

---

#### Bynder

**What it is:** One of the largest enterprise DAM platforms. Primarily serves marketing, brand, and creative teams at large organizations. Used by global brands to manage logos, campaign images, documents, and video libraries.

**Strengths:**

- Centralized asset library with rich metadata, taxonomy, and version control
- AI Agents (2025+): automated metadata enrichment, title/description/tag generation on upload
- Brand guidelines, style guides, and approval workflows built in
- Dynamic Asset Transformation: on-the-fly image resizing, format conversion, CDN delivery
- 250+ integrations (Adobe Creative Cloud, Canva, Salesforce, CMS platforms)
- Multi-portal support: role-based brand portals for different teams or clients
- Strong enterprise compliance: SOC 2, SSO, granular permissions

**Pricing:** Quote-only. Entry-level around $450/month; enterprise contracts average ~$41K/year based on procurement data.

**Gaps relevant to BENE Intelligence:**

- Built for _brand assets_ (logos, campaign images, marketing docs) — not venue intelligence
- No venue-specific extraction schema (no capacity, amenities, room configurations, restrictions)
- AI enrichment is generic (auto-tags any image/document) — no domain understanding of floor plans or spec sheets
- No document parsing pipeline designed for multi-source, multi-format venue documents
- No semantic search against _extracted_ venue attributes — only metadata search
- No cross-document aggregation or conflict resolution
- Priced for enterprise marketing teams, not event planning agencies

**Verdict:** Bynder is a sophisticated content library, not an intelligence platform. If a planner stored venue PDFs in Bynder, they'd get organized storage and generic AI tags. They would _not_ get extracted capacity data, structured room configurations, or answers to "find me a venue with a freight elevator and kosher catering." The intelligence layer BENE provides simply doesn't exist in any DAM.

---

#### Brandfolder (by Smartsheet)

**What it is:** DAM platform acquired by Smartsheet in 2020 for $155M. Now positioned as the content management arm of Smartsheet's work management suite. Serves mid-market to enterprise teams managing brand and creative assets.

**Strengths:**

- Brand Intelligence engine: proprietary AI/ML for auto-tagging, object detection, and smart content classification
- Natural language search (in rollout): find assets using conversational queries instead of exact keywords
- Smart CDN: publish and distribute assets directly to web/social channels
- Customizable brand portals for client-facing asset sharing
- Tight Smartsheet integration: approve assets in Smartsheet, publish to Brandfolder in one click
- 30+ out-of-the-box integrations
- Clean, modern UI — lower learning curve than Bynder for mid-market teams
- Powers Getty Images' Media Manager (announced Feb 2026) — handles high-volume professional asset ingestion at scale

**Pricing:** Custom, quote-based. Two tiers (Premium and Enterprise). Starting point reported around $1,600/month; median contracts around $24,700/year.

**Gaps relevant to BENE Intelligence:**

- Same fundamental limitation as Bynder: designed for _brand/marketing assets_, not venue intelligence
- Brand Intelligence tags visual content generically — no understanding of venue-specific semantics
- No structured extraction of capacity configurations, catering policies, AV specs, or floor plan geometry
- No multi-source aggregation: uploading three versions of the same venue deck doesn't reconcile conflicts
- No event planner workflow: no RFP support, no venue comparison, no sourcing history
- Smartsheet integration helps with _work management_ (tasks, approvals) — not with venue knowledge

**Verdict:** Brandfolder is Bynder's strongest mid-market competitor, with a cleaner UX and a stronger AI tagging story. Neither is a venue intelligence platform. A planner using Brandfolder gets a well-organized file library with good visual search — and nothing more. The gap BENE fills (venue-specific structured extraction, semantic search by venue attributes, multi-source reconciliation) is entirely absent from the DAM category.

---

### 1.3 AI Productivity Tools for Event Professionals

A distinct emerging category: general-purpose AI assistants built specifically for event planners. These are not venue databases or booking systems — they are workflow automation tools that wrap LLMs in event-industry-specific task templates.

---

#### Spark (sparkit.ai) — by GEVME & PCMA

**What it is:** AI productivity platform for event professionals, built by Singapore-based event tech company GEVME in partnership with PCMA (Professional Convention Management Association), the leading global business events industry association. Launched mid-2023 as "Project Spark"; now in version 2.0 with 14,500+ professionals across 150+ countries.

**Strengths:**

- 150+ pre-built AI tasks covering the full event lifecycle: agenda building, speaker bios, email copy, event descriptions, session titles, feedback analysis, surveys, post-event reports
- Spark 2.0 Agent Studio: no-code agent builder that chains multiple actions (web search → LinkedIn data pull → content generation → output) into automated workflows
- DestinAItor: AI module for venue/destination research, RFP response generation, and destination prospecting — targeted at DMOs and hotel sales teams
- Multi-model: runs across the best available LLMs rather than locking to one provider
- 30 languages supported
- PCMA co-branding gives it significant industry credibility and distribution within the business events community
- Free tier available; enterprise tier with SSO, custom integrations, and security controls
- Spark Academy, Spark Excellence (team training), SparkU (student tier): education ecosystem building adoption

**Gaps relevant to BENE Intelligence:**

- Spark is a _content generation and workflow automation_ tool — it helps planners write faster, not know their venues better
- No venue knowledge base: no venue profiles, no stored documents, no portfolio management
- DestinAItor does venue research from web/public data — it does not ingest or parse a planner's own venue documents
- No document intelligence: no PDF parsing, no floor plan extraction, no CAD support
- No structured venue metadata schema — outputs are generated text, not queryable structured data
- No multi-source aggregation or conflict resolution across documents

**Verdict:** Spark is the closest thing to a purpose-built AI assistant for event planners, and it's well-adopted (14,500+ users). But it's a _writing and workflow_ tool, not a _knowledge_ tool. A planner using Spark can draft an RFP faster — but still has no structured, searchable record of what their 50 preferred venues actually offer. BENE and Spark are complementary: Spark generates the content; BENE supplies the venue intelligence that makes that content accurate and specific.

---

### 1.4 Document Intelligence & ETL Platforms

These are the infrastructure players. They are the technical substrate that BENE's pipeline either competes with or can leverage.

---

#### Unstructured.io

**What it is:** The leading enterprise ETL platform for unstructured data. Purpose-built to prepare documents for LLMs and RAG pipelines.

**What it does:**

- Ingests: PDFs, Word, PowerPoint, HTML, emails, images, scanned docs
- Outputs: clean, chunked, LLM-ready JSON
- Connectors: S3, SharePoint, Google Drive, Confluence, 25+ sources
- Destinations: vector databases, data warehouses
- Handles: OCR, table extraction, layout detection, multi-column PDFs

**Pricing:** Free tier (15K pages, no expiry). Pay-as-you-go (~$2.66/compute hour). Enterprise subscription.

**Relevance to BENE Intelligence:**

- Unstructured.io is what BENE's ETL layer _could use as a backend_ rather than building from scratch
- Their open-source library (`unstructured`) can be self-hosted
- Handles the hardest parsing problems (scanned PDFs, multi-column layouts, tables)
- Not a product for end users — pure infrastructure/API

**Strategic insight:** BENE doesn't need to reinvent document parsing. Unstructured.io (or Docling) handles the extraction layer. BENE's value add is the _venue-specific intelligence_ on top — the domain schema, the aggregation model, the search experience, the team collaboration.

---

#### IBM Docling

**What it is:** Open-source document conversion toolkit from IBM Research Zurich, now under Linux Foundation AI & Data.

**What it does:**

- State-of-the-art PDF layout analysis (DocLayNet model)
- Table structure recognition (TableFormer model)
- OCR for scanned documents
- Exports: clean Markdown, structured JSON with full metadata
- Handles: PDFs, DOCX, PPTX, HTML, images
- Runs locally, no cloud dependency, MIT license

**Why it matters for BENE Intelligence:**

- Free, open-source, no per-page pricing
- Superior table and layout understanding vs. naive PDF parsing
- Direct integration with Spring AI via `TikaDocumentReader` (Apache Tika underneath) or custom `DocumentReader`
- IBM Granite-Docling-258M: new ultra-compact VLM for document-to-structured-format conversion
- Ideal for floor plan PDFs, spec sheets with tables, multi-column venue decks

**Strategic decision for BENE Intelligence:** Use Docling as the primary document parsing layer. It handles the structural extraction (layout, tables, text) and Spring AI's ETL pipeline then handles chunking, embedding, and vector storage.

---

#### Apache Tika

**What it is:** The Java ecosystem's gold standard for file format detection and text extraction. Detects and parses 1000+ file types through a single interface.

**What it does:**

- Text extraction from: PDF, DOC/DOCX, XLS/XLSX, PPT, images (via OCR), HTML, XML, ZIP, and 1000+ more
- Metadata extraction (author, creation date, dimensions)
- Language detection
- MIME type detection
- Tika Pipes: fault-tolerant, scalable processing (each file in a forked JVM with timeout/memory limits)

**Integration with Spring AI:**

- Spring AI ships `TikaDocumentReader` out of the box
- `TikaDocumentReader` wraps Tika behind Spring AI's `DocumentReader` interface
- Handles: PDFs, Word, Excel, PowerPoint transparently — same API regardless of file type
- Used in production for: search engine indexing, content analysis, translation pipelines

**Why it's the right choice for BENE Intelligence:**

- Battle-tested in enterprise Java for 15+ years
- DWG/DXF support via Tika's AutoCAD parser (direct path for CAD files)
- Zero extra infrastructure — runs in-process
- Tika Pipes for production safety (malformed files can't crash the service)
- Already in Spring AI's ETL pipeline — no custom integration needed

---

#### LlamaIndex / LangChain

**What they are:** Python-first AI orchestration frameworks. LlamaIndex has strong document parsing (LlamaExtract, LlamaParse). LangChain has document loaders.

**Relevance to BENE Intelligence:** These are Python-ecosystem tools. Since BENE Intelligence is Java/Spring Boot, they are not directly applicable. Spring AI is the Java equivalent and has caught up rapidly.

**Note:** If BENE ever needs a Python microservice for specialized extraction (e.g., advanced floor plan analysis), LlamaIndex's LlamaParse is best-in-class for complex PDFs.

---

#### Reducto, Raydocs, Retab (Document Intelligence Startups)

Recent well-funded entrants in the document intelligence space:

- **Reducto** — $75M Series B (a16z, Benchmark, YC). API-first structured extraction from complex documents.
- **Raydocs** — Template-based extraction with confidence scores and source links.
- **Retab** — Pre-seed $3.5M. Non-technical users building extraction templates.

**Pattern:** All these companies are _horizontal_ document intelligence APIs. BENE's opportunity is to be _vertical_ — deeply specialized for venue documents (floor plans, venue decks, CAD files, spec sheets). Horizontal tools extract generic fields. BENE extracts venue-specific intelligence with a purpose-built schema.

---

### 1.5 Floor Plan Intelligence & Spatial Data Tools

A distinct cluster of tools that deal with floor plan geometry, spatial extraction, or indoor location infrastructure. None of them are venue management or event planning platforms, but they are directly adjacent to BENE's document intelligence layer — specifically the CAD/floor plan ingestion pipeline. Understanding where they sit clarifies what BENE needs to build versus what it can leverage or ignore.

---

#### SeatPlan.io

**What it is:** Free-to-start, browser-based seating chart maker targeting wedding planners, corporate event teams, and hospitality venues. Guest list import via CSV, drag-and-drop table layout, real-time collaboration, and QR code guest self-check-in. Event Manager plan at £25/month. AI floor plan import (upload a photo or PDF, AI generates a layout draft) with CAD/DXF support at the paid tier.

**Strengths:**

- Zero friction entry: no signup required to start designing; full designer accessible before payment
- AI-assisted floor plan import: upload a venue PDF or photo and the tool generates a seating layout to review — practical head start for planners
- CAD/DXF import at Event Manager tier — exact-scale layouts from venue-supplied drawings
- Real-time collaboration: client-facing shared links with planner approval control, no per-viewer cost
- Guest check-in distribution: QR signs, Apple/Google Wallet passes, door-team check-in link — all generated from the same layout
- Reusable venue templates: build a room once, clone for every event
- Covers the full seating workflow end-to-end: import → arrange → export (A3 PDF, Excel seat list, name cards)

**Gaps relevant to BENE Intelligence:**

- Seating chart tool, not a venue intelligence platform — manages guest-to-seat assignment, not venue knowledge
- No venue knowledge base: no venue profiles, no document ingestion beyond floor plan layout import, no spec sheet extraction
- AI floor plan import produces a _layout for seating_, not structured venue metadata (capacity, amenities, restrictions)
- One venue per event — no cross-venue portfolio management or semantic search
- No multi-source aggregation: uploading three versions of a venue's deck doesn't reconcile data
- No RFP support, no catering policy extraction, no AV spec parsing

**Verdict:** SeatPlan.io solves the downstream problem — arranging guests in a space the planner has already chosen. BENE solves the upstream problem — knowing and comparing the spaces themselves. The AI floor plan import is the only genuine overlap: both tools ingest floor plan PDFs. But SeatPlan produces a seating layout; BENE extracts structured venue intelligence. Complementary: a planner uses BENE to decide which room to book, then SeatPlan.io to seat the guests.

---

#### FloorScan.ai

**What it is:** AI-powered construction takeoff tool targeting quantity surveyors, site managers, and BIM modelers. Upload a PDF construction plan, crop the area of interest, and the AI detects all architectural elements (walls, doors, windows, rooms, surfaces) with confidence scores. Outputs an annotated PDF plus an Excel report with quantities and surface areas. Native DXF/AutoCAD export. Claims 5 hours saved per project vs. manual measurement. Active user base in France, Morocco, and Portugal; used by contractors including Eiffage and Léon Grosse.

**Strengths:**

- Fast turnaround: full analysis of a construction plan in under 30 seconds
- Purpose-built computer vision models trained exclusively on architectural drawings — not a generic vision API
- Interactive validation UI: review and correct each detection before export
- Native DXF and Excel export — output lands directly in AutoCAD or a quantity spreadsheet
- Structured output with confidence scores and surface quantities — not just bounding boxes
- Free tier with 30-day Pro trial, no credit card required

**Gaps relevant to BENE Intelligence:**

- Construction takeoff tool — built for quantity surveyors estimating material volumes, not event planners managing venue portfolios
- No venue-specific extraction schema (no capacity configurations, catering policies, AV specs, access restrictions)
- No document intelligence beyond floor plan geometry — cannot parse spec sheets, venue decks, or mixed PDF documents
- No multi-source aggregation or conflict resolution across documents
- No team knowledge base, no cross-venue search, no semantic query capability
- Narrow vertical: construction estimation workflows in France/Europe — not broadly available

**Verdict:** FloorScan.ai is a construction industry tool whose core capability — AI geometry extraction from PDF floor plans — is the same technical problem BENE faces for venue CAD files. It's not a competitor but a technical reference point: the accuracy benchmark (sub-30s analysis, confidence-scored element detection, DXF export) is the quality bar BENE's floor plan parsing pipeline should target. BENE's differentiation is what happens _after_ geometric extraction: mapping room dimensions to event configurations, seating capacities, and a searchable venue intelligence schema.

---

#### FloorPlan API (floorplanapi.com)

**What it is:** Dedicated computer vision API that accepts raster floor plans (PNG, JPG, WebP, rasterized PDF) and returns pixel-aligned geometric output: walls, doors, windows, and measurements. Segmentation models trained exclusively on architectural drawings (not a general vision model). Benchmarked at 85.31% Wall IoU on CubiCasa5k; P95 sync latency under 5 seconds. Free tier: 50 API calls/month. Paid tiers from $29/month (100 calls) to $99/month (500 calls) with enterprise custom pricing.

**What it does and doesn't do:**

- Input: raster images of floor plans (must pre-rasterize PDF pages before submission)
- Output: pixel-aligned geometry (walls, doors, windows, measurements)
- Does not: parse spec sheet text, extract venue metadata, handle mixed-content PDFs, provide a venue schema

**Relevance to BENE Intelligence:**

- FloorPlan API is a potential _infrastructure component_ for BENE's floor plan pipeline — the same role Apache Tika plays for text extraction
- It handles the hardest part of floor plan processing: geometric segmentation from raster images
- The 85.31% Wall IoU benchmark on CubiCasa5k is a documented quality baseline
- REST API integration is straightforward — fits into a Spring Boot ETL pipeline as a pre-processing step before venue schema enrichment
- Per-call pricing ($0.29–$0.50/call at paid tiers) is viable for a venue intelligence platform processing hundreds of floor plans, not millions

**Strategic decision for BENE Intelligence:** FloorPlan API (or an equivalent: CubiCasa, Archilogic, or an in-house Docling-based pipeline) handles geometric extraction from raster floor plans. BENE's proprietary layer is the venue intelligence schema mapped _onto_ that geometry — room identifiers, configuration capacities, sightlines, access points — none of which FloorPlan API provides.

**Verdict:** Not a competitor. A candidate infrastructure vendor for the geometric extraction step of BENE's CAD/floor plan ingestion pipeline.

---

#### Open Location Stack (openlocationstack.com)

**What it is:** Open-source middleware project by FORMATION for indoor location and RTLS (Real-Time Location System) deployments. Built around the omlox open locating standard and OGC IMDF (Indoor Mapping Data Format). Core components: Open Location Hub (cloud-capable hub-of-hubs aggregating data from omlox-compatible on-premise hubs), Open Location Hub CLI, and a browser-based Floor Plan Editor for structured vector indoor maps. MIT licensed. Supported by RTLS hardware vendors including INDUTRAX, Sinfosy, and Synchronic. Participated in omlox Plugfest 2026.

**What it does:**

- Aggregates live location telemetry from multiple RTLS technologies (UWB, BLE, Wi-Fi RTT, RFID, GNSS, QR/barcode) across sites and vendors
- Manages structured indoor maps (IMDF) as spatial context for live location feeds
- Provides vendor-agnostic hub federation — one API surface across mixed-vendor deployments
- Floor Plan Editor: browser-based tool for authoring vector geometry, routing paths, and zones aligned to IMDF
- Designed for industrial/logistics use cases: asset tracking, workplace visibility, AI-assisted logistics

**Gaps relevant to BENE Intelligence:**

- Live asset tracking / RTLS middleware — designed for real-time position data, not document intelligence
- No document ingestion, no PDF parsing, no spec sheet extraction
- IMDF-based floor plan representation is about _live spatial context for tracking_, not venue knowledge for event planners
- No venue portfolio management, no semantic search, no RFP support
- Requires physical RTLS hardware deployment — far outside the scope of a venue intelligence platform

**Why it appears here:** The IMDF structured indoor map format is relevant to BENE's long-term architecture. IMDF is the OGC standard for portable venue maps — georeferenced vector geometry with named rooms, levels, and zones. If BENE ever moves beyond document extraction into interactive floor plan rendering (Phase 3+), IMDF is the output format to target. Open Location Stack's Floor Plan Editor demonstrates what a browser-based IMDF authoring tool looks like.

**Verdict:** No competitive overlap. Open Location Stack targets logistics and manufacturing RTLS deployments. BENE targets event planning knowledge management. The IMDF format is worth monitoring as a potential output standard for BENE's structured venue map layer.

---

### 1.6 Competitive Gap Summary

| Capability                               | Cvent            | Tripleseat      | Momentus        | Facilitron         | Propared         | Planning Pod    | VenueScanner    | VenueFindAI     | VenueArc        | Spark (GEVME/PCMA)    | Bynder            | Brandfolder         | Unstructured.io | SeatPlan.io           | FloorScan.ai      | FloorPlan API      | Open Location Stack | BENE                |
| ---------------------------------------- | ---------------- | --------------- | --------------- | ------------------ | ---------------- | --------------- | --------------- | --------------- | --------------- | --------------------- | ----------------- | ------------------- | --------------- | --------------------- | ----------------- | ------------------ | ------------------- | ------------------- |
| Venue discovery (marketplace)            | ✅ Best-in-class | ⛔              | ⛔              | ✅ (public sector) | ⛔               | ⛔              | ✅ (UK-focused) | ✅ (AI + human) | ⛔              | Partial (DestinAItor) | ⛔                | ⛔                  | ⛔              | ⛔                    | ⛔                | ⛔                 | ⛔                  | Phase 3             |
| Planner's own venue library              | ⛔               | ⛔              | ⛔              | ⛔                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | ⛔                  | ⛔              | ⛔                    | ⛔                | ⛔                 | ⛔                  | ✅                  |
| Document / asset storage                 | ⛔               | ⛔              | ⛔              | ⛔                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ✅ (brand assets) | ✅ (brand assets)   | ✅ (infra only) | ⛔                    | ⛔                | ⛔                 | ⛔                  | ✅                  |
| Document intelligence (PDF, floor plans) | ⛔               | ⛔              | ⛔              | ⛔                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | ⛔                  | ✅ (infra only) | Partial (layout only) | ✅ (construction) | ✅ (geometry only) | ⛔                  | ✅                  |
| AI metadata extraction                   | ⛔               | ⛔              | ⛔              | ⛔                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ✅ (generated text)   | ✅ (generic)      | ✅ (generic)        | ✅ (generic)    | ✅ (seating layout)   | ✅ (quantities)   | ✅ (geometry)      | ⛔                  | ✅ (venue-specific) |
| Floor plan geometry extraction           | ⛔               | ⛔              | ⛔              | ⛔                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | ⛔                  | ⛔              | ✅ (seating layout)   | ✅ (walls/rooms)  | ✅ Best-in-class   | ✅ (IMDF/vector)    | ✅ (via Tika/API)   |
| CAD / DXF support                        | ⛔               | ⛔              | ⛔              | ⛔                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | ⛔                  | ⛔              | ✅ (import)           | ✅ (export)       | ⛔ (raster only)   | ✅ (IMDF authoring) | ✅ (via Tika)       |
| Event planning workflow / content gen    | ⛔               | ⛔              | ⛔              | ⛔                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ✅ Best-in-class      | ⛔                | ⛔                  | ⛔              | ✅ (seating/check-in) | ⛔                | ⛔                 | ⛔                  | ⛔                  |
| Production planning (crew, schedule)     | ⛔               | ⛔              | ⛔              | ⛔                 | ✅ Best-in-class | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | ⛔                  | ⛔              | ⛔                    | ⛔                | ⛔                 | ⛔                  | ⛔                  |
| Semantic search (vector)                 | ⛔               | ⛔              | ⛔              | ⛔                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | Partial (NL search) | ⛔              | ⛔                    | ⛔                | ⛔                 | ⛔                  | ✅                  |
| Real-time location / RTLS                | ⛔               | ⛔              | ⛔              | ⛔                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | ⛔                  | ⛔              | ⛔                    | ⛔                | ⛔                 | ✅ Best-in-class    | ⛔                  |
| Team collaboration / shared library      | Partial          | ⛔              | ✅              | ✅ (internal)      | ✅               | ✅ (venue-side) | ⛔              | ⛔              | ⛔              | Partial               | ✅                | ✅                  | ⛔              | ✅ (event-scoped)     | ⛔                | ⛔                 | ⛔                  | ✅                  |
| Multi-tenant SaaS                        | ✅               | ✅              | ✅              | ✅                 | ✅               | ✅              | ✅              | ✅              | ✅              | ✅                    | ✅                | ✅                  | ✅              | ✅                    | ⛔ (early-stage)  | ✅ (API)           | ⛔ (open source)    | ✅                  |
| Venue-specific schema                    | ⛔               | ✅ (operations) | ✅ (operations) | ✅ (operations)    | ⛔               | ✅ (operations) | ⛔              | ⛔              | ✅ (operations) | ⛔                    | ⛔                | ⛔                  | ⛔              | ⛔                    | ⛔                | ⛔                 | ⛔                  | ✅ (intelligence)   |
| SMB-friendly pricing                     | ⛔               | Partial         | ⛔              | ✅ (no cost)       | ✅               | ✅              | ✅ (free)       | ✅ (free)       | Partial         | ✅ (free tier)        | ⛔                | ⛔                  | ✅              | ✅ (free tier)        | ✅ (free tier)    | ✅ (free tier)     | ✅ (open source)    | ✅                  |

**The gap BENE fills:** Nobody provides document intelligence specifically for event planners managing their own venue portfolio. Marketplace/sourcing tools (Cvent, VenueScanner) only know what venues self-report. AI productivity tools (Spark) generate content but have no venue knowledge base. DAM platforms (Bynder, Brandfolder) store files with generic tagging but understand nothing about venue semantics. Operations platforms (Tripleseat, Momentus, VenueArc) serve venue operators, not planners, and contain no document intelligence. Generic document APIs (Unstructured.io) handle extraction but have no venue schema. Floor plan and spatial tools (SeatPlan.io, FloorScan.ai, FloorPlan API, Open Location Stack) solve geometry and layout problems for construction or logistics — they are infrastructure building blocks, not planner intelligence platforms. BENE Intelligence is the missing layer: structured, searchable, planner-owned venue intelligence extracted from the documents planners already have.

---

## 2. ETL Pipeline & Intelligence Layer

The document intelligence infrastructure, ETL pipeline architecture, venue-specific extraction schema, multi-source aggregation, scalability, and technology decisions are covered in the dedicated reference:

→ **[Intelligence Layer & ETL Pipeline](../platform/intelligence.md)**

---

**Document type:** Competitive landscape reference
**Stage:** Pre-build design
**Audience:** Engineering, founding team

---

**Docs:** [What is BENE?](../README.md) · [Business Proposal](proposal.md) · [Competitive Landscape](comparison.md) · [Intelligence Layer](../platform/intelligence.md) · [Architecture](../platform/architecture.md)
