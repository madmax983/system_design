# Distributed Coordination Service

> **Prompt:** Design the internal coordination service that every other system at the company uses for leader election, distributed locks, service discovery, and small-configuration storage. Think ZooKeeper, etcd, or Chubby. It must be correct under partitions, and it will be the single most depended-upon system in the company.

This is the system whose failure appears in every other system's post-mortem. The interesting engineering is not the consensus algorithm (use Raft; do not invent one). It is the *API semantics* that make clients correct, the *operational posture* that makes the service boring, and the *blast-radius* decisions that keep it from becoming the company's single point of failure.

## 1. Problem Framing

**What the prompt gets right:** correctness under partitions is the whole point. A lock service that grants two holders during a partition is worse than no lock service, because clients trusted it.

**What the prompt gets wrong:**
- "Distributed locks" as a primitive is a trap. A lock service cannot make a *client* correct: the client can hold a lock, pause for a garbage collection cycle, have its lease expire, and then act as if it still holds the lock. The service's job is to make that failure *detectable* by the systems the client talks to. That is the fencing token, and it is the single most important idea in this design.
- "Every system depends on it" is stated as a fact. A principal engineer treats it as a *risk to reduce*. The design should push clients toward using the service for *rare, small, coordination* operations and away from putting it on every request's hot path. The most effective availability improvement for a coordination service is fewer dependents.
- "Small-configuration storage" invites teams to use it as a database. It is a strongly consistent store for kilobytes, updated rarely. The design has to enforce that boundary or it becomes a general-purpose database with consensus latency.

**The reframe to state out loud:** "I'm going to build this on Raft with a small, sharded fleet, and spend my design effort on three things: an API whose semantics make it possible for clients to be correct (sessions, leases, fencing tokens, watches with sequence numbers); a set of guardrails that stop it from being used as a database or as a hot-path dependency; and an operational model that treats it as the most conservative, slowest-changing system in the company."

## 2. Requirements & SLOs

### Functional
- **Key-value store** with a hierarchical namespace, strongly consistent (linearizable) reads and writes, atomic compare-and-swap, multi-key transactions within one shard
- **Sessions and leases**: a client holds a session with a lease; ephemeral keys are deleted when the lease expires
- **Leader election** as a library pattern on top of ephemeral keys + fencing tokens, not a special API
- **Watches**: a client subscribes to a key or prefix and receives ordered change notifications with sequence numbers, with the guarantee that no change is missed (or that the client is told it missed some)
- **Service discovery** as a pattern on top of ephemeral keys and watches
- Access control per namespace prefix; quotas per client identity

### Non-Functional
| Requirement | Target | Note |
|-------------|--------|------|
| Consistency | Linearizable writes; linearizable reads by default, with an explicit serializable (stale-allowed) read option | Clients must opt *in* to staleness; the default is safe |
| Availability (writes) | 99.99% per shard | Majority quorum; survives one node loss in a 3-node group, two in 5 |
| Availability (reads, stale-allowed) | 99.999% | Any replica can serve |
| Write latency | p99 < 10 ms within a region | One consensus round trip |
| Read latency (linearizable) | p99 < 5 ms | Leader lease read or read-index |
| Lease granularity | 1-60 s, default 10 s | Sets the failure-detection time for ephemeral keys |
| Watch delivery | p99 < 100 ms from commit | Notification lag becomes application failover lag |
| Data size | ≤ 1 MB per key; ≤ 8 GB per shard | Hard limits, enforced |
| Blast radius | A shard failure affects only its tenants; the fleet has no global coordinator on the request path | Sharding by namespace prefix |

### Non-Goals
- Not a database, cache, queue, or metrics store; hard limits enforce this
- Not a cross-region strongly consistent store: a shard lives in one region; cross-region use is through separate shards and application-level reconciliation
- Not providing "distributed lock" as an API name; providing leases and fencing tokens, and documenting the lock pattern with its caveats

**The dominant requirement:** *correctness under partition*, specifically: never two leaders that both believe they are current *and can both act on the world*. The service alone cannot guarantee the second half; the fencing token protocol is what does.

