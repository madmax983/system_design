# Zero-Downtime Database Migration

> **Prompt:** The core `orders` service runs on a single PostgreSQL primary that is at 80% CPU and 12 TB, with a 2 TB-per-quarter growth rate. It serves 40K QPS and is the source of truth for every order. Migrate it to a horizontally sharded store with no maintenance window, no data loss, and the ability to abort at any point.

Every principal engineer has led one of these, and interviewers at this level ask about it because it tests something a greenfield design cannot: whether you can change a running system's foundation without anyone noticing. The design is 20% target architecture and 80% *transition mechanics*.

## 1. Problem Framing

**What the prompt gets right:** no downtime, no loss, and abortability are exactly the right constraints. "Abort at any point" is the one candidates skip, and it is the one that shapes the whole plan.

**What the prompt gets wrong (or leaves open):**
- "Migrate to a sharded store" presupposes the answer. The first question a principal asks is whether sharding is necessary *now*. 80% CPU on a single primary is usually a query-shape problem or a missing-cache problem before it is a sharding problem. The honest plan has a phase 0: buy 12-18 months with cheaper interventions, and use that time to do the sharding migration without a gun to your head.
- "No data loss" needs precision: no *acknowledged write* is lost, and every read during the migration returns a value at least as fresh as the last acknowledged write *from that client*. Cross-client staleness bounds are negotiable; read-your-writes is not.
- The migration is not one migration. It is: schema changes to support sharding, a data copy, a dual-write period, a verification period, a read cutover, a write cutover, and a decommission. Each has its own rollback.

**The reframe to state out loud:** "I'll treat this as a staged migration with a rollback at every stage, driven by a per-shard (or per-tenant) traffic-routing flag rather than a big-bang switch. Before that, I'll confirm sharding is actually the bottleneck and buy time with cheaper fixes so the migration runs on a schedule instead of under pressure."

## 2. Requirements & SLOs

### Functional
- All existing APIs continue to work unchanged throughout
- Every order is readable and writable at all times
- The migration can be paused, and rolled back to the previous stage, at any stage, without data loss
- After cutover, the old database can be decommissioned with confidence that nothing still depends on it

### Non-Functional
| Requirement | Target | Note |
|-------------|--------|------|
| Availability during migration | Same as today's SLO (99.95%) | The migration is not allowed to consume the error budget |
| Latency during migration | p99 within 20% of baseline | Dual writes add latency; bound it |
| Data loss | Zero acknowledged writes | Verified continuously, not assumed |
| Divergence detection | Any old/new mismatch detected within 1 hour | Before reads are cut over, mismatches are bugs to fix; after, they are incidents |
| Rollback time | < 5 minutes from decision to traffic restored on the previous stage | A rollback that takes an hour is not a rollback; it is a second migration |
| Duration | 3-6 months end to end | Fast is not the goal; boring is |

### Non-Goals
- Not changing the API or the data model beyond what sharding requires
- Not migrating historical cold data in the same operation; cold data can move later or stay
- Not solving cross-shard transactions in this migration; queries that need them are identified up front and handled (see Deep Dive 1)

**The dominant requirement:** *abortability*. It forces the old system to remain authoritative until the last stage, and it forces every stage to be reversible by flipping a flag rather than by running a job.

## 3. Estimation

| Quantity | Estimate | Derivation |
|----------|----------|------------|
| Data | 12 TB, +2 TB/quarter | Given |
| Write QPS | ~4K | 10% of 40K |
| Read QPS | ~36K | |
| Rows | ~3B orders + ~10B line items | ~1 KB/order with items |
| Initial bulk copy | 12 TB at ~200 MB/s throttled ≈ 17 hours per full pass | Throttled to avoid loading the primary; realistically a few days with retries |
| Change stream during copy | 4K writes/s × 3 days ≈ 1B changes to apply after the snapshot | This is why the copy must be followed by CDC catch-up, not a second copy |
| Target shards | 16 initially, sized for 4x growth | 12 TB / 16 = 750 GB each; at +2 TB/quarter, ~3 years before resharding |
| Dual-write overhead | +1 round trip per write (~2-5 ms), +1 write of 4K/s to the new store | Within the 20% latency budget |
| Verification | Compare 3B rows over the dual-write period: ~1K rows/s sustained | Trivial load; the window is the constraint, not the rate |

