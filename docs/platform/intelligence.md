# Shortlisty — Intelligence Layer & ETL Pipeline

> **Audience:** Engineers, architects.
> **Purpose:** Technical reference for the document intelligence ETL pipeline, the proprietary venue-specific extraction schema, the multi-source aggregation model, and the vertical-agnostic extension strategy.

---

## 1. ETL Pipeline Architecture — Proven Foundation

### 1.1 Spring AI's ETL Pipeline

Spring AI ships a first-class, production-grade ETL pipeline with three composable stages:

```
DocumentReader  →  DocumentTransformer  →  DocumentWriter
   (Extract)           (Transform)            (Load)
```

**DocumentReaders (Extract) — available out of the box:**

| Reader                       | Handles                                         | Notes                                                       |
| ---------------------------- | ----------------------------------------------- | ----------------------------------------------------------- |
| `TikaDocumentReader`         | PDF, DOCX, XLSX, PPTX, HTML, XML, 1000+ formats | Apache Tika under the hood. **Primary reader for Shortlisty.** |
| `PagePdfDocumentReader`      | PDFs, page-by-page                              | Preserves page boundaries, useful for floor plans           |
| `ParagraphPdfDocumentReader` | PDFs, paragraph-level                           | Better semantic chunking for venue decks                    |
| `MarkdownDocumentReader`     | Markdown files                                  | Useful for structured venue specs                           |
| `JsonMetadataReader`         | JSON with metadata                              | Useful for structured imports                               |
| `JsoupDocumentReader`        | HTML pages                                      | Web scraping venue information                              |

**DocumentTransformers (Transform):**

| Transformer                | What it does                                                                                                                                                                                                                                          |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TokenTextSplitter`        | Splits large documents into chunks respecting token limits                                                                                                                                                                                            |
| `ContentFormatTransformer` | Normalizes text format                                                                                                                                                                                                                                |
| `SummaryMetadataEnricher`  | Generates document summary using LLM, stored as metadata                                                                                                                                                                                              |
| `KeywordMetadataEnricher`  | Extracts keywords using LLM, stored as metadata                                                                                                                                                                                                       |
| `VenueMetadataEnricher`    | **Venue-domain-specific** (`mi-venue-model`): extracts capacity, amenities, contacts via structured GPT-4o call against the venue canonical field set. The only non-generic component in the pipeline — everything else is reusable across verticals. |

**DocumentWriters (Load):**

| Writer               | What it does                                      |
| -------------------- | ------------------------------------------------- |
| `PgVectorStore`      | Writes chunks + embeddings to PostgreSQL pgvector |
| `SimpleVectorStore`  | In-memory (testing/dev)                           |
| `FileDocumentWriter` | Write to files (useful for debugging pipeline)    |

### 1.2 Shortlisty's Document Processing Pipeline

```
                     S3 Asset Storage
                          │
                          │ presigned URL download
                          ▼
               ┌─────────────────────┐
               │  DocumentReader     │  Spring AI / Apache Tika          [generic — mi-data-intelligence]
               │  (per asset type)   │  + IBM Docling (PDF tables, Ph.2)
               └──────────┬──────────┘
                          │  List<Document>
                          │  (raw text chunks + page metadata)
                          ▼
               ┌─────────────────────┐
               │  DocumentSplitter   │  TokenTextSplitter                [generic — mi-data-intelligence]
               │                     │  (512 tokens, 50 overlap)
               └──────────┬──────────┘
                          │  List<Document> (chunks)
                          ▼
               ┌─────────────────────┐
               │  VenueMetadata      │  GPT-4o structured output         [venue-specific — mi-venue-model]
               │  Enricher           │  → capacity, amenities, contacts
               └──────────┬──────────┘
                          │  List<Document> + venue metadata
                          ▼
               ┌─────────────────────┐
               │  EmbeddingModel     │  text-embedding-3-small           [generic — mi-data-intelligence]
               │                     │  (1536 dimensions per chunk)
               └──────────┬──────────┘
                          │  List<Document> + float[] embeddings
                          ▼
               ┌─────────────────────┐
               │  TenantAware        │  PostgreSQL + pgvector            [generic — mi-data-intelligence]
               │  PgVectorStore      │  → item_vectors (per-tenant schema)
               └──────────┬──────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │  MetadataAggregator │  Event-sourced consolidation      [venue-specific — mi-venue-model]
               │                     │  (conflict resolution)
               └─────────────────────┘
