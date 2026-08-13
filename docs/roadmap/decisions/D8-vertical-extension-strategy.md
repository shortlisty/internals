# D8 — Vertical extension strategy: strategy-pattern generic core + domain library swap

> **Audience:** Engineers, architects.
> **Purpose:** Record why the intelligence platform is split into a domain-agnostic generic core (`mi-data-intelligence`) and a swappable domain library (`mi-venue-model`) via the Strategy pattern, rather than a monolithic venue-specific service, a config-driven parameterisation, or a code-fork per vertical.

---

## Context

VenueMi Intelligence is built first for the venue intelligence vertical (event planner venue catalogs). However, the platform founders explicitly designed the underlying intelligence infrastructure to be reusable across other verticals:

- Medical: structured extraction from medical records (patient intake forms, lab reports, imaging reports) with a curated drug compendium (instead of master venue catalog).
- Agro: agricultural asset documents (soil analysis reports, satellite imagery metadata, crop yield certificates).
- Legal: contract clause extraction with a curated legal-term master catalog.
- Real estate: property listing documents with a curated building master catalog.

Each vertical swaps: canonical field set, extraction prompt, extraction structured output schema, conflict-resolution priority rules, curated-list matching algorithm, and search field semantics. The shared infrastructure (ETL pipeline stages, embeddings, vector store, metadata aggregation consumer, search orchestrator, event contracts, schema versioning) is identical.

The question is how to architect this reuse so that a pivot to a new vertical is a library swap, not a rewrite.

---

## Options considered

### Option A — Monolithic venue-specific service. Verticalisation = code fork.

Everything lives in `mi-venue-service` and `mi-venue-processing-worker`. Domain logic, extraction prompts, venue canonical fields, aggregation rules, and the ETL pipeline consumers are all in one codebase. Pivoting to medical means forking the repo, globally find-replacing "venue" with "case", deleting venue-specific fields, adding medical fields, fixing 4,000 lines of tests, and maintaining two permanently divergent codebases.

**Pros:**

- Fastest MVP delivery. No abstraction overhead. No generic contracts to design and test.
- Simple code navigation for the first 6–12 months: everything venue-related is in one service module.

**Cons:**

- **Vertical pivot = 6–12 month rewrite.** Every venue-specific class name, every `VenueMetadataExtractionStrategy`, every `MasterVenueMatchStrategy`, every table name (`venues`, `venue_assets`) must be renamed, redesigned, or replaced. Generic infrastructure is tangled with venue logic. 50% of the code is "copy with edits", 40% is new tests, 10% is de-venue-ing — effectively a greenfield rewrite. The promise of "platform reusable across verticals" is dead on arrival.
- **Bug fixes do not cross-pollinate.** A critical fix to the `MetadataAggregationConsumer` (e.g., lost-update race, or a migrator bug) in the venue fork must be manually cherry-picked into the medical fork, the agro fork, and any future fork. With 3 verticals, this is a permanent drag on engineering velocity; bugs fixed in one vertical recur in others for months.
- **Platform-level security and isolation audits do not transfer.** Foundation security auditors reviewed schema-per-tenant logic, `public.` access rules, and vector-store isolation in the venue service. In a fork-based medical vertical, every auditor must re-review the same patterns because the code path is copy-pasted with edits; the auditor cannot trust that the venue audit results apply.

### Option B — Config-driven parameterisation. One service, YAML-defined fields per vertical.

A single service supports all verticals. The canonical field set, extraction prompts, and aggregation priority rules are defined in YAML configuration loaded on startup. Tenant onboarding selects a vertical profile; per-request code dispatches to the configured YAML-defined rules via reflection/generics.

**Pros:**

- No code changes for a new vertical. A new YAML file + new extraction prompt + new Liquibase changelog = new vertical live.
- One codebase, one deploy, one set of audits.

**Cons:**

