# D6 — Tenant isolation: schema-per-tenant vs. tenant_id column

> **Audience:** Engineers, architects.
> **Purpose:** Record why tenant data is isolated via PostgreSQL schema-per-tenant (`t_{tenantKey}` per organisation) rather than a shared `tenant_id` discriminator column in shared tables, and why this pattern extends to OiQb's venue tables and vector tables.

---

## Context

The iQ Key Value foundation platform serves multiple independent tenants (customer organisations). OiQb inherits the foundation's multi-tenancy model but extends it to venue-specific tables (`venues`, `venue_assets`, `venue_groups`, `item_vectors`, `extraction_jobs`, etc.). The choice of isolation model was originally made in the foundation platform; this decision record documents the rationale explicitly for OiQb's tables and justifies why any deviation in OiQb-specific schema is rejected.

---

## Options considered

### Option A — Shared tables with tenant_id discriminator column

All tenants share one set of tables. Every row carries a `tenant_id UUID NOT NULL` column. Every query includes `WHERE tenant_id = current_tenant()`. Row-level security (RLS) policies enforce the predicate at the database level.

**Pros:**

- Simple DDL. One set of tables, one Liquibase changelog execution on bootstrap (not per-tenant).
- Easy cross-tenant analytics queries (no schema iteration needed for the analytics DB user).
- Low count of PostgreSQL objects; shared buffer cache pressure does not grow linearly with tenant count.

**Cons:**

- **Isolation relies on query predicate correctness, not structure.** A `tenant_id` predicate is code; code has bugs. A developer writing a new mapper, a new report query, a new admin batch job, or a manual debug SQL statement that forgets `WHERE tenant_id = ?` leaks cross-tenant data silently. RLS mitigates but does not eliminate: RLS must be enabled per table, policies must be written per table, the bypassrls attribute on superusers/roles must be audited, and any role with BYPASSRLS or a session-level `SET app.current_tenant_id` mis-configuration silently overrides all protections. For event-planner tenant data (pricing, contacts, custom agency notes), a cross-tenant leak is a contract-terminating security incident.
- **Index and planner quality degrade under wide tenant cardinality skew.** Tenant #1 has 10 venues. Tenant #2 has 50,000 venues. A `WHERE tenant_id = ? AND capacity.max_total >= 200` GIN index scan on JSONB that is optimal for tenant #2's 50K rows (index scan) may be catastrophically wrong for tenant #1's 10 rows (seq scan cheaper, but planner chooses index because shared statistics skew to tenant #2's distribution). PostgreSQL `ALTER TABLE ... SET (n_distinct = ...)` per-column tuning does not help because the skew is per-value-of-tenant_id, not per-column. OiQb's JSONB metadata queries are particularly sensitive to planner mis-estimates because JSONB containment operators have loose selectivity estimates.
- **pgvector per-tenant recall is structurally worse.** IVFFlat cluster centroids are computed on the shared `item_vectors` table. A 10-venue tenant's vectors are clustered alongside a 50,000-venue tenant's vectors; the 10-venue tenant gets worse cosine-distance recall because its clusters are polluted by the larger tenant's distribution. Per-tenant IVFFlat list tuning is impossible when all tenants share one index.
- **Operational tenant actions require predicates, not DDL.** GDPR "right to erasure" for one tenant = `DELETE FROM venues WHERE tenant_id = ?` (and 8 other tables) with potentially millions of row deletions, table bloat, autovacuum storms, and replication lag. Tenant export requires WHERE-clause-scoped `pg_dump` (non-standard, requires custom tooling) rather than a standard `pg_dump -n t_acme0001`. Tenant restore from a PITR snapshot of a single tenant is impossible without a complex import/filter/restore procedure.
- **Audit for cross-tenant access is code-level, not infrastructure-level.** When investigating a suspected leak, security auditors must review every query path in application code and every ad-hoc DB role grant. They cannot prove isolation via structural database inspection alone.

### Option B — Separate PostgreSQL database per tenant

Every tenant gets its own PostgreSQL database with its own users, tables, and connection pool.

**Pros:**

- Hardest possible isolation. Cross-tenant leakage is structurally impossible without network-level access to the tenant's database.
- Per-tenant tuning (work_mem, shared_buffers fraction, autovacuum thresholds) is possible.

**Cons:**

