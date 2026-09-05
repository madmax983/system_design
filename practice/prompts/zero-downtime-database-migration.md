# Prompt Card: Zero-Downtime Database Migration

**Worked example:** [examples/zero-downtime-database-migration.md](../../examples/zero-downtime-database-migration.md). Do not open it until step 4 of the [session protocol](../README.md#the-session-protocol).

## The Prompt

> The core `orders` service runs on a single PostgreSQL primary at 80% CPU and 12 TB, growing 2 TB per quarter, serving 40K QPS as the source of truth for every order. Migrate it to a horizontally sharded store with no maintenance window, no data loss, and the ability to abort at any point.

## What This Trains

Transition mechanics rather than end-state design; the difference between dual-write for freshness and CDC-plus-verification for correctness; rollback as a flag flip at every stage; the discipline to question the premise (is sharding needed now?) and buy time.

## Pacing (45 min)

| Minutes | Do |
|---------|-----|
| 0-7 | Framing. Is sharding the bottleneck? What does "abort at any point" force? What does "no data loss" precisely mean? |
| 7-10 | Copy duration versus write rate; why the copy alone cannot be consistent; shard count and the resharding horizon |
| 10-17 | The target, the data access layer as the migration, the shard key with its enumerated broken queries |
| 17-35 | Deep dives. Pick two of: the staged plan with per-stage rollback; dual-write correctness cases; continuous verification |
| 35-42 | Failure modes per stage; the one automatic "stop" trigger; who can bypass the DAL and how you stop them |
| 42-45 | What phase 0 buys; what the machinery is reused for later |

## Before You Look: Questions to Answer in Your Attempt

- What would you do *before* sharding, and how long does it buy?
- What single component makes the migration possible, and what if the service does not have one?
- Enumerate the ways dual-write silently corrupts data, and the mechanism for each
- After write cutover, is rollback still a flag flip? What has to be true for that?
- What kinds of mismatches do shadow reads find that row comparisons miss?

## Hints (reveal one at a time, only if stuck)

<details><summary>Hint 1: the premise</summary>
80% CPU on one primary is usually a query-shape or caching problem first. Phase 0 (kill top queries, cache the hot read, replicas for reporting, archive old rows, scale up) commonly buys 12-18 months, and the migration then runs on a schedule with your best people instead of in a war room.
</details>
<details><summary>Hint 2: the mechanism</summary>
A data access layer with per-virtual-shard flags for reads and writes. Cutover is 1,024 small cutovers. Symmetric dual-write after write cutover (new primary, legacy secondary, reverse CDC) keeps legacy complete so rollback stays a flag flip.
</details>
<details><summary>Hint 3: correctness</summary>
Application dual-write cannot survive a crash between the two writes. CDC from the authoritative store with idempotent, versioned apply is the correctness path; continuous verification (sampled rows, per-vshard checksums, shadow reads) is the backstop. Dual-write is for freshness.
</details>
<details><summary>Hint 4: the enforcement</summary>
Rotate legacy credentials at write cutover so only the DAL can reach it, and alarm on legacy access logs. "Nothing bypasses the DAL" is enforced, not hoped.
</details>

## Interviewer Follow-Ups (answer out loud before revealing)

<details><summary>Q1: Stage 5 (new store is primary), and legacy goes down. Can you roll back?</summary>
Not until legacy is repaired and caught up through reverse CDC. Users are unaffected because new is authoritative. The rollback option is temporarily unavailable, which is acceptable because rollback would be triggered by a problem with the new store, which is healthy.
</details>
<details><summary>Q2: A schema change ships mid-migration.</summary>
Expand/contract on both stores, new store first, as its own deploy stage. The DAL writes both old and new columns during expand. Contract only after the migration completes.
</details>
<details><summary>Q3: What did the shadow reads find?</summary>
Collation differences in ORDER BY, timestamp precision, NULL sort order, float formatting. Each becomes a fix or a documented accepted difference before reads cut over.
</details>
<details><summary>Q4: Product wants a cross-customer "all orders by status" page during the migration.</summary>
It is a scatter-gather or a read model. Build the read model from CDC in phase 0; it is faster than the legacy query anyway and outlives the migration.
</details>
<details><summary>Q5: Why virtual shards?</summary>
So that the next migration is a rebalance of vshards between physical shards using the same DAL flags and the same copy-CDC-verify-cutover loop, and so that a single hot tenant's vshard can be moved to a dedicated physical shard.
</details>

## Scoring Focus

Weight dimensions 1 (reframing: question the premise), 12 (evolution: this whole prompt is evolution), 13 (operations: rollback time), and 16 (alternatives). A principal answer says "dual-write is for freshness, not correctness" in those words or equivalent.
