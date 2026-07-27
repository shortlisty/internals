# Venue Intelligence Platform — Intelligence Layer & Competitive Landscape

> Technical and strategic reference for the document intelligence, ETL pipeline,
> and competitive positioning of IQ BENE.

**Docs:** [What is IQ BENE?](what-is-vip.md) · [Business Overview](business-overview.md) · [Competitive Landscape](intelligence-and-competitive-landscape.md) · [Architecture](architecture.md)

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

**Gaps relevant to IQ BENE:**

- Cvent is a _discovery and booking_ platform — it doesn't help you manage your own venue library
- Venues in Cvent are self-submitted by venue owners, not extracted from your own documents
- No document intelligence (no PDF/floor plan/CAD parsing)
- No team-owned venue knowledge base
- Enterprise pricing puts it out of reach for SMB agencies

**Verdict:** Not a direct competitor. Cvent is a venue marketplace. IQ BENE is an intelligence layer for your own venue portfolio. They could be _complementary_ (import discovered venues from Cvent into IQ BENE).

---

#### Tripleseat

**What it is:** Sales and catering software for restaurants, hotels, and unique venues. 20,000+ venue clients.

**Strengths:**

- Strong operational workflows (booking, contracts, invoicing)
- Just launched "Tripleseat Intelligence" — AI suite built on their dataset of millions of events
- AI for: demand forecasting, F&B inventory recommendations, conversational analytics, peer benchmarking
- Deeply embedded in hospitality operations

**Gaps relevant to IQ BENE:**

- Tripleseat is built _for venues_ to manage their events — not _for planners_ to manage their venue portfolio
- Their AI runs on their own transactional data (bookings), not on unstructured documents
- No document parsing, no cross-venue search for planners
- No support for planner's own uploaded assets

**Verdict:** Different side of the market. Tripleseat serves venues; IQ BENE serves planners. The intelligence architectures are fundamentally different: Tripleseat mines structured operational data; IQ BENE mines unstructured documents.

---

#### Momentus Technologies (formerly Ungerboeck / VenueOps)

**What it is:** Enterprise-grade venue and event management. Serves convention centers, performing arts, stadiums, universities. 700+ performing arts centers, 50+ countries.

**Strengths:**

- End-to-end: booking → operations → finance → analytics
- AI-powered platform enhancements (Feb 2026): operational insights, space optimization
- 20+ years of venue and event intelligence baked into their models
- WeTrack product for safety/sustainability/risk management

**Gaps relevant to IQ BENE:**

- Heavy enterprise product, not accessible to SMB agencies
- Focused on venue operators managing their own space, not planners curating a portfolio
- No document intelligence or ETL pipeline
- Implementation takes months, not minutes

**Verdict:** Enterprise venue ops software. No overlap with IQ BENE's document intelligence core.

---

#### Perfect Venue / Planning Pod / Event Temple

**What it is:** Lightweight venue management tools targeting independent venues, small hotels, wineries.

**Strengths:** Affordable, easy to set up, covers booking basics.

**Gaps:** No AI, no document intelligence, no team venue library concept. More CRM than intelligence platform.

**Verdict:** Irrelevant to IQ BENE's positioning. Different price/feature tier entirely.

---

#### VenueScanner

**What it is:** UK-based venue marketplace and concierge service. 19,000+ venues across the UK and internationally. Described by Forbes as "the Airbnb of venue hire." 1M+ event organisers use the platform annually.

**Strengths:**

- Large self-serve search with filters for location, capacity, price, and amenities
- Free for event organisers — revenue model is commission from venues
- VenueScanner for Business: concierge team handles corporate briefs, negotiates rates, shortlists within 24–48 hours
- AI-ranking algorithm boosts venues that respond quickly to enquiries
- Expanding beyond UK into international markets

**Gaps relevant to IQ BENE:**

- Pure _discovery and booking_ marketplace — venues are listed by venue owners, not extracted from planner-owned documents
- No planner-side venue library or knowledge base
- No document intelligence — planners can't upload their own PDFs, floor plans, or spec sheets
- AI is limited to ranking and response-time scoring, not semantic extraction
- No cross-venue comparison against a planner's own portfolio

