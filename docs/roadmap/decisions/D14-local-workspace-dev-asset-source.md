# D14 — Local shared folder as dev asset source behind a single `AssetStore` port

> **Audience:** Engineers, architects.
> **Purpose:** Record why read/parse/extract features are developed against a plain local folder mounted into the worker as a Docker volume (`WORKER_WORKSPACE`) instead of S3/MinIO, and how the seam is kept honest: one `AssetStore` port, two explicit adapters, no implicit fallback.

---

## Context

S3 (MinIO in local dev) is the storage of record for every user document. The bucket strategy, key layout, and presigned-URL upload flow are fixed in [services.md § 4a](../../platform/services.md) — nothing in this decision changes them.

The bulk of Phase 1 work is read → parse → extract: Tika/Docling parser behaviour, chunking, prompt tuning, confidence calibration. That work is iterative — the same corpus is re-parsed dozens of times per day. Driving it through the real asset path means, for every iteration:

- MinIO, PostgreSQL, and `docintelligence-catalog-service` must all be running;
- a fixture must travel through `POST /assets/initiate` → presigned `PUT` → `POST /assets/{id}/confirm`;
- any edit to a fixture file requires repeating that flow;
- reproducing a production parsing bug requires exporting the object from the bucket before debugging can start.

`docintelligence-catalog-processing-worker` is a companion worker with no inbound HTTP surface (health/ready/metrics only) and no extraction code yet. The asset-access seam is therefore being introduced now, while it is still free to choose.

The question: can read/parse/extract development run against a plain shared folder — subfolders, files saved under a fixed naming template — mounted into the worker as a Docker volume, with the path configured from outside and exposed as `WORKER_WORKSPACE`, while S3 remains the storage of record?

---

## Options considered

### Option A — One `AssetStore` port, two explicit adapters, selected by configuration

The worker's application layer depends on a single port (working name):

```java
public interface AssetStore {
  Resource open(String s3Key);                 // read — the only way the pipeline gets bytes
  void write(String s3Key, Resource content);  // derived artifacts only
}
```

Two adapters implement it:

- `FileSystemAssetStore` — resolves the key against the `WORKER_WORKSPACE` root directory (a Docker volume; the host path is configured outside the image).
- `S3AssetStore` — uses the existing `ObjectStorageConfigurationProperties` (endpoint, credentials, bucket, region) and the MinIO/S3 SDK.

The active adapter is selected by an explicit property (`worker.asset.source = filesystem | s3`) that defaults to `s3`; `application-local.yml` selects `filesystem`. There is no fallback chain: a key the filesystem adapter cannot resolve is an error, not a cue to try S3.

The workspace folder mirrors the S3 key layout one-to-one — the relative path inside `WORKER_WORKSPACE` is exactly the object key, for example `docintelligence/tenants/acme0001/venues/{venueId}/assets/{assetId}/grand-ballroom-deck.pdf`. The folder starts as a plain subfolder tree with files named by the same template the bucket uses.

**Pros:**

- The iteration loop collapses to "drop the file in the folder, re-run the job" — no catalog service, no database, no MinIO, no upload flow.
- Both adapters sit behind the same port, so parsing, chunking, extraction, and embedding code is identical in dev and runtime; only the `AssetStore` binding changes.
- Fixture trees are reproducible: `mc mirror` / `aws s3 sync` of a real bucket prefix produces a ready workspace, so production parsing bugs can be replayed locally on the exact bytes.
- A missing or malformed fixture fails loudly — the behaviour a parser-development corpus wants.
- No new service, and no change to the upload, presigning, key, or IAM design.

**Cons:**

- A second adapter that only the local profile exercises; a regression in `S3AssetStore` is not caught by local runs.
- Filesystem semantics differ from object storage (no eventual consistency, no per-request latency, no object permissions); code that accidentally relies on them passes locally and fails elsewhere.
- One more property and one more volume in the Compose topology.
- The key → path mapping is a small piece of code that must stay in sync with the key convention.

### Option B — S3/MinIO only, in every environment

Development always reads through the real `S3AssetStore`; MinIO runs locally and fixtures are uploaded through the normal flow.

**Pros:**

- One code path only — no dev-only adapter to write, test, or maintain.
- The upload flow, presigned URLs, credentials, and key layout are exercised from day one.

**Cons:**

- Every parser iteration is gated on three unrelated services being up and a three-step upload completing.
- Fixture churn is expensive (re-upload per edit), which discourages the wide corpus that table and catering extraction quality work needs.
- Reproducing a production parse failure locally requires a bucket round-trip before debugging can start.
- It puts the critical-path MVP work behind infrastructure availability it does not actually depend on.

