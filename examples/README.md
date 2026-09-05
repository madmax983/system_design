# Worked Examples (IC7 / Principal Engineer Level)

The rest of this repository catalogs patterns and building blocks. This section shows how a principal engineer *combines* them under real constraints: conflicting requirements, money on the line, multiple regions, and a system that already exists and cannot be turned off.

Every example here is written to the level expected of a Principal / IC7 candidate. That means the interesting content is not the boxes-and-arrows diagram. It is the reasoning that precedes it, the failure analysis that follows it, and the honest accounting of what was traded away.

## What "Principal Level" Means Here

| Dimension | Senior (IC5) | Staff (IC6) | Principal (IC7) |
|-----------|-------------|-------------|-----------------|
| **Requirements** | Accepts the prompt as given | Negotiates scope, names non-goals | Reframes the problem; identifies the requirement that dominates the design and the one that is secretly wrong |
| **Estimation** | Computes QPS and storage | Derives design constraints from the numbers | Identifies which number the whole design is sensitive to, and what happens when it is off by 10x |
| **Architecture** | Produces a correct design | Produces a design with explicit trade-offs | Produces a design *and its evolution path*, including how to get there from what exists today |
| **Correctness** | Handles the happy path and obvious failures | Handles partial failure, retries, idempotency | Reasons about invariants, proves where they hold, and names where they are deliberately relaxed |
| **Failure** | Adds redundancy | Analyzes failure modes and blast radius | Designs for graceful degradation; decides what to sacrifice first and who decides |
| **Operations** | Mentions monitoring | Defines SLOs and alerts | Designs the operational model: on-call load, deploy safety, runbooks, capacity planning cadence |
| **Cost** | Not discussed | Order-of-magnitude aware | Models cost as a first-class constraint; knows the dominant cost driver and the knob that controls it |
| **Organization** | Not discussed | Considers team ownership | Designs the service boundary to match the team boundary; anticipates the migration and the politics |

See [Principal Engineer Signals](../vocabulary/principal-engineer-signals.md) for the full rubric.

## The Examples

| Example | Core Tension | Hardest Sub-Problem |
|---------|-------------|---------------------|
| [Global Payments Ledger](global-payments-ledger.md) | Exactly-once money movement vs. availability | Idempotency across retries, crashes, and region failover; reconciliation as a design primitive |
| [Multi-Region Active-Active Data Platform](multi-region-active-active.md) | Low local latency vs. a single source of truth | Conflict resolution that product teams can reason about; data residency; failover without data loss |
| [Event Streaming Platform at 10M events/sec](event-streaming-platform.md) | Throughput vs. ordering and exactly-once delivery | Schema evolution, late/out-of-order data, backfills that do not starve live traffic |
| [Zero-Downtime Database Migration](zero-downtime-database-migration.md) | Changing the storage layer under live traffic with no maintenance window | Dual-write correctness, shadow verification, an always-available rollback |
| [Cell-Based Multi-Tenant Platform](cell-based-multi-tenant-platform.md) | Tenant isolation vs. utilization and operational simplicity | Shuffle sharding, cell sizing, control-plane / data-plane separation, tenant migration |
| [Distributed Coordination Service](distributed-coordination-service.md) | Correctness of locks and leases vs. availability | Fencing tokens, lease expiry under GC pauses, the service everyone depends on being the biggest blast radius |
| [Observability Pipeline at 100M active series](observability-pipeline.md) | Cardinality and cost vs. the ability to debug an outage | Tail-based trace sampling, high-cardinality metrics, the pipeline staying up when everything else is down |

## Component-Level Examples (First-Principles Angle)

Three further IC6/IC7 examples take the other angle: instead of designing the *platform around* a component, they build the component itself from a product contract, a capacity and cost model, explicit APIs, a failure-first write path, and an evolution roadmap. Each pairs with one of the platform-level designs above; practicing both angles on the same domain is the fastest way to find the gaps in your reasoning.

| Example | Builds | Pairs with | The contrast |
|---------|--------|-----------|--------------|
| [Distributed Metrics Logging & Aggregation](question-1-distributed-metrics-logging-and-aggregation.md) | The metrics store: ingestion, cardinality strategy, rollups, query planning, alerting integration | [Observability Pipeline](observability-pipeline.md) | The pipeline example treats metrics as one of three signals and optimizes the *bill*; this one goes deep on the storage engine and query path |
| [Distributed Stream Processing like Kafka](question-2-distributed-stream-processing-like-kafka.md) | The log itself: partitions, replication, ISR, consumer groups, metadata scaling | [Event Streaming Platform](event-streaming-platform.md) | The platform example assumes the log exists and designs governance, tiering, and the exactly-once boundary; this one designs the log |
| [Globally Distributed Key-Value Store](question-3-design-a-key-value-store.md) | The store: sharding, quorum reads/writes, hot-key mitigation, LSM engine, TTL, multi-region DR | [Multi-Region Active-Active](multi-region-active-active.md) | The multi-region example classifies *application* data and picks a conflict model per class; this one builds the storage primitive those classes sit on |

These follow their own numbered structure rather than the twelve-section template below. Score them with the same [scorecard](../practice/scorecard.md); the dimensions apply regardless of structure.

## The Template Each Example Follows

Every example uses the same skeleton so they can be compared side by side. In an interview you will not have time for every section; the ordering reflects the priority a principal engineer gives each concern.

```
1. Problem Framing         — what is actually being asked, and what the prompt gets wrong
2. Requirements & SLOs     — functional, non-functional, explicit non-goals, the dominant requirement
3. Estimation              — the numbers, and the design constraints they *force*
4. Architecture            — the design, with the data flow for the critical path
5. Deep Dives              — the 2-3 sub-problems where the design succeeds or fails
6. Invariants              — what must always be true, and where it is enforced
7. Failure Modes           — component, dependency, and correlated failures; blast radius; degradation order
8. Evolution Path          — v1 → v2 → v3, and how to migrate between them
9. Cost Model              — the dominant cost driver and the knob that controls it
10. What Breaks at 10x     — the first thing that falls over, and the second
11. Rejected Alternatives  — what else was considered, and the specific reason it lost
12. Level Signals          — what a senior, staff, and principal answer to this prompt looks like
```

## How to Use These

1. **Read the problem framing first, then stop.** Write down your own requirements, dominant constraint, and architecture before reading further. Compare.
2. **Argue with the deep dives.** Every deep dive picks a side. Find the case where the other side wins.
3. **Practice the evolution path.** Interviewers at this level frequently ask "you inherited v1; how do you get to v3 without downtime?" That is a different skill from designing v3 from scratch.
4. **Rehearse the rejected alternatives out loud.** Being able to say "I considered X, and it loses because of Y" in one sentence is the single most reliable principal-level signal.