**Verdict:** VenueScanner is a consumer-grade venue search engine, not a planner intelligence tool. A planner who already knows their preferred venues gets nothing from VenueScanner — it only helps with first-pass discovery of venues they haven't worked with yet. Complementary to IQ BENE, not competitive.

---

#### VenueFindAI

**What it is:** AI-powered venue discovery and concierge service (venuefindai.com). Positions itself as an intelligent assistant that delivers personalised venue recommendations for corporate events and special celebrations, backed by human experts available on demand. Free to use for organisers — no fees, no commission.

**Model:** AI-first matchmaking with a human-in-the-loop concierge layer. The AI surfaces recommendations; human specialists are available "at the touch of a button" for complex or high-value briefs. Revenue model appears to be venue-side (similar to VenueScanner, Hire Space — venues pay for leads/placement rather than planners paying for the service).

**Strengths:**

- Zero cost to planners — lowers the barrier to trial significantly
- AI + human hybrid model hedges against the accuracy limits of pure AI sourcing
- Broad scope: corporate events and celebrations, suggesting a wide venue inventory or intent to build one
- Straightforward positioning that's easy for non-technical buyers to understand

**Gaps relevant to IQ BENE:**

- Discovery platform: works from a database of venues that _have listed themselves_ — not from documents a planner already owns
- No planner-side knowledge base — recommendations are ephemeral, not stored as a team asset
- No document intelligence, no PDF/floor plan ingestion, no structured extraction
- Human concierge layer adds latency and doesn't scale to a planner's full portfolio of 50–100 known venues
- AI recommendations are only as good as what venues have self-submitted — the moment a planner needs intelligence from their own files (a venue deck sent by email, a floor plan from 2019), VenueFindAI has nothing

**Verdict:** Same quadrant as VenueScanner and Cvent — a venue marketplace/sourcing tool for _discovering_ new venues. IQ BENE solves the adjacent and complementary problem: once you've found and worked with venues, how do you manage, extract intelligence from, and search across everything you already know about them. A planner could use VenueFindAI to discover a venue, then use IQ BENE to ingest that venue's documents and build a permanent, searchable profile.

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

**Gaps relevant to IQ BENE:**

- Serves _venue operators_, not event planners — the same side-of-market distinction as Momentus and Tripleseat
- No document intelligence, no PDF/floor plan ingestion, no ETL pipeline
- No planner-facing portfolio management
- Narrow vertical: performing arts centres and theatres, not the broader corporate/social events market

**Verdict:** Niche venue operations software for a specific vertical (performing arts). No overlap with IQ BENE's core use case. Closer to a lighter-weight Momentus than anything in IQ BENE's competitive set.

---

### 1.2 Digital Asset Management (DAM) Platforms

DAM platforms are the closest adjacent category to IQ BENE — they centralize, tag, search, and distribute digital files for marketing and brand teams. Unlike the booking/CRM tools above, DAMs do handle unstructured files (images, PDFs, videos). But their intelligence is built for brand governance, not venue intelligence.

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

**Gaps relevant to IQ BENE:**

- Built for _brand assets_ (logos, campaign images, marketing docs) — not venue intelligence
- No venue-specific extraction schema (no capacity, amenities, room configurations, restrictions)
- AI enrichment is generic (auto-tags any image/document) — no domain understanding of floor plans or spec sheets
- No document parsing pipeline designed for multi-source, multi-format venue documents
- No semantic search against _extracted_ venue attributes — only metadata search
- No cross-document aggregation or conflict resolution
- Priced for enterprise marketing teams, not event planning agencies

**Verdict:** Bynder is a sophisticated content library, not an intelligence platform. If a planner stored venue PDFs in Bynder, they'd get organized storage and generic AI tags. They would _not_ get extracted capacity data, structured room configurations, or answers to "find me a venue with a freight elevator and kosher catering." The intelligence layer IQ BENE provides simply doesn't exist in any DAM.

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

**Gaps relevant to IQ BENE:**

- Same fundamental limitation as Bynder: designed for _brand/marketing assets_, not venue intelligence
- Brand Intelligence tags visual content generically — no understanding of venue-specific semantics
- No structured extraction of capacity configurations, catering policies, AV specs, or floor plan geometry
- No multi-source aggregation: uploading three versions of the same venue deck doesn't reconcile conflicts
- No event planner workflow: no RFP support, no venue comparison, no sourcing history
- Smartsheet integration helps with _work management_ (tasks, approvals) — not with venue knowledge

