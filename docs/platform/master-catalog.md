# Shortlisty — Master Venue Catalog

> **Audience:** Engineers, product.
> **Purpose:** Master Venue Catalog design — what it is, how it is seeded, how duplicate detection works, and how tenant extraction merges master catalog fields via MC_INHERIT provenance.

---

## Related Documents

- [data-model.md](data-model.md) — `master_venue`, `master_venue_alias`, `master_venue_external` table definitions
- [services.md](services.md) — `shortlisty-mc-ingest-tagvenue-scraper`, `shortlisty-master-venue-loader` services; S3 master-catalog import paths
- [etl-pipeline.md](etl-pipeline.md) — Stage 3 where `MasterVenueMatcher` runs after extraction
- [aggregation.md](aggregation.md) — MC_INHERIT priority 7 in conflict resolution
- [search.md](search.md) — cross-source search, `scope` parameter, branch B master catalog queries
- [events.md](events.md) — `admin.master-catalog.import.*` RabbitMQ events

---

## What Is the Master Catalog?

The master venue catalog (`public.master_venue`) is a **platform-level service component** — a reference dataset that operates behind the scenes to improve tenant experiences without direct tenant interaction. It is **not a tenant-facing feature**; it is **platform infrastructure** that provides autocomplete suggestions, prevents duplicates, and enriches metadata transparently.

Master Catalog functions as a background service via MC_INHERIT provenance (priority 7 — above `SCRAPE_PROVIDER` priority 4, below all AI tiers and manual/verified). When a tenant's venue creation or extraction process matches a master catalog entry, null/empty canonical fields are automatically back-filled from the catalog. The tenant sees improved data quality but never directly interacts with the master catalog itself. After the copy, the tenant record is fully independent — there is no live link or ongoing dependency.

---

## Cold-Start Population Strategy (MVP)

Three **platform admin channels** seed `public.master_venue`. **No tenant interaction** — all management happens at the platform level by platform administrators. Tenants benefit from the improved suggestions and data quality without seeing or managing the underlying reference data.

| Channel                                   | Mechanism                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | `source` value  | Tenant Visibility                                                            |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- | ---------------------------------------------------------------------------- |
| **1. Pre-provisioned seed migrations**    | Hardcoded rows in Liquibase XML changesets under `shortlisty-venue-model/src/main/resources/db/changelog/system/`. Curated shortlist of 50–200 high-signal venues (top convention centres, major hotel chains in target launch cities). Runs on first startup against `public` schema via `TenantLiquibaseRunner`. Zero code, zero S3, zero admin interaction.                                                                                                                                                                                                                                                                                                                         | `platform_seed` | **Hidden** — provides autocomplete suggestions                               |
| **2. Platform admin manual entry**        | Master Catalog Admin API (Phase 2, pulled forward for MVP as **complete CRUD interface**): `POST/GET/PUT/PATCH/DELETE /api/v1/venues/admin/master-venues`, `POST/PUT/DELETE /api/v1/venues/admin/master-venues/{id}/aliases`. `PLATFORM_ADMIN` authority only. **Full unrestricted access** — admins can create, modify, or delete any master venue record without validation constraints. Pre-write dedup check via `POST /api/v1/venues/admin/master-venues/dedup-check` returns candidate list; admin has final authority to confirm Insert, Merge, or **force any operation**. Any possible change accepted.                                                                       | `admin_import`  | **Hidden** — used for background deduplication                               |
| **3. Standalone scrapers + human review** | Standalone Node.js scrapers (`shortlisty-mc-ingest-tagvenue-scraper`, etc.) produce CSV/JSONL output, upload to S3 `shortlisty/master-catalog/imports/{importId}/`. `MasterCatalogImportOrchestrator` in `shortlisty-master-venue-loader` (Spring Boot) runs on admin RabbitMQ event `admin.master-catalog.import.dry-run` → produces a CSV audit report. Admin reviews the report (each row: action `INSERT` / `MERGE #id` / `SKIP` with name_sim + geo_distance + duplicate candidates list), edits the Action column, re-uploads reviewed CSV, fires `admin.master-catalog.import.apply` → importer applies reviewed actions exactly. No autonomous decisions on the worker's side. | `web_scrape`    | **Hidden** — enriches tenant metadata transparently                          |
| **Tenant-signal enrichment**              | After `extraction.completed`, if tenant data has high-confidence fields not in master catalog → candidate event (Phase 3, no reverse flow in MVP)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | — (not in MVP)  | **One-way** — tenant data can inform platform, but tenants don't see process |

