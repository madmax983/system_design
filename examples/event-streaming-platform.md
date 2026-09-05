# Event Streaming Platform at 10M Events/sec

> **Prompt:** Design the company-wide event streaming platform. Hundreds of producer services emit events (clicks, orders, sensor readings, audit records) at a combined peak of 10M events/sec. Hundreds of consumer teams need to subscribe, some in real time and some for batch analytics. Consumers must be able to reprocess history after a bug. Some event types require exactly-once processing.

This looks like "design Kafka." It is actually "design the *platform around* a log": schema governance, multi-tenancy, exactly-once semantics that are honest about their boundaries, and the operational model for a system that every other system depends on.

## 1. Problem Framing

**What the prompt gets right:** durability, replayability, and fan-out to many consumers are the reasons a log exists. Reprocessing after a bug is the killer feature.

**What the prompt gets wrong:**
- "Exactly-once processing" is not a property of a platform; it is a property of a *producer-log-consumer-sink* chain, and the sink is where it is won or lost. The platform can offer exactly-once *within* the log (idempotent producers, transactional consume-transform-produce). Exactly-once *into an external database* is the consumer's idempotent-write responsibility. A principal engineer draws that boundary in the first five minutes.
- "10M events/sec" is the aggregate. The design question is the *distribution*: a handful of topics (clickstream) at millions/sec with relaxed ordering, and thousands of small topics (order events) at hundreds/sec with strict per-key ordering. One cluster cannot serve both well. Tiering by workload class is the core structural decision.
- "Hundreds of consumer teams" is an organizational fact with technical consequences: schema evolution, quota enforcement, and blast-radius isolation are the actual hard problems, not throughput.

**The reframe to state out loud:** "I'm going to design a *tiered* log platform with a control plane for schema, quotas, and topic lifecycle; separate clusters for high-volume-relaxed-ordering and low-volume-strict-ordering workloads; and I'll be precise about which exactly-once guarantees the platform provides and which it delegates to consumers."

## 2. Requirements & SLOs

### Functional
- Producers publish to named topics with a registered schema; the platform rejects unregistered or incompatible schemas
- Consumers subscribe in consumer groups; partition assignment and offset tracking are managed
- Per-key ordering within a partition
- Replay from any retained offset or timestamp
- Tiered retention: hot (days) on the brokers; cold (months to years) in object storage, transparently readable
- Exactly-once for consume-transform-produce chains within the platform; idempotent producers
- Multi-tenancy: per-team quotas on throughput, storage, and partitions; a runaway producer cannot degrade others

### Non-Functional
| Requirement | Target | Note |
|-------------|--------|------|
| Ingest throughput | 10M events/s peak, ~10 GB/s | Average event ~1 KB |
| Durability | No acknowledged event lost, up to loss of 2 brokers (or one AZ) | Replication factor 3, min in-sync replicas 2, acks=all |
| Producer latency | p99 < 20 ms for acks=all | Sets the batching window |
| End-to-end latency (real-time consumers) | p99 < 500 ms | Producer → broker → consumer |
| Availability (produce path) | 99.99% | Producers cannot buffer forever; a down platform means data loss upstream |
| Hot retention | 7 days | Enough for a weekend outage plus a fix |
| Cold retention | 2 years, per-topic configurable | Analytics and audit |
| Reprocessing | A consumer can reread 30 days at ≥ 5x real-time without affecting live consumers | The isolation requirement |

### Non-Goals
- Not a stream-processing framework (Flink/Spark); those are consumers
- Not a request/response message bus; no per-message acks to producers, no priority queues
- Not a database; there is no random access by key beyond compacted topics

**The dominant requirement:** *the produce path must be up, always.* Downstream systems can be slow, replay can be throttled, but if producers cannot write, data that will never be regenerated is lost. Everything else degrades before the produce path does.

## 3. Estimation

| Quantity | Estimate | Derivation |
|----------|----------|------------|
| Peak ingest | 10M events/s, 10 GB/s | Given, ~1 KB/event |
| Average ingest | 3M events/s, 3 GB/s | Peak is ~3x average |
| Replicated write volume | 30 GB/s peak | RF=3 |
| Hot storage (7 days) | 3 GB/s × 86,400 × 7 × 3 (RF) ≈ 5.4 PB | The number that sizes the broker fleet's disks |
| Cold storage (2 years) | 3 GB/s × 63M s ≈ 190 PB raw; ~50 PB compressed | Object storage; compression ~4x |
| Fan-out | Average 5 consumer groups per topic → 15 GB/s average egress, 50 GB/s peak | Egress exceeds ingress; this is the usual surprise |
| Brokers | ~300-500 | At ~100-200 MB/s sustained write per broker with headroom and 3x replication |
| Partitions | ~50,000-100,000 across the fleet | Bounded by controller metadata and per-broker file handles |
| Topics | ~5,000 | Hundreds of teams × ~10-50 topics each |

