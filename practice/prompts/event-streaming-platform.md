# Prompt Card: Event Streaming Platform at 10M Events/sec

**Worked example:** [examples/event-streaming-platform.md](../../examples/event-streaming-platform.md). Do not open it until step 4 of the [session protocol](../README.md#the-session-protocol).

## The Prompt

> Design the company-wide event streaming platform. Hundreds of producer services emit events at a combined peak of 10M events/sec. Hundreds of consumer teams need to subscribe, some in real time and some for batch analytics. Consumers must be able to reprocess history after a bug. Some event types require exactly-once processing.

## What This Trains

Drawing the honest boundary of a guarantee (exactly-once stops at the platform's edge); treating an SDK and a schema registry as *organizational* mechanisms; designing replay as an isolated operation; finding the cost knob in a system where the obvious costs are not the real ones.

## Pacing (45 min)

| Minutes | Do |
|---------|-----|
| 0-7 | Framing. What does "exactly-once" mean and where does the platform's responsibility end? What is the produce-path availability requirement and why does it dominate? |
| 7-10 | Ingest, replicated write volume, hot storage, and the egress-versus-ingress surprise |
| 10-17 | Architecture: tiers by workload, the produce path with idempotency, the mandatory SDK |
| 17-35 | Deep dives. Pick two of: exactly-once and its boundary; schema evolution across teams; replay without starving live consumers |
| 35-42 | Failure modes; the degradation order ending with what is never rejected; the correlated failure |
| 42-45 | Evolution; cost knobs |

## Before You Look: Questions to Answer in Your Attempt

- What are the three distinct guarantees the platform can offer, precisely, and what happens at an external sink?
- Why one cluster is wrong, and what the tiering criterion is
- How a consumer reprocesses 30 days at 5x real-time without touching the brokers that serve live traffic
- What a producer must do to rename a field, and what the platform does to stop them breaking 40 consumers
- What is never sacrificed under overload, and why

## Hints (reveal one at a time, only if stuck)

<details><summary>Hint 1: the boundary</summary>
Idempotent produce within a producer session; transactional consume-transform-produce between topics; and *nothing* for external sinks, where the consumer must make the write idempotent or store the offset in the sink's own transaction. Say this precisely and the interviewer relaxes.
</details>
<details><summary>Hint 2: the numbers</summary>
Egress exceeds ingress by the fan-out factor (~5x). The bottleneck is network, not disk. Tiered storage to object storage cuts broker disk by 5x and is the mechanism that isolates replay from live traffic.
</details>
<details><summary>Hint 3: replay</summary>
Cold reads bypass the brokers entirely. Replay is a declared, quota-limited mode with its own consumer group. The sink must be idempotent on event ID *and* version-checked, or replay overwrites newer state with older.
</details>
<details><summary>Hint 4: degradation</summary>
Throttle replay, then batch, then low-priority producers by quota tier, then pause offload, then shed real-time consumers. Never reject transactional-tier produces while the in-sync replica minimum is satisfiable, because upstream data that cannot be produced is lost forever.
</details>

## Interviewer Follow-Ups (answer out loud before revealing)

<details><summary>Q1: A team says they need exactly-once into Postgres.</summary>
Store the consumer offset in Postgres in the same transaction as the data. The SDK ships a helper. Otherwise, upsert by event ID with a version check. The platform cannot do this for them and should not claim to.
</details>
<details><summary>Q2: A producer needs to change its partition key.</summary>
New topic. Dual-publish. Migrate consumers. There is no in-place path because existing data is partitioned by the old key. Say so before they build something that depends on one.
</details>
<details><summary>Q3: Migrate a topic between clusters without consumers noticing.</summary>
SDK-level dual-publish flag from the topic catalog (no producer redeploy); consumer group offset translation by timestamp; a cutover window where consumers read both and dedupe by event ID.
</details>
<details><summary>Q4: Object storage is down for six hours.</summary>
Segment offload backs up; brokers have 2x hot-window headroom so there is days of margin; cold replays fail; live traffic is unaffected. Alert at one hour of offload lag.
</details>
<details><summary>Q5: Why make the SDK mandatory?</summary>
Schema validation, idempotent-producer configuration, and spool-on-failure become guarantees instead of best-practice documents. The first incident without it is a team that turned off acks for throughput.
</details>

## Scoring Focus

Weight dimensions 1 (reframing), 7 (deep dive depth), 11 (degradation order), and 14 (cost). A principal answer draws the exactly-once boundary in the first ten minutes and names hot-window length and consumer locality as the cost knobs.
