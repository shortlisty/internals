# D5 — Metadata aggregation concurrency: RabbitMQ FIFO routing per venue

> **Audience:** Engineers, architects.
> **Purpose:** Record why concurrent metadata aggregation writes for the same venue are serialized at the RabbitMQ messaging layer (per-venue FIFO routing), rather than using database-level locks, optimistic locking with retry, or distributed coordination services.

---

## Context

Metadata aggregation is a read-modify-write operation:

```
1. SELECT venues.metadata, venues.metadata_aggregated_at FROM venues WHERE id = ?
2. Merge all unprocessed venue_metadata_events into metadata via conflict-resolution rules
3. UPDATE venues SET metadata = ?, metadata_sources = ?, metadata_aggregated_at = NOW() WHERE id = ?
```

If three extraction jobs for the same venue complete almost simultaneously, three `extraction.completed` events arrive on RabbitMQ. Three consumer threads can read the same stale `metadata` row in parallel, each merge one PDF's fields, and each write back — two of the three writes are lost (Lost Update anomaly). The resulting `metadata` JSONB contains fields from only one of the three PDFs.

The question is how to serialize per-venue aggregation safely and with minimal operational overhead.

---

## Options considered

### Option A — Optimistic locking (version column + retry loop)

Add a numeric `_aggregation_version` column to `venues`. Step 3 becomes:

```sql
UPDATE venues SET metadata = ?, metadata_sources = ?, metadata_aggregated_at = NOW(), _aggregation_version = _aggregation_version + 1
WHERE id = ? AND _aggregation_version = <read_version>
```

If the row count is 0 (another writer won), the consumer sleeps with exponential backoff, re-reads, re-merges, and retries up to N times before DLQ.

**Pros:**

- No messaging-layer configuration. Works with any queue topology.
- Standard pattern, well-understood by engineers.

**Cons:**

- **Live lock risk under sustained parallel load.** If 5 events arrive for the same venue at exactly the same time, four threads retry. Retry backoff overlap can yield repeated collisions; with 8 events the probability of N-fold retries consuming the thread pool becomes non-trivial, especially when combined with the 5-second debounce window also running inside the transaction.
- **Merge is not idempotent.** Re-reading metadata after a lost race and re-applying the same extraction event is logically correct only if the merge function is idempotent per event. If aggregation logic later introduces non-idempotent behavior (e.g., "increment a counter per source"), the retry loop double-applies. The current merge is idempotent by coincidence, not by design contract.
- **Test surface is large and subtle.** Correctness tests require multi-threaded harnesses with precise timing control to reproduce the race. Edge cases: (a) DLQ after N retries leaves the venue with partial data; (b) a very slow merge transaction holds the version lock for longer than the retry backoff, causing all other writers to exhaust their retry budget.
- **No observability for collision frequency without custom metrics.** Optimistic locking failures appear in logs as retry-loop WARN lines; aggregating them into an alertable metric requires custom code.

### Option B — Pessimistic database lock (SELECT ... FOR UPDATE)

Step 1 becomes `SELECT ... FROM venues WHERE id = ? FOR UPDATE`. The row-level exclusive lock serializes all three aggregation transactions for the same venue.

**Pros:**

- Simple, no retries. No race condition at any concurrency level.
- Works with any queue topology.

**Cons:**

- **Contention explodes under debounced rapid events.** If three events for venue X arrive within the 5-second debounce window, the first consumer acquires the lock, runs the full aggregation, commits (releasing the lock). The second consumer acquires the lock, sees `metadata_aggregated_at` within the debounce window, and immediately no-ops and commits. The third consumer same. In the worst case, two consumers blocked on a lock for ~20 ms only to no-op when they finally acquire it. With 10 rapid events, 9 consumers waste their held thread pool slot waiting for a lock only to no-op.
- **Lock hold time includes the merge.** The merge itself is microseconds, but when combined with `VenueMetadataMigrator.ensureCurrent()` (which chains N schema migrations for a stale document), the `FOR UPDATE` lock can be held for 50–200 ms per row. Under parallel load this creates visible queue-depth backpressure.
- **Deadlock corner cases with adjacent writes.** A future feature that writes `venue_metadata_events` inside the same transaction as the aggregation creates a risk of AB→BA deadlock with ingestion-worker writers inserting event rows. The code today avoids this, but the pattern is fragile under maintenance.

