# D18 — Self-hosted inference stack: open-source models (Phi-4, Qwen) on own infrastructure as primary AI layer

> **Audience:** Engineers, architects.
> **Purpose:** Record the decision to run AI inference — structured extraction, embeddings, and AI-assisted pitch features — on self-hosted open-source models rather than calling external AI APIs as the primary path.

---

## Context

The initial architecture documented in `docs/platform/intelligence.md` specifies GPT-4o (OpenAI API) for structured venue metadata extraction and `text-embedding-3-small` (OpenAI API) for vector embeddings. This was the correct starting assumption for rapid prototyping: strong baseline accuracy, no infrastructure overhead, low day-one cost.

Three forces push toward self-hosted inference:

1. **Data security and NDA obligations.** Agencies sign NDAs that create legal obligations around client information. Every document that leaves the tenant's logical boundary and passes through a third-party API — even under enterprise data processing terms that exclude training — is an exposure surface. Self-hosted inference eliminates that surface entirely: documents are processed on infrastructure the operator controls, and no venue data or client information ever leaves the network perimeter.

2. **Data sovereignty and regulatory fit.** EU-based agencies face GDPR requirements on data residency. US agencies serving government or healthcare clients may have FISMA or HIPAA-adjacent requirements. A self-hosted stack provides a single, demonstrable answer: "all processing happens on our servers, in our data centre, under our control." No dependency on a third-party's compliance posture.

3. **Unit economics at scale.** OpenAI API costs scale linearly with usage — every extraction call, every embedding, every AI-assisted pitch response has a marginal cost. At 100 agencies with active catalogs and frequent pitch generation, the AI processing line is manageable. At 1,000 agencies it becomes a significant margin pressure. Self-hosted inference has high upfront infrastructure cost but near-zero marginal cost per call.

---

## Options considered

### Option A — Retain OpenAI API as the primary inference layer

Continue with GPT-4o for extraction and `text-embedding-3-small` for embeddings. Offer Azure OpenAI as an Enterprise add-on for data residency requirements.

**Pros:** No infrastructure complexity. GPT-4o extraction quality is well-understood and benchmarked. Fast iteration — no GPU provisioning, no model management.

**Cons:** Every document passes through a third-party API. Linear cost scaling. Dependency on OpenAI pricing and availability. The Azure "private endpoint" answer still routes data through Microsoft's infrastructure — it does not satisfy the strongest data sovereignty requirements.

### Option B — Self-hosted open-source models as primary layer (chosen)

Deploy open-source LLMs (Phi-4, Qwen family) on own VPS/server infrastructure using an OpenAI-compatible inference server (Ollama, vLLM, or llama.cpp). Open-source embedding models (BGE-M3, E5-mistral, or nomic-embed-text) replace `text-embedding-3-small`. The Spring AI abstraction layer (`ChatModel`, `EmbeddingModel`) is already model-agnostic — swapping the endpoint is a configuration change, not a code change.

**Pros:** No data leaves the operator's infrastructure. Zero marginal cost per inference call after infrastructure is provisioned. Strong answer to NDA and data sovereignty concerns. Model choice is not locked to a single vendor. Models can be fine-tuned on venue-domain data over time.

**Cons:** GPU infrastructure provisioning and management overhead. Model performance must be benchmarked against the venue extraction schema before production use. Smaller open-source models may require prompt engineering investment to match GPT-4o structured output quality on complex venue documents. Requires operational expertise (model serving, load balancing, failover).

### Option C — Hybrid: self-hosted for extraction, API for fallback

Run self-hosted models as the primary path; fall back to OpenAI API for documents where the local model confidence falls below a threshold.

**Pros:** Balances cost and quality. High-confidence extractions never leave the perimeter; only difficult documents are escalated.

**Cons:** More complex routing logic. The fallback still creates an API dependency and a data-leaving-the-perimeter path — weakening the security story. Threshold calibration requires ongoing maintenance.

---

## Decision

**Option B.** Self-hosted open-source models on own infrastructure (VPS / dedicated servers) as the primary inference layer for both extraction and embeddings.

**Primary candidates:**

- Extraction: Phi-4 (Microsoft, 14B) and Qwen2.5 family (Alibaba, 7B–72B) — both support structured JSON output and demonstrate strong performance on document understanding tasks.
- Embeddings: BGE-M3 (BAAI) — multilingual, supports dense + sparse + colbert retrieval modes; strong candidate for replacing `text-embedding-3-small` given the multilingual venue document corpus.
- Inference server: Ollama (development and smaller deployments) or vLLM (production, higher throughput, OpenAI-compatible API).

Spring AI's `ChatModel` and `EmbeddingModel` abstractions are already endpoint-agnostic. The only required change is the base URL configuration — all application code remains unchanged.

---

## Rationale

The primary driver is the data security and NDA story. Agencies that sign NDAs with their clients need a demonstrable answer to "where is our data processed?" Self-hosted inference gives that answer cleanly: on our servers, under our control, nothing leaves the network. No third-party data processing agreement, no audit dependency on OpenAI or Microsoft, no exposure surface in transit.

The unit economics argument strengthens over time. The infrastructure investment (GPU server or high-VRAM VPS) is a fixed cost. At 50 agencies that cost is hard to justify. At 500 it pays for itself many times over. Building toward self-hosted inference from the start — rather than migrating later — avoids a disruptive architectural change at scale.

The open-source model landscape in 2026 makes this decision viable in a way it was not in 2023. Phi-4 and Qwen2.5 produce structured JSON output that is competitive with GPT-4o on constrained extraction tasks. The venue extraction schema (§2.1 in intelligence.md) is a well-defined, bounded prompt — not an open-ended generation task. Smaller, purpose-tuned models are often more consistent than large general-purpose models on this class of task.

---

## Consequences

- `docs/platform/intelligence.md` references to GPT-4o as the extraction model and `text-embedding-3-small` are updated to reflect the self-hosted stack as primary, with OpenAI API noted as a fallback option during the transition period.
- The extraction pipeline's `VenueMetadataEnricher` is configured against a local OpenAI-compatible endpoint — no application code change required.
- Benchmark task added before production deployment: run the venue extraction schema against 50 real venue documents on Phi-4 and Qwen2.5; measure per-field accuracy against a human-labelled ground truth. Accept production use only when accuracy meets the threshold established by the GPT-4o baseline.
- Infrastructure requirements: minimum one GPU-capable VPS or dedicated server with ≥24GB VRAM for Qwen2.5-14B or Phi-4 at full precision; or ≥16GB VRAM with 4-bit quantisation. BGE-M3 embedding model runs on CPU with acceptable latency for batch processing.
- The data-security objection answer in `objections.md` is strengthened: "all AI processing runs on our own servers — no document ever leaves our infrastructure."
- OpenAI API remains documented as a development-time fallback (useful when running without GPU hardware locally). It is not a production dependency.
- If benchmark results show a significant accuracy gap on complex scanned PDFs or design-heavy venue decks, Option C (hybrid routing for difficult documents) is the documented fallback — create a new decision record at that point.

---

## Status

**Status:** Accepted

---

**Docs:** [Decisions index](README.md) · [Intelligence Layer](../../platform/intelligence.md) · [Architecture](../../platform/README.md) · [Vision](../vision.md)