---

## Alias Normalisation

Before any name comparison (INSERT-time dedup check, scraper dry-run report, extraction-time gap-fill), both sides pass through the same normalisation function so that trivial differences do not break the match:

```
normalize(name):
  1. Trim leading/trailing whitespace
  2. Strip case-insensitive leading articles: "^the\s+", "^a\s+", "^an\s+"
  3. Lowercase
  4. Strip any character that is not alphanumeric or whitespace (keep ASCII spaces only)
  5. Collapse consecutive whitespace into single space
  6. Trim again
```

Example: `"The  Bowery-Hotel, NYC!"` → `"bowery hotel nyc"`.

Both the original name and the normalised form are stored. `master_venue_alias` holds the original-name alias, and the normalised form is computed on-the-fly during comparison (or stored redundantly for index-friendliness).

---

## Scraper Import — Dry-Run Dedup Check

Channel 3 scraper dry-run and channel 2 admin INSERT pre-write validation share one deduplication rule set. **Never auto-apply.**

```sql
-- For a candidate import row (name_raw, address_raw, country_code, location_raw):
candidates = SELECT r.id, r.name, r.location
             FROM master_venue r
             LEFT JOIN master_venue_alias a ON a.master_venue_id = r.id
             WHERE normalize(name_raw) % normalize(COALESCE(a.alias, r.name))
             ORDER BY similarity DESC
             LIMIT 5

-- For each candidate:
name_sim = similarity( normalize(name_raw), normalize(COALESCE(a.alias, r.name)) )
if both locations available:
    geo_distance_m = ST_Distance( location_raw, r.location )::int
    geo_ok         = geo_distance_m <= 150
else:
    geo_distance_m = null
    geo_ok         = null

-- Action recommendation (for human review, non-binding):
if (name_sim >= 0.90 AND geo_ok = true)
   OR (geo_distance_m is null AND name_sim >= 0.95):
    -> MERGE candidate #id (show confidence)
else if name_sim >= 0.75 OR geo_distance_m <= 300:
    -> REVIEW CANDIDATE #id (list top 3, show sim + dist table)
else:
    -> INSERT (new)
```

Admin always has the final say via the reviewed CSV action column or the confirm-insert API call.

---

## Platform Admin Full Control

Platform administrators (`PLATFORM_ADMIN` authority) have **complete unrestricted access** to the master venue catalog:

### Full CRUD Operations

- **CREATE**: Add new master venues without any validation constraints
- **READ**: Access all master venue data, metadata, aliases, and external provider records
- **UPDATE**: Modify any field in any master venue record, including metadata, confidence scores, and provenance
- **DELETE**: Remove master venues, aliases, or external provider records

### Override Capabilities

- **Force operations** that bypass deduplication warnings
- **Manual merge** decisions overriding similarity thresholds
- **Confidence score adjustments** for any venue record
- **Metadata schema version** management and force-migration
- **Bulk operations** for data cleanup and management

### No Restrictions

- **Any possible change is accepted** — no business logic constraints on admin operations
- **Bypass all validation** — admins can create venues that fail normal validation rules
- **Override system suggestions** — final authority on all master catalog decisions
- **Emergency data fixes** — complete access for operational issues

Platform admins are trusted with full control because master venue data quality directly impacts all tenant experiences through autocomplete, deduplication, and metadata enrichment services.

---

## Extraction-Time MC_INHERIT Merge — `MasterVenueMatcher`

After a tenant document finishes extraction, `MasterVenueMatcher` runs as Step 3 in [etl-pipeline.md § Stage 3](etl-pipeline.md). It compares the tenant's `Venue` record against `public.master_venue`. **No LLM calls. No embedding similarity. Pure PostgreSQL `pg_trgm` + PostGIS, zero external cost.**

### Algorithm