**Verdict:** Brandfolder is Bynder's strongest mid-market competitor, with a cleaner UX and a stronger AI tagging story. Neither is a venue intelligence platform. A planner using Brandfolder gets a well-organized file library with good visual search — and nothing more. The gap IQ BENE fills (venue-specific structured extraction, semantic search by venue attributes, multi-source reconciliation) is entirely absent from the DAM category.

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

**Gaps relevant to IQ BENE:**

- Spark is a _content generation and workflow automation_ tool — it helps planners write faster, not know their venues better
- No venue knowledge base: no venue profiles, no stored documents, no portfolio management
- DestinAItor does venue research from web/public data — it does not ingest or parse a planner's own venue documents
- No document intelligence: no PDF parsing, no floor plan extraction, no CAD support
- No structured venue metadata schema — outputs are generated text, not queryable structured data
- No multi-source aggregation or conflict resolution across documents

**Verdict:** Spark is the closest thing to a purpose-built AI assistant for event planners, and it's well-adopted (14,500+ users). But it's a _writing and workflow_ tool, not a _knowledge_ tool. A planner using Spark can draft an RFP faster — but still has no structured, searchable record of what their 50 preferred venues actually offer. IQ BENE and Spark are complementary: Spark generates the content; IQ BENE supplies the venue intelligence that makes that content accurate and specific.

---

### 1.4 Document Intelligence & ETL Platforms

These are the infrastructure players. They are the technical substrate that IQ BENE's pipeline either competes with or can leverage.

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

**Relevance to IQ BENE:**

- Unstructured.io is what IQ BENE's ETL layer _could use as a backend_ rather than building from scratch
- Their open-source library (`unstructured`) can be self-hosted
- Handles the hardest parsing problems (scanned PDFs, multi-column layouts, tables)
- Not a product for end users — pure infrastructure/API

**Strategic insight:** IQ BENE doesn't need to reinvent document parsing. Unstructured.io (or Docling) handles the extraction layer. IQ BENE's value add is the _venue-specific intelligence_ on top — the domain schema, the aggregation model, the search experience, the team collaboration.

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

**Why it matters for IQ BENE:**

- Free, open-source, no per-page pricing
- Superior table and layout understanding vs. naive PDF parsing
- Direct integration with Spring AI via `TikaDocumentReader` (Apache Tika underneath) or custom `DocumentReader`
- IBM Granite-Docling-258M: new ultra-compact VLM for document-to-structured-format conversion
- Ideal for floor plan PDFs, spec sheets with tables, multi-column venue decks

**Strategic decision for IQ BENE:** Use Docling as the primary document parsing layer. It handles the structural extraction (layout, tables, text) and Spring AI's ETL pipeline then handles chunking, embedding, and vector storage.

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

**Why it's the right choice for IQ BENE:**

