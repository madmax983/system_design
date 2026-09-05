# Multi-Region Active-Active Data Platform

> **Prompt:** Your consumer product has 500M users across North America, Europe, and Asia-Pacific. Today everything runs in one US region and p99 latency for APAC users is 800 ms. Design the move to an active-active multi-region architecture. Users must be able to read and write from their nearest region, and a region outage must not take down the product.

This prompt sounds like an infrastructure problem. It is a *data semantics* problem. The infrastructure (global load balancing, replication) is well understood. The hard part is deciding, per piece of data, what "correct" means when two regions accept writes at the same time.

## 1. Problem Framing

**What the prompt gets right:** latency is a real problem, and regional isolation for availability is a real requirement.

**What the prompt gets wrong:**
- "Active-active for everything" is not a requirement; it is an implementation the prompt has pre-selected. Most data has a natural home. A user's profile, settings, and private content are written almost exclusively by that one user, from one place. That data wants a *home region* with local reads and writes, not multi-master. Only genuinely shared, concurrently-written data (a group chat, a collaborative document, a follower count) needs a multi-master story.
- "Read and write from the nearest region" conflates two goals. Reads from the nearest region is cheap and always right. Writes to the nearest region is only right if the write's *owner* is nearby, which for home-region data it is by construction.
- "A region outage must not take down the product" does not mean every feature must survive. It means the *core* product must. A principal engineer forces the product team to rank features by what may degrade during a region outage.

**The reframe to state out loud:** "I'm going to partition the data model into three classes: home-region data (the majority: single writer, replicated for reads and failover), globally-replicated reference data (small, rarely written, read everywhere), and genuinely multi-master data (the minority: designed case by case with an explicit conflict-resolution strategy). Then I'll design the routing, replication, and failover for each class. The interesting design is in the third class and in failover."

## 2. Requirements & SLOs

### Functional
- Users read and write their own data with local-region latency
- Users interact with shared data (groups, threads, shared documents) with other users in any region
- A user can travel: their experience follows them, with a degraded-but-working mode when far from home
- Data residency: EU users' personal data is stored and processed in EU regions (a hard legal requirement, not a preference)

### Non-Functional
| Requirement | Target | Note |
|-------------|--------|------|
| Read latency (own data) | p99 < 100 ms | Served from the local region |
| Write latency (own data) | p99 < 150 ms | Home region, which is local for most users |
| Shared-data write latency | p99 < 300 ms | May involve cross-region coordination for some data classes |
| Availability during single-region outage | Core product 99.9%; full product best-effort | Ranked degradation, see failure modes |
| Cross-region replication lag | p99 < 2 s | Sets the staleness bound for reads of another user's data |
| Data loss on region failure | Zero for home-region data with sync intra-region replication; bounded and reconcilable for multi-master data | Different classes, different guarantees |
| Data residency | 100% compliance, provable | Auditable placement |

### Non-Goals
- Not designing the global CDN for static assets (assumed)
- Not designing the identity provider; it is a globally-replicated reference service consumed by this design
- Not solving the "user physically moves continents permanently" case in v1: home-region migration is a v2 tool

**The dominant requirement:** *data residency*. It is legal, it is binary, and it forces a home-region model for personal data regardless of latency considerations. Everything else is a trade-off; this one is not.

## 3. Estimation

| Quantity | Estimate | Derivation |
|----------|----------|------------|
| Users | 500M | Given; ~200M NA, 150M EU, 150M APAC |
| Daily active | 150M | 30% |
| Reads/s peak | ~1.5M | 150M DAU × 100 reads/day × 5x peak / 86,400 |
| Writes/s peak | ~150K | 10:1 read:write |
| Multi-master writes/s | ~15K | ~10% of writes touch genuinely shared data |
| Per-user data | ~10 KB hot, ~1 MB total | Profile, settings, recent activity vs. full history |
| Hot dataset | ~5 TB | 500M × 10 KB |
| Cross-region replication bandwidth | ~150K writes/s × 1 KB × 2 destinations = ~300 MB/s | Continuous, per direction pair |