## 3. Estimation

| Quantity | Estimate | Derivation |
|----------|----------|------------|
| Dependent services | ~2,000 | Every service at a large company |
| Client sessions | ~500K | ~250 instances per service on average |
| Lease renewals | ~50K/s | 500K sessions / 10 s lease |
| Reads (linearizable) | ~20K/s | Mostly discovery lookups that should be cached; enforced down over time |
| Reads (stale-allowed) | ~200K/s | Cached-ish discovery; served by any replica |
| Writes | ~2K/s | Config changes, election churn, ephemeral registration |
| Watches active | ~5M | Many clients watch many prefixes |
| Total data | ~50 GB across the fleet | Small keys, many of them |
| Shards (Raft groups) | ~20-50 | By namespace prefix; 5 nodes each |

### Constraints the numbers force

1. **50K lease renewals/s is the dominant load**, and it is pure overhead. Renewals must be cheap: batched per client connection, handled by the leader without a consensus round (a lease renewal is a leader-local operation; only lease *expiry* needs to be agreed, and the leader decides it with a lease of its own).
2. **5M watches with 100 ms delivery** means the watch fan-out is a bigger engineering problem than consensus. Watches are served by followers from their replicated log, not by the leader; the leader does consensus, followers do fan-out.
3. **A single Raft group cannot serve the whole company.** Raft throughput is bounded by the leader's single-threaded log append and fsync, ~10-50K writes/s at best. At 2K writes/s that is fine, but the *blast radius* of one group is the whole company. Shard by namespace prefix so that a bad client or a group failure affects a subset.
4. **50 GB is small.** It fits in memory on every node. Reads never touch disk; the disk is for the log. This is what makes 5 ms linearizable reads possible.

## 4. Architecture

```
                         ┌─────────────── Fleet Control Plane ────────────────┐
                         │  Shard map (prefix → Raft group)                   │
                         │  Membership changes, split/merge, key rotation     │
                         │  (NOT on the request path; clients cache the map)  │
                         └──────────────────────────┬─────────────────────────┘
                                                    │
        Clients (SDK)                               │
        ┌──────────────┐        ┌───────────────────▼──────────────────────────────┐
        │ - shard map  │        │  Shard A (prefix /services/*)   5 nodes, 3 AZs   │
        │   cache      │───────▶│  ┌────────┐ ┌────────┐ ┌────────┐ ┌───┐ ┌───┐    │
        │ - session &  │        │  │ Leader │ │Follower│ │Follower│ │ F │ │ F │    │
        │   lease      │        │  │ (Raft, │ │ (log,  │ │ (log,  │ │   │ │   │    │
        │   keepalive  │        │  │  lease │ │  stale │ │  stale │ │   │ │   │    │
        │ - watch      │        │  │  mgmt) │ │  reads,│ │  reads,│ │   │ │   │    │
        │   resume by  │        │  │        │ │ watches│ │ watches│ │   │ │   │    │
        │   revision   │        │  └────────┘ └────────┘ └────────┘ └───┘ └───┘    │
        │ - fencing    │        └──────────────────────────────────────────────────┘
        │   token      │        ┌──────────────────────────────────────────────────┐
        │   plumbing   │───────▶│  Shard B (prefix /config/*)                      │
        └──────────────┘        └──────────────────────────────────────────────────┘
                                ┌──────────────────────────────────────────────────┐
                        ───────▶│  Shard C (prefix /locks/*)                       │
                                └──────────────────────────────────────────────────┘
                                                   ...
```

### Data model

```
Key:      /services/orders/instances/i-0abc   (hierarchical; prefix operations supported)
Value:    ≤ 1 MB bytes
Revision: global-per-shard monotonic counter, incremented on every write (this is the fencing token source)
Lease:    optional; if the owning session's lease expires, the key is deleted (ephemeral)
Metadata: create_revision, mod_revision, version (per-key write count)
```

The *revision* is the load-bearing concept. Every write to the shard gets a unique, monotonically increasing revision. Every read returns the revision it observed. Every watch event carries its revision. Every ephemeral key's create-revision is a fencing token.