### Option C — Filesystem first, silent fallback to S3

Read the local folder; if the key is absent, fetch it from S3.

**Pros:**

- Least friction for a developer: some assets come from disk, some from S3, with no visible difference.

**Cons:**

- Behaviour depends on what happens to be on the local disk — the same key resolves to different sources on different machines.
- A typo in a fixture path silently becomes a network read (or an auth failure) instead of an error.
- Tests become non-deterministic: an unrelated local file changes the outcome.
- Two sources of truth for one asset; nothing in the result reveals which one was read.
- Contradicts the storage-of-record invariant: the disk quietly becomes a partial replica of the bucket.

---

## Decision made

**Option A.** Read/parse/extract development runs against a local shared folder exposed as `WORKER_WORKSPACE`, but only behind a single `AssetStore` port with two explicit adapters:

1. **One port, no fallback.** `AssetStore` is the only way the worker obtains asset bytes. An unresolvable key is a hard failure. Option C is rejected outright.
2. **Explicit selection.** The adapter is chosen by configuration (`worker.asset.source`, default `s3`), never by probing the environment at runtime.
3. **Layout parity.** The folder mirrors the S3 key layout one-to-one; files are stored under the same naming template and subfolder structure as the bucket.
4. **Dev fixtures only.** The workspace is a development corpus — not a storage of record, not a cache, and not an authority for anything the platform persists. S3 remains authoritative and no asset is dual-written.

---

## Rationale

- **The critical path is parser and extraction iteration, not upload plumbing.** Cost per iteration dominates Phase 1 throughput, and the cheapest way to widen the corpus is a folder of files.
- **The seam has to exist anyway.** `TikaDocumentReader` has no auto-configuration and takes a `Resource`, so _something_ must resolve key → bytes. Making that something a port costs nothing and keeps S3 SDK calls out of the extraction pipeline.
- **Explicit selection with no fallback keeps dev mode honest.** Option C buys a small convenience with non-deterministic behaviour and a hidden second source of truth — a convenience path nobody can reason about later.
- **Layout parity makes the mode switch almost meaningless.** A real bucket prefix copied into the workspace is a valid corpus, and a file that parses locally parses in sit on the same bytes; no translation layer between folder structure and object key needs maintaining.
- **S3 stays the storage of record.** The two-phase presigned upload, key layout, bucket strategy, and credentials in [services.md § 4a](../../platform/services.md) are untouched; this decision only governs how the worker reads bytes.
- **Option B's single code path is genuinely simpler**, but it puts the whole parsing effort behind MinIO, the catalog service, and the upload flow. The maintenance cost of one adapter (path resolution plus a stream open) is far smaller than the velocity lost across the Phase 1 extraction corpus.

---

## Consequences

- `docintelligence-catalog-processing-worker` gains two configuration inputs: `worker.asset.source` (`filesystem` | `s3`, default `s3`) and `WORKER_WORKSPACE` (absolute path inside the container, host path injected from outside).
- `application-local.yml` selects `filesystem`; sit/uat/prd keep `s3`, which is also the default in `application.yml`.
- `vds-shortlisty-service-via-docker/compose.yaml` gains a bind mount for the workspace folder (for example `../var/worker-workspace:/workspace`) plus the matching `WORKER_WORKSPACE` variable.
- The workspace folder is git-ignored and never committed — it holds customer-like content of arbitrary size.
- The `AssetStore` port must land before the first extraction code; retrofitting it later means auditing every read site in the pipeline.
- The `S3AssetStore` path is not exercised by local runs. At least one sit smoke check against a real asset is required before an extraction change is considered done — local-only success is not acceptance.
- Parsing tests need no MinIO and no container orchestration; a temporary directory and the filesystem adapter are sufficient.
- Fixture corpora live outside the repositories (they are inputs, not deliverables) and are described in the worker README.
- Nothing is superseded by this record. It complements [D2](D2-tika-vs-docling-phase1.md) (parser choice) and leaves the storage design in [services.md § 4a](../../platform/services.md) intact.

---

## Status

**Accepted.** The local workspace is a development aid for reading assets only. S3 remains the storage of record in every environment.

---

**Docs:** [Vision](../vision.md) · [Intelligence Layer](../../platform/intelligence.md) · [Services / S3 layout](../../platform/services.md) · [Architecture](../../platform/README.md)
