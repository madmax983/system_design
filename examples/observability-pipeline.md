# Observability Pipeline at 100M Active Series

> **Prompt:** Design the metrics, logs, and traces pipeline for an engineering organization of 3,000 engineers running ~5,000 services. Peak ingest: 100M active metric series, 5 TB/day of logs, and 50K traces/sec. Engineers need to debug an outage in the first five minutes using this data. Cost is a serious constraint: the current bill is growing faster than the business.

Observability is the system that has to work *when nothing else does*. It is also the system whose cost grows superlinearly with the organization, because every engineer adds telemetry and nobody removes it. The principal-level design solves both: a pipeline that is more available than the systems it observes, and a cost model where the marginal series has an owner and a price.

## 1. Problem Framing

**What the prompt gets right:** "first five minutes of an outage" is the correct SLO framing. Observability that is complete but slow, or that is down when production is down, has failed at its only job.

**What the prompt gets wrong:**
- "Metrics, logs, and traces" as three parallel pipelines is how the bill got out of control. They are three *views* of the same events. A design that treats them as one ingest path with three storage backends, and that lets each signal *reduce* the volume of the others (traces sample logs; metrics are derived from traces), is what actually bends the cost curve.
- "100M active series" is not the scale problem; it is the *cardinality* problem. 100M series at 10-second resolution is ~10M samples/sec, which is manageable. What breaks systems is that those 100M series are 1,000 metrics × 100,000 label combinations, and one engineer adding `user_id` as a label turns it into 10 billion. Cardinality control is the core design problem.
- "Cost is a serious constraint" is stated but not sized. The design needs to make cost *per team* visible and *per signal* controllable, so that the constraint is enforced by the people generating the data rather than by the platform team saying no.
- The pipeline's own dependency graph is the trap. If the observability pipeline depends on the service mesh, the coordination service, or the main object store, then an outage in those takes observability down at the moment it is needed. The pipeline must be built on a *separate, smaller, more boring* dependency set.

**The reframe to state out loud:** "I'm going to design one ingest path with three backends, with cardinality and volume controls at the edge, tail-based sampling for traces that drives log sampling, and a strict rule that the pipeline depends on nothing the rest of production depends on. Cost will be attributed per team and per signal, and the platform's job is to make the expensive thing visible, not to forbid it."

## 2. Requirements & SLOs

### Functional
- Ingest metrics (push and scrape), structured logs, and distributed traces from every service via a standard agent/SDK (OpenTelemetry-shaped)
- Query metrics with a PromQL-like language; query logs by structured fields and full text; view traces by ID and search by attributes
- Alerting on metrics with burn-rate SLO alerts as the default template
- Correlation: from a metric spike to the traces in that window; from a trace to its logs; from a log line to its trace
- Retention: metrics 15 days at full resolution then downsampled to 13 months; logs 30 days hot then 1 year cold; traces 7 days sampled, 30 days for error/slow traces
- Per-team cost attribution and quotas

### Non-Functional
| Requirement | Target | Note |
|-------------|--------|------|
| Pipeline availability | 99.99%, **measured during production incidents** | The only availability number that matters |
| Ingest to queryable (metrics) | p99 < 30 s | Alerts fire on data ≤ 30 s old |
| Ingest to queryable (logs, traces) | p99 < 60 s | Debugging, not alerting |
| Query latency (dashboards, recent data) | p99 < 2 s | An incident dashboard that takes 20 s to load is not used |
| Data loss under pipeline overload | Bounded and *visible*: sampled, never silently dropped | Engineers must know when they are looking at a sample |
| Cardinality | Hard per-metric and per-team limits; new label keys require registration | The control that prevents 10B series |
| Cost | Growth ≤ business growth; per-team attribution accurate to 5% | The business constraint stated as an SLO |
| Dependencies | The pipeline does not depend on any production system it observes | Enforced architecturally |

### Non-Goals
- Not designing the alerting UI or on-call paging integration beyond the alert evaluation
- Not a security information and event management (SIEM) system; security logs are a separate tenant with its own retention
- Not a business analytics warehouse; product analytics events are not telemetry and go to the [event streaming platform](event-streaming-platform.md)

**The dominant requirement:** *availability during incidents*, which forces the separate-dependency rule and the graceful-degradation design. Close second: cardinality control, which is what makes the cost SLO achievable.