- **Operationally prohibitive at the target scale.** OiQb targets 100 paying agencies at sustainability and 1,000 at scale. 1,000 PostgreSQL databases = 1,000 connection pools × `pool_size=10` minimum = 10,000 persistent PostgreSQL backends. PostgreSQL process-per-connection architecture collapses at this scale without PgBouncer per-database (which adds a routing layer that itself becomes a failure domain). 1,000 separate WAL streams, 1,000 separate base backups, 1,000 separate failover targets. The ops cost per tenant would exceed tenant ACV at this price point.
- **Cross-tenant admin queries (customer support, billing metrics, platform-wide observability) require federation.** Support agents cannot list a venue across all tenants to debug "where did this PDF come from?" without a cross-db query layer.
- **Foundation platform compatibility break.** The existing foundation services (IAM, billing, gateway, audit) all run against a single PostgreSQL instance with schema-per-tenant. Introducing OiQb as "separate database per tenant" means OiQb is the first service in the platform with its own DB fleet topology, requiring dedicated provisioning, backup, failover, and SRE playbooks that do not exist.

### Option C — Schema-per-tenant inside one shared PostgreSQL instance

Each tenant gets its own named schema `t_{tenantKey}`. Tables are created per-schema: `t_acme0001.venues`, `t_acme0001.venue_assets`, `t_acme0001.item_vectors`, etc. A `MyBatisSchemaInterceptor` sets `SET search_path = t_acme0001, public` on each connection before the SQL executes. Queries reference `venues` unqualified — the connection's search_path resolves it. `public` schema contains only the `venue_registry` (platform-curated, read-only) and infrastructure tables.

**Pros:**

- **Structural isolation with auditable proof.** There is no `tenant_id` predicate to forget. A schema is a PostgreSQL namespace object; a role `acme_app` can be granted `USAGE, CREATE ON SCHEMA t_acme0001` and `NOINHERIT`-denied access to every other tenant's schema via `ALTER DEFAULT PRIVILEGES`. Security auditors can enumerate schema grants in one `SELECT schema_name, grantee, privilege_type FROM information_schema.role_schema_grants` query and receive a mathematically complete picture of tenant isolation. Cross-tenant leakage via SQL error is not merely unlikely; it is structurally forbidden by PostgreSQL's grant system unless the DBA explicitly revokes a deny grant.
- **Per-tenant index statistics, planner estimates, and vector index quality.** `t_acme0001.venues` has planner statistics computed only on acme's 200 rows. JSONB GIN, tsvector GIN, PostGIS GIST, and pgvector IVFFlat indexes are all built on the 200-row distribution. A neighbouring tenant with 50,000 venues has its own `t_other02.venues` table with its own indexes, its own statistics, and its own IVFFlat cluster centroids. No cross-tenant statistics pollution.
- **Operational tenant actions are DDL-level, not predicate-level.** GDPR tenant erasure: `DROP SCHEMA t_acme0001 CASCADE` → instant, zero row-deletion bloat, zero autovacuum impact. Tenant export: `pg_dump -n t_acme0001` → standard tooling, one command. Tenant restore from backup: `pg_restore -n t_acme0001` into a clean schema. Bulk tenant migration between PostgreSQL shards: export a schema and import it. All of these are standard PostgreSQL operations with existing platform playbooks.
- **pgvector per-tenant tuning.** IVFFlat `lists` parameter depends on row count: `ceil(sqrt(N))`. Schema-per-tenant allows `ALTER INDEX t_acme0001.item_vectors_cosine_idx SET (lists = 14)` for tenant acme (200 venues × 20 chunks = 4,000 rows) while a larger tenant gets `lists = 224` for its 50,000-venue library. Shared-table Option A has one global `lists` value that is wrong for 90% of tenants.
- **Foundation compatibility.** This is the existing pattern in foundation services (IAM, billing). `foundation-tenancy` library exposes `TenantContext.getCurrentTenant()`, `MyBatisSchemaInterceptor`, `TenantLiquibaseRunner` (runs per-tenant Liquibase changelogs on tenant onboarding). OiQb reuses all of this verbatim with zero custom tenancy code.

**Cons:**

