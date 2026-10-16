# Shortlisty — Intelligence Layer & ETL Pipeline

> **Audience:** Engineers, architects.
> **Purpose:** Technical reference for the document intelligence ETL pipeline, the proprietary venue-specific extraction schema, and the multi-source aggregation model.

---

## 1. ETL pipeline architecture — proven foundation

### 1.1 Spring AI's ETL pipeline

Spring AI ships a first-class, production-grade ETL pipeline with three composable stages:

```
DocumentReader  →  DocumentTransformer  →  DocumentWriter
   (Extract)           (Transform)            (Load)
```

**DocumentReaders (Extract) — available out of the box:**

| Reader                       | Handles                                         | Notes                                                          |
| ---------------------------- | ----------------------------------------------- | -------------------------------------------------------------- |
| `TikaDocumentReader`         | PDF, DOCX, XLSX, PPTX, HTML, XML, 1000+ formats | Apache Tika under the hood. **Primary reader for Shortlisty.** |
| `PagePdfDocumentReader`      | PDFs, page-by-page                              | Preserves page boundaries, useful for floor plans              |
| `ParagraphPdfDocumentReader` | PDFs, paragraph-level                           | Better semantic chunking for venue decks                       |
| `MarkdownDocumentReader`     | Markdown files                                  | Useful for structured venue specs                              |
| `JsonMetadataReader`         | JSON with metadata                              | Useful for structured imports                                  |
| `JsoupDocumentReader`        | HTML pages                                      | Web scraping venue information                                 |

**DocumentTransformers (Transform):**

| Transformer                | What it does                                                                                                                                                                                                                                                                                                                                                  |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TokenTextSplitter`        | Splits large documents into chunks respecting token limits                                                                                                                                                                                                                                                                                                    |
| `ContentFormatTransformer` | Normalizes text format                                                                                                                                                                                                                                                                                                                                        |
| `SummaryMetadataEnricher`  | Generates document summary using LLM, stored as metadata                                                                                                                                                                                                                                                                                                      |
| `KeywordMetadataEnricher`  | Extracts keywords using LLM, stored as metadata                                                                                                                                                                                                                                                                                                               |
| `VenueMetadataEnricher`    | **Venue-domain-specific**: extracts capacity, amenities, contacts via structured call against the venue canonical field set. Runs against a self-hosted OpenAI-compatible endpoint (Phi-4 / Qwen2.5 via Ollama or vLLM — see D12). Implemented inside `venueintelligence-catalog-processing-worker`. |

**DocumentWriters (Load):**

| Writer               | What it does                                      |
| -------------------- | ------------------------------------------------- |
| `PgVectorStore`      | Writes chunks + embeddings to PostgreSQL pgvector |
| `SimpleVectorStore`  | In-memory (testing/dev)                           |
| `FileDocumentWriter` | Write to files (useful for debugging pipeline)    |

### 1.2 Shortlisty's document processing pipeline

```
                     S3 Asset Storage
                          │
                          │ presigned URL download
                          ▼
               ┌─────────────────────┐
               │  DocumentReader     │  Spring AI / Apache Tika
               │  (per asset type)   │  + IBM Docling (PDF tables, Ph.2)
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
               │  VenueMetadata      │  Self-hosted LLM structured output
               │  Enricher           │  Phi-4 / Qwen2.5 via Ollama or vLLM
               │                     │  → capacity, amenities, contacts
               └──────────┬──────────┘
                          │  List<Document> + venue metadata
                          ▼
               ┌─────────────────────┐
               │  EmbeddingModel     │  BGE-M3 (self-hosted) or
               │                     │  text-embedding-3-small (fallback)
               └──────────┬──────────┘
                          │  List<Document> + float[] embeddings
                          ▼
               ┌─────────────────────┐
               │  TenantAware        │  PostgreSQL + pgvector
               │  PgVectorStore      │  → item_vectors (per-tenant schema)
               └──────────┬──────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │  MetadataAggregator │  Event-sourced consolidation
               │                     │  (conflict resolution)
               └─────────────────────┘
```

**Java implementation sketch** — every type below lives inside `venueintelligence-catalog-processing-worker`. No pipeline type is shared with another service:

```java
package com.iqkv.venueintelligence.catalog.processing;

// venueintelligence-catalog-processing-worker — Spring wiring only, no domain logic here
@Service
public class AssetExtractionOrchestrator<M> {

