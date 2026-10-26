# D16 — Shared venue model and MyBatis utilities as published artifacts

> **Audience:** Engineers, architects.
> **Purpose:** Record why the docintelligence suite now shares code through published Maven artifacts, and what is still duplicated by design.

---

## Context

[D1](D1-one-service-vs-two.md) split the venue domain into a synchronous service and an asynchronous sidecar worker. [D8](D8-master-venue-seeding-infrastructure-first.md) then extended that split into a rule about code: as written at the time, each service was to carry its own domain classes, event payloads, and Liquibase changelogs, and duplication across services was accepted in exchange for zero cross-service compile coupling. [D15](D15-dedicated-master-venue-worker.md) added a third Java service under the same rule.

That rule was written for services, but it was read as a rule about code. The cost of that reading showed up twice.

| Duplicated code                                                               | Copies in the suite | What it cost                                                                                       |
| ----------------------------------------------------------------------------- | ------------------- | -------------------------------------------------------------------------------------------------- |
| MyBatis type handlers (UUID, raw JSON to `jsonb`, `List<String>` to `text[]`) | 8 Java services     | One handler change means eight edits, eight builds, eight releases, and eight chances to drift     |
| `MasterVenue`, `MasterVenueMapper`, `MasterVenueMapper.xml`                   | 3 Java services     | The same statements reviewed three times, differing only in `namespace` and the `typeHandler` FQCN |

The third service shows where the rule leads. `docintelligence-master-venue-worker` began as a copy of `docintelligence-catalog-processing-worker`, venue model and mapper included, because the rule left no other way to give a new service the types of a table it already had to write.

What is not in question is the boundary the rule protects: no service compiles against another service, and no service can break another's build or release. The open question is whether that boundary also forbids libraries that belong to no service.

---

## Options considered

### Option A — Keep the code duplicated per service

Every service keeps its own copy of the venue model, the mapper interface, the mapper XML, and the type handlers it needs.

**Pros:**

- No artifact to version, release, or coordinate across repositories
- Each service can diverge from the shared shape without a release cycle
- Nothing new to publish, sign, or maintain on Maven Central

**Cons:**

- A change to one table's mapping is an N-repository change with N reviews and no compiler help between copies
- Handlers with no service-specific behaviour are maintained as many times as there are services
- A new service starts as a copy of an existing one, which is how the duplication grew to three copies in the first place

### Option B — Publish the genuinely shared code as Maven artifacts

Two artifacts, split by scope:

- `com.iqkv:foundation-mybatis-util` — platform level, no product knowledge: the MyBatis type handlers that map UUID, raw JSON to PostgreSQL `jsonb`, and `List<String>` to `text[]`.
- `com.iqkv:docintelligence-venue` — product level: the `MasterVenue` model, the `MasterVenueMapper` interface, and its mapper XML, shared by all three Java services of the product.

**Pros:**

- One place to change the shape of `public.master_venue` and its persistence layer
- Handlers that carry no service-specific logic are written once
- New services depend on the model instead of copying it
- The published artifact is exactly the contract the ownership rules already assume

**Cons:**

- Two more releases in the suite's release train, both gated by Maven Central publication
- Version bumps travel through `boot-parent-pom`, so adopting a model change in three services is three dependency bumps
- Mapper XML moves out of the consuming service's resources, which changes how MyBatis discovers it

---

## Decision made

**Option B: publish the shared code as Maven artifacts, split by scope.**

`foundation-mybatis-util` holds code that is generic to the platform. `docintelligence-venue` holds the venue bounded context of the product. Services depend on both as ordinary managed dependencies.

The boundary the rule protects is unchanged and now stated more precisely:

- **No service depends on another service.** `docintelligence-catalog-service`, `docintelligence-catalog-processing-worker`, and `docintelligence-master-venue-worker` still share no code, no build, and no image.
- **Shared code lives only in libraries.** An artifact is eligible for extraction only if it has no Spring Boot application, no HTTP surface, no scheduled or event-driven entry point, and no deployment of its own.
- **Extracting a new artifact follows this record.** A new candidate that meets the eligibility test above needs its own decision record before it is created.

Versions are managed centrally: `boot-parent-pom`'s `dependencyManagement` sets both artifact versions, and consumers declare the dependency with no `<version>` of their own.

---

## Rationale

- **The rule protects a deployment boundary, not a source layout.** What D1 wanted was that neither service can break the other at build or deploy time. A library that is never deployed cannot do that, and every service in the suite already consumes `foundation-tenancy` and `foundation-mybatis-util` under exactly this arrangement.
- **Not everything shared between services is a contract.** Event payloads are a contract between two services and must stay independent on each side. A row type and its mapper are the shape of one table, which has one owner, one DDL source, and one meaning. Duplicating the second is not isolation, it is three copies of one model.
- **The duplication was already being paid for.** The three copies of the venue mapper differed only in the values that had to be local, `namespace` and the `typeHandler` FQCN, so no copy could be reviewed as an independent design decision. Every real change had to be made three times.
- **Ownership is untouched.** [services.md § Table ownership](../../platform/services.md) still decides which service may write which table, DDL still lives in `docintelligence-catalog-service`'s changelogs, and the artifact carries no migrations, no scheduler, and no configuration.
- **Cost is bounded and visible.** A model change becomes a release plus three dependency bumps instead of one commit, but it is one edit in one place with the compiler on every consumer's side.

---

## Consequences

- `com.iqkv:foundation-mybatis-util` carries the shared type handlers. Services that kept local copies drop them and reference `com.iqkv.foundation.mybatis.util.*`. Handlers with service-specific behaviour are not eligible and stay local.
- `com.iqkv:docintelligence-venue` carries `MasterVenue`, `MasterVenueMapper`, and the mapper XML. The model and mapper no longer exist in any service's source tree, and each consumer's `@MapperScan` points at `com.iqkv.docintelligence.venue`.
- **Mapper XML now ships inside a jar.** Consumer mapper locations must use `classpath*:mappers/**/*.xml`. The single-star `classpath:mappers/**/*.xml` form resolves only the service's own resources, so the interface is bound without its statements and the first call fails with `Invalid bound statement (not found)`. `docintelligence-catalog-service` was corrected, and this pattern is the one to check when a new consumer is added.
- Removing a local mapper XML does not remove its copy in `target/classes`, so a stale copy survives a non-clean build and surfaces as a duplicate `namespace` error. Rebuild with `clean` after deleting mapper resources.
- One shared mapper interface now serves every consumer, reads and writes alike. Which service may call which method stays a review rule in [services.md](../../platform/services.md) plus the connection-level grants in [data-model.md](../../platform/data-model.md); it is no longer expressible as an interface split.
- Both artifacts are released and all consumers are migrated. [D1](D1-one-service-vs-two.md), [D8](D8-master-venue-seeding-infrastructure-first.md), and [D15](D15-dedicated-master-venue-worker.md) are amended to point here for the code-sharing rule; their service topology decisions are unchanged.

---

## Status

**Accepted.** The two artifacts are published on Maven Central, `boot-parent-pom` manages both versions, and every consumer that carried a local copy has been migrated.

---

**Docs:** [Decisions index](README.md) · [Architecture](../../platform/README.md) · [Service architecture](../../platform/services.md)