## 3. Estimation

| Quantity | Estimate | Derivation |
|----------|----------|------------|
| Active series | 100M | Given |
| Samples/s | ~10M | 100M / 10 s scrape interval |
| Sample size (compressed) | ~1.5 bytes | Gorilla-style delta-of-delta compression; ~16 bytes raw |
| Metrics ingest | ~15 MB/s compressed, ~160 MB/s raw | The bandwidth is not the problem |
| Metrics storage (15 days full res) | 10M × 1.5 B × 1.3M s ≈ 20 TB, ×2 replicas ≈ 40 TB | Fits on a modest fleet |
| Series index | 100M series × ~200 B of labels ≈ 20 GB | Must be in memory for query; this is the cardinality cost |
| Logs ingest | 5 TB/day ≈ 60 MB/s average, ~300 MB/s peak | The volume driver |
| Logs storage (30 days hot, ~10x compressed) | 15 TB hot; 180 TB/year cold | Cheap in storage, expensive in indexing |
| Traces | 50K/s × ~20 spans × ~500 B ≈ 500 MB/s raw | **Larger than logs.** This is the number that forces sampling |
| Traces after tail sampling (keep ~5%) | ~25 MB/s | And all errors and slow traces are in the 5% |
| Query load | ~10K queries/s (dashboards), spiking 10x during incidents | Dashboards auto-refresh; incident spikes are the peak to size for |

### Constraints the numbers force

1. **The series index, not the samples, is the metrics cost.** 100M series is 20 GB of label metadata that every query touches. At 1B series it is 200 GB and queries over "all series matching this label" take minutes. Cardinality control is not optional.
2. **Raw traces exceed logs in volume**, and most of them are uninteresting successful requests. Head-based sampling (decide at the root span) is cheap but throws away the interesting traces at the same rate as boring ones. Tail-based sampling (decide after the trace completes) keeps every error and outlier at the cost of buffering every trace for a few seconds. The buffering cost (~500 MB/s × 10 s = 5 GB in memory across the sampler fleet) is trivial. Tail-based it is.
3. **Incident query load is 10x normal.** The query tier must be sized for incidents, not for normal operation, or it will be slow exactly when it is needed. This is the failover-headroom problem from the [multi-region example](multi-region-active-active.md), applied to observability.
4. **5 TB/day of logs at 10x compression is cheap to store and expensive to index.** Full-text indexing of everything is the log-cost driver. Index structured fields; store the raw line compressed; full-text search is a scan over a time-bounded, field-filtered subset.

## 4. Architecture

```
   Services (5,000)                      ┌──────────────────── Separate dependency set ────────────────────┐
   ┌────────────────┐                    │  (own Kafka, own object store bucket + account, own DNS zone,   │
   │ OTel SDK       │                    │   own network path; NO service mesh, NO coordination service)   │
   │ + local agent  │                    │                                                                 │
   │  - batching    │      ┌─────────────▼──────────────┐                                                 │
   │  - cardinality │─────▶│  Edge Collectors (per AZ)  │  first line of defense:                         │
   │    pre-check   │      │  - auth, tenant tagging    │  - reject unregistered label keys               │
   │  - disk buffer │      │  - cardinality enforcement │  - per-team volume quotas                        │
   │    (5 min)     │      │  - volume quotas           │  - sampling *decisions* stamped, not applied     │
   └────────────────┘      │  - protocol normalization  │                                                 │
                           └──────┬──────┬──────┬───────┘                                                 │
                                  │      │      │                                                         │
                     ┌────────────▼┐  ┌──▼────────────┐  ┌▼───────────────────┐                          │
                     │ Metrics     │  │ Logs           │  │ Traces             │                          │
                     │ Ingest      │  │ Ingest         │  │ Tail Sampler       │                          │
                     │ (Kafka)     │  │ (Kafka)        │  │ (Kafka + stateful  │                          │
                     └──────┬──────┘  └───────┬────────┘  │  sampler fleet,    │                          │
                            │                 │           │  keyed by trace ID)│                          │
                            │                 │           └────────┬───────────┘                          │
                            │                 │      sampling      │                                       │
                            │                 │◀──── decisions ────┘  (kept traces → keep their logs;     │
                            │                 │      (trace ID set)    dropped traces → sample their logs) │
                            ▼                 ▼                    ▼                                       │
                     ┌─────────────┐  ┌───────────────┐  ┌────────────────────┐                          │
                     │ TSDB        │  │ Log Store     │  │ Trace Store        │                          │
                     │ - hot: 15d  │  │ - hot: 30d    │  │ - 7d sampled       │                          │
                     │   in-memory │  │   columnar,   │  │ - 30d errors/slow  │                          │
                     │   index +   │  │   field-      │  │ - object storage   │                          │
                     │   local SSD │  │   indexed     │  │   with trace-ID    │                          │
                     │ - downsample│  │ - cold: 1y    │  │   index            │                          │
                     │   → 13mo    │  │   obj storage │  │                    │                          │
                     └──────┬──────┘  └───────┬───────┘  └─────────┬──────────┘                          │
                            └─────────────────┼────────────────────┘                                      │
                                              ▼                                                           │
                              ┌──────────────────────────────┐   ┌──────────────────────┐                 │
                              │  Query Federation            │   │  Alert Evaluator     │                 │
                              │  (sized for 10x incident     │   │  (burn-rate SLO      │                 │
                              │   load; correlation joins    │   │   templates; runs    │                 │
                              │   across signals by trace ID,│   │   against TSDB only; │                 │
                              │   service, time window)      │   │   independent of the │                 │
                              └──────────────────────────────┘   │   query tier)        │                 │
                                                                 └──────────────────────┘                 │
                                                                                                          │
                              ┌──────────────────────────────┐                                            │
                              │  Cost Attribution            │  per team, per signal, per metric:         │
                              │  (series count × retention,  │  daily report; quota proposals             │
                              │   log bytes, trace spans)    │                                            │
                              └──────────────────────────────┘                                            │
                                                                                                          │
                              └───────────────────────────────────────────────────────────────────────────┘
```

