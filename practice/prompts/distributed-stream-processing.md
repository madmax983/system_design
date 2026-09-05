# Prompt Card: Distributed Stream Processing like Kafka

**Worked example:** [examples/question-2-distributed-stream-processing-like-kafka.md](../../examples/question-2-distributed-stream-processing-like-kafka.md). Do not open it until step 4 of the [session protocol](../README.md#the-session-protocol). **Pairs with:** the [event streaming platform card](event-streaming-platform.md); attempt them in consecutive sessions.

## The Prompt

> Design a distributed, replayable log system like Kafka. Producers across regions write to topics and partitions; consumer groups read by offset and can replay from any offset or timestamp; ordering is preserved within a partition; the system must sustain hundreds of gigabytes per second of ingress with a durability guarantee that an acknowledged write is not lost.

## What This Trains

Building the log itself: the durability contract and what `acks=all` actually promises; in-sync replica mechanics and unclean election; partition-local ordering as the deliberate limit; consumer-group coordination and backpressure; metadata scaling as the hidden bottleneck.

## Pacing (45 min)

| Minutes | Do |
|---------|-----|
| 0-7 | Framing. What precisely is the durability contract? What is the ordering contract, and what is deliberately *not* promised? |
| 7-10 | Ingress, replication multiplier, broker count, partition count, and the metadata that partition count implies |
| 10-17 | Data plane (brokers, partitions, replicas) and control plane (metadata, leader election, assignment) |
| 17-35 | Deep dives. Pick two of: the write path with ISR, min in-sync replicas, and leader failover; consumer groups, offsets, and rebalancing; metadata scaling and why partition count is the ceiling |
| 35-42 | Failure modes: broker loss, AZ loss, unclean election, controller failure; backpressure under a slow consumer |
| 42-45 | Trade-offs: queue-only, object-storage-only, global ordering, exactly-once everywhere; roadmap |

## Before You Look: Questions to Answer in Your Attempt

- Walk an acknowledged write from producer to durable replicas. At which byte is it "acknowledged," and what can still be lost?
- A leader dies. Which replica may become leader, and what happens if none is in sync?
- What does a consumer group rebalance cost, and how do you make it cheaper?
- Why is global ordering rejected, and what does a producer do that needs related events ordered?
- What limits the number of partitions in one cluster?

## Hints (reveal one at a time, only if stuck)

<details><summary>Hint 1: the contract</summary>
Acknowledged means replicated to the in-sync replica minimum with the leader's log flushed to the point that the configured durability requires. Unclean leader election trades durability for availability; disable it for critical topics and say so.
</details>
<details><summary>Hint 2: ordering</summary>
Partition-local only. Global ordering would serialize through one sequencer and cap throughput and availability. Related events share a key; everything else uses event-time in the consumer.
</details>
<details><summary>Hint 3: metadata</summary>
Every partition is leader state, ISR state, and offsets across consumer groups. Controller failover time and per-broker file handles scale with partition count; that is the ceiling, and the answer past it is more clusters.
</details>
<details><summary>Hint 4: exactly-once</summary>
Idempotent producers plus transactions between topics on the platform. Not across an external sink. Most pipelines get a better cost-reliability balance from at-least-once plus idempotent consumers.
</details>

## Interviewer Follow-Ups (answer out loud before revealing)

<details><summary>Q1: Why not build it on object storage alone?</summary>
Excellent durability and cost, poor low-latency fetch and no consumer-group coordination semantics. The right use is as the cold tier behind the brokers, not as the log.
</details>
<details><summary>Q2: A consumer is ten times slower than the producer. What happens over an hour?</summary>
Lag grows; retention is the buffer; the consumer's lag alert fires; nothing is lost until retention expires. Backpressure to the producer is a policy decision per topic, not a default.
</details>
<details><summary>Q3: You need to add brokers and move partitions. What does that cost?</summary>
Replica reassignment copies partition data across the network under a throttle; leadership moves at the end; consumers see a brief pause. It is a rebalance, not a migration, and it is throttled so live traffic is not starved.
</details>
<details><summary>Q4: How does a cross-region setup work?</summary>
Mirror topics between regional clusters; offsets are cluster-local, so consumers resume by timestamp after a region failover. Compare with the platform example's multi-region stage.
</details>

## Scoring Focus

Weight dimensions 5 (critical path: the write path byte by byte), 8 (invariants: the durability contract), 10 (failure modes), and 15 (what breaks at 10x: metadata). A principal answer states what is deliberately not promised as clearly as what is.