### Constraints the numbers force

1. **5.4 PB of hot storage on broker-local disks is the cost driver.** Tiered storage (offload segments to object storage after hours, not days) cuts the broker fleet's disk requirement by ~5x and is the single biggest architectural decision. The brokers become a hot cache in front of an object store.
2. **Egress > ingress by 5x** means the network, not the disk, is the broker bottleneck. Broker placement must be rack-aware and consumers should read from the nearest replica (follower fetching) to avoid cross-AZ egress cost.
3. **100K partitions on one cluster is past the comfortable limit** for controller metadata, leader election time, and per-broker file handles. This forces multiple clusters, which is convenient because workload tiering wants multiple clusters anyway.
4. **Reprocessing at 5x real-time for one consumer = 15 GB/s of extra egress** from cold storage. That must come from the object store, not the brokers, or it starves live consumers. Tiered storage with direct cold reads is not an optimization; it is the isolation mechanism.

## 4. Architecture

```
                    ┌────────────────────── Control Plane ──────────────────────┐
                    │  Schema Registry   Topic Catalog   Quota Service   ACLs   │
                    │  (compatibility    (ownership,     (per-tenant     (per-  │
                    │   rules, versions)  tier, retention) throughput)   topic) │
                    └────────────────────────────┬──────────────────────────────┘
                                                 │ config, not on the data path
                                                 │ (cached in clients & brokers)
     Producers                                   ▼                                     Consumers
   ┌──────────┐    ┌───────────────────────────────────────────────────────┐    ┌──────────────┐
   │ Service A│───▶│  Tier 1: High-volume clusters (clickstream, telemetry)│───▶│ Real-time    │
   │ Service B│    │  - RF=3, acks=all, min.isr=2                          │    │ (Flink, apps)│
   │   ...    │    │  - large partitions, relaxed ordering, 3-day hot      │    │              │
   └──────────┘    ├───────────────────────────────────────────────────────┤    ├──────────────┤
        │          │  Tier 2: Transactional clusters (orders, ledger, audit)│───▶│ Batch        │
        │          │  - RF=3, acks=all, min.isr=2, idempotent + txn        │    │ (Spark, DW)  │
        │          │  - many small partitions, strict key ordering, 7-day  │    │              │
        │          ├───────────────────────────────────────────────────────┤    ├──────────────┤
        │          │  Tier 3: Compacted / state clusters (CDC, configs)    │───▶│ Replay /     │
        │          │  - log compaction, infinite retention of latest       │    │ Backfill jobs│
        │          └────────────┬──────────────────────────────────────────┘    └──────┬───────┘
        │                       │ segment offload after N hours                       │
        │                       ▼                                                     │
        │          ┌───────────────────────────┐   direct cold reads for replay        │
        │          │  Tiered Object Storage    │◀──────────────────────────────────────┘
        │          │  (S3-class; 2-year        │
        │          │   retention; Parquet-ized │
        │          │   copy for analytics)     │
        │          └───────────────────────────┘
        │
        ▼
   Client SDK (mandatory): schema validation, idempotent producer config,
   quota-aware backoff, local disk spool on broker unavailability
```

### The produce path

```
1. Producer SDK fetches schema ID for the topic from the registry (cached, TTL 5 min)
2. Serializes event with schema ID prefix; validates against schema locally
3. Batches by partition (linger 5-10 ms or 64 KB); compresses (zstd, ~4x)
4. Sends with idempotent producer enabled: (producer_id, epoch, sequence) per partition
5. Leader broker: dedupes by sequence, appends to log, replicates to followers
6. Ack after min.isr=2 replicas have the batch (acks=all)
7. On broker unavailability: SDK spools to local disk (bounded, e.g. 1 GB), retries; emits a metric
```

The **mandatory SDK** is a platform decision with organizational teeth: it is the only way to guarantee schema validation, idempotency configuration, and spool-on-failure across hundreds of teams. Raw client access is for the platform team only.

## 5. Deep Dives

### Deep Dive 1: Exactly-once, and exactly where it stops

The platform offers three guarantees. State each precisely.

**1. Idempotent produce (at-least-once delivery becomes effectively-once within a producer session).**
The producer gets a `producer_id` and per-partition sequence numbers. The broker rejects duplicate sequences. Covers: network retries, broker failover mid-send. Does not cover: the producer *process* crashing and restarting with a new `producer_id`. That gap is closed by the producer's own idempotency key in the event payload (see the [ledger's outbox](global-payments-ledger.md), which is exactly this).