### Constraints the numbers force

1. **150K writes/s globally is ~50K/s per region.** Each region needs a sharded primary store. The shard key is user ID for home-region data (see the [ledger example](global-payments-ledger.md) for why shard key choice is a one-way door).
2. **300 MB/s of cross-region replication is cheap in bandwidth but expensive in *lag variance*.** Replication lag is what determines the user-visible staleness of other people's data. A 2 s p99 lag target means the replication pipeline needs headroom: size for 3x steady-state throughput.
3. **15K/s of multi-master writes is small enough** that a per-data-class strategy (CRDTs for some, single-leader-per-object for others) is affordable. If this number were 150K/s the design would need to be more uniform.
4. **5 TB of hot data replicated to three regions is 15 TB.** Fine. The cold tier (500 TB) is replicated only for durability, not for serving, and can live in two regions.

## 4. Architecture

```
                      ┌──────────────── Global Control Plane ───────────────┐
                      │  Placement Service (user → home region, shard)      │
                      │  Config / Feature Flags (globally replicated)       │
                      │  Global Traffic Manager (GeoDNS + Anycast)          │
                      └──────────────────────────┬──────────────────────────┘
                                                 │ (read-only cache in every region;
                                                 │  regions function if control plane is down)
        ┌──────────────────────┐   ┌─────────────┴──────────┐   ┌──────────────────────┐
        │     Region: NA       │   │      Region: EU        │   │     Region: APAC      │
        │                      │   │                        │   │                      │
        │  Edge / API Gateway  │   │  Edge / API Gateway    │   │  Edge / API Gateway  │
        │        │             │   │        │               │   │        │             │
        │  Request Router ─────┼───┼─── forwards non-home ──┼───┼──▶ (home region)     │
        │        │             │   │        writes          │   │                      │
        │  Home-Region Store   │◀──┼── async replication ───┼──▶│  Home-Region Store   │
        │  (sharded; primary   │   │   (CDC → replication   │   │  (sharded)           │
        │   for NA users)      │   │    bus → apply)        │   │                      │
        │        │             │   │                        │   │                      │
        │  Replica of EU/APAC  │   │  Replica of NA/APAC    │   │  Replica of NA/EU    │
        │  (read-only)         │   │  (read-only)           │   │  (read-only)         │
        │                      │   │                        │   │                      │
        │  Multi-Master Store  │◀──┼── bidirectional, ──────┼──▶│  Multi-Master Store  │
        │  (per data class)    │   │   conflict-resolving   │   │                      │
        │                      │   │                        │   │                      │
        │  Reference Data      │◀──┼── globally replicated ─┼──▶│  Reference Data      │
        └──────────────────────┘   └────────────────────────┘   └──────────────────────┘
```

### The three data classes

| Class | Examples | Writes | Reads | Replication | Conflict model |
|-------|----------|--------|-------|-------------|----------------|
| **Home-region** | Profile, settings, private content, purchase history, personal data subject to residency | Home region only (router forwards) | Any region (async replica; staleness marker) | Async CDC, one direction per object | None: single writer |
| **Reference** | Product catalog, config, feature flags, geo data, currency tables | One designated region, low rate | Everywhere | Async, all regions | None: single writer, tolerant of seconds of lag |
| **Multi-master** | Group membership, chat threads, collaborative docs, counters, presence | Any region | Any region | Bidirectional | **Per data class, chosen explicitly** (Deep Dive 1) |

### Request routing for a home-region write from a traveling user