  // ── pipeline infrastructure ──────────────────────────────────────────────
  private final TikaDocumentReader.Factory tikaFactory;
  private final TokenTextSplitter splitter;
  private final EmbeddingModel embeddingModel;
  private final VectorStore vectorStore;              // writes to item_vectors

  // ── venue strategies ─────────────────────────────────────────────────────
  private final MetadataExtractionStrategy<M> extractionStrategy;   // VenueMetadataExtractionStrategy
  private final MetadataAggregationStrategy<M> aggregationStrategy; // VenueMetadataAggregationStrategy
  private final MetadataMigrator migrator;                           // VenueMetadataMigrator
  private final CuratedListMatchStrategy matchStrategy;              // MasterVenueMatchStrategy

  public AssetExtractionOrchestrator(
      TikaDocumentReader.Factory tikaFactory,
      TokenTextSplitter splitter,
      EmbeddingModel embeddingModel,
      VectorStore vectorStore,
      MetadataExtractionStrategy<M> extractionStrategy,
      MetadataAggregationStrategy<M> aggregationStrategy,
      MetadataMigrator migrator,
      CuratedListMatchStrategy matchStrategy) {
    this.tikaFactory = tikaFactory;
    this.splitter = splitter;
    this.embeddingModel = embeddingModel;
    this.vectorStore = vectorStore;
    this.extractionStrategy = extractionStrategy;
    this.aggregationStrategy = aggregationStrategy;
    this.migrator = migrator;
    this.matchStrategy = matchStrategy;
  }

  public void process(ItemAsset asset, byte[] content) {
    // 1. Parse — Tika handles PDF, DOCX, XLSX, images via OCR, DWG
    var rawDocs = tikaFactory
        .create(new ByteArrayResource(content), asset.getContentType())
        .get();

    // 2. Tag with source context
    var taggedDocs = rawDocs.stream()
        .map(doc -> doc.mutate()
            .metadata("item_id",   asset.getItemId().toString())
            .metadata("asset_id",  asset.getId().toString())
            .metadata("asset_type", asset.getType().name())
            .metadata("tenant_id", TenantContext.getCurrentTenant())
            .build())
        .toList();

    // 3. Split into semantic chunks
    var chunks = splitter.apply(taggedDocs);

    // 4. Domain-specific: extract structured metadata (strategy supplies prompt + output type)
    //    Runs against self-hosted OpenAI-compatible endpoint (Phi-4 / Qwen2.5 — see D12)
    var result = extractionStrategy.extract(chunks, ExtractionContext.of(asset));

    // 5. Embed + write to item_vectors
    vectorStore.add(chunks);

    // 6. Domain-specific: aggregate into item profile via strategy + migrator
    aggregationStrategy.applyExtractionResult(asset.getItemId(), result, migrator);

    // 7. Domain-specific: gap-fill from curated list
    matchStrategy.matchAndCopy(asset.getItemId(), result);
  }
}
```

### 1.3 Asset-type processing matrix

| Asset Type            | Asset Type Enum | Parser                            | OCR Needed  | Structured Extraction     | `table_data` | Vector Indexed    |
| --------------------- | --------------- | --------------------------------- | ----------- | ------------------------- | ------------ | ----------------- |
| PDF Deck (text)       | `PDF_DECK`      | Tika → ParagraphPdfDocumentReader | No          | Yes (GPT-4o)              | No           | Yes               |
| PDF Deck (scanned)    | `PDF_DECK`      | Tika + OCR (Tesseract)            | Yes         | Yes (GPT-4o)              | No           | Yes               |
| Floor Plan (PDF)      | `FLOOR_PLAN`    | Docling (layout-aware, Phase 2)   | No          | Yes (GPT-4o vision)       | No           | Yes               |
| Floor Plan (image)    | `FLOOR_PLAN`    | GPT-4o vision direct              | Yes (LLM)   | Yes                       | No           | Yes               |
| Photos                | `PHOTO`         | GPT-4o vision                     | No          | Amenity + category detect | No           | Yes               |
| Spec Sheet            | `SPEC_SHEET`    | TikaDocumentReader                | No          | Yes (GPT-4o)              | No           | Yes               |
| Menu                  | `MENU`          | TikaDocumentReader                | No          | Catering fields only      | No           | Yes               |
| Price List (PDF)      | `PRICE_LIST`    | TikaDocumentReader                | No          | Pricing fields only       | No           | Yes               |
| Price List (CSV/XLSX) | `PRICE_LIST`    | Tika → structured rows            | No          | Pricing fields only       | **Yes**      | Metadata only     |
| Data Table (CSV/XLSX) | `DATA_TABLE`    | Tika → structured rows            | No          | No                        | **Yes**      | Metadata only     |
| CAD File              | `CAD_FILE`      | Tika AutoCAD parser               | No          | Metadata only             | No           | Metadata only     |
| Video                 | `VIDEO`         | Extract thumbnail + audio         | Via Whisper | Partial (Phase 2)         | No           | Partial (Phase 2) |
| Other                 | `MISC`          | Tika best-effort                  | If needed   | Best-effort text only     | No           | Yes               |

`table_data` is populated synchronously in the worker before text extraction is queued — it is a fast CSV/XLSX parse, not an AI call. See [data-model.md §2c](data-model.md) for the `table_data` JSONB schema and size constraints (max 500 rows × 20 columns).

### 1.4 Chunking strategy

Document chunking significantly impacts retrieval quality. Shortlisty uses a hybrid strategy:

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

### 1.5 Why Apache Tika is the right foundation

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

### 1.6 Docling integration for high-fidelity PDF parsing

For venue decks with complex layouts, tables, and multi-column structures, Docling outperforms basic Tika PDF parsing:

**When to use Docling over Tika for PDFs:**

- Multi-column layouts (most venue decks)
- Capacity tables (banquet vs. theater vs. classroom setups)
- Mixed text + diagram pages (floor plans embedded in PDFs)
- Scanned PDFs requiring layout-aware OCR

**Integration approach:**

```java
package com.iqkv.venueintelligence.catalog.processing;