**2. Transactional consume-transform-produce.**
A consumer reads from topic A, transforms, writes to topic B, and commits its offset on A as part of the same transaction. If it crashes mid-way, the transaction aborts; the output on B is invisible to `read_committed` consumers, and the offset on A is not advanced. This is real exactly-once *between topics on the platform*. Cost: ~10-20% throughput, higher end-to-end latency (transactions commit in batches of ~100 ms).

**3. Nothing, for external sinks.**
When a consumer writes to a database, a search index, or an external API, the platform cannot make that atomic with the offset commit. The consumer must either:
- Make the sink write idempotent (upsert by event ID, the common case), or
- Store the offset *in the sink*, in the same transaction as the data (the correct case for relational sinks), or
- Accept at-least-once and deduplicate downstream

**The principal-level framing:** "The platform's exactly-once guarantee has a boundary, and the boundary is the platform's edge. I'll document the three sink patterns and the SDK will ship helpers for the first two. Teams that claim exactly-once without one of them are wrong, and the platform team's job is to make the correct pattern the easy pattern."

### Deep Dive 2: Schema evolution across hundreds of teams

Schemas are the API contract between teams who do not talk to each other. The platform enforces the contract so that humans do not have to.

**Registry rules:**
- Every topic has a schema subject with a compatibility mode. Default: `BACKWARD_TRANSITIVE` (a new schema can read all old data; consumers upgrade before or independently of producers).
- Producers cannot publish a schema that violates the mode. The check is in the registry, and the SDK refuses to serialize with an unregistered schema.
- Schema IDs are embedded in every event (4-byte prefix). A consumer reading a 2-year-old cold segment resolves the schema by ID. The registry is therefore an *append-only* store; a schema is never deleted while any retained data references it.

**Evolution patterns the platform supports and documents:**

| Change | Allowed under BACKWARD? | Migration path |
|--------|------------------------|----------------|
| Add optional field with default | Yes | Just do it |
| Remove a field | Yes (consumers ignore it) | Deprecate first; registry warns if any consumer group's registered reader schema still requires it |
| Rename a field | **No** | Add new, dual-write both for one retention window, remove old |
| Change a field's type | **No** | New field, same as rename |
| Split a topic into two | N/A | Producer dual-publishes; consumers migrate; old topic retention runs out |
| Change the partition key | **Ordering-breaking** | New topic; there is no in-place path because existing data is partitioned by the old key |

**The "who consumes this?" problem.** Producers evolving a schema need to know their consumers. The platform tracks consumer groups per topic and their last-read schema ID, and exposes it. A producer removing a field sees "3 consumer groups still read schema v4, which requires this field" before they break anyone. This turns a cross-team coordination problem into a dashboard.

### Deep Dive 3: Replay and backfill without starving live traffic

A consumer with a bug needs to reprocess 30 days. Naively, they reset offsets and read from the brokers at maximum speed, which evicts hot pages and starves live consumers of the same partitions.

**The design:**

1. **Cold reads bypass the brokers.** Segments older than the hot window are in object storage. The SDK's consumer, when asked for an offset older than the hot window, reads directly from object storage using the segment index. The brokers see none of that traffic.
2. **Replay is a declared, quota-limited operation.** A consumer group enters "replay mode" through the control plane, with a throughput quota that is separate from its live quota. The default is 5x the topic's live rate; more requires a ticket.
3. **Replay and live are separate consumer groups.** The replay group processes history into the sink; the live group continues. When replay catches up to the hot window, the replay group is stopped and the live group's offset is *not* touched. This requires the sink to be idempotent (Deep Dive 1), which it must be anyway.
4. **Parquet-ized cold copy for analytics.** Batch consumers reading 30 days should not deserialize 30 days of Avro/Protobuf events. A background job converts cold segments to columnar files partitioned by topic and hour. Analytics reads those; they never touch the log.

**The subtle point:** replay and live processing interleaving in the *same* sink produces out-of-order writes. The sink must be idempotent on event ID *and* ordered by event timestamp (last-write-wins on a version field), or replay will overwrite newer state with older. The SDK helper for idempotent sinks includes the version check. Teams get this wrong constantly; the platform's job is to make it hard to get wrong.

### Deep Dive 4: Late and out-of-order data

Events arrive late (mobile devices offline, retries, replay). The platform does not solve this, but it must not make it worse, and it must give consumers what they need.