## 5. Deep Dives

### Deep Dive 1: Cardinality control at the edge

Cardinality is controlled at *three* points, because any single point can be bypassed.

**1. Registration.** Every metric name and every label *key* is registered by a team before it is accepted. Registration is a config change in the team's repo (a YAML file reviewed like code), not a ticket. Unregistered metrics are dropped at the edge collector *with a metric counting the drops per team*, so the team sees it immediately. Label *values* are not registered (that would be unworkable) but are bounded.

**2. Per-metric cardinality limits.** Each registered metric declares its expected cardinality (default 10K series). The edge collector tracks active series per metric per team with a sketch (HyperLogLog is enough) and, above the limit, stops accepting *new* series for that metric while continuing to accept existing ones. The team gets a page-severity alert: "metric X exceeded 10K series; new series dropped." This is the control that prevents `user_id` from becoming a label in production.

**3. Per-team series budget.** Sum of all series across the team's metrics, enforced the same way. The budget is the cost lever: a team that wants more series pays for them from its budget allocation, which appears on its cost report.

**What makes this a principal design and not a policy:** the enforcement point is the edge collector, which is *before* the expensive storage; the failure mode is "new series dropped, loudly" rather than "the TSDB fell over for everyone"; and the mechanism for a team to get more capacity is self-service with a price, not a negotiation with the platform team.

**The escape hatch for high-cardinality debugging:** exemplars. A metric sample can carry a trace ID; the trace has the high-cardinality context (user, request, etc.). "I need `user_id` on this metric" is almost always "I need to get from this metric to a trace," and exemplars do that with zero cardinality cost.

### Deep Dive 2: Tail-based sampling that drives everything

The tail sampler buffers all spans of a trace until the trace completes (root span ends, or a 10-second timeout), then decides:

| Keep if | Rate |
|---------|------|
| Any span has an error | 100% |
| Trace duration > p99 for its root operation (tracked per operation, per hour) | 100% |
| Trace touches a service currently flagged "under investigation" (set by on-call) | 100% |
| Random | 1-5%, adjusted per service to hit a per-team span budget |

**Statefulness.** All spans of a trace must reach the *same* sampler instance. The edge collectors partition by trace ID onto a Kafka topic; each sampler consumes a partition set; the trace's spans are colocated. A sampler crash loses the in-flight buffer for its partitions (~10 s of traces); that is acceptable and stated. A more durable design would checkpoint the buffer and is not worth it.

