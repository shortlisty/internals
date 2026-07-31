# Venue Intelligence Platform — Intelligence Layer & ETL Pipeline

> Technical reference for the document intelligence, ETL pipeline, and the proprietary
> intelligence layer of BENE.

**Docs:** [What is BENE?](../README.md) · [Business Proposal](../business/proposal.md) · [Competitive Landscape](../business/comparison.md) · [Intelligence Layer](intelligence.md) · [Architecture](architecture.md)

---

## 1. ETL Pipeline Architecture — Proven Foundation

### 1.1 Spring AI's ETL Pipeline

Spring AI ships a first-class, production-grade ETL pipeline with three composable stages:

```
DocumentReader  →  DocumentTransformer  →  DocumentWriter
   (Extract)           (Transform)            (Load)
```

**DocumentReaders (Extract) — available out of the box:**

| Reader                       | Handles                                         | Notes                                                    |
| ---------------------------- | ----------------------------------------------- | -------------------------------------------------------- |
| `TikaDocumentReader`         | PDF, DOCX, XLSX, PPTX, HTML, XML, 1000+ formats | Apache Tika under the hood. **Primary reader for BENE.** |
| `PagePdfDocumentReader`      | PDFs, page-by-page                              | Preserves page boundaries, useful for floor plans        |
| `ParagraphPdfDocumentReader` | PDFs, paragraph-level                           | Better semantic chunking for venue decks                 |
| `MarkdownDocumentReader`     | Markdown files                                  | Useful for structured venue specs                        |
| `JsonMetadataReader`         | JSON with metadata                              | Useful for structured imports                            |
| `JsoupDocumentReader`        | HTML pages                                      | Web scraping venue information                           |

**DocumentTransformers (Transform):**

| Transformer                    | What it does                                                                      |
| ------------------------------ | --------------------------------------------------------------------------------- |
| `TokenTextSplitter`            | Splits large documents into chunks respecting token limits                        |
| `ContentFormatTransformer`     | Normalizes text format                                                            |
| `SummaryMetadataEnricher`      | Generates document summary using LLM, stored as metadata                          |
| `KeywordMetadataEnricher`      | Extracts keywords using LLM, stored as metadata                                   |
| Custom `VenueMetadataEnricher` | **BENE-specific:** extracts capacity, amenities, contacts via structured LLM call |

**DocumentWriters (Load):**

| Writer               | What it does                                      |
| -------------------- | ------------------------------------------------- |
| `PgVectorStore`      | Writes chunks + embeddings to PostgreSQL pgvector |
| `SimpleVectorStore`  | In-memory (testing/dev)                           |
| `FileDocumentWriter` | Write to files (useful for debugging pipeline)    |

### 1.2 BENE's Document Processing Pipeline

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

### 1.3 Asset-Type Processing Matrix

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

### 1.4 Chunking Strategy

Document chunking significantly impacts retrieval quality. BENE uses a hybrid strategy:

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

### 1.5 Why Apache Tika is the Right Foundation

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

### 1.6 Docling Integration for High-Fidelity PDF Parsing

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

## 2. The Intelligence Layer BENE Owns

Everything above (Tika, Docling, Spring AI ETL) is infrastructure. BENE's proprietary intelligence sits on top:

### 2.1 Venue-Specific Extraction Schema

Generic document intelligence tools extract generic fields. BENE extracts fields that matter for event professionals:

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

This schema is what makes BENE a _venue intelligence platform_, not just a document storage system. Every competitor either has operational data (bookings, invoicing) or generic extraction. No one has this schema purpose-built for event planners.

### 2.2 Confidence-Sourced Metadata Model

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

### 2.3 Multi-Source Aggregation (The Hard Problem Nobody Solves)

Venues send the same venue in multiple formats — a marketing deck, a floor plan PDF, a technical spec sheet, a photo set. Each source may have conflicting or complementary data.

BENE's aggregation engine:

1. Collects all extraction events per venue (event log)
2. Applies priority rules: `manual_override > verified > high_confidence_AI > low_confidence_AI`
3. For arrays (amenities, restrictions): set-union with confidence weighting
4. Surfaces conflicts in the UI: "AI found two different capacity values — which is correct?"
5. Allows one-click resolution

This is a genuine product moat. No other platform in the event space does this.

---

## 3. Scalability Architecture

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

At the $0.001/venue cost of GPT-4o extraction + embedding generation, BENE can process 1 million venues for approximately $1,000 in AI costs. This is not a cost problem.

### Vector Search Scaling

pgvector with IVFFlat index:

- Sub-10ms semantic search at 1M venues
- Scales to ~5M vectors per instance before needing optimization (HNSW index or Pgvector cloud)
- Multi-tenant isolation via schema-per-tenant — no cross-tenant data leakage

---

## 4. Technology Decisions Summary

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

**Principle:** Use proven infrastructure that already exists in the IQ Key Value foundation. Introduce the minimum number of new services. The only truly new infrastructure is pgvector (a PostgreSQL extension, not a new service) and optionally a self-hosted Docling container for advanced PDF parsing.

---

## 5. Open Questions for Implementation

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

**Docs:** [What is BENE?](../README.md) · [Business Proposal](../business/proposal.md) · [Competitive Landscape](../business/comparison.md) · [Intelligence Layer](intelligence.md) · [Architecture](architecture.md)
