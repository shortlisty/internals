# Shortlisty — ETL Pipeline

> **Audience:** Engineers.
> **Purpose:** The full async document processing pipeline inside `mi-venue-processing-worker` — parse, transform, load stages, asset type routing, and processing SLAs.

---

## Related Documents

- [services.md](services.md) — service responsibilities, table ownership, `mi-data-intelligence` library
- [data-model.md](data-model.md) — `ExtractionJob`, `VenueAsset`, `item_vectors` table definitions
- [aggregation.md](aggregation.md) — what happens after `ExtractionCompletedEvent` is published
- [master-catalog.md](master-catalog.md) — Stage 3 MC_INHERIT merge (MasterVenueMatcher)
- [events.md](events.md) — `asset.uploaded` trigger, `extraction.*` event contracts
- [observability.md](observability.md) — extraction metrics and latency histograms

---

## 5. ETL Pipeline (`mi-venue-processing-worker`)

Built on **Spring AI's ETL framework**. Three composable stages:

```
DocumentReader  →  DocumentTransformer  →  DocumentWriter
  (parse)            (chunk + enrich)        (embed + store)
```

The pipeline contracts — `ExtractionJob`, `ExtractionStatus`, `ExtractorType`, `AssetType`, event POJOs (`AssetUploadedEvent`, `ExtractionCompletedEvent`, `ExtractionFailedEvent`) — are defined in `mi-data-intelligence` (see [services.md § 4c](services.md)). The venue-specific enricher (`VenueMetadataEnricher`) and its output type (`VenueMetadata`) come from `mi-venue-model` (see [services.md § 4a](services.md)).

---

### Stage 1 — Parse (per asset type)

| Asset type                   | Asset type enum | Reader                                    | `table_data` | Notes                                   |
| ---------------------------- | --------------- | ----------------------------------------- | ------------ | --------------------------------------- |
| PDF (text-based)             | `PDF_DECK`      | `TikaDocumentReader`                      | No           | Apache Tika, ships with Spring AI       |
| PDF (scanned)                | `PDF_DECK`      | `TikaDocumentReader` + Tesseract OCR      | No           | Tika bundles OCR                        |
| PDF (complex layout, tables) | `PDF_DECK`      | Docling sidecar → custom `DocumentReader` | No           | Phase 2; better table fidelity          |
| Spec sheet (PDF/DOCX)        | `SPEC_SHEET`    | `TikaDocumentReader`                      | No           | Structured field extraction             |
| Menu (PDF/DOCX)              | `MENU`          | `TikaDocumentReader`                      | No           | Catering field enrichment only          |
| Price list (PDF)             | `PRICE_LIST`    | `TikaDocumentReader`                      | No           | Pricing field enrichment only           |
| Price list (CSV/XLSX)        | `PRICE_LIST`    | Tika → structured rows                    | **Yes**      | `table_data` populated before embedding |
| Data table (CSV/XLSX)        | `DATA_TABLE`    | Tika → structured rows                    | **Yes**      | `table_data` only — no text extraction  |
| DOCX / PPTX                  | `MISC`          | `TikaDocumentReader`                      | No           | Same reader, 1000+ formats              |
| Photos (JPG, PNG, WEBP)      | `PHOTO`         | GPT-4o vision direct                      | No           | Category inferred when not provided     |
| Floor plan (PDF/image)       | `FLOOR_PLAN`    | Docling layout-aware → GPT-4o vision      | No           | Phase 2                                 |
| DWG / DXF (CAD)              | `CAD_FILE`      | `TikaDocumentReader` (AutoCAD parser)     | No           | Extracts metadata; visual in Phase 2    |
| Video                        | `VIDEO`         | Out of scope Phase 1                      | No           | Phase 2: keyframe extraction via ffmpeg |
| Other                        | `MISC`          | `TikaDocumentReader` best-effort          | No           |                                         |

**`table_data` population** (CSV/XLSX path only): runs as a fast synchronous parse step inside `onAssetUploaded` before text extraction is queued. Uses Tika's structured row output — not an AI call. Result is written directly to `venue_assets.table_data` JSONB. Size constraint: max 500 rows × 20 columns (see [data-model.md §2c](data-model.md)); larger files skip `table_data` and set an `extraction_status = COMPLETED` + `size_exceeded` flag in `extracted_text`.

---

### Stage 2 — Transform

1. **Chunk** — `TokenTextSplitter` (512 tokens, 50-token overlap). Spec-sheet tables use 256-token chunks to preserve row precision.
2. **Tag** — attach `venue_id`, `asset_id`, `asset_type`, `tenant_id` as Document metadata.
3. **Extract** — `VenueMetadataEnricher` (custom `DocumentTransformer`, venue-specific, from `mi-venue-model`): calls GPT-4o with structured output schema matching the venue canonical field set (see [data-model.md § Canonical Field Set](data-model.md)), returns `VenueMetadata` POJO with confidence scores per field. The enricher is the **only venue-specific component** in the pipeline — all surrounding plumbing is generic.

---