- **Aggregation and matching rules are not declaratively specifiable in YAML at the fidelity the product requires.** Venue conflict resolution uses priority: `MANUAL_OVERRIDE > VERIFIED_EXTRACTION > HIGH_CONFIDENCE_AI > MEDIUM_CONFIDENCE_AI > LOW_CONFIDENCE_AI > MC_INHERIT > SCRAPE_PROVIDER`. MC_INHERIT (priority 7) sits above SCRAPE_PROVIDER (4), not lowest — see architecture §3. Array fields use set-union with confidence ≥ 0.6 threshold. Some fields (`capacity.configurations`) use numeric max-tiebreak-with-source-authority logic, not priority-ordered overwrite. Medical case aggregation requires per-field override semantics where a lab-report source overrides a patient-intake source specifically for `blood_pressure` but not for `diagnosis_notes` — i.e., field-level custom rules that are logic, not configuration. Encoding field-level logic in YAML produces a Turing-complete YAML dialect that no engineer can debug and no static analyser can type-check.
- **Extraction strategy output types are Java POJOs, not maps.** The venue extraction GPT-4o call returns a `VenueMetadata` POJO with nested types (`VenueCapacity`, `VenueCatering`, typed enums for `CateringPolicy`). This POJO is validated via Bean Validation on deserialisation. A config-driven map-based approach throws away compile-time type safety: missing nested keys, wrong numeric types, and enum typos surface as runtime `ClassCastException` and null-pointer bugs that are far harder to diagnose than a compile error.
- **Performance.** Config-driven field-level rules require runtime reflection per field per aggregation call. Aggregation is in the hot path; adding microseconds of reflection per field across 100+ fields per venue on every extraction multiplies into visible latency.
- **Testing is combinatorial explosion.** A config-driven system has N verticals × M rule types. Writing unit tests for every vertical × every rule type interaction is a matrix that grows without bound; in practice, critical cross-vertical interactions are never tested and break in production.

### Option C — Strategy pattern: generic core library (`mi-data-intelligence`) + domain library swap (`mi-venue-model`)

Split into three layers:

**Layer 1: `mi-data-intelligence` (generic, compile-only JAR, no runtime, no Spring beans)**

- Defines generic Java interface contracts:
  - `MetadataExtractionStrategy<M>` — extract structured metadata from chunks
  - `MetadataAggregationStrategy<M>` — merge + detect conflicts
  - `MetadataMigrator` — schema versioning chain runner
  - `CuratedListMatchStrategy` — platform curated list gap-fill
  - `SearchBranchExecutor<R>` — one parallel search branch
- Defines generic final consumers wired with injected strategies:
  - `AssetExtractionOrchestrator<M>` — full ETL pipeline, uses strategies
  - `MetadataAggregationConsumer<M>` — aggregation, uses strategy + migrator
  - `SearchOrchestrator<R>` — parallel search + RRF merge, uses 2 branch executors
- Defines infrastructure Liquibase changelogs (extraction_jobs, item_vectors, item_metadata_events, ai_cost_tracking)
- Defines event POJOs (`AssetUploadedEvent`, `ExtractionCompletedEvent`, etc.) with generic `item_id` field names.
- **No venue concept. No medical concept. No vertical noun in any class name.**

**Layer 2: `mi-venue-model` (venue domain, compile JAR, imports mi-data-intelligence)**

- Implements all 5 strategy interfaces for venues:
  - `VenueMetadataExtractionStrategy implements MetadataExtractionStrategy<VenueMetadata>`
  - `VenueMetadataAggregationStrategy implements MetadataAggregationStrategy<VenueMetadata>`
  - `VenueMetadataMigrator implements MetadataMigrator` (D7)
  - `MasterVenueMatchStrategy implements CuratedListMatchStrategy`
  - `TenantVenueSearchBranch` + `MasterVenueSearchBranch` implement `SearchBranchExecutor<VenueSummaryView>`
- Defines venue-specific POJOs (`Venue`, `VenueAsset`, `VenueMetadata`, canonical field set enums)
- Defines venue Liquibase changelogs (`venues`, `venue_assets`, `master_venue` in `public`)

**Layer 3: Spring Boot services (venue-service + processing-worker)**

- Import both libraries.
- Their sole job: `@Bean` registration wiring venue strategies into generic consumers.
- `@RabbitListener` configuration: queue names, concurrency settings, ack modes.
- REST controllers translating HTTP DTOs into domain POJOs.
- **Zero venue-domain decisions in service code. Zero extraction logic. Zero aggregation logic.** If a class contains "venue" in a business-logic identifier (not just a bean name), it belongs in Layer 2.

