# VenueMi — Architecture Overview

> **Audience:** Engineers, architects.
> **Purpose:** Platform context, technology decisions, and implementation patterns. Entry point for the full architecture reference.

---

## Document Map

| Document                                           | Contents                                                                                      |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **This file**                                      | Platform context, foundation reuse, tech stack decisions, implementation patterns             |
| [data-model.md](data-model.md)                     | Domain model, canonical field set, schema versioning, database schema & indexes               |
| [services.md](services.md)                         | Service decomposition, shared libraries (`mi-venue-model`, `mi-data-intelligence`), S3 layout |
| [aggregation.md](aggregation.md)                   | Metadata aggregation, conflict resolution, FIFO race-condition prevention                     |
| [master-catalog.md](master-catalog.md)             | Master Venue Catalog — cold start, alias normalisation, MC_INHERIT merge algorithm            |
| [etl-pipeline.md](etl-pipeline.md)                 | ETL pipeline (parse → transform → load), Spring AI stages, processing SLAs                    |
| [search.md](search.md)                             | Search architecture — keyword, semantic, geo, hybrid, cross-source orchestration              |
| [api.md](api.md)                                   | REST API surface — all endpoints, DTOs, error responses                                       |
| [events.md](events.md)                             | RabbitMQ event contracts, plan entitlement mapping                                            |
| [observability.md](observability.md)               | Prometheus metrics, Grafana dashboards, security model                                        |
| [roadmap-decisions.md](roadmap-decisions.md)       | Open decisions, pre-Sprint 1 tasks, Phase 2/3 design backlog                                  |
| [ui-venue-management.md](ui-venue-management.md)   | UI: venue CRUD form, list, field registry, component structure, themes/skins, addon placement |

---

## 1. Platform Context

VenueMi Intelligence is a new product service built **on top of the iQ Key Value foundation**. It does not replace or fork any existing service. It reuses:

| Foundation service           | What VenueMi inherits                                                                   |
| ---------------------------- | --------------------------------------------------------------------------------------- |
| `foundation-gateway-service` | JWT validation, tenant routing, header propagation — no changes needed                  |
| `foundation-iam-service`     | Auth, multi-tenancy, team invitations, SSO, presigned S3 upload pattern                 |
| `foundation-billing-service` | Plan entitlements (`max_venues`, `ai_extraction_enabled`, etc.), subscription lifecycle |
| `foundation-audit-service`   | Compliance log — consumes VenueMi events passively, no code changes                     |
| `foundation-ui-app`          | Extended (not forked) with new `/venues/*` routes under FSD architecture                |
| `foundation-tenancy`         | Schema-per-tenant isolation library reused directly                                     |

**New services introduced by VenueMi Intelligence:**

- `mi-data-intelligence` — platform-level shared library (JAR). Domain-agnostic extraction pipeline contracts, metadata versioning mechanism, provenance model, event POJOs, and Liquibase migrations for infrastructure tables (`extraction_jobs`, `item_vectors`, `item_metadata_events`, `ai_cost_tracking`). No Spring beans, no business logic, no venue-specific fields. The domain-agnostic layer — reusable across verticals (venue, medical, agro, etc.). Imported by both services and by `mi-venue-model`.
- `mi-venue-model` — venue-domain shared library (JAR). Venue-specific domain model (`Venue`, `VenueMetadata`, `MasterVenue`), canonical field set, venue metadata migrations, and Liquibase migrations for venue tables (`venues`, `venue_assets`, `master_venue`). Depends on `mi-data-intelligence`. Imported by both services.
- `mi-venue-service` — core domain: venues, assets, metadata, search, plan enforcement, master catalog backdrop lookup. Synchronous request/response only.
- `mi-venue-processing-worker` — async sidecar: document ETL for tenant uploads, embedding generation, metadata aggregation, scheduled maintenance jobs. Scraping (e.g. Tagvenue) is extracted to standalone Node.js scrapers (`mi-mc-ingest-<source>-scraper`). Master Catalog population runs in `mi-mc-loader` (Spring Boot). No inbound HTTP — event-driven only. Shares the same PostgreSQL schema as `mi-venue-service`.

**New infrastructure introduced by VenueMi Intelligence:**

- pgvector extension on existing PostgreSQL (not a new service)
- PostGIS extension on existing PostgreSQL (not a new service)
- IBM Docling (optional self-hosted container, Phase 2 only)

---

## 14. Technology Decisions