### Constraints the numbers force

1. **A bulk copy cannot be consistent by itself.** 17+ hours of copying against 4K writes/s means the copy is stale before it finishes. The copy must be a *snapshot plus change stream*: take a consistent snapshot (Postgres snapshot export or a replica frozen at an LSN), copy it, then apply the CDC stream from that LSN forward until caught up. This is a standard technique; naming the LSN is the detail that shows you have done it.
2. **16 shards with a 4x growth horizon is ~3 years.** Choosing the shard count is a one-way door only if you cannot reshard. So the target store must support resharding (consistent hashing with virtual shards, or a directory-based mapping) from day one. Pick 1,024 virtual shards mapped to 16 physical, so that the next migration is a rebalance rather than a redesign.
3. **Dual writes at 4K/s are cheap; dual-write *correctness* is expensive.** See Deep Dive 2.

## 4. Architecture: The Target and the Transition

### Target
```
                 ┌────────────────────┐
  Orders API ───▶│  Data Access Layer │
                 │  (DAL)             │
                 │  - shard router    │
                 │  - traffic flags   │
                 │  - dual-write      │
                 │  - shadow-read     │
                 └──┬──────────────┬──┘
                    │              │
        ┌───────────▼───┐    ┌─────▼──────────────────────────────┐
        │ Legacy        │    │ Sharded Store                      │
        │ Postgres      │    │  1024 vshards → 16 physical shards │
        │ (12 TB)       │    │  (Postgres per shard, or a         │
        │               │    │   distributed SQL store)           │
        └───────┬───────┘    └────────────────────────────────────┘
                │ CDC (logical replication)
                ▼
        ┌───────────────┐
        │ Migration     │  bulk copy + CDC apply + continuous verify
        │ Pipeline      │
        └───────────────┘
```

**The DAL is the migration.** Every read and write goes through one code path that consults a per-vshard flag: `{legacy_only, dual_write_legacy_primary, dual_write_new_primary, new_only}` for writes and `{legacy, shadow_compare, new}` for reads. If the service already has a DAL, the migration starts there. If it does not, building one is phase 0 and is the largest engineering cost of the whole project. Services that construct SQL in fifty places cannot be migrated safely; that is a finding to report, not to work around.

### Shard key choice

`customer_id`. Justification, in the order that matters:
1. **Every latency-sensitive query is customer-scoped** ("my orders", "place order for customer X"). Co-locating a customer's orders on one shard makes those single-shard.
2. **Blast radius**: a shard outage affects 1/16 of customers entirely rather than 1/16 of every customer's orders.
3. **The queries it breaks** are enumerated up front: "orders for merchant Y" and "orders by status across all customers" become scatter-gather or need a secondary index. Both are reporting-shaped, not transactional; they move to a read model fed by CDC. That is the price, and it is stated.

The rejected key, `order_id` (hash), gives perfect distribution and makes *every* customer query cross-shard. Distribution is a smaller problem than query locality.

## 5. Deep Dives

### Deep Dive 1: The staged plan, with a rollback at every stage

| Stage | What changes | Authoritative store | Rollback | Exit criterion |
|-------|-------------|--------------------|---------|----------------|
| **0. Prepare** | Build/fix the DAL; add `customer_id` to every table that lacks it (backfill); enumerate cross-shard queries and build read models for them; buy time (indexes, cache, read replicas) | Legacy | N/A (nothing risky yet) | Every query goes through the DAL; every table has the shard key; p99 headroom restored |
| **1. Bulk copy + CDC catch-up** | Snapshot at LSN, copy, apply CDC until lag < 1 s | Legacy | Delete the new store | CDC lag stable under 1 s for 24 h; row counts match |
| **2. Dual write, legacy primary** | DAL writes to legacy (sync, authoritative), then to new (sync, best-effort with retry queue) | Legacy | Flag → `legacy_only`; new store is discarded or re-synced | Dual-write failure rate < 0.01% for 1 week; verification clean |
| **3. Shadow reads** | DAL reads legacy (served), also reads new, compares, logs mismatches; **no user-visible change** | Legacy | Flag → `legacy` reads | Mismatch rate < 1 in 10^6 for 1 week, with every remaining mismatch explained |
| **4. Read cutover, per vshard** | Reads served from new; legacy still written first | Legacy (writes) / New (reads) | Flag → `legacy` reads, instant | Latency and error SLOs hold for 1 week at 100% of vshards |
| **5. Write cutover, per vshard** | DAL writes to new (sync, authoritative), then to legacy (sync, best-effort); **reverse CDC** from new → legacy keeps legacy complete | New | Flag → `dual_write_legacy_primary`; legacy is complete because of reverse dual-write | 2 weeks stable; legacy shows zero reads in access logs |
| **6. Decommission** | Stop writing to legacy; snapshot it; keep read-only for 90 days; delete | New | Restore from snapshot (this is the only stage without a fast rollback, which is why it waits 90 days) | Nothing has read legacy in 90 days |