## 5. Deep Dives

### Deep Dive 1: Fencing tokens, or why the lock service cannot make you correct

The scenario, which happens in production regularly:

```
1. Client A acquires the lock on /locks/job-42 (creates an ephemeral key; gets revision 100)
2. Client A begins a 30-second write to the storage system
3. Client A pauses (GC, VM migration, network hiccup) for 15 seconds
4. A's lease (10 s) expires; the service deletes A's ephemeral key
5. Client B acquires the lock (creates the key; gets revision 117)
6. Client B begins writing to the storage system
7. Client A resumes, still believes it holds the lock, and completes its write
   → A and B both wrote. The lock service did nothing wrong. A is simply wrong about the world.
```

No lease duration fixes this; a longer lease just makes step 4 rarer and failover slower. The fix is to give the *storage system* a way to reject A:

```
Fencing protocol:
- The lock grant returns the key's create_revision as a fencing token (A: 100, B: 117)
- Every write to the protected resource carries the token
- The protected resource stores the highest token it has seen and rejects any write with a lower one
- A's late write with token 100 is rejected because the resource has seen 117
```

**Consequences for the design:**
- The service's API *must* expose revisions on every operation, including lock/lease grants, because the fencing token *is* the revision.
- The SDK's lock helper returns a token and refuses to be used without one being plumbed to the protected resource. It is a compile-time (or at least construction-time) requirement, not a documentation note.
- The protected resource must support conditional writes on the token. For a database, that is a `WHERE fence_token < :token` clause; for object storage, a conditional put on a version; for a system that cannot, the honest answer is "this system cannot be safely protected by a lock" and the pattern is replaced with idempotent operations.
- Documentation calls the primitive a *lease*, not a *lock*, and the first paragraph explains the scenario above. Naming matters here because "lock" imports intuitions from single-process programming that are false.

### Deep Dive 2: Leases, clocks, and what "expired" means

Lease expiry is decided by the *leader*, using the leader's clock. Three details make this correct:

1. **The leader has its own lease from the Raft group.** A leader that is partitioned from the majority stops being leader after its leader-lease expires (a few heartbeat intervals) and stops making decisions. So a client lease is only expired by a node that is currently, provably, the leader. During the window where an old leader has not yet noticed it lost leadership, it also cannot expire client leases, because expiry requires committing a deletion through Raft, which requires a majority it no longer has.

2. **Clients renew with margin.** A 10-second lease is renewed every 3 seconds. The SDK treats *two* missed renewals as "assume expired; stop acting as if you hold anything; re-establish the session." The client's assumption is conservative relative to the server's decision, so the client gives up *before* the server evicts it. The remaining race (client thinks it has the lease, server has expired it) is exactly the fencing-token case.

3. **Clock skew between leader and client does not matter**, because lease timing is *relative* (the leader measures elapsed time since the last renewal on its own clock; the client measures elapsed time since its last successful renewal on its own clock). Neither compares absolute timestamps. Clock *rate* skew matters slightly; the renewal margin absorbs it.

**The failure to state:** a leader whose clock *jumps* (NTP step, VM live-migration) can expire every lease at once. Mitigation: the leader uses a monotonic clock for lease timing, never wall-clock; and a leader that observes a monotonic-clock discontinuity beyond a threshold steps down rather than acting on it.

### Deep Dive 3: Watches that do not lose events

A watch on `/services/orders/` must deliver every change, in order, or tell the client it has fallen behind. "Best-effort notifications" are useless for discovery, because a missed "instance removed" event means routing to a dead instance forever.

