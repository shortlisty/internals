# VenueMi — ETL Pipeline

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

| Asset type                   | Reader                                    | Notes                                   |
| ---------------------------- | ----------------------------------------- | --------------------------------------- |
| PDF (text-based)             | `TikaDocumentReader`                      | Apache Tika, ships with Spring AI       |
| PDF (scanned)                | `TikaDocumentReader` + Tesseract OCR      | Tika bundles OCR                        |
| PDF (complex layout, tables) | Docling sidecar → custom `DocumentReader` | Phase 2; better table fidelity          |
| DOCX / XLSX / PPTX           | `TikaDocumentReader`                      | Same reader, 1000+ formats              |
| Images (JPG, PNG)            | GPT-4o vision direct                      | No text reader needed                   |
| Floor plan (PDF/image)       | Docling layout-aware → GPT-4o vision      | Phase 2                                 |
| DWG / DXF (CAD)              | `TikaDocumentReader` (AutoCAD parser)     | Extracts metadata; visual in Phase 2    |
| Video                        | Out of scope Phase 1                      | Phase 2: keyframe extraction via ffmpeg |

---

### Stage 2 — Transform

1. **Chunk** — `TokenTextSplitter` (512 tokens, 50-token overlap). Spec-sheet tables use 256-token chunks to preserve row precision.
2. **Tag** — attach `venue_id`, `asset_id`, `asset_type`, `tenant_id` as Document metadata.
3. **Extract** — `VenueMetadataEnricher` (custom `DocumentTransformer`, venue-specific, from `mi-venue-model`): calls GPT-4o with structured output schema matching the venue canonical field set (see [data-model.md § Canonical Field Set](data-model.md)), returns `VenueMetadata` POJO with confidence scores per field. The enricher is the **only venue-specific component** in the pipeline — all surrounding plumbing is generic.

---

### Stage 3 — Load

1. **Embed** — `EmbeddingModel` (`text-embedding-3-small`, 1536 dims). Generic — from `mi-data-intelligence`.
2. **Store** — `TenantAwarePgVectorStore` writes chunks + embeddings to `item_vectors` table in the tenant's schema. Table defined in `mi-data-intelligence` changelog (see [services.md § 4c](services.md)).
3. **Master catalog match** — `MasterVenueMatcher` runs the full MC_INHERIT merge algorithm (trigram name similarity via `pg_trgm` GIN index on `master_venue_alias`, PostGIS `ST_DWithin` 200m radius, combined confidence formula, thresholds 0.75 with geo / 0.90 name-only, ambiguity delta guard ≥ 0.08). No LLM calls. Full algorithm, thresholds, and field-copy semantics in [master-catalog.md](master-catalog.md).
4. **Aggregate** — publishes `ExtractionCompletedEvent` (from `mi-data-intelligence`) → `MetadataAggregationConsumer` in `mi-venue-service` updates `venues.metadata` via `VenueMetadataMigrator.ensureCurrent()` (from `mi-venue-model`). Full aggregation design in [aggregation.md](aggregation.md).

---

### Processing SLA

| Asset type           | Target latency |
| -------------------- | -------------- |
| PDF / DOCX (text)    | < 30s          |
| Images / floor plans | < 60s          |
| CAD files            | < 2 min        |

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
    private final EmbeddingModel embeddingModel;
    private final TenantAwarePgVectorStore vectorStore;
    private final MasterVenueMatcher masterVenueMatcher;
    private final RabbitTemplate rabbitTemplate;

    @RabbitListener(queues = "${venuemi.queues.asset-uploaded}")
    public void onAssetUploaded(AssetUploadedEvent event) {
        // 1. Parse
        var documents = tikaReader.read(s3Key(event));
        // 2. Chunk + tag
        var chunks = splitter.apply(documents).stream()
            .map(d -> tagWithMetadata(d, event))
            .toList();
        // 3. Extract
        var enriched = enricher.apply(chunks);
        // 4. Embed + store
        vectorStore.accept(enriched);
        // 5. MC_INHERIT match
        masterVenueMatcher.matchAndMerge(event.itemId(), event.tenantId());
        // 6. Signal aggregation
        rabbitTemplate.convertAndSend("iqkv.events", "extraction.completed",
            new ExtractionCompletedEvent(jobId, event.assetId(), event.itemId(), event.tenantId()));
    }
}
```

**Rule:** if a class in `mi-venue-processing-worker` or `mi-venue-service` contains the word `venue` in its business logic (not just in a tag string), ask whether it belongs in `mi-venue-model` instead. The worker and service should contain Spring wiring, `@Bean` registrations, and `@RabbitListener` configuration — not domain decisions.

---

**Docs:** [Architecture Index](README.md) · [Services](services.md) · [Data Model](data-model.md) · [Aggregation](aggregation.md) · [Master Catalog](master-catalog.md) · [Events](events.md) · [Observability](observability.md)
