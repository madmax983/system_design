# Prompt Card: Multi-Region Active-Active Data Platform

**Worked example:** [examples/multi-region-active-active.md](../../examples/multi-region-active-active.md). Do not open it until step 4 of the [session protocol](../README.md#the-session-protocol).

## The Prompt

> Your consumer product has 500M users across North America, Europe, and Asia-Pacific. Today everything runs in one US region and p99 latency for APAC users is 800 ms. Design the move to an active-active multi-region architecture. Users must be able to read and write from their nearest region, and a region outage must not take down the product.

## What This Trains

Classifying data by its consistency needs instead of applying one model everywhere; per-data-class conflict resolution you can defend to a product manager; recognizing a legal constraint as the dominant one; designing an evolution where each stage is independently valuable.

## Pacing (45 min)

| Minutes | Do |
|---------|-----|
| 0-7 | Framing. Is "active-active for everything" a requirement or a pre-selected answer? What is the non-negotiable requirement? |
| 7-10 | Estimate reads, writes, the fraction of writes that are genuinely multi-writer, and replication bandwidth |
| 10-17 | Architecture: the data classes, the routing for a home-region write from a traveling user |
| 17-35 | Deep dives. Pick two of: per-class conflict resolution; the replication pipeline; region failover including what happens to the failed region's data |
| 35-42 | Failure modes; the correlated failure; ranked degradation agreed with product |
| 42-45 | Evolution from single-region; the cost surprise |

## Before You Look: Questions to Answer in Your Attempt

- What fraction of the data is written concurrently from multiple regions? What model does the rest want?
- For each class of shared data, what is the merge function? Can you write it down?
- What does read-your-writes cost, and how do you provide it for a traveling user?
- During a region outage, do users homed there get to write? Where does that write go? What happens when the region returns?
- What is the dominant *cost* of active-active, and is it what leadership expects?

## Hints (reveal one at a time, only if stuck)

<details><summary>Hint 1: the reframe</summary>
Three data classes: home-region (single writer, the majority), reference (globally replicated, rarely written), and genuinely multi-master (the minority, designed case by case). "Active-active" is a property of the third class only.
</details>
<details><summary>Hint 2: the dominant constraint</summary>
Data residency. It is legal and binary. It forces home regions for personal data regardless of latency, and it forces at least two regions per residency zone for failover.
</details>
<details><summary>Hint 3: conflict resolution</summary>
Counters → PN-counter CRDT. Sets → OR-set. Independent fields → per-field LWW on hybrid logical clocks, never wall clocks. Text → sequence CRDT or OT. Anything with a hard invariant (inventory) → not multi-master; single leader per object. Money → not in this system.
</details>
<details><summary>Hint 4: failover</summary>
During a region outage, home-region data of the failed region becomes temporarily multi-master (writes queued in the surviving region, merged on return). So every data class needs a merge function even if only used during failover, and for some classes the function is "refuse writes."
</details>

## Interviewer Follow-Ups (answer out loud before revealing)

<details><summary>Q1: A user moves from the US to Germany permanently. What happens?</summary>
Their data is now subject to EU residency. A home-region migration: freeze the user's writes briefly, copy, flip the placement record, unfreeze, verify, then purge from the US region. The purge is the step that gets forgotten and the one the regulator asks about.
</details>
<details><summary>Q2: Two users in different regions rename the same group at the same moment.</summary>
Per-field LWW on HLC; the later edit wins deterministically; the loser's change is reverted within replication lag. If product finds that unacceptable, the group name becomes single-leader and cross-region edits pay ~150 ms.
</details>
<details><summary>Q3: How do you test the merge functions?</summary>
Property-based tests for commutativity, associativity, and idempotency per class; plus replaying a day of production writes through simulated regions with artificial partitions and comparing final states.
</details>
<details><summary>Q4: Why not auto-promote on health-check failure?</summary>
Health checks flap. A false positive creates two home regions for the same users: split brain. Promotion is human-gated with a runbook; the temporarily-multi-master mode makes the window safe enough that a few minutes of human latency is acceptable.
</details>
<details><summary>Q5: What is the dominant cost?</summary>
Failover headroom. Each region must absorb a neighbor's load, so the fleet runs at ~50% utilization, across five or six regions once residency forces two per zone. Not storage, not bandwidth.
</details>

## Scoring Focus

Weight dimensions 1 (reframing), 9 (relaxed invariants), 12 (evolution), and 14 (cost). A principal answer identifies residency as dominant and the headroom as the cost driver without being asked.