**Design:**
- Every watch is established *at a revision*: "send me every change to this prefix after revision R." The client's first call is a read (which returns the current state and revision R), then a watch from R+1. There is no gap.
- The follower serving the watch reads its replicated log from R+1 forward and streams events in revision order. The log is retained for a window (say, 1 hour or 1M revisions, whichever is larger); a *compaction* boundary is advertised.
- If a client requests a watch from a revision older than the compaction boundary, it receives `ERR_COMPACTED` with the boundary revision. The SDK handles this by re-reading the current state and re-establishing the watch. The application sees a full "resync" callback, not a stream of individual events. This is the "you missed some; here is everything" path, and it must be exercised in tests, because it will happen in production during any follower restart.
- Watch fan-out is on followers. A leader change does not disrupt watches; followers keep streaming from their log. A follower failure moves the client's watch to another follower, resumed at the last delivered revision (the SDK tracks it).

**The subtle guarantee:** a client that reads at revision R and then watches from R+1 sees a linearizable history. A client that uses stale reads *and* watches can see a state at revision R (stale) and then an event at revision R-5 (from a watch established earlier): the SDK must not deliver events older than the client's last observed revision. This is a real bug class in real clients.

### Deep Dive 4: Guardrails against becoming the company's database

The service will be misused. The design assumes it and defends:

| Guardrail | Enforcement | Why |
|-----------|------------|-----|
| 1 MB max value, 8 GB max shard | Hard reject | Keeps everything in memory; keeps snapshots fast |
| Per-client-identity write quota (e.g., 100 writes/s) | Reject with backoff | A misbehaving client cannot saturate a leader's log |
| Per-client watch limit (e.g., 1,000) | Reject | Fan-out is the scaling bottleneck |
| Linearizable reads rate-limited per client; stale reads are cheap | Reject beyond quota, with a message suggesting stale reads | Pushes discovery traffic to followers |
| No range scans over more than 10K keys | Reject | A scan holds a read transaction on the leader |
| Every key has an owner (namespace prefix → team); unowned prefixes cannot be written | Reject | Turns "who is doing this?" into a lookup |
| SDK-level local cache for discovery with watch-driven invalidation | Default on | The service should see ~1 read per client per key per *change*, not per *request* |

**The organizational mechanism:** the platform team publishes per-team usage and cost. The team that does 40% of linearizable reads gets a conversation, not an outage.

## 6. Invariants

| Invariant | Where enforced | Verification |
|-----------|---------------|--------------|
| At most one leader per Raft group per term | Raft | Jepsen-style testing in CI against the actual binary, continuously |
| Revisions are unique and monotonic per shard | Assigned by the leader at log-append; committed through Raft | Any client observing a revision decrease is a P0; the SDK asserts it |
| An ephemeral key is deleted only after its session's lease has expired on the leader's monotonic clock | Leader-only expiry; committed through Raft | Chaos: pause the leader; verify no expiry occurs during the pause beyond the leader-lease bound |
| A watch from revision R delivers every change > R in order, or returns `ERR_COMPACTED` | Follower log replay | Test harness compares watch stream to log |
| A linearizable read reflects every write committed before the read began | Read-index or leader lease | Jepsen linearizability checker |
| No shard depends on another shard or on the fleet control plane to serve requests | Architecture; network policy | Control plane taken down in game days |

**Deliberately relaxed:** stale reads may return data up to the follower's replication lag old (typically < 50 ms). Clients opt in per call. The SDK's discovery cache uses stale reads because staleness of 50 ms in a service list is harmless *given watches keep it current*.

## 7. Failure Modes

