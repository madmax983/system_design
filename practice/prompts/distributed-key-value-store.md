# Prompt Card: Globally Distributed Key-Value Store

**Worked example:** [examples/question-3-design-a-key-value-store.md](../../examples/question-3-design-a-key-value-store.md). Do not open it until step 4 of the [session protocol](../README.md#the-session-protocol). **Pairs with:** the [multi-region active-active card](multi-region-active-active.md); attempt them in consecutive sessions.

## The Prompt

> Design a globally distributed key-value store supporting PUT, GET, DELETE, TTLs, and conditional writes. Target 20M reads/sec and 5M writes/sec globally, p99 read latency under 30 ms in-region, an acknowledged write never lost, and namespace isolation per tenant. It must survive region failure.

## What This Trains

Consistency as an explicit per-API contract rather than a global setting; quorum mechanics and where `W + R > N` is not enough; hot-key mitigation; tombstones and TTL lifecycle in an LSM engine; tail-latency engineering; multi-region disaster recovery for a storage primitive.

## Pacing (45 min)

| Minutes | Do |
|---------|-----|
| 0-7 | Framing. Which consistency levels are offered, per API? What does "survive region failure" mean for writes in flight? |
| 7-10 | Reads, writes, key count, bytes, replication factor, node count, hot-key skew assumptions |
| 10-17 | Architecture: routing, partitioning, replicas, the storage engine; the versioning model for conditional writes |
| 17-35 | Deep dives. Pick two of: quorum reads/writes and the cases where `W + R > N` still returns stale data; hot-key detection and mitigation; the LSM engine with tombstones and TTL compaction; tail-latency techniques |
| 35-42 | Failure modes: node, AZ, region; hinted handoff and read repair; anti-entropy |
| 42-45 | Trade-offs: LSM versus B-tree, quorum defaults, TTL-heavy workloads, global latency versus strong consistency; the recommendation matrix |

## Before You Look: Questions to Answer in Your Attempt

- For each API, which consistency level is default and which are available? Can you defend each?
- Name a case where a quorum read returns stale data despite `W + R > N`.
- A single key gets 5% of all traffic. What happens without mitigation, and what is the mitigation?
- A DELETE races a PUT across regions. How does the tombstone resolve it, and when may the tombstone be removed?
- What is the p99 read latency budget, and which technique buys the most?

## Hints (reveal one at a time, only if stuck)

<details><summary>Hint 1: the contract</summary>
Offer consistency per call (for example ONE, QUORUM, and a linearizable option) and recommend defaults per workload class. Do not pretend global low latency and strong consistency are both free; end with a recommendation matrix.
</details>
<details><summary>Hint 2: quorums</summary>
`W + R > N` is necessary, not sufficient: sloppy quorums, hinted handoff to non-home replicas, and replicas serving stale local reads can all violate it. Name the mechanism that closes each gap.
</details>
<details><summary>Hint 3: hot keys</summary>
Detect with a sketch at the router; mitigate by replicating the hot key's reads to more replicas, caching at the router, or salting the key with client-side fan-in for writes. The mitigation must not require a data migration.
</details>
<details><summary>Hint 4: tombstones</summary>
A DELETE is a write of a tombstone with a version; it must replicate like any write. It can be compacted away only after every replica has seen it (a grace period tied to anti-entropy), or a stale replica resurrects the key.
</details>

## Interviewer Follow-Ups (answer out loud before revealing)

<details><summary>Q1: LSM or B-tree, and why?</summary>
LSM for the write-heavy target; sequential writes and cheap inserts. B-tree simplifies the read amplification profile and wins for read-dominant, update-in-place workloads. State the workload assumption that decides it.
</details>
<details><summary>Q2: A TTL-heavy tenant writes 1M keys/sec with a 1-hour TTL. What breaks?</summary>
Tombstone and expired-entry accumulation bloats storage and slows reads until compaction catches up. Needs time-bucketed SSTables or a TTL-aware compaction strategy so whole files drop at once.
</details>
<details><summary>Q3: Region failover: what happens to a write acknowledged one millisecond before the region dies?</summary>
If the quorum was in-region, it may be lost unless cross-region replication was synchronous for that namespace. Offer per-namespace choice; state the loss window for the async default. Compare with the ledger and multi-region examples' treatment of the same trade.
</details>
<details><summary>Q4: How do you rebalance shards when adding nodes?</summary>
Virtual nodes or a directory so that adding a node moves a small fraction of keys; streaming under a throttle; reads served from old and new during the move; cutover per vshard. Same machinery as the migration example.
</details>

## Scoring Focus

Weight dimensions 8 (invariants: the consistency contract per API), 7 (deep dive depth: quorum edge cases), 10 (failure modes), and 16 (alternatives). A principal answer ends with a recommendation matrix rather than a single consistency setting.