```
1. APAC user (home = APAC) is in Europe, opens the app
2. GeoDNS sends them to the EU region
3. EU edge authenticates; the token carries home_region=APAC (cached from placement service)
4. Read of own profile: served from EU's replica of APAC data, with staleness header (typically < 2 s)
5. Write to own profile: EU request router forwards to APAC over the inter-region backbone
   (~150-250 ms added); response returns through EU
6. Subsequent read in EU may not yet reflect the write (replication lag)
   → read-your-writes: the write response returns a version token; the client sends it on the
     next read; if the EU replica is behind that version, the router forwards the read to APAC
```

Read-your-writes via version token is the cheapest way to hide replication lag from the one person who would notice it. It does not help user B see user A's write faster; that is bounded by replication lag, and the product accepts it.

## 5. Deep Dives

### Deep Dive 1: Multi-master conflict resolution, per data class

There is no single correct conflict strategy. A principal answer enumerates the data classes, picks a strategy for each, and can defend each one to a product manager.

| Data | Strategy | Why | What the user sees on conflict |
|------|----------|-----|-------------------------------|
| **Counters** (likes, view counts) | CRDT: PN-Counter (per-region increment/decrement, merged by sum) | Commutative by nature; exact eventual value; no coordination | Count may briefly differ by region; converges within replication lag |
| **Sets** (group membership, followers) | CRDT: OR-Set (observed-remove) | Add and remove commute correctly; concurrent add/remove resolves to "add wins," which matches user intent | Rare: user removed in one region and re-added in another ends up added |
| **Last-writer-wins fields** (status message, display name if edited from two devices) | LWW with hybrid logical clock, per field | Fields are independent; the loser is the user's own older write | The most recent edit wins, per field, not per record |
| **Chat messages / append-only feeds** | Append with per-region sequence + HLC for global order; no conflicts possible on insert | Inserts never conflict; ordering is resolved by (HLC, region_id) deterministically | Messages may briefly appear in different order across regions, then settle |
| **Collaborative documents** | Operational transform or sequence CRDT (RGA / Yjs-style) | The only correct answer for concurrent text editing | Both edits are preserved; the merge is deterministic |
| **Inventory / anything with a hard constraint** | **Not multi-master.** Single leader per object, with the leader placed in the region where most writes originate | A constraint like "stock ≥ 0" cannot be maintained by merging; someone must serialize | Cross-region write pays ~150 ms; that is the cost of the constraint |
| **Money** | Not in this system. See the [ledger example](global-payments-ledger.md) | A ledger has invariants that no merge function preserves | N/A |

**The principal-level point:** the hardest part is not the algorithms; it is *refusing* to make something multi-master when it has an invariant that merging cannot preserve. The question to ask for each data class is: "Write down the merge function. Does it preserve every invariant the product depends on? If you cannot write it down, the data is not multi-master."

**Hybrid logical clocks, and why not wall clocks:** wall clocks across regions disagree by milliseconds to seconds. LWW on wall-clock time means a region with a fast clock always wins, which is a silent data-loss bug. HLC combines a physical clock with a logical counter so that causally-related events are always ordered correctly, and concurrent events are ordered deterministically. Clock sync (NTP with monitoring, or better) is a hard operational dependency, and it belongs on the list of "dependencies everyone forgets."

### Deep Dive 2: The replication pipeline and its failure modes

Each region's home-region store emits a CDC stream. That stream is published to a replication bus (a Kafka cluster per region, mirrored) and applied to replicas in other regions.

**Design points that matter:**

- **Per-shard ordering.** Each shard's CDC stream is one partition. Cross-shard order is not guaranteed and does not need to be, because home-region data has one writer per object and objects live on one shard.
- **Idempotent apply.** Each change carries the source shard's log position. The applier records the high-water mark per (source region, shard) in the same transaction as the apply. Replays after failure are safe.
- **Schema evolution travels with the data.** A schema change in the home region is emitted as a CDC event *before* the first row that uses it; appliers block on unknown schema versions rather than guessing. See the [event streaming example](event-streaming-platform.md) for schema registry mechanics.
- **Lag is a first-class SLI.** Per (source, destination, shard). Alert at 5 s, page at 30 s. Lag is the number that becomes user-visible staleness and, during failover, becomes data loss.
- **Backpressure, not unbounded buffering.** If a destination cannot keep up, the bus retains (7 days) and the applier falls behind visibly. The source never slows down for a replica.