- Every event carries two timestamps: `event_time` (when it happened, producer's clock) and `ingest_time` (broker's clock at append). The platform stamps the second; the SDK requires the first.
- Partitions are ordered by ingest, not event time. Consumers needing event-time semantics use watermarks in their stream processor; the platform exposes per-partition ingest-time high-water marks so watermarks can be derived without scanning.
- Retention is by ingest time. An event with an `event_time` 30 days old and an `ingest_time` of now is retained for the full window from now.

## 6. Invariants

| Invariant | Where enforced | Verification |
|-----------|---------------|--------------|
| An acknowledged event is durable to 2 replicas | acks=all, min.isr=2; unclean leader election disabled | Chaos: kill leader after ack, verify event on new leader |
| Per-partition order matches append order | Single leader per partition; idempotent producer sequence | Sequence gap detection in the SDK |
| Every event references a registered schema | SDK refuses otherwise; broker-side validation on Tier 2 | Sample-scan for unresolvable schema IDs |
| No schema is deleted while data references it | Registry is append-only; deletion requires retention proof | Registry audit |
| A tenant cannot exceed its quota | Broker-side quota enforcement by client ID; SDK backoff | Per-tenant throttle metrics |
| Cold-tier segments are byte-identical to hot-tier segments | Checksum on offload; consumer verifies on read | Periodic re-verification job |

**Deliberately relaxed:** cross-partition ordering. Stated loudly, because every team eventually asks for it. The answer is "put related events under the same key" and, when that is not possible, "use a stream processor with event-time watermarks."

## 7. Failure Modes

| Failure | Blast radius | Behavior | Mitigation |
|---------|-------------|----------|------------|
| Broker dies | Partitions it led (leader election ~seconds) | Producers retry; consumers pause | RF=3; rack-aware placement; idempotent producers make retry safe |
| AZ dies | 1/3 of replicas | min.isr=2 still satisfiable with 2 AZs | Rack-aware replica placement guarantees no partition has all replicas in one AZ |
| Controller failure | Metadata operations (new topics, elections) stall; data path continues | Existing partitions keep serving | KRaft/quorum controller with fast failover; metadata operations are not on the data path |
| Object storage unavailable | Cold reads fail; segment offload stalls; **broker disks fill** | Offload backlog grows | Alert on offload lag; brokers sized for 2x the hot window so a day-long object-store outage does not fill disks |
| Schema registry down | New schema registrations fail; existing cached schemas work | Producers with cached schemas continue; new topics blocked | Registry is replicated; SDK cache TTL is long enough (hours) to ride out an outage |
| Runaway producer (bug emits 100x) | Its topic's partitions; without quotas, its whole cluster | Quota throttles it at the broker | Per-tenant quotas; alert on throttle; Tier separation bounds the blast radius to one cluster |
| Consumer group rebalance storm | That group's lag | Processing pauses during rebalance | Cooperative (incremental) rebalancing; static group membership for stable consumers |
| Disk corruption on a broker | One replica | Detected by checksums; replica rebuilt from leader | Segment checksums; corrupted replica is dropped from ISR |
| **Correlated: bad SDK release to all producers** | Every producer | Could break serialization or spooling everywhere | SDK releases are staged by team cohort over a week; the SDK's own metrics are the canary |
| **Correlated: the platform is down and *nobody* can emit events** | Every dependent system | Upstream data loss | This is why the SDK spools to local disk; and why the produce path degrades last |

**Degradation order:**
1. Throttle replay consumer groups to zero
2. Throttle batch/analytics consumers
3. Throttle low-priority producers (telemetry) by quota tier
4. Pause segment offload (brokers have 2x headroom)
5. Shed real-time consumers (they lag; they catch up)
6. **Never** reject acks=all produces from Tier 2 while min.isr is satisfiable

## 8. Evolution Path

**v1: one cluster, mandatory SDK, schema registry from day one.** A single well-configured cluster handles ~1 GB/s. The SDK and registry are the parts that cannot be retrofitted across hundreds of teams; they exist before there are hundreds of teams.

**v2: tiered storage.** Offload segments to object storage. This is the moment retention becomes cheap and replay becomes isolated. Introduce replay mode in the control plane.

**v3: workload-tiered clusters.** Split high-volume from transactional. Migrate topics by dual-publishing from the SDK (a flag in the topic catalog; producers do not redeploy) and migrating consumer groups with offset translation. This is a months-long, per-topic migration and it is fine.

**v4: multi-region.** Mirror Tier 2 topics across regions for disaster recovery; Tier 1 stays regional (clickstream does not need cross-region durability). Offset translation between mirrored clusters is the hard part; consumers must treat offsets as cluster-local and use timestamps for cross-cluster resume.

## 9. Cost Model