### Option C — Distributed lock (Redis/Spring Integration Lock Registry)

Before step 1, acquire a distributed red lock `bene:agg:venue:<venueId>` with TTL. Release after commit.

**Pros:**

- Database is not involved in lock coordination. Lock TTL bounds worst-case stall if a consumer dies mid-transaction.

**Cons:**

- **New infrastructure dependency.** Redis is not in the iQ Key Value foundation stack today. Adding it for one lock pattern expands monitoring, backup, failover, and dependency surface permanently.
- **Correctness of red locks under network partition is famously non-trivial.** The Martin Kleppmann vs. antirez debate aside, engineering teams that infrequently debug distributed-lock issues regularly introduce bugs: TTL shorter than the transaction, no lock fencing token, no renewal heartbeat — each yields silent lost writes when the lock expires mid-operation.
- **Same no-op waste as Option B.** Under rapid debounced events, 8 of 10 consumers acquire and release the lock only to no-op. Thread pool churn is identical to pessimistic DB locking.

### Option D — RabbitMQ FIFO routing per venue_id

Serialize per-venue work before the message ever reaches the consumer. Two variants share the same conceptual model; variant A1 is MVP.

**Variant A1 (MVP):** One queue, one serialising consumer thread for all aggregation events. `@RabbitListener(concurrency = "1", prefetchCount = "1")` on `oiqb.metadata.aggregation`. Exactly one thread processes all aggregation events sequentially across all tenants and all venues.

**Variant A2 (immediately available):** N hash-partitioned queues with per-queue single-threaded consumption. Publisher computes `slot = Math.abs(venueId.hashCode() % SLOT_COUNT)`; routes to `oiqb.metadata.aggregation.{slot}`; 16 consumer threads each bind to one slot queue with `prefetchCount = 1`. Same venue always maps to same slot = strict FIFO per venue. Different venues always process in parallel.

**Pros:**

- **Zero new dependencies.** RabbitMQ is in the platform foundation stack. No Redis, no distributed lock algorithm, no custom optimistic-lock retry library.
- **Zero race conditions by construction.** With `concurrency = 1` per queue and `prefetchCount = 1`, the consumer never pulls a batch and interleaves processing; it takes one message, finishes, acks, and takes the next. Two events for the same venue are processed in published order. No lost updates, no retries, no deadlocks, no TTL fencing, no lock-expiry bugs. The only correctness requirement is single-threaded consumption per queue — a feature RabbitMQ exposes as a first-class listener configuration.
- **5-second debounce composes naturally with FIFO.** Three rapid events for the same venue arrive in order. Event 1 runs aggregation, updates `metadata_aggregated_at = NOW()`. Event 2 is consumed next, detects `metadata_aggregated_at` within 5 s, no-ops and acks. Event 3 same. Outcome: one SQL UPDATE instead of three. The redundant work is eliminated before it reaches the database, not contended inside it.
- **Extraction stays parallel.** Only the final microsecond-scale metadata merge is serialised per venue. The three PDFs' extraction work (parse → chunk → GPT-4o call → embedding generation) runs on separate ingestion-worker threads concurrently, completing at different times. Only the final aggregation step is serialised.
- **Horizontal scalability path is code-free upgrade from A1 to A2.** If queue depth breaches threshold, a config change (slot count from 1 to 16 + publisher routing key suffix) migrates the topology with no consumer code changes and no message loss. Both variants use the exact same consumer transaction boundary code.

**Cons:**