| Concern              | Decision                                                             | Rationale                                                                                                                 |
| -------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Document parsing     | Apache Tika via Spring AI `TikaDocumentReader`                       | 1000+ formats, DWG support, fault-tolerant Tika Pipes, zero extra infra                                                   |
| PDF layout / tables  | IBM Docling (Phase 2, self-hosted)                                   | State-of-the-art table reconstruction, MIT license, zero per-page cost                                                    |
| AI framework         | Spring AI 1.0                                                        | Java-native, provider-agnostic, ETL pipeline built-in, Micrometer integration                                             |
| LLM (extraction)     | OpenAI GPT-4o                                                        | Best structured output + multimodal (vision for images/floor plans)                                                       |
| Embeddings           | OpenAI text-embedding-3-small                                        | 1536 dims, $0.02/1M tokens, good quality/cost ratio                                                                       |
| Vector store         | pgvector (PostgreSQL extension)                                      | No new service, transactional, tenant-isolated via schema                                                                 |
| Full-text search     | PostgreSQL tsvector                                                  | Unified with relational data, no new service                                                                              |
| Geo search           | PostGIS (PostgreSQL extension)                                       | No new service                                                                                                            |
| Async processing     | RabbitMQ (existing foundation)                                       | Priority queues, DLQ, already in platform                                                                                 |
| File storage         | S3 / MinIO (existing foundation)                                     | Presigned URL pattern already proven in IAM                                                                               |
| Shared library split | `mi-data-intelligence` (generic) + `mi-venue-model` (venue-specific) | Enables pivot to other verticals without refactoring infrastructure contracts. Full design in [services.md](services.md). |

Full rationale and competitor analysis: see [`../business/Digital_Sales_Room_for_Events/comparison.md`](../business/Digital_Sales_Room_for_Events/comparison.md).

---

## 16. Implementation Patterns

This section records how core cross-cutting concerns are implemented in the existing foundation services. VenueMi must follow these patterns exactly — they are not aspirational, they are the actual running code.

### Stack

| Concern          | Technology                                 | Notes                                                      |
| ---------------- | ------------------------------------------ | ---------------------------------------------------------- |
| Web tier         | Spring MVC (`spring-boot-starter-web`)     | Blocking servlet stack — no WebFlux except in the gateway  |
| Persistence      | MyBatis (`mybatis-spring-boot-starter`)    | No JPA, no Hibernate, no `@Entity` annotations anywhere    |
| Messaging        | RabbitMQ (`spring-boot-starter-amqp`)      | No Kafka, no Spring Cloud Stream                           |
| Schema migration | Liquibase XML changesets                   | No Flyway, no SQL scripts, no YAML changesets              |
| Domain objects   | Plain Java POJOs                           | No `@Entity`, no `@Column`, no ORM annotations of any kind |
| SQL mapping      | MyBatis `@Mapper` interfaces + XML mappers | `ResultMap` per entity, `SET search_path` via interceptor  |
| Security         | Spring Security OAuth2 Resource Server     | JWT validation via `NimbusJwtDecoder`, RS256 only          |

---

### Multi-tenancy

Tenancy is **schema-level**, not row-level. There is no `tenant_id` column on any table.

**Schema naming:** each tenant gets a PostgreSQL schema named `t_{tenantKey}` (e.g. `t_acme0001`). The `public` schema holds system-level tables (tenants registry, platform users).

**Routing mechanism — `MyBatisSchemaInterceptor`:**

Before every MyBatis statement execution, the interceptor sets:

```sql
SET search_path TO t_{tenantKey}, public
```

This is done via a MyBatis `StatementHandler.prepare` interceptor that reads `TenantContext.getCurrentTenant()`. If no tenant context is active (system-level operation), the interceptor skips and leaves the default `public` search path in place.

**TenantContext:** `ThreadLocal<String>` holder in `foundation-tenancy`. Must be set before any DB call and cleared in a `finally` block after every request.

```java
// Set by TenantExtractionFilter before the request reaches the controller
TenantContext.setCurrentTenant(tenantKey);
try {
    filterChain.doFilter(request, response);
} finally {
    TenantContext.clear();  // always, even on exception
}
```

**`TenantExtractionFilter`** — `@Order(HIGHEST_PRECEDENCE + 1)`, present in every service:

1. Check `X-Tenant-ID` header (set by the Gateway — priority 1)
2. Decode Bearer JWT, read `tenant_id` claim (priority 2)
3. If neither resolves → `400 Bad Request` with inline `application/problem+json` body, request stops
4. Set `TenantContext`, continue filter chain
5. `finally` → `TenantContext.clear()`

**`TenantLiquibaseRunner`** — `ApplicationRunner` startup sequence:

1. Migrate `public` schema (system changelog)
2. If `iqkv.liquibase.upgrade-existing-tenants=true`: discover all `t_*` schemas from `information_schema.schemata` and apply pending tenant changesets to each — failures are logged and skipped, they do not abort startup
3. Apply tenant migrations to any `iqkv.liquibase.bootstrap-tenants` list (idempotent)
4. Always migrate the `platform` tenant (landing zone for new users before org creation)

**When to use `public` schema instead of tenant schema:**

- Use `public` when data is platform-level and shared across all tenants (tenants registry, platform users, master venue catalog, locale lists, token denylist), data volume is small, or cross-tenant queries are needed.
- Use `t_{tenantKey}` when data is tenant-owned business content (venues, assets, CMS pages, extraction jobs), data volume per tenant can grow independently, or hard isolation is required.

---

### Security

**`SecurityConfig` pattern** (identical structure in every consumer service):

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**").permitAll()
                .requestMatchers("/api-docs/**", "/swagger-ui/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .addFilterBefore(correlationIdFilter, BearerTokenAuthenticationFilter.class)
            .addFilterAfter(tenantExtractionFilter, CorrelationIdFilter.class);
        return http.build();
    }

    @Bean
    public JwtDecoder jwtDecoder() {
        if (jwksUri is configured) {
            return NimbusJwtDecoder.withJwkSetUri(jwksUri).build();
        }
        return NimbusJwtDecoder.withPublicKey(loadRsaPublicKey(publicKeyPath)).build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        var converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            List<String> authorities = jwt.getClaimAsStringList(JwtClaimNames.AUTHORITIES);
            if (authorities == null) return List.of();
            return authorities.stream()
                .map(SimpleGrantedAuthority::new)
                .map(a -> (GrantedAuthority) a)
                .toList();
        });
        return converter;
    }
}
```

Key points:

- RS256 only — no HS256, no symmetric keys
- Authority strings are read directly from JWT `authorities` claim — no `ROLE_` prefix
- No `UserContext` class — authority is read from Spring Security's `Authentication` object
- Consumer services validate using JWKS URI (deployed) or local PEM (dev), never a shared secret

**JWT claim names** (wire names from `JwtClaimNames.java`):

| Constant               | Wire key               | Notes                                             |
| ---------------------- | ---------------------- | ------------------------------------------------- |
| `USER_ID`              | `user_id`              | snake_case — not `userId`                         |
| `EMAIL`                | `email`                |                                                   |
| `FIRST_NAME`           | `first_name`           | snake_case — not `firstName`                      |
| `LAST_NAME`            | `last_name`            | snake_case — not `lastName`                       |
| `TENANT_ID`            | `tenant_id`            | 8-char nanoid `tenant_key`, not the internal UUID |
| `AUTHORITIES`          | `authorities`          | `List<String>`, bare strings                      |
| `PLAN_CODE`            | `plan_code`            | Active plan code, used for entitlement checks     |
| `EMAIL_VERIFIED`       | `email_verified`       |                                                   |
| `ONBOARDING_COMPLETED` | `onboarding_completed` |                                                   |
| `PROFILE_COMPLETED`    | `profile_completed`    |                                                   |
| `TYPE`                 | `type`                 | `"access"` or `"refresh"`                         |

**Authority strings** (from `UserServiceConstants.java`):

| Authority        | Scope          | Who holds it                                          |
| ---------------- | -------------- | ----------------------------------------------------- |
| `PLATFORM_ADMIN` | Platform-level | Platform operator. Cross-tenant. Never auto-assigned. |
| `TENANT_OWNER`   | Tenant-scoped  | Tenant owner — full control within their tenant       |
| `ADMIN`          | Tenant-scoped  | Tenant admin — elevated access within their tenant    |
| `MEMBER`         | Tenant-scoped  | Regular authenticated tenant member                   |

`USER` does not exist on this platform. Use `MEMBER` for regular authenticated users. Use `hasAnyAuthority(...)` in `@PreAuthorize`. Never `hasRole(...)`.

---

### REST Controllers

**Pattern 1 — Tenant-scoped (standard).** Controller at `/api/v1/venues/**`. `TenantExtractionFilter` sets `TenantContext` from JWT before the request hits the controller. No tenant path variable.

```java
@RestController
@RequestMapping("/api/v1/venues")
@Tag(name = "Venues", description = "Venue management")
@SecurityRequirement(name = "bearerAuth")
@PreAuthorize("hasAnyAuthority('TENANT_OWNER', 'ADMIN', 'MEMBER')")
public class VenueRestResource {

    @PostMapping
    public ResponseEntity<VenueDtos.VenueResponse> create(
            @Valid @RequestBody VenueDtos.CreateVenueRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(venueService.create(request));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAnyAuthority('TENANT_OWNER', 'ADMIN')")
    public ResponseEntity<Void> delete(@PathVariable UUID id) {
        venueService.archive(id);
        return ResponseEntity.noContent().build();
    }
}
```

**Pattern 2 — Admin cross-tenant.** Controller at `/api/v1/venues/admin/{tenantKey}/**`. `TenantExtractionFilter` excluded for `/admin/` paths. Controller sets `TenantContext` manually in try/finally.

```java
@RestController
@RequestMapping("/api/v1/venues/admin/{tenantKey}")
@PreAuthorize("hasAuthority('PLATFORM_ADMIN')")
public class AdminVenueRestResource {

    @GetMapping
    public ResponseEntity<VenueDtos.VenueSummaryListResponse> getAll(
            @PathVariable String tenantKey,
            @RequestParam(defaultValue = "20") int limit,
            @RequestParam(defaultValue = "0") int offset) {
        try {
            TenantContext.setCurrentTenant(tenantKey);
            return ResponseEntity.ok(venueService.getAllSummary(limit, offset));
        } finally {
            TenantContext.clear();
        }
    }
}
```

---

### DTO Pattern

All DTOs are Java **records**, grouped in a single `{Domain}Dtos.java` container class. Response wrappers for lists include `items` and `totalElements` — no Spring `Page<T>` in API responses.

```java
public final class VenueDtos {
    private VenueDtos() {}

    public record CreateVenueRequest(
        @NotBlank @Size(max = 255) String name,
        String address,
        String description) {}

    public record VenueResponse(
        UUID id,
        String name,
        String address,
        VenueStatus status,
        Object metadata,
        LocalDateTime createdAt,
        LocalDateTime updatedAt) {}

    public record VenueSummaryListResponse(
        List<VenueResponse> items,
        long totalElements) {}
}
```

---

### MyBatis Mapper Pattern

```java
@Mapper
public interface VenueMapper {
    Optional<Venue> findById(UUID id);
    List<Venue> findAll(@Param("limit") int limit, @Param("offset") int offset);
    long countAll();
    void insert(Venue venue);
    void update(Venue venue);
    void updateStatus(@Param("id") UUID id, @Param("status") String status);
    boolean existsById(UUID id);
}
```

SQL in XML mapper. `search_path` is set to `t_{tenantKey}` by `MyBatisSchemaInterceptor` before every statement — mappers write plain table names with no schema prefix.

---

### GlobalExceptionHandler Pattern

One `@RestControllerAdvice` per service. Every handler uses the same `problem(type, title, status, detail, request)` helper that sets `type = "about:blank"`, adds `correlationId` (from MDC) and `requestId` (generated short UUID). No `https://api.iqkv.site/errors/` URIs — the actual implementation uses `about:blank` throughout.

---

## UI Integration (`foundation-ui-app`)

Extend `foundation-ui-app` — do **not** fork. New VenueMi features live under:

```
src/
├── addons/
│   └── venue-management/              ← isolated addon: field registry + utils + CSS tokens
│       ├── index.ts
│       └── registry/
│           ├── field-registry.types.ts
│           ├── venue-field-registry.ts
│           └── registry.utils.ts
├── entities/
│   └── venue/                         ← VenueSummary, VenueDetail, VenueMetadata, VenueAnnotation
├── features/
│   ├── create-venue/
│   ├── edit-venue-metadata/           ← tabbed metadata form, driven by field registry
│   ├── upload-venue-asset/
│   └── venue-quick-fill/              ← inline SEEDED→ENRICHED nudge
├── widgets/
│   ├── venue-list/                    ← list + filters + search bar
│   └── venue-profile/                 ← header, tabs, asset gallery
└── pages/
    ├── venues/                        ← /venues
    └── venues_.$id/                   ← /venues/:id
```

New routes added to TanStack Router:

| Path                   | Auth   | Description          |
| ---------------------- | ------ | -------------------- |
| `/venues`              | Member | Venue list / search  |
| `/venues/new`          | Member | Create venue         |
| `/venues/:id`          | Member | Venue profile        |

Reuse without modification: auth flows, session management, token refresh, team management, billing/entitlements (`FeatureGate`, `useEntitlements`), notification bell.

Full component structure, field registry, theme/skin pattern, and API integration: see [ui-venue-management.md](ui-venue-management.md).

---

**Docs:** [Architecture Index](README.md) · [Data Model](data-model.md) · [Services](services.md) · [Aggregation](aggregation.md) · [Master Catalog](master-catalog.md) · [ETL Pipeline](etl-pipeline.md) · [Search](search.md) · [API](api.md) · [Events](events.md) · [Observability](observability.md) · [Roadmap & Decisions](roadmap-decisions.md)