| Component | Dominant driver | The knob |
|-----------|----------------|----------|
| Broker fleet (300-500 nodes) | Network egress and hot disk | **Hot window length.** 7 days → 1 day of local retention (with tiered storage) cuts broker disk by 7x; egress is cut by follower fetching from the same AZ |
| Object storage (~50 PB compressed) | Retention length × compression | Per-topic retention; default 90 days, 2 years only for topics that justify it |
| Cross-AZ network | Replication (unavoidable) + consumer fetch (avoidable) | Rack-aware consumers reading from local replicas; this is often 30% of the bill |
| Control plane | Negligible | N/A |

**The insight:** at this scale the bill is network and hot disk, and both are controlled by the hot-window length and consumer locality. A platform team that measures cost per topic and shows it to topic owners gets 30-50% reductions with no engineering, because most 2-year retentions are set by someone who did not know the price.

## 10. What Breaks at 10x

At 100M events/s, 100 GB/s:

**First: the controller and partition count.** ~1M partitions across the fleet. Leader election on a broker failure now takes minutes. Fix: many more, smaller clusters (a cluster per tenant cohort) behind a routing layer in the SDK; the topic catalog maps topic → cluster.

**Second: the schema registry's consumer tracking.** Thousands of consumer groups × thousands of topics × schema versions is a hot metadata store. Fix: it becomes a proper database with its own sharding, and lookups are cached aggressively in the SDK.

**Third: cold-read egress from object storage during replays.** Multiple concurrent 30-day replays at 5x real-time is hundreds of GB/s from the object store, which has its own throttles. Fix: replay quotas become a global budget, scheduled, and the Parquet copy absorbs analytics replays entirely.

## 11. Rejected Alternatives

| Alternative | Why it loses |
|-------------|-------------|
| **One giant cluster** | Partition-count limits, blast radius, and the impossibility of tuning one cluster for both clickstream and transactional workloads |
| **Pull-based per-message queue (RabbitMQ/SQS-style) for everything** | No replay, no fan-out without duplication, no per-key ordering at scale. Fine for work queues; not a log |
| **Optional SDK / raw client access for all teams** | Schema validation, idempotency, and spooling become "best practice documents" instead of guarantees; the first incident is a team that turned off acks=all for throughput |
| **Infinite retention on brokers** | 190 PB on local NVMe. No |
| **Platform-provided exactly-once to external sinks** | Cannot be done generally; claiming it produces incorrect systems. The honest boundary plus SDK helpers is strictly better |
| **Cross-partition ordering via a global sequencer** | A single sequencer at 10M/s is a single point of failure and the throughput cap. Related events share a key; everything else uses event-time in the processor |
| **Schema-less JSON topics** | Works for a year; then a producer renames a field and 40 consumers break at 3am. The registry costs less than that one incident |

## 12. Level Signals

**A senior answer** picks Kafka, sets RF=3 and acks=all, adds a schema registry, and discusses consumer groups and retention. Solid.

**A staff answer** tiers storage to object storage, separates clusters by workload, explains idempotent producers and transactions, and designs quotas and rack-aware placement. Names the egress > ingress surprise.

**A principal answer** does all of that, and additionally:
- Draws the exactly-once boundary at the platform's edge and specifies the three sink patterns, making the correct one the easy one via the SDK
- Treats the mandatory SDK and the schema registry as *organizational* mechanisms and justifies them by the cross-team coordination they replace
- Designs replay as an isolated, quota-limited, cold-storage-served operation, and identifies the replay-overwrites-newer-state bug
- Identifies the hot-window length and consumer locality as the cost knobs, and turns cost visibility per topic into a feature
- Says "never reject Tier 2 produces" as the last line of the degradation order and explains why produce availability dominates
- Names the bad-SDK-release as the correlated failure and stages SDK rollouts accordingly

**Interviewer follow-ups to expect:**
- "A team says they need exactly-once into Postgres. What do you tell them?" (Store the consumer offset in Postgres, in the same transaction as the data. The SDK has a helper. If they cannot, upsert by event ID with a version check.)
- "A producer needs to change its partition key." (New topic. Dual-publish. Migrate consumers. There is no in-place path, and you should say so before they design a system that depends on one.)
- "How do you migrate a topic between clusters without consumers noticing?" (SDK-level dual-publish flag; consumer group offset translation by timestamp; a cutover window where the consumer reads both and dedupes by event ID.)
- "Object storage is down for 6 hours. What happens?" (Offload backs up; broker disks have 2x hot-window headroom, so ~7 days of margin at 7-day hot retention; cold replays fail; live traffic is unaffected. The alert fires at 1 hour of offload lag.)
