# Prompt Card: Observability Pipeline at 100M Active Series

**Worked example:** [examples/observability-pipeline.md](../../examples/observability-pipeline.md). Do not open it until step 4 of the [session protocol](../README.md#the-session-protocol).

## The Prompt

> Design the metrics, logs, and traces pipeline for 3,000 engineers running ~5,000 services. Peak ingest: 100M active metric series, 5 TB/day of logs, 50K traces/sec. Engineers need to debug an outage in the first five minutes using this data. Cost is a serious constraint: the current bill is growing faster than the business.

## What This Trains

Designing a system that must be more available than what it observes, with a disjoint dependency set; treating cardinality as the real scale problem; using one signal's sampling decision to cut another signal's cost; making cost a product feature rather than a policy.

## Pacing (45 min)

| Minutes | Do |
|---------|-----|
| 0-7 | Framing. Why are three separate pipelines the cause of the bill? What is the real scale problem behind "100M series"? What must the pipeline *not* depend on? |
| 7-10 | Samples/s versus index size; raw traces exceeding logs; incident query load at 10x |
| 10-17 | One ingest path, edge collectors, three backends, the alert evaluator separate from the query tier |
| 17-35 | Deep dives. Pick two of: cardinality control at the edge; tail-based sampling driving log sampling; staying up when everything else is down |
| 35-42 | Failure modes including the ingest spike during an incident and the deploy during an incident; degradation order |
| 42-45 | Cost attribution as a product; what to do first for 30% |

## Before You Look: Questions to Answer in Your Attempt

- Where is cardinality enforced, what is the failure mode when a team exceeds it, and what is the escape hatch for high-cardinality debugging?
- Why tail-based sampling, what state does it require, and how does its decision reduce log volume?
- List the production dependencies the pipeline must avoid and the substitute for each
- How do you know the alert evaluator is alive?
- What is the cheapest 30% cost reduction?

## Hints (reveal one at a time, only if stuck)

<details><summary>Hint 1: the reframe</summary>
Three views of one event stream. One ingest path with three backends, and sampling decisions that flow from traces to logs. "100M series" is an index-size and cardinality problem, not a sample-rate problem.
</details>
<details><summary>Hint 2: cardinality</summary>
Registration of metric names and label keys in the team's repo; per-metric limits enforced at the edge collector by dropping *new* series loudly; per-team budgets with a price. Exemplars link a metric sample to a trace so high-cardinality context costs nothing.
</details>
<details><summary>Hint 3: the cost bend</summary>
Tail sampler keeps 100% of errors and outliers, ~1-5% of the rest. Log lines carry trace IDs; INFO logs are kept only for kept traces; WARN and above always. 60-80% log reduction with no loss of the logs anyone would look at.
</details>
<details><summary>Hint 4: independence</summary>
Own Kafka, own object-storage account, static DNS instead of the mesh, no coordination service, local token validation instead of the IdP, static config instead of the config service, deploys on a different schedule with an automatic freeze during incidents. A tiny separate meta-monitor whose only job is "is the main pipeline up?"
</details>

## Interviewer Follow-Ups (answer out loud before revealing)

<details><summary>Q1: An engineer needs user ID on a latency metric to debug one customer.</summary>
Exemplars: the sample links to a trace that carries the user ID. Zero cardinality cost. Per-user aggregates are an analytics query over traces, not a metric.
</details>
<details><summary>Q2: How do you know the pipeline is up during a production outage?</summary>
A separate, tiny meta-monitor checks ingest and query end-to-end every 30 seconds and pages the pipeline on-call through a path that does not use the main pipeline's alerting.
</details>
<details><summary>Q3: A team's logs jump 50x from a stack-trace-per-request bug.</summary>
Errors are kept at 100% by trace-driven sampling; the team's edge volume quota samples the excess with a visible counter; their cost report shows it tomorrow; nobody else is affected. Soft quota with sampling, not a hard cap with drops, so they do not lose error logs mid-incident.
</details>
<details><summary>Q4: First thing you would do to cut the bill 30%?</summary>
Ship the unqueried-series report and delete what it finds (typically 30-50% of series). Then trace-driven log sampling. Neither needs new infrastructure.
</details>
<details><summary>Q5: Why is the alert evaluator separate from the query tier?</summary>
Incident dashboard load is 10x normal, exactly when alert evaluation latency matters most. Reserved capacity, separate process pool, never shed.
</details>

## Scoring Focus

Weight dimensions 1 (reframing), 10 (failure modes: correlated), 11 (degradation order), and 14 (cost). A principal answer enumerates the avoided dependencies specifically and treats the cost report as the first deliverable.