**The decision propagates to logs.** Log lines carry a trace ID when emitted within a traced request (the SDK does this automatically). The log ingest path holds lines for the sampling window and keeps:
- 100% of lines for kept traces
- 100% of lines at WARN and above regardless
- A configurable sample (default 1%) of INFO and below for dropped traces

This is the cost bend. Logs are 5 TB/day because every INFO line of every successful request is stored. With trace-driven sampling, a successful request's INFO logs are kept only if the trace was kept. In practice this cuts log volume by 60-80% with *zero* loss of debuggability for the requests anyone will ever look at, because the requests anyone looks at are the errors and outliers, and those are kept at 100%.

**The honest cost:** an engineer looking for "the INFO logs of a specific successful request from an hour ago" will usually not find them. That is stated in the UI ("this request was not sampled; here is the aggregate for its operation") rather than discovered. And on-call can flip a service to 100% during an investigation.

### Deep Dive 3: Staying up when everything else is down

The pipeline must not share failure modes with production. The rule is architectural and enforced:

| Production dependency | Pipeline substitute | Why |
|----------------------|--------------------|-----|
| Service mesh / internal load balancers | Direct connections from agents to edge collectors by static DNS, with client-side load balancing | A mesh control-plane outage would blind observability |
| Coordination service (etcd/ZooKeeper) | None. Collectors and samplers are stateless or use Kafka's own coordination | See the [coordination service example](distributed-coordination-service.md) for why it is the biggest blast radius |
| Main Kafka cluster | The pipeline's own Kafka, in its own account, with its own on-call | The [event streaming platform's](event-streaming-platform.md) outage should be *visible in* observability, not *cause* an observability outage |
| Main object storage bucket/account | Separate account, separate bucket, separate credentials | Account-level throttling or a credential rotation mistake must not cascade |
| Main identity provider | Agent auth by long-lived per-service tokens validated locally; UI auth by the IdP with a *break-glass* local account | The IdP outage must not lock on-call out of the dashboards |
| Deploy tooling | The pipeline deploys with the same tooling but on a *different schedule*, never in the same window as a large production deploy | A bad deploy tooling release must not hit both |
| Configuration service / feature flags | Pipeline config is static files shipped with the binary, plus a small on-call override mechanism | A config-push outage is the most common cause of correlated failure |

**Degradation under overload** (a production incident *is* an ingest spike: error logs and retried requests multiply):
1. Metrics are never sampled; they are the alerting signal. Cardinality limits already bound them
2. Trace sampling rate drops adaptively; errors and slow traces are still 100% until the sampler is at capacity, then errors are prioritized
3. INFO logs are sampled harder; WARN+ are preserved
4. Agents buffer to local disk (5 min) if collectors reject; then drop oldest, with a counter
5. Query tier sheds dashboard auto-refresh (extends refresh intervals) before shedding ad-hoc queries; the alert evaluator has reserved capacity and is never shed

**The pipeline observes itself** with a tiny, entirely separate stack (a handful of nodes, a different vendor or a different open-source tool) that answers exactly one question: is the main pipeline ingesting and queryable? That meta-monitor pages the pipeline's on-call. It has no dashboards worth speaking of; it is a heartbeat.

### Deep Dive 4: Cost attribution as a product feature

The platform's cost problem is a *visibility* problem. Engineers add telemetry because it is free to them. The fix is to make it not free:

- **Every series, log byte, and span has an owning team** (from registration and from the service catalog).
- **A daily report per team**: series count, log GB, span count, and the dollar cost of each, with a trend and the top 10 metrics/loggers by cost.
- **Budgets, not caps**: a team's budget is set with its director; exceeding it does not cut the team off but appears on the director's report. Hard caps exist only for the pathological cases (cardinality explosion).
- **The platform team's job is the price list, the report, and the tools to reduce cost** (a "show me my useless metrics" view: series with no dashboard, alert, or query touching them in 30 days).

In practice, the first such report finds that 30-50% of series have never been queried. Deleting them requires no engineering and is the biggest single cost reduction the platform will ever ship.

## 6. Invariants