### Stage 3 — Load

1. **Embed** — `EmbeddingModel` (`text-embedding-3-small`, 1536 dims). Generic — from `mi-data-intelligence`. Skipped for `DATA_TABLE` and `PRICE_LIST` (CSV/XLSX) — `table_data` is already populated; only metadata stored in `item_vectors`.
2. **Store** — `TenantAwarePgVectorStore` writes chunks + embeddings to `item_vectors` table in the tenant's schema. Table defined in `mi-data-intelligence` changelog (see [services.md § 4c](services.md)).
3. **Master catalog match** — `MasterVenueMatcher` runs the full MC_INHERIT merge algorithm (trigram name similarity via `pg_trgm` GIN index on `master_venue_alias`, PostGIS `ST_DWithin` 200m radius, combined confidence formula, thresholds 0.75 with geo / 0.90 name-only, ambiguity delta guard ≥ 0.08). No LLM calls. Full algorithm, thresholds, and field-copy semantics in [master-catalog.md](master-catalog.md).
4. **Aggregate** — publishes `ExtractionCompletedEvent` (from `mi-data-intelligence`) → `MetadataAggregationConsumer` in `mi-venue-service` updates `venues.metadata` via `VenueMetadataMigrator.ensureCurrent()` (from `mi-venue-model`) and recalculates `profile_stage` (`SEEDED → ENRICHED → CURATED → READY`) in the same DB transaction. Full aggregation design including `profile_stage` transition rules in [aggregation.md](aggregation.md).

---

### Processing SLA

| Asset type              | Target latency | Notes                        |
| ----------------------- | -------------- | ---------------------------- |
| PDF / DOCX (text)       | < 30s          |                              |
| Images / floor plans    | < 60s          |                              |
| CSV / XLSX (table data) | < 5s           | Fast parse only — no AI call |
| CAD files               | < 2 min        |                              |
| Video                   | Phase 2        |                              |

Retry on failure: 3 attempts with exponential backoff. After 3 failures → `ExtractionFailedEvent` (from `mi-data-intelligence`) → user notification.

---

### Spring Wiring — Worker Entry Point

```java
// mi-venue-processing-worker — Spring wiring only, no domain logic here
@Service
@RequiredArgsConstructor
public class AssetExtractionConsumer {

    private final DocumentReader tikaReader;
    private final TokenTextSplitter splitter;
    private final VenueMetadataEnricher enricher;   // from mi-venue-model
    private final TableDataParser tableDataParser;  // from mi-data-intelligence
    private final EmbeddingModel embeddingModel;
    private final TenantAwarePgVectorStore vectorStore;
    private final MasterVenueMatcher masterVenueMatcher;
    private final VenueAssetRepository assetRepository;
    private final RabbitTemplate rabbitTemplate;

    @RabbitListener(queues = "${shortlisty.queues.asset-uploaded}")
    public void onAssetUploaded(AssetUploadedEvent event) {
        // 0. Table data (CSV/XLSX only) — fast parse, no AI, written before embedding
        if (tableDataParser.supports(event.assetType())) {
            var tableData = tableDataParser.parse(s3Key(event), event.contentType());
            assetRepository.setTableData(event.assetId(), tableData);
            // signal done — no further extraction for pure table assets
            if (event.assetType() == AssetType.DATA_TABLE) {
                publishCompleted(event);
                return;
            }
        }
        // 1. Parse
        var documents = tikaReader.read(s3Key(event));
        // 2. Chunk + tag
        var chunks = splitter.apply(documents).stream()
            .map(d -> tagWithMetadata(d, event))
            .toList();
        // 3. Extract (photo_category inferred for PHOTO assets when not set by user)
        var enriched = enricher.apply(chunks, ExtractionContext.of(event));
        if (event.assetType() == AssetType.PHOTO && event.photoCategory() == null) {
            assetRepository.setPhotoCategory(event.assetId(), enriched.inferredPhotoCategory());
        }
        // 4. Embed + store
        vectorStore.accept(enriched.documents());
        // 5. MC_INHERIT match
        masterVenueMatcher.matchAndMerge(event.itemId(), event.tenantId());
        // 6. Signal aggregation
        publishCompleted(event);
    }

    private void publishCompleted(AssetUploadedEvent event) {
        rabbitTemplate.convertAndSend("iqkv.events", "extraction.completed",
            new ExtractionCompletedEvent(event.jobId(), event.assetId(),
                                         event.itemId(), event.tenantId()));
    }
}
```

**Rule:** if a class in `mi-venue-processing-worker` or `mi-venue-service` contains the word `venue` in its business logic (not just in a tag string), ask whether it belongs in `mi-venue-model` instead. The worker and service should contain Spring wiring, `@Bean` registrations, and `@RabbitListener` configuration — not domain decisions.

---

**Docs:** [Architecture Index](README.md) · [Services](services.md) · [Data Model](data-model.md) · [Aggregation](aggregation.md) · [Master Catalog](master-catalog.md) · [Events](events.md) · [Observability](observability.md)
