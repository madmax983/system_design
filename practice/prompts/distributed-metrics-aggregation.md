# Prompt Card: Distributed Metrics Logging & Aggregation

**Worked example:** [examples/question-1-distributed-metrics-logging-and-aggregation.md](../../examples/question-1-distributed-metrics-logging-and-aggregation.md). Do not open it until step 4 of the [session protocol](../README.md#the-session-protocol). **Pairs with:** the [observability pipeline card](observability-pipeline.md); attempt them in consecutive sessions.

## The Prompt

> Design a distributed metrics logging and aggregation system. Services and hosts across many regions emit counters, gauges, and histograms with high-cardinality dimensions. The system must serve dashboards, ad-hoc queries, and alert evaluation with a freshness target under ten seconds, retain data across multiple horizons at different costs, and isolate tenants.

## What This Trains

Building a storage and query system from a product contract; the raw-versus-rollup retention trade; cardinality as a *data model* decision rather than an edge policy; a failure-first write path; making percentiles correct rather than approximately plausible.

## Pacing (45 min)

| Minutes | Do |
|---------|-----|
| 0-7 | Framing. What is the product contract: which query shapes, which freshness, which accuracy for percentiles? Which requirement dominates the storage design? |
| 7-10 | Points/sec, active series, bytes/point after compression, raw versus rollup storage across horizons |
| 10-17 | Data plane and control plane; the write path from agent to durable storage; the read path with query planning |
| 17-35 | Deep dives. Pick two of: the cardinality strategy and its data model; the write path's failure handling (at-least-once with idempotent writes); rollups and how percentiles survive them |
| 35-42 | Failure modes; tenant isolation on the read path; what the alerting path must not share with dashboards |
| 42-45 | Trade-offs: raw-only, rollup-only, exactly-once, warehouse-as-primary; the roadmap |

## Before You Look: Questions to Answer in Your Attempt

- What is stored per point, and how does a histogram stay mergeable across rollups and across regions?
- Where does a duplicate point get deduplicated, and what is the key?
- How does a long-range query avoid scanning raw data, and what accuracy does it lose?
- A tenant runs an expensive ad-hoc query during an incident. What protects alert evaluation?
- What is the cost per active series per month, and which horizon dominates?

## Hints (reveal one at a time, only if stuck)

<details><summary>Hint 1: the retention shape</summary>
Raw at full resolution for a short window, then rollups at increasing intervals for longer horizons. Raw-only is unsustainable for long-range queries; rollup-only loses debugging fidelity and percentile accuracy. The design needs both and a query planner that picks the tier.
</details>
<details><summary>Hint 2: percentiles</summary>
Store mergeable sketches (histogram buckets or a quantile sketch), never precomputed p99 values. A p99 of p99s is not a p99. Sketches merge across time windows, hosts, and regions with bounded error.
</details>
<details><summary>Hint 3: the write path</summary>
At-least-once from the agent through the broker to storage, with idempotent writes keyed on (series, timestamp, source sequence). Exactly-once across broker, storage, and aggregation is expensive and brittle; say so and pick idempotency.
</details>
<details><summary>Hint 4: isolation</summary>
Per-tenant query quotas and a separate execution pool for alert evaluation. The dashboard tier and the alerting tier must not share a query queue.
</details>

## Interviewer Follow-Ups (answer out loud before revealing)

<details><summary>Q1: Why not put everything in the OLAP warehouse?</summary>
Fine for offline analytics; poor fit for sub-ten-second freshness and streaming alert evaluation. The warehouse is a downstream consumer of rollups, not the primary store.
</details>
<details><summary>Q2: A region's ingestion is down for twenty minutes. What do dashboards show, and what happens on recovery?</summary>
Agents buffer locally with a bound; dashboards show a gap or a "partial" marker for that region; on recovery, the backlog replays through the same idempotent path and fills the gap. Alerts that depend on that region's data should be marked as insufficient-data rather than firing on the absence.
</details>
<details><summary>Q3: How do you roll out a change to the rollup function?</summary>
New rollups are computed alongside old ones for a validation window; the query planner reads old until the new tier is verified; then flip. Rollup changes are a migration, not a deploy.
</details>
<details><summary>Q4: Where do you put the cardinality limit, and what does exceeding it do?</summary>
At ingestion, per tenant and per metric, dropping new series loudly rather than sampling existing ones. Compare with the observability pipeline example's edge enforcement; the data model here (series identity and index) is what makes the limit necessary.
</details>

## Scoring Focus

Weight dimensions 5 (critical path), 7 (deep dive depth: the data model), 14 (cost across horizons), and 16 (alternatives). The worked example's own trade-off section is a good check on dimension 16.