// Spring AI custom DocumentReader wrapping Docling HTTP API
@Component
public class DoclingDocumentReader implements DocumentReader {

  private final DoclingClient doclingClient; // REST client to Docling service

  public DoclingDocumentReader(DoclingClient doclingClient) {
    this.doclingClient = doclingClient;
  }

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

## 2. The intelligence layer Shortlisty owns

Everything above (Tika, Docling, Spring AI ETL) is infrastructure. Shortlisty's proprietary intelligence sits on top:

### 2.1 Venue-specific extraction schema

Generic document intelligence tools extract generic fields. Shortlisty extracts fields that matter for event professionals.

This schema is the **venue canonical field set** — defined as `VenueMetadata` in `venueintelligence-catalog-service` (see §2 of [Architecture](README.md)). It defines what a structured venue document looks like. The extraction prompt sent to the inference endpoint is derived directly from this schema.

> **Inference stack:** Extraction calls run against a self-hosted OpenAI-compatible endpoint — Phi-4 (Microsoft, 14B) or Qwen2.5 (Alibaba, 7B–72B) served via Ollama or vLLM on own VPS/server infrastructure. Spring AI's `ChatModel` abstraction makes the endpoint a configuration value, not a code dependency. See [D12](../roadmap/decisions/D12-self-hosted-inference-stack.md) for the full rationale.

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
      "vegan_available": false,
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
      "curfew_time": "23:00",
      "setup_hours_before": 3,
      "teardown_hours_after": 2
    },
    "outdoor_space": {
      "available": true,
      "covered": false,
      "sqm": 400
    },
    "exclusivity": {
      "exclusive_use": true,
      "shared_space": false
    },
    "restrictions": ["no_open_flame", "no_confetti", "no_outside_alcohol"],
    "contacts": [{ "name": "...", "role": "venue_sales", "email": "...", "phone": "..." }],
    "pricing": {
      "minimum_spend": 10000,
      "currency": "USD",
      "rental_fee_indicative": 5000
    },
    "social": {
      "instagram_handle": "grand_ballroom_nyc",
      "google_place_id": "ChIJN1t_tDeuEmsRUsoyG83frY4"
    },
    "ratings": {
      "google_rating": 4.7,
      "google_review_count": 312
    }
  }
}
```

This schema is what makes Shortlisty a _venue intelligence platform_, not just a document storage system. Every competitor either has operational data (bookings, invoicing) or generic extraction. No one has this schema purpose-built for event planners.

### 2.2 Confidence-sourced metadata model

Each field carries full provenance:

```json
"capacity.max_total": {
  "value": 500,
  "confidence": 0.94,
  "source_type": "PDF_DECK",
  "source_page": 4,
  "extraction_model": "phi-4",
  "alternatives": [
    { "value": 480, "confidence": 0.72, "source_type": "FLOOR_PLAN" }
  ],
  "overridden_by": null
}
```

No existing venue tool surfaces this level of data provenance. Users see not just the value but _why_ the system believes it.

### 2.3 Semantic layer and versioned event mappings

Raw venue documents use inconsistent field names across sources and across time. A marketing deck from 2023 may call it "maximum occupancy"; a 2026 spec sheet calls it "standing capacity." Both map to the same canonical field `capacity.max_total`. Shortlisty solves this with a **semantic layer**: a set of canonical business definitions that remain stable regardless of how raw source documents phrase or rename the underlying data.

**Canonical event contracts.** The venue extraction schema in §2.1 defines canonical field names that never change from the application's perspective. Every extraction call, every aggregation step, and every search query operates against these canonical names — not against the raw phrasing found in documents.

**Versioned mappings, not in-place updates.** When a source document uses a new phrasing or a new vendor introduces a different field naming convention, a versioned mapping is added — not an in-place mutation of the existing definition. This preserves historical accuracy: a query run against past extractions still returns the same result because the mapping at the time of extraction is recorded alongside the extracted value.

```json
"capacity.max_total": {
  "value": 500,
  "source_raw_field": "maximum_occupancy",   // what the document said
  "mapping_version": "v2",                   // which mapping resolved it
  "extraction_model": "phi-4",
  "confidence": 0.94
}
```

**Schema versioning as the enforcement mechanism.** The `_schema_version` field on every venue metadata record (see D7) is the runtime contract between the canonical definitions and the stored data. When a canonical field set evolves — a new sub-field is added, a field is split — the migration chain increments the version and upgrades existing records lazily on read, eagerly on write. No historical record is silently corrupted by a definition change.

**Why this matters for AI-generated insights.** An AI layer querying venue data — whether for search ranking, anomaly detection, or pattern extraction — can only produce deterministic, auditable results if the underlying field definitions are stable and versioned. Without this, a model trained or prompted on `capacity.max_total` today may silently return different results next year because a mapping changed without a version bump. The semantic layer and versioned mapping pattern guarantee that insight queries are reproducible.

### 2.4 Multi-source aggregation (the hard problem nobody solves)

Documents arrive for the same item in multiple formats — a marketing deck, a floor plan PDF, a technical spec sheet, a photo set. Each source may have conflicting or complementary data.

The aggregation engine (`MetadataAggregationConsumer` in `venueintelligence-catalog-service`) operates on `JsonNode` + `metadata_sources` provenance entries and delegates conflict decisions to `MetadataAggregationStrategy`:

1. Collects all extraction events per item (event log)
2. Delegates priority resolution to `MetadataAggregationStrategy.aggregate()` — venue impl applies: `manual_override > verified > high_confidence_AI > low_confidence_AI`
3. For array fields: set-union with confidence weighting (strategy-defined)
4. `ConflictReport` surfaced in the UI: "AI found two different capacity values — which is correct?"
5. One-click resolution writes a `MANUAL_OVERRIDE` event, re-triggers aggregation

This is a genuine product moat. No other platform in the event space does this.

---

## 4. Scalability architecture

### Event-driven, horizontally scalable

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

### Cost scaling

| Scale            | AI Cost | Infrastructure                         |
| ---------------- | ------- | -------------------------------------- |
| 100 venues (MVP) | ~$0.10  | Single container                       |
| 10K venues       | ~$10    | 2-3 containers                         |
| 1M venues        | ~$1,000 | Auto-scaled, still manageable          |
| 100M venues      | ~$100K  | Optimize with cheaper models + caching |

At the $0.001/venue cost of GPT-4o extraction + embedding generation, Shortlisty can process 1 million venues for approximately $1,000 in AI costs. This is not a cost problem.

### Vector search scaling

pgvector with IVFFlat index:

- Sub-10ms semantic search at 1M venues
- Scales to ~5M vectors per instance before needing optimization (HNSW index or Pgvector cloud)
- Multi-tenant isolation via schema-per-tenant — no cross-tenant data leakage

---

## 5. Technology decisions summary

| Layer                   | Choice                                                                                                                                                                                                                                               | Rationale                                                                                             |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Document parsing**    | Apache Tika (via Spring AI `TikaDocumentReader`)                                                                                                                                                                                                     | 1000+ formats, DWG support, fault-tolerant Pipes, built into Spring AI                                |
| **PDF layout analysis** | IBM Docling (self-hosted, open source)                                                                                                                                                                                                               | State-of-the-art table/layout extraction, MIT license, no per-page cost                               |
| **AI framework**        | Spring AI 1.0                                                                                                                                                                                                                                        | Java-native, provider-agnostic, ETL pipeline built-in, Micrometer metrics                             |
| **LLM extraction**      | OpenAI GPT-4o                                                                                                                                                                                                                                        | Best structured output, multimodal (vision for images/floor plans)                                    |
| **Embeddings**          | OpenAI text-embedding-3-small                                                                                                                                                                                                                        | 1536 dims, $0.02/1M tokens, excellent quality/cost ratio                                              |
| **Vector store**        | pgvector (PostgreSQL extension)                                                                                                                                                                                                                      | No extra service, transactional, tenant-isolated, production-ready                                    |
| **Full-text search**    | PostgreSQL tsvector                                                                                                                                                                                                                                  | Unified with relational data, no extra service                                                        |
| **Geo search**          | PostGIS (PostgreSQL extension)                                                                                                                                                                                                                       | Mature, no extra service                                                                              |
| **Async processing**    | RabbitMQ (existing foundation)                                                                                                                                                                                                                       | Already in platform, priority queues, DLQ                                                             |
| **File storage**        | S3 / MinIO (existing foundation)                                                                                                                                                                                                                     | Already in IAM service, same pattern                                                                  |

**Principle:** Use proven infrastructure that already exists in the iQ Key Value foundation. Introduce the minimum number of new services. The only truly new infrastructure is pgvector (a PostgreSQL extension, not a new service) and optionally a self-hosted Docling container for advanced PDF parsing.

---

## 6. Open questions for implementation

- [x] **Docling vs. pure Tika:** ~~Start with Tika for MVP speed. Add Docling for Phase 2 when floor plan fidelity matters.~~ **Decided:** Tika-only for Phase 1 (MVP). Docling sidecar added in Phase 2 for floor plan and table fidelity. See §15 of [Architecture](README.md).
- [x] **OCR strategy:** ~~Tika bundles Tesseract for basic OCR. GPT-4o vision handles complex cases.~~ **Decided:** If Tika OCR confidence < 0.7 on a scanned PDF, escalate to GPT-4o vision. Threshold is fixed, not configurable in Phase 1.
- [x] **Chunking overlap for capacity tables:** ~~50-token overlap is standard.~~ **Decided:** Venue-specific content (capacity tables, spec sheets) uses 256-token chunks with 30-token overlap. General venue deck text uses 512 tokens with 50-token overlap. Configured per `asset_type` in `VenueMetadataExtractionStrategy`.
- **CAD files:** Tika extracts metadata from DWG/DXF (dimensions, layers). For Phase 1, expose raw metadata. Phase 2: convert to PNG via LibreCAD/ODA, then GPT-4o vision for layout understanding.
- **Video walkthroughs:** Out of scope for Phase 1. Phase 2: extract keyframes (ffmpeg), run GPT-4o vision on representative frames.
- **Photo category inference:** When `asset_type = PHOTO` and `photo_category` is not provided by the user, the worker should infer it from GPT-4o vision output (`EXTERIOR`, `INTERIOR`, `SETUP`, etc.). Confidence threshold for auto-assignment TBD — below threshold defaults to `OTHER`.
- **Embedding freshness:** Re-embed when manual overrides change the consolidated metadata. Trigger via `MetadataAggregatedEvent`. Don't re-embed unchanged chunks.
- **`google_place_id` enrichment pipeline:** When `social.google_place_id` is set (via MC_INHERIT or manual entry), a background job can pull structured data from Google Places API — address, photos, rating, hours — and apply as `SCRAPE_PROVIDER` provenance events. Priority 4, overridden by any AI or manual source. Design TBD.

---

**Docs:** [Architecture Index](README.md) · [ETL Pipeline](etl-pipeline.md) · [Search](search.md) · [Data Model](data-model.md)