Scraping and master catalog population are further decomposed into mi-mc-ingest-<source>-scraper (Node.js per provider) and mi-mc-loader (Spring Boot) — see architecture §4.

**Pivot to medical:**

- Create `mi-med-model` (Layer 2 medical library, imports `mi-data-intelligence` unchanged).
- Implement 5 medical strategy classes.
- Add medical POJOs + medical Liquibase changelogs.
- Create `mi-med-service` + `mi-med-processing-worker` (Layer 3) with `@Bean` wiring.
- **0 lines changed in `mi-data-intelligence`. 0 lines changed in existing venue modules.**

**Pros:**

- **Vertical pivot cost is linear in domain complexity, not proportional to platform size.** A medical pivot is ~20 Java classes (5 strategies + POJOs + migrations) + 5 Liquibase changelogs + 2 thin service modules. The entire 50,000-line ETL/aggregation/search infrastructure is reused unchanged. The venue-platform security audit, incident-response playbooks, and monitoring all transfer because the consumer code paths are byte-identical.
- **Bug fixes in generic core benefit all verticals simultaneously.** One fix to `MetadataAggregationConsumer` transaction boundary (or one performance improvement to `SearchOrchestrator` RRF weights) is available to venue, medical, agro, and legal the next time they upgrade `mi-data-intelligence` version. No cherry-picking, no divergence, no cross-fork bug recurrence.
- **Type safety and test quality.** Domain strategies return typed POJOs, not maps. Aggregation rules are Java logic with compile-time type checking. Each `VenueMetadataAggregationStrategy` unit test has 3 fixtures and 5 assertions; the equivalent config-driven test across verticals is 15 YAML files and 100 lines of reflection-heavy harness.
- **Separation of concerns is clean.** Strategy interfaces define what each layer is responsible for. Code reviewers can audit `mi-data-intelligence` for generic correctness (transaction boundaries, thread safety, interface contracts) without understanding venue catering policy enums. Venue domain reviewers audit the strategies for event-planner semantics without understanding RabbitMQ listener ack mode internals. This separation matters for security audit velocity: foundation auditors review the generic core once and then only spot-check domain libraries, rather than re-reviewing 100% of code per vertical.
- **Compile-only dependency on mi-data-intelligence means zero bloat.** The library has no Spring Boot runtime, no application properties, no auto-configuration classpath pollution. Services are lean; classpath is identical to a monolithic service of the same size.

**Cons:**

- **MVP delivery is slower than monolithic Option A by 1–2 weeks.** The 5 generic interfaces, the parameterised consumers, and the compile-vs-runtime separation require careful design. A junior engineer without Strategy-pattern familiarity can write a monolithic venue-specific aggregation consumer in 2 days but may take 5 days to design the generic `MetadataAggregationStrategy<M>` interface correctly and write tests for it. This is a one-time cost that pays for itself on the first vertical pivot.
- **Java generics complexity budget consumed.** `AssetExtractionOrchestrator<M>` parameterised on `<M extends MetadataBase>` combined with `@Bean` parameterised Spring wiring produces compiler warnings that require `@SuppressWarnings` in a small number of well-documented locations. IDE code-navigation from a strategy interface usage to its venue implementation requires resolving the generic parameter first; new joiners need a 30-minute architecture onboarding to understand the flow.
- **Naming discipline is a social contract, not enforced by Java.** Nothing mechanically prevents a developer from adding a `venue_id` field to a generic event POJO in `mi-data-intelligence` "just to make the venue service easier." Once the first vertical noun leaks into the generic core, the second one will, and within 6 months Option C devolves into Option A with extra layers. Enforcement is code-review discipline + a documented list of "what does not go in mi-data-intelligence" (D7 README.md §4c has this table).

---

## Decision made

**Option C: Strategy pattern, generic core library + swappable domain library.**

Three-layer architecture is mandatory:

1. `mi-data-intelligence` — generic contracts + final consumers + infra changelogs + event POJOs. Zero vertical nouns.
2. `mi-venue-model` — venue strategies implementing generic contracts + venue POJOs + venue changelogs. No Spring beans for infrastructure; only domain `@Component` strategy implementations.
3. Spring Boot services — `@Bean` wiring only; zero domain business logic.

New vertical must follow the same pattern. Any code change that introduces a vertical noun into a generic core interface or consumer class is rejected at code review with a link to this decision record.

---

## Rationale

- **The explicit multi-vertical intent makes this the only viable long-term choice.** The founders have explicitly stated that the venue vertical is a beachhead; medical and agro verticals are within the 24-month horizon. Option A means a full rewrite per vertical. Option B means a Turing-complete YAML nightmare that even the original authors cannot debug after 12 months. Only Option C delivers on the platform promise.
- **The Strategy pattern is the textbook answer to "same algorithm structure, different per-step details."** The ETL pipeline has a fixed structure (parse → split → extract → embed → aggregate → match). The per-step details differ by vertical. This is the canonical use case for Strategy; every senior engineer has seen this pattern before. Designing the interfaces correctly requires 2 meetings and 5 days of implementation; the alternative of "we'll refactor later into Strategy when we need medical" underestimates how much venue logic will have leaked into the consumer code by then (estimated 3–6 months to refactor-extract the generic core from a monolith, vs. a 5-day upfront design).
- **Bug propagation across verticals is eliminated at zero cost.** A production incident in mi-venue-service caused by an aggregation consumer bug typically takes 4 hours to diagnose, fix, deploy, and hot-patch. With Option C's monolithic fork (Option A), the same bug exists identically in the medical fork and is independently discovered 3 months later during medical UAT, taking another 4 hours to re-diagnose ("we fixed this in venue already, why is it happening here?"). The accumulated engineering cost of cross-fork bug re-discoveries across 3 verticals over 3 years easily exceeds the upfront 5-day Strategy design cost by an order of magnitude.
- **Audit and compliance for medical vertical require structurally identical consumer code paths.** HIPAA-audited medical vertical must demonstrate that "the aggregation consumer transaction boundary is byte-identical to the venue vertical's transaction boundary that already passed foundation audit." Under Option C, this is proven by showing that both services import the same version of `mi-data-intelligence` and therefore run the same `MetadataAggregationConsumer.class` bytecode. Under Option A monolithic forks, auditors demand a 100% line-by-line re-review: 2–4 weeks of auditor time and $20–50K in audit fees.

---

## Consequences

- README.md §4c explicitly lists "What does NOT go here — common mistakes at code review." This table is the enforcement checklist for code review. Any PR adding a vertical-specific field, POJO, migration, or prompt to `mi-data-intelligence` is rejected with a comment linking to the table in §4c and linking to this decision record.
- `mi-data-intelligence` package structure has a zero vertical-noun rule. `find . -name '*.java' | xargs grep -E 'venue|medical|agro|legal' -i` inside the `mi-data-intelligence` module must return zero matches, excluding test comments that explicitly reference a domain for illustration. A CI grep check enforces this pre-merge.
- New vertical onboarding SOP is documented: (1) create domain library, (2) implement 5 strategy interfaces, (3) add domain POJOs and changelogs, (4) create thin services with `@Bean` wiring, (5) verify `mi-data-intelligence` grep rule still passes, (6) deploy. The SOP explicitly states: "If you must modify `mi-data-intelligence` to ship your vertical, your strategy implementations are missing an interface contract. File a design review to extend the generic interface before modifying the core."
- Service code review rule: any class in `mi-venue-service` or `mi-venue-processing-worker` that contains more than `@Bean` definitions, `@RabbitListener` annotations, and HTTP DTO translation must be challenged. If the class contains an `if (venue.getCatering() == ...)` business logic line, it belongs in `mi-venue-model`, not the service.

---

## Status

**Accepted.** Generic core + venue-domain library implemented. Vertical-pivot architectural invariant.

---

**Docs:** [Vision](../vision.md) · [Intelligence Layer](../../platform/intelligence.md) · [Architecture](../../platform/README.md) · [Epics](../epics/README.md)