| Invariant | Where enforced | Verification |
|-----------|---------------|--------------|
| Every alert-driving metric is ingested unsampled | Metrics path has no sampling stage; cardinality limits drop *new series*, never samples | Synthetic metric with known values; alert on any missing sample |
| Every error trace and every WARN+ log is retained | Tail sampler and log sampler rules | Synthetic error request per minute; verify its trace and logs are queryable within 60 s |
| The pipeline shares no dependency with observed production systems | Dependency register, reviewed quarterly; network policy | Game day: main Kafka, mesh, and coordination service taken down; observability must stay up |
| Data that is sampled is labeled as sampled in query results | Sampling decisions are stored with the data | UI test |
| Every stored series/log/span has an owning team | Registration; unowned data is rejected at the edge | Daily report shows 0 unattributed cost |
| Alert evaluation is not affected by dashboard query load | Reserved capacity; separate process pool | Load test: 10x dashboard load; alert evaluation latency unchanged |

**Deliberately relaxed:** INFO-level logs for successful, unsampled requests are not retained. Trace completeness under sampler failure (~10 s loss). Both are stated in the platform's documentation as contractual, not as incidents.

## 7. Failure Modes

| Failure | Blast radius | Behavior | Mitigation |
|---------|-------------|----------|------------|
| Edge collector AZ fails | 1/3 of ingest capacity | Agents fail over to other AZs' collectors via DNS; buffer briefly | Collectors sized N+1 per AZ; agent disk buffer |
| Pipeline Kafka degraded | All signals delayed | Agents buffer 5 min then drop oldest with counter | Separate on-call; over-provisioned because it is small relative to production Kafka |
| TSDB node fails | Its series shard, for the replication-catch-up window | Queries return partial data with a "partial" flag | 2 replicas; partial-result flag is a UI requirement |
| Tail sampler fleet overloaded | Sampling degrades toward head-based (random) | Errors still prioritized until capacity is exhausted | Adaptive rates; the sampler reports its own drop rate |
| Cardinality explosion from one team | That team's new series are dropped | Everyone else is unaffected | The whole point of edge enforcement |
| Query tier overloaded during an incident | Dashboards slow | Auto-refresh shed first; alert evaluator unaffected | Sized for 10x; the sizing is checked quarterly against actual incident load |
| Object storage (cold tier) unavailable | Cold queries fail; hot tier unaffected | Hot tier holds 15-30 days | Alert; not an incident-time concern |
| Alert evaluator fails | **No alerts fire** | Silent | The meta-monitor's heartbeat alert is evaluated by the *meta* stack; a dead evaluator is detected within 2 minutes by the absence of its heartbeat |
| **Correlated: production incident causes 10x error logs and traces** | Ingest spike exactly when needed | Degradation order above | This is the load test scenario, run quarterly with replayed incident traffic |
| **Correlated: pipeline deploy during a production incident** | Could take down observability mid-incident | Deploy freeze is automatic when any P0/P1 incident is open | The freeze is enforced by the deploy tooling, not by policy |

## 8. Evolution Path

**v1: metrics with cardinality registration; logs with field indexing; head-sampled traces.** The registration mechanism is the part that cannot be retrofitted (every team would have to register thousands of existing metrics), so it exists from the start even when it feels like bureaucracy for a small org.

**v2: separate dependency set.** Usually forced by the first incident where observability went down with production. Doing it before that incident is the principal move.

**v3: tail-based sampling and trace-driven log sampling.** This is the cost inflection. Requires trace IDs in logs everywhere, which requires the SDK to have been mandatory since v1.

**v4: cost attribution and budgets.** Requires ownership metadata to be complete, which requires registration since v1.

**v5: correlation UI and exemplars.** The debugging experience that makes "first five minutes" real: metric spike → exemplar trace → its logs, in three clicks.

## 9. Cost Model

| Component | Dominant driver | The knob | Typical share |
|-----------|----------------|----------|---------------|
| Metrics | Series count (index) and retention | Cardinality limits; deleting unqueried series; downsampling age | 30% |
| Logs | Ingest volume × index breadth | Trace-driven sampling (60-80% reduction); indexing only registered fields; hot retention length | 40% |
| Traces | Span volume after sampling | Sampling rate; span attribute size limits | 15% |
| Query tier | Sized for incident load | Auto-refresh intervals; pre-aggregated recording rules for dashboards | 10% |
| Pipeline (collectors, Kafka, samplers) | Ingest volume | Follows from the above | 5% |

**The insight:** logs are the cost, sampling driven by traces is the fix, and the *prerequisite* for that fix (trace IDs in every log line) is an SDK decision made years earlier. The second insight: 30-50% of metrics cost is series nobody queries, and deleting them is a report, not a project.