**The two ideas that make this work:**
- **Per-vshard flags.** Cutover is 1,024 small cutovers, not one. Start with 1 vshard (0.1% of customers), then 10, then 100. A problem at 1% is an incident for 1% of customers and a flag flip; at 100% it is a headline.
- **Symmetric dual-write.** In stage 5, the new store is primary and legacy is the secondary, written synchronously. That means rollback to stage 4 or 2 is a flag flip, because legacy never fell behind. Most failed migrations skipped this step, cut writes over with legacy going stale, and discovered that "rollback" meant "re-migrate backwards."

### Deep Dive 2: Dual-write correctness

Dual writes are where migrations silently corrupt data. The failure cases, and the mechanism for each:

| Case | What goes wrong | Mechanism |
|------|----------------|-----------|
| Write to legacy succeeds, write to new fails | New store is missing the row | The DAL enqueues the failed write (with the legacy row's version) to a **repair queue**; a worker retries; the verifier catches anything the queue misses |
| Two concurrent writes to the same row, applied in different orders on the two stores | Stores diverge permanently | Every row carries a monotonic `version` (legacy's `xmin`-derived or an explicit column). Writes to the new store are conditional: `WHERE version < :incoming`. Out-of-order application is rejected and repaired from legacy |
| Write to legacy is in a transaction that later rolls back, but the new write already committed | New store has a phantom row | Dual write happens *after* the legacy commit, not inside its transaction. The cost is the window where legacy has the row and new does not, which the repair queue and verifier cover. The alternative (2PC across stores) is rejected for the same reasons as everywhere else |
| Process crashes between the legacy commit and the new write | Same as row 1 | Same: the verifier is the backstop. **CDC from legacy is the better backstop**: it sees the committed row regardless of process state and applies it to the new store idempotently. In practice, the dual-write path is for latency and the CDC path is for correctness; both run |
| Schema drift: a column exists in one store and not the other | Writes fail or drop data | Schema changes during the migration go through expand/contract on *both* stores, new first, as a separate deploy stage |

**The honest summary:** dual-write from the application is *not* sufficient for correctness. It is sufficient for keeping the new store fresh enough to serve reads. Correctness comes from CDC (idempotent, versioned apply) plus continuous verification. State this; candidates who present dual-write as the correctness mechanism are the ones who have not done it.

### Deep Dive 3: Continuous verification

The verifier is the component that turns "we think it worked" into "we know it worked." Three layers:

1. **Row-level compare, sampled.** Every minute, pick 1,000 random primary keys, read both stores, compare canonical serializations. Log every mismatch with both versions. Target: < 1 in 10^6, and every mismatch has a known cause.
2. **Aggregate checksums, per vshard.** Hourly, compute `count(*)`, `sum(hash(row))`, and `max(updated_at)` per vshard on both stores. A mismatch localizes the problem to one vshard. Runs against replicas so it never loads the primary.
3. **Read-path shadow compare (stage 3).** The real query workload, compared in production. This is the only layer that catches *query-semantic* differences (collation, timezone handling, float rounding, NULL ordering), which sampled row compares miss because they compare storage, not results.

Mismatch triage during stage 3 finds real bugs every time. Typical findings: a timestamp column with different precision; a text column with a different collation making `ORDER BY name` differ; a `NULL`s-first vs `NULL`s-last default. Each becomes a fix or a documented, accepted difference before reads cut over.

### Deep Dive 4: Buying time (phase 0 in detail)

Because this is what a principal actually does first:

| Intervention | Typical gain | Cost | Duration bought |
|-------------|-------------|------|----------------|
| Kill the top 5 queries by total time (usually missing indexes or N+1) | 30-50% CPU | Days | 3-6 months |
| Add a read-through cache for the hottest read path (order lookups by ID) | 20-40% of read QPS | Weeks | 6 months |
| Move read-heavy reporting queries to a read replica | 10-30% CPU | Weeks | 6 months |
| Archive orders older than 2 years to cold storage with a stub row | 30-50% of data | Weeks | Slows growth; helps copy time |
| Vertical scale-up | 2x headroom | Days | 6-12 months, once |

Done together, these commonly buy 12-18 months. That time is spent building the DAL and running the migration on a schedule, with the team's best people, rather than in a war room. The principal-level framing: "the migration's biggest risk is being rushed; phase 0 removes the rush."

## 6. Invariants

| Invariant | Where enforced | Verification |
|-----------|---------------|--------------|
| Exactly one store is authoritative for each vshard at any moment | The per-vshard flag; the DAL reads it once per request and holds it for the request's lifetime | Flag change log; a request never spans two flag states |
| An acknowledged write is durable in the authoritative store | The DAL acks only after the authoritative store commits | Standard |
| The non-authoritative store converges to the authoritative one | CDC with idempotent, versioned apply; repair queue; verifier | Verifier mismatch rate |
| Rollback never loses acknowledged writes | The non-authoritative store is kept complete via synchronous dual-write plus CDC | Before each cutover step, verifier confirms the *other* store is complete |
| Read-your-writes across the cutover | Reads and writes for a vshard flip together; the flag flips reads first (stage 4) while writes still go to legacy first, so a reader on the new store sees writes that already replicated; the residual lag (< 1 s) is covered by the DAL routing reads to legacy for a key written in the last 2 s by the same client | Synthetic probe per vshard |

## 7. Failure Modes

| Failure | Stage | Behavior | Response |
|---------|-------|----------|----------|
| New store down | 2-4 | Dual writes fail → repair queue grows; reads (stage 4) fail → DAL falls back to legacy automatically | Alert on repair queue depth; automatic read fallback is a design requirement, not a runbook step |
| Legacy down | 5 | Dual writes to legacy fail → repair queue grows; new is authoritative so users are unaffected | Rollback to stage 4 is now *not* possible until legacy is repaired and caught up; that is acceptable and stated |
| CDC pipeline stalls | 1-5 | Non-authoritative store falls behind | Lag alert at 10 s; cutover steps are blocked while lag > 1 s |
| Verifier finds a mismatch spike | 3+ | Something is systematically wrong | Automatic: pause any in-progress vshard flips. Human: triage. This is the only "stop the migration" trigger and it must be automatic |
| Flag service down | Any | The DAL cannot read routing flags | DAL caches the last-known flags locally and continues; a flag-service outage freezes the migration but not the product |
| Hot vshard (one huge customer) | 4-5 | One physical shard overloaded | Directory-based mapping lets a single vshard be moved to a dedicated physical shard; this is the reason for 1,024 vshards over 16 physical |
| A developer bypasses the DAL with a direct query | Any | Reads stale data from legacy after cutover, or writes to legacy only | Legacy database credentials are rotated at stage 5 so only the DAL has them; access logs on legacy are alarmed |
| Long-running transaction on legacy blocks the snapshot LSN | 1 | Copy cannot start consistently | Identify and terminate; this is a pre-flight check |

## 8. Evolution Path

This example *is* an evolution path. After decommission:

- **Resharding (year 3):** move vshards between physical shards using the same DAL flags and the same copy-CDC-verify-cutover loop, per vshard. The machinery built for the migration is the machinery for every future rebalance. That is the argument for building it properly rather than as a one-off script.
- **Cross-shard queries:** the CDC-fed read models built in phase 0 become the reporting layer permanently.
- **The DAL becomes the service's storage abstraction** and the next migration (say, to a different engine) starts at stage 1, not stage 0.

## 9. Cost Model

| Cost | Driver | Note |
|------|--------|------|
| Running two stores for 3-6 months | Double storage and compute for the duration | Budget it explicitly; it is the price of abortability |
| Engineering time | The DAL (if absent), the verifier, the migration pipeline | Typically 3-5 engineers for two quarters; the DAL is the majority |
| Dual-write latency | +2-5 ms p50 on writes | Within the budget, but announce it |
| Verification | Trivial compute | Runs on replicas |
| Legacy retention (90 days read-only) | One snapshot plus a small instance | Cheap insurance |

**The insight:** the expensive part is not infrastructure; it is building the DAL and the verifier properly. Both are reusable. A migration done cheaply (no DAL, no verifier, big-bang cutover) is the one that costs the most, once.

## 10. What Breaks at 10x

At 120 TB and 400K QPS:

**First: bulk copy duration.** 120 TB at 200 MB/s is a week per pass, and the CDC backlog during that week is 40K writes/s × 600K s = 24B changes. Fix: copy per vshard in parallel from replicas, each with its own LSN, so the CDC catch-up per vshard is bounded.

**Second: verification coverage.** Sampling 1,000 rows/minute against 30B rows is 1 in 50,000 coverage per month. Fix: verify by CDC (every change is verified once, shortly after it applies) instead of by random sampling; plus the aggregate checksums.

**Third: dual-write throughput.** 40K/s of synchronous dual writes with retries under load. Fix: the application dual-write becomes async through the outbox, and CDC is the only correctness path; read cutover waits for lag, which is now the critical metric.

## 11. Rejected Alternatives

| Alternative | Why it loses |
|-------------|-------------|
| **Big-bang cutover in a maintenance window** | The prompt forbids downtime, but more importantly: there is no rollback. If a query-semantic difference surfaces after cutover, you are stuck |
| **Migrate by table instead of by vshard** | Cross-table transactions span both stores mid-migration; correctness is impossible to reason about. Migrate by *partition of data*, never by *partition of schema* |
| **Application-level dual-write as the correctness mechanism** | Covered in Deep Dive 2: it cannot survive crashes between writes. CDC + verification is the mechanism; dual-write is for freshness |
| **2PC across old and new stores** | Latency, blocking, and the coordinator becoming the failure mode; and it does not solve query-semantic drift, which is where the real bugs are |
| **Shard by order_id for perfect distribution** | Makes every customer-scoped query cross-shard. Locality beats distribution |
| **Skip phase 0 and start sharding now** | The migration runs under pressure, with a service that has no DAL and fifty ad-hoc SQL call sites. This is how migrations fail |
| **Logical replication tools alone (no DAL, no flags)** | Gets data across but gives no per-vshard traffic control and no shadow reads; the cutover is still big-bang |

## 12. Level Signals

**A senior answer** picks a shard key, copies the data with a replication tool, dual-writes, and switches over. Might mention a rollback plan.

**A staff answer** stages the migration, uses CDC for catch-up, dual-writes with a repair queue, verifies with sampling, and cuts over reads before writes. Has a rollback per stage.

**A principal answer** does all of that, and additionally:
- Questions whether sharding is needed now, and buys time with phase 0 so the migration runs on a schedule
- Identifies the DAL as the real project and the absence of one as the real risk
- Makes cutover per-vshard, and makes the symmetric reverse dual-write the mechanism that keeps rollback a flag flip even after write cutover
- States plainly that application dual-write is for freshness, not correctness, and that CDC plus continuous verification is the correctness mechanism
- Uses shadow reads to find query-semantic drift, and can name the kinds of drift that actually show up
- Chooses 1,024 virtual shards so that the *next* migration is a rebalance, and reuses the machinery for it
- Rotates legacy credentials at write cutover so the invariant "nothing bypasses the DAL" is enforced, not hoped

**Interviewer follow-ups to expect:**
- "You're at stage 5 and legacy goes down. Can you roll back?" (Not until legacy is repaired and caught up via reverse CDC. Users are unaffected because new is authoritative. The rollback option is temporarily unavailable, and that is acceptable because the trigger for rollback would be a problem with the new store, which is healthy.)
- "How do you handle a schema change that ships mid-migration?" (Expand/contract on both stores, new store first, as its own deploy. The DAL writes both old and new columns during the expand phase. Contract only after the migration completes.)
- "What did the shadow reads find?" (Collation ordering, timestamp precision, NULL sort order, float formatting. Each is fixed or documented as an accepted difference before stage 4.)
- "The product team wants a cross-customer 'all orders by status' page during the migration." (It is a scatter-gather or a read model. Build the read model from CDC in phase 0. It is faster than the legacy query anyway.)