- **Table count grows linearly with tenants.** 1,000 tenants × 8 OiQb tables = 8,000 tables. PostgreSQL catalog tables (`pg_class`, `pg_attribute`) grow proportionally. PostgreSQL comfortably handles 100,000+ tables on modern hardware. A `VACUUM FREEZE` on a 1,000-tenant cluster is well-documented and operationally routine; the foundation already runs this pattern.
- **Liquibase changelog execution runs once per tenant on onboarding.** With 8 tables and ~20 changelog files, this is a ~500 ms one-shot cost per tenant. The platform already has `TenantLiquibaseRunner` with an async onboarding job that runs this outside the request/response cycle; no user-visible latency impact.
- **Cross-tenant admin queries (customer support, billing metrics) require schema iteration.** An admin CTE or PL/pgSQL function enumerates schemas and unions results — ~30 lines of SQL for the "find venue across all tenants" support use case. This is an explicit, auditable code path (an admin-only function with `SECURITY DEFINER` and audit logging) rather than an implicit query predicate that can be omitted.

---

## Decision made

**Option C: Schema-per-tenant inside one shared PostgreSQL instance.**

OiQb inherits the foundation's `foundation-tenancy` library unchanged. All OiQb tenant-specific tables (`venues`, `venue_assets`, `venue_groups`, `extraction_jobs`, `item_vectors`, `venue_metadata_events`, `ai_cost_tracking`) are created inside `t_{tenantKey}`. The `venue_registry` platform reference table lives in `public` and is read-only to tenant connections.

Only one mapper (`RegistryEntryQueryMapper`) may explicitly qualify `public.` in SQL. All other mappers rely on `search_path` set by `MyBatisSchemaInterceptor` and never reference schema names.

---

## Rationale

- **Event-planner data is sensitive and legally risky.** OiQb stores per-venue custom agency pricing, contact names of venue coordinators, internal notes ("this venue manager is unreliable"), and client-approval snapshots. Cross-tenant data leakage is not a theoretical bug; it is an event that destroys customer trust, triggers GDPR notification obligations, and terminates contracts. Structural schema-grant-based isolation provides mathematically auditable proof that no query path can leak data absent explicit DBA-level grant tampering (which is itself auditable). Option A's predicate-based isolation, even with RLS, relies on correct code in every mapper, every report, every ad-hoc debug query — it is not provably correct in the presence of human error.
- **Per-tenant vector and JSONB index quality is a real product requirement, not a premature optimisation.** IVFFlat recall quality depends on cluster centroids matching the query's vector distribution. A 10-venue event agency and a 50,000-venue enterprise AMC have fundamentally different vector distributions. One shared IVFFlat index serves both poorly. pgvector documentation explicitly recommends per-corpus tuning when corpora differ dramatically. Schema-per-tenant makes this tuning free (one ALTER INDEX per tenant).
- **Foundation pattern reuse is the highest-leverage engineering choice.** `foundation-tenancy` already implements schema interceptor, per-tenant Liquibase runner, tenant context propagation, and cross-schema admin query helpers. Reusing this verbatim means OiQb ships zero custom tenancy code, inherits all existing platform security audits and tenancy bug fixes, and is operationally identical to other foundation services from day one. Any other isolation model would require a platform-wide pattern exception and a dedicated tenancy implementation.

---

## Consequences

- **No cross-schema SQL.** No query may JOIN or UNION ALL across tenant schemas in a single statement. Cross-source search (Option D in D4) explicitly runs two separate queries via parallel `CompletableFuture`, not a UNION ALL. This is a documented and static-analysis-enforced rule: `RegistryEntryQueryMapper` is the only code path with `public.` qualification.
- **Per-tenant changelogs are immutable and append-only.** Liquibase changesets in `oiqb-venue-model` must never be edited or reordered after merge. `TenantLiquibaseRunner` runs the changelog sequentially on each tenant schema; editing an old changeset creates checksum mismatches that require tenant-by-tenant manual intervention.
- **PostgreSQL max_locks_per_transaction is tuned upward.** Schema-per-tenant with 1,000 schemas × 8 tables = 8,000 tables; access patterns that touch many tables in one transaction (e.g., the admin cross-tenant search function) require higher lock table capacity. The foundation PostgreSQL image already ships with this tuned.
- **Index maintenance on new tenants is tenant-scoped.** When a changelog adds a new index to `venues`, `TenantLiquibaseRunner` creates it per tenant in the async onboarding job. New tenants get the index on creation. Existing tenants' indexes are created during a rolling window where tenant activity is low; a `CREATE INDEX CONCURRENTLY` wrapper in Liquibase avoids locking the table during the build.

---

## Status

**Accepted.** Foundation platform invariant. OiQb inherits without deviation.

---

**Docs:** [Vision](../vision.md) · [Architecture](../../platform/architecture.md) · [Intelligence Layer](../../platform/intelligence.md) · [Epics](../epics/README.md)