## 10. What Breaks at 10x

At 1B series, 50 TB/day of logs, 500K traces/s:

**First: the series index.** 200 GB of label metadata does not fit in memory on query nodes. Fix: the index is sharded by metric name across the TSDB fleet and queries fan out; "all series matching label X" across metrics becomes a federated query with a cost limit. And cardinality limits get tighter, because the *fleet* cannot absorb what the *limits* allow.

**Second: tail sampler state.** 5 GB of in-flight buffer becomes 50 GB across the fleet, still fine, but the partition count and rebalance behavior of the sampler's Kafka consumer group become the operational problem. Fix: sticky partition assignment and a sampler fleet that scales with partitions, not with traffic.

**Third: query federation across signals.** Correlation queries (metric → traces → logs) at incident time across 10x the data. Fix: pre-computed correlation indexes (trace ID → log segment; service+time → trace IDs) built at ingest, so correlation is a lookup, not a scan.

## 11. Rejected Alternatives

| Alternative | Why it loses |
|-------------|-------------|
| **Three separate pipelines, three vendors** | No cross-signal sampling, no correlation, three bills, three on-calls |
| **Head-based trace sampling only** | Throws away errors and outliers at the same rate as successes; the interesting 1% is lost with the boring 99% |
| **Index everything in logs (full-text on all fields)** | The log cost driver. Field-indexed with time-bounded scans for full text is 5-10x cheaper with no practical loss |
| **No cardinality registration ("engineers should be careful")** | The 10B-series incident is a matter of when. Registration is a YAML file in the team's repo; the friction is minutes and the protection is total |
| **Run observability on the same Kafka/mesh/etcd as production** | Saves a small amount of money and guarantees observability is down during the outages that matter |
| **Sample metrics under load** | Sampled metrics make alerts unreliable; metrics are small enough that cardinality control alone bounds them |
| **Hard caps on team cost** | Cuts off a team mid-incident when their error logs spike. Budgets with visibility, plus hard caps only for cardinality explosion |
| **A single monolithic query tier for alerts and dashboards** | Dashboard load during an incident delays alert evaluation, which delays the next page |

## 12. Level Signals

**A senior answer** picks Prometheus-style metrics, an Elasticsearch-style log store, and a tracing backend; adds Kafka in front; sets retention; adds dashboards and alerts.

**A staff answer** enforces cardinality at ingest, uses tail-based sampling, tiers storage, sizes the query tier for incidents, and separates alert evaluation from dashboards.

**A principal answer** does all of that, and additionally:
- Treats the three signals as one ingest path and uses trace sampling decisions to drive log sampling, which is the cost inflection
- Insists on a separate dependency set and can enumerate the specific production dependencies it avoids and why
- Puts cardinality enforcement at the edge, makes the failure mode "new series dropped, loudly, for one team," and offers exemplars as the escape hatch
- Designs cost attribution as a product feature with self-service budgets, and knows the first report will find 30-50% waste
- Identifies which decisions (SDK mandatory, registration, trace IDs in logs) must be made in v1 because v3 and v4 depend on them
- Names the deploy-during-incident and the ingest-spike-during-incident as the correlated failures, and makes the deploy freeze automatic
- Runs a separate meta-monitor whose only job is to detect that the alert evaluator is alive

**Interviewer follow-ups to expect:**
- "An engineer says they need `user_id` on a latency metric to debug a customer complaint." (Exemplars: the metric sample links to a trace, the trace has the user ID. Zero cardinality cost. If they truly need per-user aggregates, that is an analytics query over traces, not a metric.)
- "How do you know the pipeline is up during a production outage?" (The meta-monitor: a separate, tiny stack that checks ingest and query end-to-end every 30 s and pages the pipeline on-call by a path that does not go through the main pipeline's alerting.)
- "A team's logs jump 50x because a bug is logging a stack trace per request. What happens?" (Their trace-driven sampling keeps errors at 100%, so the spike is partly kept; their per-team volume quota at the edge kicks in and samples the excess with a counter; their cost report shows it the next day; the platform is unaffected. If the quota were a hard cap, they would lose the error logs mid-incident, which is why it is a soft quota with sampling rather than a cap with drops.)
- "What is the first thing you would do to cut the bill by 30%?" (Ship the unqueried-series report and delete what it finds. Then trace-driven log sampling. Neither requires new infrastructure.)
