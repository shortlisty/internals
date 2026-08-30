# D14 — Master Venue Seeding Infrastructure First

> **Audience:** Engineering, product.
> **Purpose:** Document the decision to prioritize master venue catalog infrastructure before implementing tenant venue management features.

---

## Context

With Group A Platform Foundation complete (user signup, tenant isolation, account settings), the next logical step would be to implement Group B Personal Venue Catalog features (create venue, upload documents, view profiles). However, analysis revealed that venue management requires a robust master venue catalog system as a prerequisite.

The master venue catalog serves as:

- A **platform-wide reference dataset** for venue deduplication and gap-filling (not tenant-facing)
- A **background service** for MC_INHERIT provenance during extraction pipeline
- An **autocomplete/suggestion engine** for tenant venue creation UI
- A **deduplication service** to prevent tenants from creating duplicate venues
- A **metadata enrichment source** that operates transparently during venue processing

## Decision

**We will implement the complete master venue seeding infrastructure before proceeding to tenant venue management features.**

This means the following infrastructure must be in place before any tenant can create or manage venues:

### 1. Shared Libraries Foundation

- `shortlisty-data-intelligence` - Domain-agnostic extraction and metadata infrastructure
- `shortlisty-venue-model` - Venue-specific domain models, migrations, and contracts

### 2. Database Schema and Migrations

- `public.master_venue` - Platform-owned reference dataset
- `public.master_venue_alias` - Alternative names for deduplication
- `public.master_venue_external` - External provider record linkage
- All tenant schema tables with proper indexing strategy

### 3. Master Venue Data Model

- JSONB metadata with `_schema_version` field (platform-managed reference data)
- `VenueMetadataMigrator` with versioning chain (backend service component)
- Canonical field set for venue metadata (shared between platform and tenant schemas)
- Provenance tracking via `metadata_sources` (transparent to tenants)

### 4. Seeding Infrastructure

- Liquibase seed migrations with curated venue data (platform bootstrap)
- Platform admin CRUD API for master catalog management (full unrestricted access - any change accepted)
- Deduplication logic using pg_trgm similarity and PostGIS geo-distance (background service with admin override)
- Alias normalization functions (transparent venue name matching with admin controls)

### 5. Storage and Policies

- S3 bucket structure with tenant isolation
- Master catalog import/export paths
- Lifecycle rules and access policies

## Rationale

### Why Infrastructure First?

1. **Platform Service Foundation**: Master venue catalog provides behind-the-scenes services (autocomplete, deduplication, metadata enrichment) that improve tenant UX from day one without exposing platform complexity.

2. **Transparent Deduplication**: Without master catalog matching running in the background, tenant venues will accumulate duplicates that are expensive to clean up later. Tenants benefit from deduplication without seeing the complexity.

3. **Architecture Validation**: Building the complete platform service pipeline validates the schema versioning, provenance tracking, and background enrichment design before tenant-facing features arrive.

4. **Operational Readiness**: Platform admin tooling and seeding infrastructure must be operational before launching tenant venue management, since tenant data quality depends on platform reference data.

5. **Development Efficiency**: Shared libraries and platform services provide stable backend foundation for implementing tenant features without architectural changes.

### Alternative Considered

**Tenant Features First**: Implement venue CRUD without master catalog infrastructure, add master catalog later.

**Rejected because:**

- Would require significant schema changes when platform service infrastructure is added later
- Tenant venues created early would lack proper background deduplication and enrichment services
- Platform service retrofit is more complex than building the foundation first
- Backend architecture assumes platform reference data availability for optimal tenant experience

## Implementation Order

1. ✅ Group A Platform Foundation (Complete)
2. 🚧 Master Venue Seeding Infrastructure (Current Priority)
   - Shared libraries foundation
   - Database schema and migrations
   - Master venue data model and versioning
   - CRUD API endpoints with deduplication
   - Seeding infrastructure for pre-provisioned venues
   - Alias normalization and deduplication system
   - Consistency verification endpoints
   - S3 storage structure and policies
3. ⏳ Group B Personal Venue Catalog (Next)
   - Create/edit/delete venue profiles
   - Document upload and asset management
   - Basic profile views

## Success Criteria

Master venue seeding infrastructure is complete when:

- [ ] 50-200 curated venues are seeded via Liquibase migrations (platform reference data)
- [ ] Platform admin can manage master catalog via API (full CRUD access - create, update, delete any venue data without restrictions)
- [ ] Deduplication prevents >99% of obvious duplicates transparently during tenant venue creation
- [ ] Metadata schema versioning handles forward/backward compatibility across platform and tenant data
- [ ] S3 storage enforces tenant isolation with proper lifecycle rules (platform manages master catalog separately)
- [ ] All database indexes support expected query patterns (platform service performance)
- [ ] Shared libraries provide stable contracts for platform services and tenant-facing applications
- [ ] Tenant venue creation UI can suggest venues from platform catalog without exposing platform complexity

## Impact on Timeline

This infrastructure-first approach adds approximately **2-3 weeks** to the v0.1 MVP timeline but provides:

- **Transparent tenant UX improvements**: Autocomplete, deduplication, metadata enrichment without complexity
- **Stable platform service architecture**: Backend foundation supports tenant features seamlessly
- **Reduced technical debt**: Platform services handle venue data quality behind the scenes
- **Better tenant onboarding**: High-quality venue suggestions and automatic metadata population

The investment pays dividends as soon as tenant venue management features are implemented, providing a polished experience that leverages platform intelligence without exposing platform complexity.

---

**Docs:** [Architecture](../../platform/README.md) · [Data Model](../../platform/data-model.md) · [Master Catalog](../../platform/master-catalog.md) · [Feature Checklist](../feature-checklist.md) · [Milestones](../milestones/README.md)