- Battle-tested in enterprise Java for 15+ years
- DWG/DXF support via Tika's AutoCAD parser (direct path for CAD files)
- Zero extra infrastructure — runs in-process
- Tika Pipes for production safety (malformed files can't crash the service)
- Already in Spring AI's ETL pipeline — no custom integration needed

---

#### LlamaIndex / LangChain

**What they are:** Python-first AI orchestration frameworks. LlamaIndex has strong document parsing (LlamaExtract, LlamaParse). LangChain has document loaders.

**Relevance to IQ BENE:** These are Python-ecosystem tools. Since IQ BENE is Java/Spring Boot, they are not directly applicable. Spring AI is the Java equivalent and has caught up rapidly.

**Note:** If IQ BENE ever needs a Python microservice for specialized extraction (e.g., advanced floor plan analysis), LlamaIndex's LlamaParse is best-in-class for complex PDFs.

---

#### Reducto, Raydocs, Retab (Document Intelligence Startups)

Recent well-funded entrants in the document intelligence space:

- **Reducto** — $75M Series B (a16z, Benchmark, YC). API-first structured extraction from complex documents.
- **Raydocs** — Template-based extraction with confidence scores and source links.
- **Retab** — Pre-seed $3.5M. Non-technical users building extraction templates.

**Pattern:** All these companies are _horizontal_ document intelligence APIs. IQ BENE's opportunity is to be _vertical_ — deeply specialized for venue documents (floor plans, venue decks, CAD files, spec sheets). Horizontal tools extract generic fields. IQ BENE extracts venue-specific intelligence with a purpose-built schema.

---

### 1.5 Competitive Gap Summary

| Capability                               | Cvent            | Tripleseat      | Momentus        | VenueScanner    | VenueFindAI     | VenueArc        | Spark (GEVME/PCMA)    | Bynder            | Brandfolder         | Unstructured.io | IQ BENE             |
| ---------------------------------------- | ---------------- | --------------- | --------------- | --------------- | --------------- | --------------- | --------------------- | ----------------- | ------------------- | --------------- | ------------------- |
| Venue discovery (marketplace)            | ✅ Best-in-class | ⛔              | ⛔              | ✅ (UK-focused) | ✅ (AI + human) | ⛔              | Partial (DestinAItor) | ⛔                | ⛔                  | ⛔              | Phase 3             |
| Planner's own venue library              | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | ⛔                  | ⛔              | ✅                  |
| Document / asset storage                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ✅ (brand assets) | ✅ (brand assets)   | ✅ (infra only) | ✅                  |
| Document intelligence (PDF, floor plans) | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | ⛔                  | ✅ (infra only) | ✅                  |
| AI metadata extraction                   | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔              | ✅ (generated text)   | ✅ (generic)      | ✅ (generic)        | ✅ (generic)    | ✅ (venue-specific) |
| Event planning workflow / content gen    | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔              | ✅ Best-in-class      | ⛔                | ⛔                  | ⛔              | ⛔                  |
| CAD file support (DWG/DXF)               | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | ⛔                  | ⛔              | ✅ (via Tika)       |
| Semantic search (vector)                 | ⛔               | ⛔              | ⛔              | ⛔              | ⛔              | ⛔              | ⛔                    | ⛔                | Partial (NL search) | ⛔              | ✅                  |
| Team collaboration / shared library      | Partial          | ⛔              | ✅              | ⛔              | ⛔              | ⛔              | Partial               | ✅                | ✅                  | ⛔              | ✅                  |
| Multi-tenant SaaS                        | ✅               | ✅              | ✅              | ✅              | ✅              | ✅              | ✅                    | ✅                | ✅                  | ✅              | ✅                  |
| Venue-specific schema                    | ⛔               | ✅ (operations) | ✅ (operations) | ⛔              | ⛔              | ✅ (operations) | ⛔                    | ⛔                | ⛔                  | ⛔              | ✅ (intelligence)   |
| SMB-friendly pricing                     | ⛔               | Partial         | ⛔              | ✅ (free)       | ✅ (free)       | Partial         | ✅ (free tier)        | ⛔                | ⛔                  | ✅              | ✅                  |

**The gap IQ BENE fills:** Nobody provides document intelligence specifically for event planners managing their own venue portfolio. Marketplace/sourcing tools (Cvent, VenueScanner) only know what venues self-report. AI productivity tools (Spark) generate content but have no venue knowledge base. DAM platforms (Bynder, Brandfolder) store files with generic tagging but understand nothing about venue semantics. Operations platforms (Tripleseat, Momentus, VenueArc) serve venue operators, not planners, and contain no document intelligence. Generic document APIs (Unstructured.io) handle extraction but have no venue schema. IQ BENE is the missing layer: structured, searchable, planner-owned venue intelligence extracted from the documents planners already have.

---

## 2. ETL Pipeline Architecture — Proven Foundation

### 2.1 Spring AI's ETL Pipeline

Spring AI ships a first-class, production-grade ETL pipeline with three composable stages:

```
DocumentReader  →  DocumentTransformer  →  DocumentWriter
   (Extract)           (Transform)            (Load)
```

**DocumentReaders (Extract) — available out of the box:**

| Reader                       | Handles                                         | Notes                                                       |
| ---------------------------- | ----------------------------------------------- | ----------------------------------------------------------- |
| `TikaDocumentReader`         | PDF, DOCX, XLSX, PPTX, HTML, XML, 1000+ formats | Apache Tika under the hood. **Primary reader for IQ BENE.** |
| `PagePdfDocumentReader`      | PDFs, page-by-page                              | Preserves page boundaries, useful for floor plans           |
| `ParagraphPdfDocumentReader` | PDFs, paragraph-level                           | Better semantic chunking for venue decks                    |
| `MarkdownDocumentReader`     | Markdown files                                  | Useful for structured venue specs                           |
| `JsonMetadataReader`         | JSON with metadata                              | Useful for structured imports                               |
| `JsoupDocumentReader`        | HTML pages                                      | Web scraping venue information                              |

**DocumentTransformers (Transform):**

| Transformer                    | What it does                                                                         |
| ------------------------------ | ------------------------------------------------------------------------------------ |
| `TokenTextSplitter`            | Splits large documents into chunks respecting token limits                           |
| `ContentFormatTransformer`     | Normalizes text format                                                               |
| `SummaryMetadataEnricher`      | Generates document summary using LLM, stored as metadata                             |
| `KeywordMetadataEnricher`      | Extracts keywords using LLM, stored as metadata                                      |
| Custom `VenueMetadataEnricher` | **IQ BENE-specific:** extracts capacity, amenities, contacts via structured LLM call |

**DocumentWriters (Load):**

| Writer               | What it does                                      |
| -------------------- | ------------------------------------------------- |
| `PgVectorStore`      | Writes chunks + embeddings to PostgreSQL pgvector |
| `SimpleVectorStore`  | In-memory (testing/dev)                           |
| `FileDocumentWriter` | Write to files (useful for debugging pipeline)    |

### 2.2 IQ BENE's Document Processing Pipeline

```
                     S3 Asset Storage
                          │
                          │ presigned URL download
                          ▼
               ┌─────────────────────┐
               │  DocumentReader     │  Spring AI / Apache Tika
               │  (per asset type)   │  + IBM Docling (PDF tables)
               └──────────┬──────────┘
                          │  List<Document>
                          │  (raw text chunks + page metadata)
                          ▼
               ┌─────────────────────┐
               │  DocumentSplitter   │  TokenTextSplitter
               │                     │  (512 tokens, 50 overlap)
               └──────────┬──────────┘
                          │  List<Document> (chunks)
                          ▼
               ┌─────────────────────┐
               │  VenueMetadata      │  GPT-4o structured output
               │  Enricher           │  → capacity, amenities, contacts
               └──────────┬──────────┘
                          │  List<Document> + venue metadata
                          ▼
               ┌─────────────────────┐
               │  EmbeddingModel     │  text-embedding-3-small
               │                     │  (1536 dimensions per chunk)
               └──────────┬──────────┘
                          │  List<Document> + float[] embeddings
                          ▼
               ┌─────────────────────┐
               │  TenantAware        │  PostgreSQL + pgvector
               │  PgVectorStore      │  per-tenant schema
               └──────────┬──────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │  MetadataAggregator │  Event-sourced consolidation
               │                     │  (conflict resolution)
               └─────────────────────┘
```

**Java implementation sketch:**

```java
@Service
@RequiredArgsConstructor
public class VenueAssetProcessingPipeline {

  private final TikaDocumentReader.Factory tikaFactory;
  private final TokenTextSplitter splitter;
  private final VenueMetadataEnricher enricher;   // custom Spring AI DocumentTransformer
  private final EmbeddingModel embeddingModel;
  private final VectorStore vectorStore;
  private final MetadataAggregationService aggregationService;

  public void process(VenueAsset asset, byte[] content) {
    // 1. Extract — Tika handles PDF, DOCX, XLSX, images via OCR, DWG
    var reader = tikaFactory.create(new ByteArrayResource(content), asset.getContentType());
    var rawDocs = reader.get();

    // 2. Enrich raw docs with source metadata
    var taggedDocs = rawDocs.stream()
      .map(doc -> doc.mutate()
        .metadata("venue_id", asset.getVenueId())
        .metadata("asset_id", asset.getId())
        .metadata("asset_type", asset.getType())
        .metadata("tenant_id", TenantContext.getCurrentTenantId())
        .build())
      .toList();

    // 3. Split into semantic chunks
    var chunks = splitter.apply(taggedDocs);

    // 4. Enrich with venue-specific metadata (LLM structured call)
    var enrichedChunks = enricher.apply(chunks);

    // 5. Embed + write to pgvector
    vectorStore.add(enrichedChunks);

    // 6. Aggregate extracted metadata into venue profile
    var extractedMetadata = enricher.getLastExtractionResult();
    aggregationService.applyExtractionEvent(asset, extractedMetadata);
  }
}
```

### 2.3 Asset-Type Processing Matrix

| Asset Type             | Parser                            | OCR Needed  | Structured Extraction | Vector Indexed    |
| ---------------------- | --------------------------------- | ----------- | --------------------- | ----------------- |
| PDF Deck (text)        | Tika → ParagraphPdfDocumentReader | No          | Yes (GPT-4o)          | Yes               |
| PDF Deck (scanned)     | Tika + OCR (Tesseract)            | Yes         | Yes (GPT-4o)          | Yes               |
| Floor Plan (PDF)       | Docling (layout-aware)            | No          | Yes (GPT-4o vision)   | Yes               |
| Floor Plan (image)     | GPT-4o vision direct              | Yes (LLM)   | Yes                   | Yes               |
| Photos                 | GPT-4o vision                     | No          | Amenity detection     | Yes               |
| DOCX / Technical Spec  | TikaDocumentReader                | No          | Yes (GPT-4o)          | Yes               |
| XLSX (capacity tables) | TikaDocumentReader                | No          | Structured parsing    | Yes               |
| DWG / DXF (CAD)        | Tika AutoCAD parser               | No          | Metadata only         | Metadata only     |
| Video (walkthrough)    | Extract thumbnail + audio         | Via Whisper | Partial               | Partial (Phase 2) |

### 2.4 Chunking Strategy

Document chunking significantly impacts retrieval quality. IQ BENE uses a hybrid strategy:

**For venue decks (PDFs):**

- `ParagraphPdfDocumentReader`: preserves paragraph structure
- Chunk size: 512 tokens with 50-token overlap
- Metadata per chunk: page number, section heading, asset ID, confidence tier

**For floor plans (images/PDFs with diagrams):**

- Page-level chunking (one Document per page)
- Attach full-page image for GPT-4o vision processing
- Extract: room names, dimensions, capacity annotations

**For spec sheets (tables):**

- Docling's `TableFormer` model reconstructs table cells
- Each table row becomes a document with column headers as metadata
- Preserves relational structure: `{"room": "Grand Ballroom", "capacity_banquet": 400, "capacity_theater": 600}`

**For photos:**

- No text chunking — pass directly to GPT-4o vision
- Single Document per image with vision-extracted metadata

### 2.5 Why Apache Tika is the Right Foundation

Apache Tika has been the Java ecosystem's battle-tested document parser since 2007:

- **1000+ file formats** — one interface regardless of file type
- **Tika Pipes** — each file processed in forked JVM; a malformed or malicious file cannot crash the service
- **Production-proven** — used by Elasticsearch, Solr, Apache Nutch, enterprise search systems globally
- **DWG/DXF support** — AutoCAD format parser built in (rare capability — most alternatives don't support this)
- **Direct Spring AI integration** — `TikaDocumentReader` ships with Spring AI, no custom code needed
- **Zero cloud dependency** — runs in-process, no API call needed for text extraction

**Tika Pipes configuration for production safety:**

```java
@Bean
public TikaDocumentReader.Factory tikaReaderFactory() {
  return TikaDocumentReader.Factory.builder()
    .withTikaConfig(TikaConfig.getDefaultConfig())
    .withForkParser(true)          // Each file in isolated JVM
    .withParseTimeout(60)          // 60s max per file
    .withMaxMemory(256 * 1024 * 1024)  // 256MB per file
    .build();
}
```

### 2.6 Docling Integration for High-Fidelity PDF Parsing

For venue decks with complex layouts, tables, and multi-column structures, Docling outperforms basic Tika PDF parsing:

**When to use Docling over Tika for PDFs:**

- Multi-column layouts (most venue decks)
- Capacity tables (banquet vs. theater vs. classroom setups)
- Mixed text + diagram pages (floor plans embedded in PDFs)
- Scanned PDFs requiring layout-aware OCR

**Integration approach:**

```java
// Spring AI custom DocumentReader wrapping Docling HTTP API
@Component
public class DoclingDocumentReader implements DocumentReader {

  private final DoclingClient doclingClient; // REST client to Docling service

  @Override
  public List<Document> get() {
    var doclingResult = doclingClient.convert(resource, ConversionOptions.builder()
      .withTableExtraction(true)
      .withOcr(resource.getContentType().contains("image"))
      .build());

    return doclingResult.getChunks().stream()
      .map(chunk -> Document.builder()
        .content(chunk.getText())
        .metadata("page", chunk.getPage())
        .metadata("element_type", chunk.getType()) // TEXT, TABLE, FIGURE
        .metadata("table_data", chunk.getTableJson())
        .build())
      .toList();
  }
}
```

**Docling deployment (self-hosted, zero cost):**

```yaml
# docker-compose.yaml addition
docling-service:
  image: ds4sd/docling-serve:latest
  ports:
    - "5001:5001"
  environment:
    - DOCLING_WORKER_THREADS=4
```

**IBM Granite-Docling-258M** (released 2026): An ultra-compact VLM that converts documents to structured formats while preserving layout, tables, equations. Can replace Docling's heavy ML models with a lighter inference endpoint for cost-sensitive deployments.

---

## 3. The Intelligence Layer IQ BENE Owns

Everything above (Tika, Docling, Spring AI ETL) is infrastructure. IQ BENE's proprietary intelligence sits on top:

### 3.1 Venue-Specific Extraction Schema

Generic document intelligence tools extract generic fields. IQ BENE extracts fields that matter for event professionals:

```json
{
  "venue_profile": {
    "capacity": {
      "max_total": 500,
      "configurations": {
        "banquet": 300,
        "theater": 500,
        "classroom": 200,
        "cocktail": 450,
        "conference": 150
      }
    },
    "venue_type": ["conference_center", "hotel_ballroom"],
    "location": {
      "address": "...",
      "neighborhood": "Midtown",
      "proximity_notes": "3 blocks from Grand Central"
    },
    "catering": {
      "policy": "in_house_exclusive",
      "kosher_available": true,
      "halal_available": false,
      "outside_catering_allowed": false
    },
    "av_tech": {
      "built_in_av": true,
      "projector_lumens": 5000,
      "screens": 2,
      "rigging_points": true,
      "internet_bandwidth_mbps": 1000
    },
    "accessibility": {
      "ada_compliant": true,
      "elevator_access": true,
      "accessible_restrooms": true,
      "wheelchair_stage_access": false
    },
    "logistics": {
      "load_in_access": "freight_elevator",
      "parking_spaces": 200,
      "valet_available": true,
      "curfew_time": "23:00"
    },
    "restrictions": ["no_open_flame", "no_confetti", "no_outside_alcohol"],
    "contacts": [{ "name": "...", "role": "venue_sales", "email": "...", "phone": "..." }],
    "pricing": {
      "minimum_spend": 10000,
      "currency": "USD",
      "rental_fee_indicative": 5000
    }
  }
}
```

This schema is what makes IQ BENE a _venue intelligence platform_, not just a document storage system. Every competitor either has operational data (bookings, invoicing) or generic extraction. No one has this schema purpose-built for event planners.

### 3.2 Confidence-Sourced Metadata Model

Each field carries full provenance:

```json
"capacity.max_total": {
  "value": 500,
  "confidence": 0.94,
  "source_type": "PDF_DECK",
  "source_page": 4,
  "extraction_model": "gpt-4o-2024-08-06",
  "alternatives": [
    { "value": 480, "confidence": 0.72, "source_type": "FLOOR_PLAN" }
  ],
  "overridden_by": null
}
```

No existing venue tool surfaces this level of data provenance. Users see not just the value but _why_ the system believes it.

### 3.3 Multi-Source Aggregation (The Hard Problem Nobody Solves)

Venues send the same venue in multiple formats — a marketing deck, a floor plan PDF, a technical spec sheet, a photo set. Each source may have conflicting or complementary data.

IQ BENE's aggregation engine:

1. Collects all extraction events per venue (event log)
2. Applies priority rules: `manual_override > verified > high_confidence_AI > low_confidence_AI`
3. For arrays (amenities, restrictions): set-union with confidence weighting
4. Surfaces conflicts in the UI: "AI found two different capacity values — which is correct?"
5. Allows one-click resolution

This is a genuine product moat. No other platform in the event space does this.

---

## 4. Scalability Architecture

### Event-Driven, Horizontally Scalable

```
Upload → S3 → AssetUploadedEvent → RabbitMQ → N consumers → Processing Pipeline
                                                    ↑
                                              Scale consumers
                                              based on queue depth
```

- Each extraction job is independent — N workers can run in parallel
- Priority queues by plan tier (Enterprise processes first)
- Dead-letter queue catches failures — automatic retry with backoff
- No single point of failure in the processing path

### Cost Scaling

| Scale            | AI Cost | Infrastructure                         |
| ---------------- | ------- | -------------------------------------- |
| 100 venues (MVP) | ~$0.10  | Single container                       |
| 10K venues       | ~$10    | 2-3 containers                         |
| 1M venues        | ~$1,000 | Auto-scaled, still manageable          |
| 100M venues      | ~$100K  | Optimize with cheaper models + caching |

At the $0.001/venue cost of GPT-4o extraction + embedding generation, IQ BENE can process 1 million venues for approximately $1,000 in AI costs. This is not a cost problem.

### Vector Search Scaling

pgvector with IVFFlat index:

- Sub-10ms semantic search at 1M venues
- Scales to ~5M vectors per instance before needing optimization (HNSW index or Pgvector cloud)
- Multi-tenant isolation via schema-per-tenant — no cross-tenant data leakage

---

## 5. Technology Decisions Summary

| Layer                   | Choice                                           | Rationale                                                                 |
| ----------------------- | ------------------------------------------------ | ------------------------------------------------------------------------- |
| **Document parsing**    | Apache Tika (via Spring AI `TikaDocumentReader`) | 1000+ formats, DWG support, fault-tolerant Pipes, built into Spring AI    |
| **PDF layout analysis** | IBM Docling (self-hosted, open source)           | State-of-the-art table/layout extraction, MIT license, no per-page cost   |
| **AI framework**        | Spring AI 1.0                                    | Java-native, provider-agnostic, ETL pipeline built-in, Micrometer metrics |
| **LLM extraction**      | OpenAI GPT-4o                                    | Best structured output, multimodal (vision for images/floor plans)        |
| **Embeddings**          | OpenAI text-embedding-3-small                    | 1536 dims, $0.02/1M tokens, excellent quality/cost ratio                  |
| **Vector store**        | pgvector (PostgreSQL extension)                  | No extra service, transactional, tenant-isolated, production-ready        |
| **Full-text search**    | PostgreSQL tsvector                              | Unified with relational data, no extra service                            |
| **Geo search**          | PostGIS (PostgreSQL extension)                   | Mature, no extra service                                                  |
| **Async processing**    | RabbitMQ (existing foundation)                   | Already in platform, priority queues, DLQ                                 |
| **File storage**        | S3 / MinIO (existing foundation)                 | Already in IAM service, same pattern                                      |

**Principle:** Use proven infrastructure that already exists in the IQKV foundation. Introduce the minimum number of new services. The only truly new infrastructure is pgvector (a PostgreSQL extension, not a new service) and optionally a self-hosted Docling container for advanced PDF parsing.

---

## 6. Open Questions for Implementation

- **Docling vs. pure Tika:** Start with Tika for MVP speed. Add Docling for Phase 2 when floor plan fidelity matters.
- **OCR strategy:** Tika bundles Tesseract for basic OCR. GPT-4o vision handles complex cases. Threshold: if Tika OCR confidence < 0.7, escalate to GPT-4o vision.
- **CAD files:** Tika extracts metadata from DWG/DXF (dimensions, layers). For Phase 1, expose raw metadata. Phase 2: convert to PNG via LibreCAD/ODA, then GPT-4o vision for layout understanding.
- **Video walkthroughs:** Out of scope for Phase 1. Phase 2: extract keyframes (ffmpeg), run GPT-4o vision on representative frames.
- **Chunking overlap:** 50-token overlap is standard. Venue-specific content (e.g., capacity tables) should use smaller chunks (256 tokens) to preserve row-level precision.
- **Embedding freshness:** Re-embed when manual overrides change the consolidated metadata. Trigger via `MetadataAggregatedEvent`. Don't re-embed unchanged chunks.

---

**Document type:** Technical intelligence reference
**Stage:** Pre-build design
**Audience:** Engineering, founding team

---

**Docs:** [What is IQ BENE?](what-is-vip.md) · [Business Overview](business-overview.md) · [Competitive Landscape](intelligence-and-competitive-landscape.md) · [Architecture](architecture.md)