**Failure: replication bus partition between two regions.** Home-region writes continue. Replicas in the cut-off region go stale, and the staleness marker on reads grows. When the partition heals, the applier catches up from the retained log. Nothing is lost; users saw stale data for the duration. The product decision is: at what staleness does a replica *refuse* to serve and instead forward reads to the home region? A reasonable default is 60 s for most data and "never refuse" for data where stale is better than unavailable (a profile picture).

### Deep Dive 3: Region failover and the placement service

A region outage has three sub-problems: routing traffic away, deciding what to do about the failed region's home-region data, and doing it without the control plane becoming the next single point of failure.

**Routing.** GeoDNS TTLs are set to 60 s; anycast at the edge handles the fast path. Clients also carry a fallback region list so that a DNS-level failure is not a total failure. Within ~2 minutes, traffic from the failed region's users lands in the next-nearest region.

**Home-region data of the failed region.** This is the same durability-versus-availability decision as in the ledger, with a different answer because the data is different:

| Data | Policy during the failed region's outage |
|------|-----------------------------------------|
| Reads of failed-region users' data | Served from the async replica in the surviving region, with a staleness marker; the last ~2 s of writes before the failure may be missing |
| Writes by failed-region users | **Accepted** in the surviving region into a *write-ahead queue* keyed by user, applied to the replica for read-your-writes, and replayed to the home region when it returns |
| Conflict when the home region returns | The failed region's last-2-seconds and the queue's writes are merged per data class using the same strategies as Deep Dive 1; home-region data is effectively multi-master *only during the failover window* |
| Data residency | An EU user's writes during an EU outage are queued in a *second EU region*, never in NA. This is why the design has at least two regions per residency zone, and it is a cost the prompt did not anticipate |

This "temporarily multi-master" mode is the reason the multi-master conflict strategies must exist even for home-region data classes. A principal engineer says: "every data class needs a merge function, even if it is only used during failover, and the answer for some classes is 'writes are refused during failover' because no safe merge exists."

**Placement service availability.** Every region keeps a full read-only cache of user → home-region mappings (500M × ~20 bytes = 10 GB; trivial). New user creation during a control-plane outage assigns a home region locally and reconciles later. The control plane is *never* on the request path.

**The rehearsal.** State that failover is exercised quarterly by actually draining a region during business hours. A failover procedure that has never run does not exist.

## 6. Invariants

| Invariant | Where enforced | Verification |
|-----------|---------------|--------------|
| Every home-region object has exactly one writer region at any moment (except during a declared failover window) | Request router forwards non-home writes; the store rejects writes whose region tag does not match the object's home | Audit: count of rejected writes should be ~0 outside failover |
| EU personal data is never persisted outside EU regions | Placement service assigns EU users to EU regions; replication topology excludes personal-data tables from non-EU destinations at the CDC filter | Continuous scan of non-EU stores for EU user IDs; any hit is a P0 and a legal event |
| Multi-master data converges: all regions reach the same state given the same set of writes | CRDT/merge-function correctness per data class | Periodic cross-region checksum of a sample of objects; divergence is a bug in the merge function |
| Replication is at-least-once and apply is idempotent | High-water mark in the same transaction as the apply | Replay a day of CDC into a test replica; compare |
| Read-your-writes for the writing user | Version token round trip | Synthetic probe: write, then read from every region with the token |

**Deliberately relaxed:** read-your-writes for a *different* user is not guaranteed; cross-user visibility is bounded by replication lag. Stated as a product contract: "changes are visible to others within seconds."

## 7. Failure Modes