- **A1 MVP: global serialisation limits worst-case throughput.** Aggregation per event is ~1 ms (merge + UPDATE). 1,000 events/s sustained = 1,000 ms backlog. With the 5 s debounce window, users never see this lag, so it is acceptable for MVP. A1→A2 migration eliminates this concern when the product reaches the scale where 1,000 concurrent uploads across all tenants is plausible.
- **Publisher routing key logic (A2 only) must be hash-stable across deployments.** Changing the slot count later requires a double-write migration window (publish to both old and new topology until in-flight messages drain) to avoid same-venue events routing to different slots and interleaving. This is a one-time documented migration, not an ongoing cost.

---

## Decision made

**Option D, RabbitMQ FIFO routing per venue_id.**

Start with Variant A1 (MVP): one queue `oiqb.metadata.aggregation`, single serialising consumer `concurrency = 1, prefetchCount = 1`. Variant A2 (hash-partitioned slots) is documented as the immediate horizontal-scale upgrade path with identical consumer code. The publisher does not change for A1; slot routing is added only when the A1→A2 migration is executed.

The listener container uses `MANUAL` ack mode and encloses the entire aggregation read-modify-write in a single DB transaction. RabbitMQ ack is only issued after DB COMMIT.

---

## Rationale

- **The FIFO pattern eliminates the entire race condition class by construction.** Engineering teams regularly introduce subtle bugs in retry loops, backoff math, lock TTLs, and fencing tokens when using Options A/B/C. With single-threaded consumption per queue, there is no race and therefore no race-condition surface. This is provably correct rather than "probably correct under testing."
- **Aggregation latency is 1 ms per event; global serialisation is invisible at MVP scale.** Even a pessimistic MVP upper bound of 10 concurrent uploads × 3 assets each = 30 events batch = 30 ms backlog. The 5-second debounce window absorbs any jitter; user-visible latency to see extracted fields is governed by the extraction pipeline (20–60 s per PDF), not the 1 ms aggregation step. Global serialisation has zero product impact at MVP and for the foreseeable future.
- **The debounce + FIFO synergy eliminates redundant work before it reaches the database.** Under rapid parallel events for the same venue, Options A/B/C all serialise at the transaction level — meaning 2 of 3 consumers wastefully execute the full transaction (BEGIN, SELECT, merge, rollback or retry or no-op UPDATE and COMMIT) inside the database. FIFO + debounce eliminates the waste before the consumer opens a transaction. Under a burst of 10 same-venue events, Options A/B/C open ~10 DB transactions (9 of which are wasted). FIFO opens 1 transaction, runs aggregation, then no-ops the remaining 9 without DB work.
- **RabbitMQ has platform-proven single-listener semantics.** The foundation billing and audit services already use single-threaded single-prefetch consumers for correctness-critical event processing. The ops team has existing playbooks for queue depth alerting, dead-letter inspection, and consumer restart procedures for this exact pattern.

---

## Consequences

- Consumer transaction boundary is non-negotiable: BEGIN → SELECT (metadata, aggregated_at) → debounce check → merge + migrator.ensureCurrent() → UPDATE → DELETE processed events → COMMIT → RabbitMQ ack. Ack before COMMIT is a merge-time data loss bug and is blocked by ArchUnit/static-analysis rule.
- A1 queue depth alert threshold set at 1,000 messages sustained for 2 minutes. Breaching triggers engineering review for A1→A2 slot migration.
- A2 slot count is a power of two (16 default) and never changed without a documented double-write migration window. The hash function (`Math.abs(venueId.hashCode() % SLOT_COUNT)`) is a utility method shared by publisher and tests; no ad-hoc slot computation.
- Unit tests for the consumer use a seeded in-memory RabbitMQ (testcontainers) and publish three same-venue events simultaneously; the assertion is that the final `venues.metadata` contains merged fields from all three sources. No `CountDownLatch` or multi-threaded test harness is needed (unlike Option A/B lock-based tests where race reproduction is timing-dependent).

---

## Status

**Accepted.** Variant A1 for MVP. Variant A2 documented as the immediate scale-up path.

---

**Docs:** [Vision](../vision.md) · [Architecture](../../platform/architecture.md) · [Epics](../epics/README.md) · [Milestones](../milestones/README.md)