```

**Java implementation sketch** — the orchestrator is generic; venue-specific behaviour is injected via strategies (see §3 Extension Model):

```java
// mi-venue-processing-worker — Spring wiring only, no domain logic here
@Service
@RequiredArgsConstructor
public class AssetExtractionOrchestrator<M> {

  // ── generic contracts from mi-data-intelligence ──────────────────────
  private final TikaDocumentReader.Factory tikaFactory;
  private final TokenTextSplitter splitter;
  private final EmbeddingModel embeddingModel;
  private final VectorStore vectorStore;              // writes to item_vectors

  // ── domain strategies injected from mi-venue-model ───────────────────
  private final MetadataExtractionStrategy<M> extractionStrategy;   // VenueMetadataExtractionStrategy
  private final MetadataAggregationStrategy<M> aggregationStrategy; // VenueMetadataAggregationStrategy
  private final MetadataMigrator migrator;                           // VenueMetadataMigrator
  private final CuratedListMatchStrategy matchStrategy;              // MasterVenueMatchStrategy

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

### 1.3 Asset-Type Processing Matrix

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

### 1.4 Chunking Strategy

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

## 2. The Intelligence Layer Shortlisty Owns

Everything above (Tika, Docling, Spring AI ETL) is infrastructure. Shortlisty's proprietary intelligence sits on top:

### 2.1 Venue-Specific Extraction Schema

Generic document intelligence tools extract generic fields. Shortlisty extracts fields that matter for event professionals.

This schema is the **venue canonical field set** — defined as `VenueMetadata` in `mi-venue-model` (see §2 of [Architecture](README.md)). It is the venue-domain's answer to the question "what does a structured document look like for this vertical?". The extraction prompt sent to GPT-4o is derived directly from this schema. If the platform pivots to a different vertical (medical, agro), the domain library is swapped — the extraction pipeline, embedding, and search infrastructure remain identical.

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

Documents arrive for the same item in multiple formats — a marketing deck, a floor plan PDF, a technical spec sheet, a photo set. Each source may have conflicting or complementary data.

The aggregation engine (`MetadataAggregationConsumer` in `mi-data-intelligence`) is generic — it does not know about venues or capacity fields. It operates on `JsonNode` + `metadata_sources` provenance entries and delegates conflict decisions to the domain's `MetadataAggregationStrategy`:

1. Collects all extraction events per item (event log)
2. Delegates priority resolution to `MetadataAggregationStrategy.aggregate()` — venue impl applies: `manual_override > verified > high_confidence_AI > low_confidence_AI`
3. For array fields: set-union with confidence weighting (strategy-defined)
4. `ConflictReport` surfaced in the UI: "AI found two different capacity values — which is correct?"
5. One-click resolution writes a `MANUAL_OVERRIDE` event, re-triggers aggregation

This is a genuine product moat. No other platform in the event space does this. And the engine itself is reusable across verticals — only the priority rules and field semantics change per domain.

---

## 3. Extension Model — Generic Core, Domain Strategies

The platform separates **infrastructure contracts** (reusable across any document-intelligence vertical) from **domain strategies** (venue-specific, swapped per vertical). This is the mechanism that makes a pivot — from venues to medical records, agro assets, legal documents, or any other domain — a library swap rather than a rewrite.

### 3.1 Contracts defined in `mi-data-intelligence`

```java
// ── Extraction ────────────────────────────────────────────────────────────

/**
 * Extracts structured domain metadata from a set of document chunks.
 * Implement once per vertical. Supplies the LLM prompt and output type.
 *
 * @param <M> the domain metadata type (e.g. VenueMetadata, CaseMetadata)
 */
public interface MetadataExtractionStrategy<M> {
    /**
     * Run structured extraction against the given chunks.
     * @param chunks   tokenised, tagged document chunks from the splitter
     * @param context  asset-level context (item id, asset type, tenant)
     * @return extracted domain metadata with per-field confidence scores
     */
    M extract(List<Document> chunks, ExtractionContext context);

    /** The concrete metadata class this strategy produces. Used for type-safe deserialization. */
    Class<M> getMetadataType();
}

// ── Aggregation ───────────────────────────────────────────────────────────

/**
 * Merges a new extraction result into the current consolidated metadata.
 * Defines conflict-resolution priority rules for the domain.
 *
 * @param <M> the domain metadata type
 */
public interface MetadataAggregationStrategy<M> {
    /**
     * Merge incoming extraction into current state.
     * Called inside a single DB transaction by MetadataAggregationConsumer.
     */
    M aggregate(M current, M incoming, AggregationContext context);

    /**
     * Identify fields where current and incoming values disagree above threshold.
     * Returned report is persisted and surfaced in the UI for human resolution.
     */
    ConflictReport detectConflicts(M current, M incoming);
}

// ── Schema migration ──────────────────────────────────────────────────────

/**
 * Migrates a raw JSONB metadata document from any historical version to current.
 * Implement once per domain. Register in the domain library; runner is generic.
 */
public interface MetadataMigrator {
    /** Upgrade raw node to current schema version. Safe to call on every read. */
    JsonNode migrateToCurrent(JsonNode raw);

    /** Upgrade + stamp _schema_version = CURRENT. Call before every DB write. */
    JsonNode ensureCurrent(JsonNode node);

    int getCurrentVersion();
}

// ── Curated list matching ─────────────────────────────────────────────────

/**
 * Finds candidates in the platform curated list and copies gap-fill fields
 * into the tenant item record. Implements the trigram + proximity algorithm;
 * the source table and field-copy rules are domain-specific.
 */
public interface CuratedListMatchStrategy {
    /**
     * Query the domain's curated list for candidate matches.
     * @param name      normalised item name
     * @param location  geo point (may be null — triggers name-only path)
     * @param limit     max candidates to return
     */
    List<MatchCandidate> findCandidates(String name, GeoPoint location, int limit);

    /**
     * Copy fields from the matched curated entry into the tenant item metadata.
     * Only copies leaf fields that are null/empty on the tenant side.
     * Sets metadata_sources[field].source = MC_INHERIT.
     */
    CopyResult copyFields(JsonNode tenantMetadata, JsonNode curatedMetadata, double confidence);
}

// ── Search ────────────────────────────────────────────────────────────────

/**
 * Executes one branch of a parallel search query (tenant items or curated list).
 * Two implementations are wired per vertical; results are merged by RRF.
 *
 * @param <R> the result summary type (e.g. VenueSummaryView)
 */
public interface SearchBranchExecutor<R> {
    List<ScoredResult<R>> execute(SearchQuery query);

    /** Branch label used in metrics (shortlisty_search_latency_seconds{branch=...}). */
    String branchName();
}
```

### 3.2 Generic consumers in `mi-data-intelligence`

These classes contain no domain knowledge. They are final implementations wired with domain strategies via Spring DI:

```java
// Orchestrates the full ETL pipeline for one asset.
// domain strategies are injected — see §3.3 for venue wiring.
@Service
public final class AssetExtractionOrchestrator<M> { ... }

// Listens on RabbitMQ, runs SELECT → debounce → aggregate → UPDATE per venue.
// Concurrency=1 / prefetchCount=1 per slot queue (see README.md §3).
@RabbitListener
public final class MetadataAggregationConsumer<M> {
    private final MetadataAggregationStrategy<M> strategy;
    private final MetadataMigrator migrator;
    // SELECT venues.metadata → strategy.aggregate() → migrator.ensureCurrent() → UPDATE
}

// Parallel search orchestrator. Runs branchA + branchB via CompletableFuture,
// merges with Reciprocal Rank Fusion, appends origin="TENANT"|"MASTER_CATALOG".
@Service
public final class SearchOrchestrator<R> {
    private final SearchBranchExecutor<R> tenantBranch;
    private final SearchBranchExecutor<R> curatedBranch;
}
```

### 3.3 Venue implementations in `mi-venue-model`

```java
// Extraction: GPT-4o structured call against venue canonical field set (§2.1)
@Component
public class VenueMetadataExtractionStrategy
        implements MetadataExtractionStrategy<VenueMetadata> { ... }

// Aggregation priority: MANUAL_OVERRIDE(10) > VERIFIED(9) > HIGH_CONF_AI(8) > MC_INHERIT(7) > MEDIUM_CONF_AI(6) > LOW_CONF_AI(5) > SCRAPE_PROVIDER(4)
@Component
public class VenueMetadataAggregationStrategy
        implements MetadataAggregationStrategy<VenueMetadata> { ... }

// Migration chain: VenueMetadataMigrationV0ToV1, V1ToV2, …
@Component
public class VenueMetadataMigrator implements MetadataMigrator { ... }

// Curated list: trigram GIN on master_venue_alias + PostGIS ST_DWithin 200m
@Component
public class MasterVenueMatchStrategy implements CuratedListMatchStrategy { ... }

// Search branch A: tenant venues, full 5-mode hybrid (keyword + semantic + structured + geo + RRF)
@Component
public class TenantVenueSearchBranch implements SearchBranchExecutor<VenueSummaryView> { ... }

// Search branch B: public.master_venue, 3-mode MVP (keyword + structured + geo)
@Component
public class MasterCatalogSearchBranch implements SearchBranchExecutor<VenueSummaryView> { ... }
```

### 3.4 Dependency and flow

```
mi-data-intelligence
  ├── interfaces:  MetadataExtractionStrategy<M>
  │                MetadataAggregationStrategy<M>
  │                MetadataMigrator
  │                CuratedListMatchStrategy
  │                SearchBranchExecutor<R>
  │
  └── consumers:   AssetExtractionOrchestrator<M>   ← wires all 4 strategies
                   MetadataAggregationConsumer<M>   ← wires strategy + migrator
                   SearchOrchestrator<R>            ← wires 2 branch executors
                          │
                          ▼ (compile dependency)
        mi-venue-model
          ├── VenueMetadataExtractionStrategy   implements MetadataExtractionStrategy<VenueMetadata>
          ├── VenueMetadataAggregationStrategy  implements MetadataAggregationStrategy<VenueMetadata>
          ├── VenueMetadataMigrator             implements MetadataMigrator
          ├── MasterVenueMatchStrategy        implements CuratedListMatchStrategy
          ├── TenantVenueSearchBranch           implements SearchBranchExecutor<VenueSummaryView>
          └── MasterCatalogSearchBranch             implements SearchBranchExecutor<VenueSummaryView>
                          │
                          ▼ (compile dependency)
        mi-venue-service / mi-venue-processing-worker
          └── @Bean registrations wire venue strategies into generic consumers


  ── vertical extension example ──────────────────────────────────────────────

        mi-data-intelligence          (unchanged)
                 │
                 ▼
        mi-med-model
          ├── MedCaseExtractionStrategy   implements MetadataExtractionStrategy<CaseMetadata>
          ├── MedCaseAggregationStrategy  implements MetadataAggregationStrategy<CaseMetadata>
          ├── MedCaseMigrator             implements MetadataMigrator
          ├── DrugCompendiumMatchStrategy implements CuratedListMatchStrategy
          └── …
                 │
                 ▼
        mi-med-service / mi-med-processing-worker
```

**Rule:** if a class in `mi-venue-processing-worker` or `mi-venue-service` contains the word `venue` in its business logic (not just in a tag string), ask whether it belongs in `mi-venue-model` instead. The worker and service should contain Spring wiring, `@Bean` registrations, and `@RabbitListener` configuration — not domain decisions.

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

At the $0.001/venue cost of GPT-4o extraction + embedding generation, Shortlisty can process 1 million venues for approximately $1,000 in AI costs. This is not a cost problem.

### Vector Search Scaling

pgvector with IVFFlat index:

- Sub-10ms semantic search at 1M venues
- Scales to ~5M vectors per instance before needing optimization (HNSW index or Pgvector cloud)
- Multi-tenant isolation via schema-per-tenant — no cross-tenant data leakage

---

## 5. Technology Decisions Summary

| Layer                   | Choice                                                                                                                                                                                                                                 | Rationale                                                                                             |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Document parsing**    | Apache Tika (via Spring AI `TikaDocumentReader`)                                                                                                                                                                                       | 1000+ formats, DWG support, fault-tolerant Pipes, built into Spring AI                                |
| **PDF layout analysis** | IBM Docling (self-hosted, open source)                                                                                                                                                                                                 | State-of-the-art table/layout extraction, MIT license, no per-page cost                               |
| **AI framework**        | Spring AI 1.0                                                                                                                                                                                                                          | Java-native, provider-agnostic, ETL pipeline built-in, Micrometer metrics                             |
| **LLM extraction**      | OpenAI GPT-4o                                                                                                                                                                                                                          | Best structured output, multimodal (vision for images/floor plans)                                    |
| **Embeddings**          | OpenAI text-embedding-3-small                                                                                                                                                                                                          | 1536 dims, $0.02/1M tokens, excellent quality/cost ratio                                              |
| **Vector store**        | pgvector (PostgreSQL extension)                                                                                                                                                                                                        | No extra service, transactional, tenant-isolated, production-ready                                    |
| **Full-text search**    | PostgreSQL tsvector                                                                                                                                                                                                                    | Unified with relational data, no extra service                                                        |
| **Geo search**          | PostGIS (PostgreSQL extension)                                                                                                                                                                                                         | Mature, no extra service                                                                              |
| **Async processing**    | RabbitMQ (existing foundation)                                                                                                                                                                                                         | Already in platform, priority queues, DLQ                                                             |
| **File storage**        | S3 / MinIO (existing foundation)                                                                                                                                                                                                       | Already in IAM service, same pattern                                                                  |
| **Vertical isolation**  | Strategy pattern — `MetadataExtractionStrategy`, `MetadataAggregationStrategy`, `MetadataMigrator`, `CuratedListMatchStrategy`, `SearchBranchExecutor` interfaces in `mi-data-intelligence`; venue implementations in `mi-venue-model` | Pivot to new domain = new domain library + `@Bean` wiring. Zero changes to generic consumers. See §3. |

**Principle:** Use proven infrastructure that already exists in the iQ Key Value foundation. Introduce the minimum number of new services. The only truly new infrastructure is pgvector (a PostgreSQL extension, not a new service) and optionally a self-hosted Docling container for advanced PDF parsing.

---

## 6. Open Questions for Implementation

- [x] **Docling vs. pure Tika:** ~~Start with Tika for MVP speed. Add Docling for Phase 2 when floor plan fidelity matters.~~ **Decided:** Tika-only for Phase 1 (MVP). Docling sidecar added in Phase 2 for floor plan and table fidelity. See §15 of [Architecture](README.md).
- [x] **OCR strategy:** ~~Tika bundles Tesseract for basic OCR. GPT-4o vision handles complex cases.~~ **Decided:** If Tika OCR confidence < 0.7 on a scanned PDF, escalate to GPT-4o vision. Threshold is fixed, not configurable in Phase 1.
- [x] **Chunking overlap for capacity tables:** ~~50-token overlap is standard.~~ **Decided:** Venue-specific content (capacity tables, spec sheets) uses 256-token chunks with 30-token overlap. General venue deck text uses 512 tokens with 50-token overlap. Configured per `asset_type` in `VenueMetadataExtractionStrategy`.
- **CAD files:** Tika extracts metadata from DWG/DXF (dimensions, layers). For Phase 1, expose raw metadata. Phase 2: convert to PNG via LibreCAD/ODA, then GPT-4o vision for layout understanding.
- **Video walkthroughs:** Out of scope for Phase 1. Phase 2: extract keyframes (ffmpeg), run GPT-4o vision on representative frames.
- **Photo category inference:** When `asset_type = PHOTO` and `photo_category` is not provided by the user, the worker should infer it from GPT-4o vision output (`EXTERIOR`, `INTERIOR`, `SETUP`, etc.). Confidence threshold for auto-assignment TBD — below threshold defaults to `OTHER`.
- **Embedding freshness:** Re-embed when manual overrides change the consolidated metadata. Trigger via `MetadataAggregatedEvent`. Don't re-embed unchanged chunks.
- **`google_place_id` enrichment pipeline:** When `social.google_place_id` is set (via MC_INHERIT or manual entry), a background job can pull structured data from Google Places API — address, photos, rating, hours — and apply as `SCRAPE_PROVIDER` provenance events. Priority 4, overridden by any AI or manual source. Design TBD.

---

**Docs:** [What is Shortlisty?](../README.md) · [Business Proposal](../business/digital-sales-room-for-events/proposal.md) · [Competitive Landscape](../business/digital-sales-room-for-events/comparison.md) · [Architecture](README.md) · [Data Model](data-model.md) · [Vision](../roadmap/vision.md)