```
Query strategy:
  1. Fetch top-5 master catalog candidates via trigram GIN index on
     master_venue_alias.alias (normalised match).
  2. For each candidate:
       name_sim = similarity( normalize(venue.name),
                              normalize(best alias OR master_venue.name) )
       if venue.location not null AND candidate.location not null:
           geo_within_200m = ST_DWithin( venue.location, candidate.location, 200 )
           geo_weight = 1.0
       else:
           geo_within_200m = null   // signal unavailable
           geo_weight = 0.0

Combined confidence:
  if geo_within_200m is not null:
      combined = 0.60 * name_sim  +  0.40 * (geo_within_200m ? 1.0 : 0.0)
  else:
      combined = 1.00 * name_sim   // name-only threshold is much stricter
```

### Outcome and Thresholds

```
MATCH (MC_INHERIT merge fields):
    (geo_within_200m is not null AND combined >= 0.75)
    OR
    (geo_within_200m is null     AND combined >= 0.90)
    AND
    the top candidate has >= 0.08 confidence gap over the 2nd-place
    candidate (not ambiguous).
  →
    For every canonical metadata field where:
      (a) tenant venue.metadata.{field} is null/empty AND
      (b) candidate.metadata.{field} is not null AND
      (c) the field is a leaf key (not a nested dict merge):
        copy field value
        set metadata_sources.{field} = { source: "MC_INHERIT",
                                          source_id: master_venue.id,
                                          confidence: combined,
                                          applied_at: now() }

NO MATCH / AMBIGUOUS (everything else, including < 0.08 delta
between top-2 candidates above 0.60):
    → silent no-op. No venue_metadata_events row written.
      Master catalog is secondary; candidates are not surfaced to UI in MVP.
```

After a successful copy, `venues.metadata_aggregated_at` is set to `NOW()` so the subsequent aggregation step (see [aggregation.md](aggregation.md)) sees the MC_INHERIT source and applies conflict-resolution priority correctly.

Also populates `venues.master_venue_id` on unambiguous MATCH — used by cross-source search dedup in [search.md](search.md).

`social.google_place_id` is a priority MC_INHERIT field. When the master catalog record carries a `google_place_id`, it is copied on MATCH even if the tenant venue already has other `social` sub-fields set — because it unlocks cheap downstream enrichment (Google Places API: address, photos, rating, hours) at `SCRAPE_PROVIDER` priority 4, without any AI cost.

---

## Before Sprint 1 — Threshold Calibration Dry-Run

Before any production tenant has access, run a mandatory calibration pass:

1. Assemble a fixture set of **50 real-world venue PDFs** from target launch cities. Manually attach the "correct" `master_venue.id` ground truth to each row (or "no master catalog match" if none applies).
2. Run `MasterVenueMatcher` in **dry-run mode**: no writes to tenant schema. Log: top-5 candidates with individual `name_sim`, `geo_within_200m`, `combined`, final outcome `matched|ambiguous|no_match`.
3. Build confusion matrix against ground truth: TP, FP, FN counts.
4. **Acceptance criterion: FP rate ≤ 1 %.** A wrong copy poisons tenant data; an FN is just a missed gap-fill that the user fills manually. If FP > 1 %, raise thresholds incrementally (e.g. 0.75 → 0.78 → 0.80) until criterion passes. Document final calibrated thresholds in CHANGELOG.md.
5. Delete the calibration run log from any environment that stores real tenant PII.

---

## Observability for Match Quality

`MasterVenueMatcher` records Micrometer metrics on every extraction run (see [observability.md](observability.md)):

| Metric                                          | Tags                            | Purpose                                                                                   |
| ----------------------------------------------- | ------------------------------- | ----------------------------------------------------------------------------------------- |
| `shortlisty_mc_match_total`                     | `stage="extraction"`, `outcome` | `matched` vs `no_match` vs `ambiguous` counts. Tracks how often master catalog is useful. |
| `shortlisty_mc_match_confidence_seconds` (hist) | `stage="extraction"`            | Distribution of `combined` score across all runs. Calibrate thresholds from this.         |

The scraper dry-run report CSV is retained in S3 for 90 days as an audit trail of every master catalog population decision.

---

**Docs:** [Architecture Index](README.md) · [Data Model](data-model.md) · [Services](services.md) · [ETL Pipeline](etl-pipeline.md) · [Aggregation](aggregation.md) · [Search](search.md) · [Observability](observability.md)