| Failure | Blast radius | Behavior | Mitigation |
|---------|-------------|----------|------------|
| Region outage | Users homed there: degraded; everyone else: stale reads of those users' data | Failover as in Deep Dive 3 | Rehearsed quarterly |
| Inter-region link degraded (high loss, not partitioned) | Replication lag grows; forwarded writes slow | Reads stale; writes for traveling users slow | Multiple backbone paths; lag alerting; replica staleness cutoff |
| Replication applier bug corrupts a replica | One region's copy of one shard | Reads from that replica are wrong | Cross-region checksums; rebuild replica from source snapshot + log; this is why the bus retains 7 days |
| Placement service down | New user creation | Regions assign locally, reconcile later | Never on the read/write path |
| Clock skew beyond HLC tolerance (a node's clock jumps forward hours) | Multi-master LWW fields written from that node win incorrectly | Silent bad merges | HLC implementations reject physical-clock jumps beyond a bound and refuse to issue timestamps; that node is drained |
| Schema change deployed to one region before others | Appliers in other regions block on the unknown schema | Replication lag grows until the schema is everywhere | Schema changes are their own deploy stage, all regions, before any code uses them (expand/contract) |
| **Correlated: global config push breaks all regions** | Everything | Full outage | Config is a globally replicated *data* store with staged rollout by region and automated rollback on SLO burn; the config service is the most common cause of multi-region outages in practice |
| **Correlated: GeoDNS misconfiguration** | Routing | Users sent to the wrong or a dead region | Client-side fallback region list; DNS changes go through the same staged rollout as code |

**Ranked degradation during a region outage (agreed with product in advance):**
1. Full function for users homed in surviving regions
2. Read-only-ish mode for failed-region users: reads (stale), plus writes to data classes with a safe merge
3. Refuse writes to data classes with no safe merge; show a clear "temporarily unavailable" state
4. Non-core features (recommendations, analytics-driven surfaces) disabled globally to free capacity for the core

## 8. Evolution Path

**v1 (today): single region.** The design should start by *classifying the data* even though everything is in one region, and by giving every user a home-region attribute (all "NA" today). This costs almost nothing and is the difference between a 6-month and an 18-month migration later.

**v2: read replicas in EU and APAC.** Reads are served locally with a staleness marker; all writes forward to NA. This alone fixes most of the 800 ms latency problem, since reads dominate. It also exercises the replication pipeline, lag SLIs, and read-your-writes tokens before any write is multi-region. Ship this and measure.

**v3: home regions.** Migrate EU users to EU home regions (required for residency anyway), then APAC. Per-user migration: freeze the user's writes for ~1 s, copy, flip the placement record, unfreeze, verify. The [migration playbook](zero-downtime-database-migration.md) applies per shard.

**v4: multi-master for the shared data classes.** By now the data classification has been in place for a year and the merge functions have been designed and tested against replayed traffic. Roll out per data class, starting with counters (lowest risk, highest visibility of correctness).

**Ordering rationale:** each stage is independently valuable, each exercises the machinery the next one depends on, and the residency requirement forces v3 to happen regardless.

## 9. Cost Model

| Component | Dominant driver | The knob |
|-----------|----------------|----------|
| Storage (hot data × 3 regions, cold × 2) | Replica count × data size | Which tables replicate for *serving* vs. only for *durability*; not every table needs a serving replica everywhere |
| Inter-region bandwidth | Write volume × destinations | Compression of the CDC stream (~4x); filtering out tables that need not replicate |
| Compute duplicated per region | Peak load per region + failover headroom | **Failover headroom is the big one:** each region must absorb a neighbor's load, so the fleet runs at ~50% utilization. This is the real cost of active-active, and it is often the number that surprises leadership |
| Replication bus (Kafka) per region | Retention × throughput | 7 days retention is the disaster-recovery buffer; do not cut it to save money |

**The insight:** the dominant cost is the failover headroom, not the storage or bandwidth. A two-region-per-residency-zone design means five or six regions, each at 50% utilization. The principal-level conversation is whether *core* traffic needs that headroom and non-core features can be shed instead; that decision can cut the compute bill by a third.

## 10. What Breaks at 10x

At 1.5M writes/s, 5B users:

**First: replication lag variance.** At 3 GB/s of cross-region CDC, the tail of the replication pipeline becomes the user-visible staleness bound. Fix: prioritize replication streams (user-visible data first, analytics tables last) and shard the replication bus by priority.

**Second: the multi-master merge cost.** At 150K/s of multi-master writes, the CRDT metadata (OR-Set tombstones, version vectors) grows faster than it is garbage-collected. Fix: bounded version vectors keyed by region rather than by node, and a tombstone GC protocol that requires a quorum of regions to acknowledge.

**Third: the placement cache.** 5B × 20 bytes is 100 GB per region: no longer trivially in memory on every router. Fix: a two-level cache with a bloom filter for "is this user homed here?" and a lookup for the rest.

## 11. Rejected Alternatives

| Alternative | Why it loses |
|-------------|-------------|
| **Everything multi-master with LWW** | Silent data loss on every concurrent write to data with invariants; wall-clock LWW is a data-loss bug with a delay |
| **Everything single-region-writer with global read replicas (stop at v2)** | Fixes latency but not residency, and a NA outage takes down writes for the whole world |
| **Globally consistent database (Spanner/Cosmos strong mode) for everything** | Every write pays cross-region consensus latency; the product does not need that guarantee for 90% of its data, and the bill is proportional to the guarantee |
| **Per-user sync replication to a second region** | Zero-loss failover for home-region data. Rejected for the general case because it doubles write latency; offered per data class (e.g., for purchase history) |
| **Automatic region promotion on health-check failure** | Health checks flap; automatic promotion on a false positive creates a split-brain with two home regions for the same users. Promotion is human-gated with a runbook, and the "temporarily multi-master" mode handles the window |
| **Storing residency-restricted data in a global store with encryption keys held regionally** | Regulators have not consistently accepted this; and it does not survive a key-management mistake. Physical placement is provable; crypto-shredding is a debate |

## 12. Level Signals

**A senior answer** adds read replicas in each region, uses GeoDNS, and picks a multi-master database (or LWW) for writes. Latency improves; conflict handling is hand-waved.

**A staff answer** classifies data into home-region and shared, designs the replication pipeline with lag SLIs and idempotent apply, uses CRDTs for counters and sets, and discusses region failover with a runbook.

**A principal answer** does all of that, and additionally:
- Identifies data residency as the dominant, non-negotiable requirement and shows it forces home regions regardless of latency
- Insists on a written merge function per data class and refuses multi-master for classes that have none
- Recognizes that home-region data becomes multi-master *during failover*, and designs for that window explicitly
- Notices that residency forces two regions per zone, and that failover headroom, not storage, is the dominant cost
- Orders the evolution so that each stage is independently valuable and exercises the next stage's machinery
- Names the config push and the DNS change as the correlated failures that actually cause multi-region outages

**Interviewer follow-ups to expect:**
- "A user moves from the US to Germany permanently. What happens?" (Their data is now subject to EU residency. A home-region migration tool: freeze, copy, flip placement, verify, then purge from NA. The purge is the part people forget.)
- "Two users in different regions edit the same group's name at the same moment." (Per-field LWW with HLC; the later edit wins deterministically; the loser sees their change reverted within replication lag. If that is unacceptable to product, the group name becomes single-leader.)
- "How do you test the merge functions?" (Property-based tests for commutativity, associativity, idempotency; and replay of a day of production writes with artificial partitions, comparing final states across simulated regions.)
- "What does the on-call see during a region outage?" (A single dashboard: per-region health, per-shard replication lag, forwarded-write queue depth, and the failover state machine's current step. Pages are on lag and queue depth, not on region health, because region health is already known.)