| Failure | Blast radius | Behavior | Mitigation |
|---------|-------------|----------|------------|
| One node fails (5-node group) | None visible | Quorum holds; if it was the leader, election in ~1-2 s during which writes block | Leader election tuned to ~1 s; clients retry; watches unaffected (on followers) |
| Two nodes fail (5-node group) | None visible | Quorum holds at 3 | Alert: the group is one failure from unavailability |
| AZ loss (2 of 5 nodes) | Same as above | Quorum holds | 3-AZ placement, 2-2-1 |
| Majority loss | That shard: writes fail; stale reads continue; **leases cannot expire** | Ephemeral keys are frozen; discovery goes stale; elections cannot change | Clients' SDKs stop renewals, assume expired, and stop acting: the system fails *safe*. Restore quorum from surviving nodes or from snapshot + log |
| Leader with a network partition from clients but not peers | Clients cannot renew; leader still expires their leases | Ephemeral keys churn; elections flap | Clients connect to any node and are forwarded; the leader's *client* reachability is a health signal that can trigger a voluntary step-down |
| Slow disk on the leader | Write latency spikes for the shard | fsync is on the commit path | Leader transfer to a healthy node on disk latency SLO breach; this is automated |
| Runaway client (writes, watches, scans) | Its shard, degraded | Quotas throttle; owner is identified | Quotas; per-team dashboards; the ability to *ban a client identity* in seconds |
| Log compaction deletes a revision a slow watcher needed | That watcher | `ERR_COMPACTED` → resync | Tested path; retention window is generous |
| Snapshot restore after total loss of a shard | That shard's data as of the snapshot; ephemeral keys are gone (correct: their sessions are gone too) | Clients resync via watch resync | Snapshots every few minutes to durable storage; the log since the snapshot is also shipped |
| **Correlated: a bad SDK release** | Every client | Could break renewals (mass lease expiry) or fencing | SDK releases roll out over weeks; the SDK reports its version and the fleet dashboard shows the mix |
| **Correlated: a config change to the fleet** | Every shard | The coordination service's own config is the most dangerous config in the company | Fleet config changes are one shard at a time, with a soak, by hand, by two people |
| **Correlated: every dependent service retries in lockstep after an outage** | The service, on recovery | Thundering herd of session re-establishment and watch resync | SDK jittered exponential backoff; the service sheds load by rejecting new sessions before rejecting renewals of existing ones |

**Degradation order:**
1. Reject range scans and non-essential linearizable reads (stale reads continue)
2. Reject new watches (existing continue)
3. Reject new sessions (existing renewals continue; this is what keeps the world stable)
4. Reject writes from clients over quota, then from clients without an owner, then all non-platform writes
5. **Never** stop processing lease renewals for existing sessions while quorum holds; losing renewals converts a coordination-service incident into a company-wide leader-election storm

## 8. Evolution Path

**v1: one Raft group, one region, the SDK.** The API semantics (revisions, leases, fencing tokens, watch-from-revision) are all in v1 because they are the contract and cannot change. Everything else can.

**v2: sharding by prefix.** Add the shard map, the control plane, and the SDK's routing. Migrate prefixes by making the new shard a learner of the old one's log for that prefix, then flipping the map. Cross-shard transactions are not supported, and were never promised.

**v3: watch fan-out on followers; guardrails.** Usually forced by the first incident where watch load took down a leader.

**v4: regional fleets.** One fleet per region; no cross-region replication. Cross-region coordination is an application-level problem, deliberately.

**The one-way doors:** the API semantics in v1. Revisions as fencing tokens, watch-from-revision, and "lease, not lock" naming. Every client in the company will code against them.

## 9. Cost Model

Small in infrastructure; enormous in *risk*. The cost model that matters here is different:

| Cost | Driver | Note |
|------|--------|------|
| Fleet compute (~250 nodes: 50 shards × 5) | Shard count | Small |
| Engineering: SDK quality | The SDK runs in 2,000 services | An SDK bug is a company-wide incident; the SDK deserves the best engineers and the slowest release cadence |
| Engineering: continuous verification (Jepsen-style) | Correctness | Runs in CI on every build; cheaper than one split-brain incident |
| On-call | The most conservative on-call rotation in the company | Nobody does a "quick fix" on the coordination service at 3am |
| **Dependent-service risk** | Number of services on the hot path | The knob is *reducing dependents*: SDK caching, stale-read defaults, and a review gate for any service that wants the coordination service on its request path |

## 10. What Breaks at 10x

At 5M sessions, 500K renewals/s, 50M watches:

**First: renewals.** 500K/s of leader-handled renewals across 50 shards is 10K/s per leader; feasible, but with batching per connection and a proxy tier that aggregates renewals from many client processes into one stream per shard. The proxy tier is a new component with its own failure modes (it must not hold sessions alive for dead clients).

**Second: watch fan-out.** 50M watches on followers with 100 ms delivery. Fix: a dedicated watch-serving tier that tails the followers' logs and does fan-out, decoupling it from the Raft group entirely. Each shard's log becomes a stream that a fan-out tier consumes; at that point the design converges toward "Raft for state, a log for notifications."

**Third: the shard map.** Thousands of prefixes across hundreds of shards; the map is now a real dataset with its own consistency concerns. Fix: the shard map lives in a root shard, and the SDK caches it with a watch, like everything else.

## 11. Rejected Alternatives

| Alternative | Why it loses |
|-------------|-------------|
| **A custom consensus protocol** | Raft is proven, understood, and has reference implementations with years of bug-fixing. Novelty here is negligence |
| **Multi-Paxos** | Equivalent correctness, harder to reason about and to hire for. Raft's explicit leader and log matching make operational reasoning easier |
| **Single global Raft group** | Company-wide blast radius; leader throughput cap |
| **Cross-region replicated groups (a 5-node group across 3 regions)** | Every write pays cross-region latency (~100 ms); leader election during a region partition is slow and frequent. Regional fleets with application-level cross-region coordination is the honest design |
| **Locks without fencing tokens** | Incorrect under GC pause, and the incorrectness is silent. See Deep Dive 1 |
| **Time-based locks (hold for N seconds) with wall clocks** | Clock skew makes "N seconds" mean different things on different machines. Leases are relative and leader-decided |
| **Best-effort watches** | A missed removal event in discovery routes to dead instances forever. Watch-from-revision with `ERR_COMPACTED` is the only correct contract |
| **Using the coordination service for high-rate reads without caching** | Turns it into a hot-path dependency for every request in the company. Guardrails and the SDK cache exist to prevent it |
| **Building on a general database with strong consistency** | Possible, but you lose ephemeral keys, ordered watches, and the small-and-in-memory property that gives 5 ms reads. The primitives are the product |

## 12. Level Signals

**A senior answer** picks Raft or etcd, describes leader election and locks with TTLs, and adds replicas for availability.

**A staff answer** shards by prefix, explains leader leases and linearizable reads via read-index, designs watches with revisions, serves watches from followers, and adds quotas.

**A principal answer** does all of that, and additionally:
- Leads with fencing tokens and explains that the service cannot make clients correct, only make client incorrectness detectable
- Renames "lock" to "lease" and treats the API semantics as the one-way door
- Explains lease expiry as leader-decided on a monotonic clock and shows why a partitioned old leader cannot expire anyone
- Specifies the watch contract precisely, including `ERR_COMPACTED` and the stale-read-plus-watch ordering bug
- Treats "reduce the number of dependents" as the primary availability strategy and designs guardrails and SDK defaults to achieve it
- Puts renewals of *existing* sessions last in the degradation order and explains that losing them causes a company-wide election storm
- Describes the operational posture (slowest release cadence, two-person config changes, continuous Jepsen-style testing) as part of the design

**Interviewer follow-ups to expect:**
- "A client holds a lease and wants to write to S3. How is that fenced?" (Conditional put on a version or ETag that the client reads first; the fencing token is compared to a token stored in the object's metadata. If the object store cannot do conditional writes, the answer is "you cannot safely fence that resource; make the operation idempotent instead.")
- "Quorum is lost on a shard for 10 minutes. What happened to the ephemeral keys?" (Nothing. They are frozen. Clients' SDKs have already assumed their own leases expired and stopped acting. When quorum returns, the leader expires sessions whose clients did not reconnect. Discovery was stale for 10 minutes; applications using cached discovery kept working against possibly-dead instances, which is why they have their own health checks.)
- "How do you migrate a prefix to a new shard with live watches on it?" (Watchers receive a special `ERR_MOVED` with the new shard and the revision at which the move happened; the SDK re-establishes on the new shard from that revision. Revisions on the new shard start from a value greater than the old shard's last revision for that prefix, to preserve monotonicity for fencing.)
- "Why should the SDK be slower to release than the service?" (The service is 250 nodes you control; the SDK is 2,000 services you do not. A service rollback takes minutes; an SDK rollback takes weeks and requires every team to redeploy.)
