# Prompt Bank

Additional principal-level prompts without full worked examples. Use them to practice the *method* rather than recall a solution. Each has hidden hints: the dominant constraint, the trap, and the sub-problem to deep-dive. Attempt first; reveal after.

Score each with the [scorecard](scorecard.md). Where a worked example is close, it is linked; read it *after* your attempt.

---

## 1. Global Feature Flag and Configuration Delivery

> Design the service that delivers feature flags and runtime configuration to 5,000 services and 200M mobile clients. A flag change must reach servers in under 5 seconds and clients within a minute. A bad flag value has, historically, caused the company's worst outages.

<details><summary>Dominant constraint</summary>
The last sentence. This is a blast-radius and change-safety problem, not a delivery-latency problem. Staged rollout of *config* with automatic rollback on SLO burn is the design; sub-5-second delivery is the easy part.
</details>
<details><summary>The trap</summary>
Putting the flag service on the request path. Every consumer must cache locally and function indefinitely on last-known values; the service pushes, nobody pulls per request. See the control-plane rule in the <a href="../examples/cell-based-multi-tenant-platform.md">cell-based example</a>.
</details>
<details><summary>Deep dive here</summary>
Rollout mechanics: percentage-based, cohort-based, and region-staged flag changes with automatic halt on error-rate burn; the evaluation model (server-side vs. client-side) and its consistency guarantee ("a user sees the same value for a session"); and the audit trail that answers "who changed what, when, and what happened."
</details>

## 2. Idempotent Webhook Delivery at Scale

> Design outbound webhook delivery for a platform that sends 500M webhooks/day to 100K customer endpoints. Customers' endpoints are slow, flaky, and sometimes malicious. Delivery must be at-least-once with ordering per customer, and one slow customer must not delay others.

<details><summary>Dominant constraint</summary>
Isolation between customers. Per-customer queues with per-customer concurrency limits and circuit breakers; the shared component is the scheduler, not the delivery workers.
</details>
<details><summary>The trap</summary>
A single global queue. One customer with a 30-second endpoint occupies every worker. Also: retry storms after a platform outage, when every customer's backlog is retried at once.
</details>
<details><summary>Deep dive here</summary>
Ordering per customer under retries (a failed delivery blocks the customer's queue, or is it skipped and the customer told?); retry schedule with jitter and a dead-letter policy the customer can see; signing and replay protection; the customer-facing "redeliver from timestamp" feature and what it requires of the storage layer.
</details>

## 3. Distributed Job Scheduler (Cron at Company Scale)

> Design the scheduler that runs 1M scheduled jobs (from every-minute to monthly) for thousands of teams. A job must run at most once per scheduled time, must not be silently skipped, and a misfire (the scheduler was down at the scheduled time) must be handled according to a per-job policy.

<details><summary>Dominant constraint</summary>
"At most once per scheduled time" under scheduler failover. The scheduler is a leader-elected component, and the fencing token problem from the <a href="../examples/distributed-coordination-service.md">coordination service example</a> applies directly: a paused old leader must not fire jobs the new leader already fired.
</details>
<details><summary>The trap</summary>
Time. Timezones, DST transitions ("run at 2:30am" on a day that has no 2:30am), leap seconds, and clock skew between scheduler nodes. Every real scheduler has a DST incident.
</details>
<details><summary>Deep dive here</summary>
Sharding jobs across scheduler instances (by job ID hash, with per-shard leadership) and what happens during rebalancing; the misfire policy matrix (fire immediately / skip / fire once for all missed); execution tracking so that "did it run?" is answerable; and the thundering herd at :00 when 200K jobs share a schedule.
</details>

## 4. Real-Time Collaborative Document Editing

> Design the backend for a collaborative document editor: 100M documents, up to 100 concurrent editors per document, sub-100 ms echo of a keystroke to other editors, full history, and offline editing with later merge.

<details><summary>Dominant constraint</summary>
Convergence: every editor must reach the same document state given the same operations, in any order. That is a CRDT or OT decision, and it is a one-way door. The <a href="../examples/multi-region-active-active.md">multi-region example's</a> conflict-resolution table is the starting point.
</details>
<details><summary>The trap</summary>
Treating the document as a blob with last-writer-wins. Also: forgetting that "full history" at keystroke granularity is a storage and compaction problem (snapshots plus operation logs, with a compaction policy that preserves the ability to reconstruct any version).
</details>
<details><summary>Deep dive here</summary>
The per-document session: one coordinator per active document (leader election, fencing, failover with client reconnection and operation replay); the storage model (snapshot every N ops, operation log, GC of tombstones); and offline merge (the client has a vector clock or a version; the merge is the same CRDT/OT path with a long delay).
</details>

## 5. Search Indexing Pipeline with Freshness SLOs

> Design the indexing pipeline for a marketplace search engine: 500M listings, 50K updates/sec, and a requirement that a price or availability change is searchable within 10 seconds while a full re-index (for a ranking model change) completes within 24 hours without affecting freshness.

<details><summary>Dominant constraint</summary>
Two write paths with different SLOs into the same index: a real-time path for small, urgent updates and a bulk path for re-indexing. They must not compete. The <a href="../examples/event-streaming-platform.md">replay-isolation design</a> from the streaming example is the pattern.
</details>
<details><summary>The trap</summary>
Re-indexing in place. A ranking model change means every document changes; doing that on the live index starves the freshness path. Build the new index alongside, swap atomically, and keep applying real-time updates to *both* during the build.
</details>
<details><summary>Deep dive here</summary>
The dual-index swap (alias flip, real-time updates applied to both, the window where they diverge and how it is bounded); segment/shard design so that price updates are cheap (separate doc-values from the inverted index; partial updates); and the freshness SLI itself (how do you measure "searchable within 10 s" end-to-end?).
</details>

## 6. Secrets Management and Rotation

> Design the service that stores, distributes, and rotates secrets (database credentials, API keys, TLS certificates) for 5,000 services. Rotation must be automatic, a leaked secret must be revocable within a minute, and the service must not become a single point of failure for the company.

<details><summary>Dominant constraint</summary>
The last clause. Like the <a href="../examples/distributed-coordination-service.md">coordination service</a>, this is the most-depended-upon system, so the design's first job is to make services resilient to its absence (cached secrets with a validity window; the service pushes rotations; nothing fetches per request).
</details>
<details><summary>The trap</summary>
Rotation that breaks running systems: a database credential rotated while a connection pool still uses the old one. Every secret needs an overlap window where both old and new are valid, and the consumer needs to know how to reload. Certificate expiry is the outage that actually happens.
</details>
<details><summary>Deep dive here</summary>
The rotation state machine (generate new, distribute new, activate new, deactivate old, with a per-secret overlap policy and verification that consumers have picked up the new one before the old is revoked); the trust bootstrap (how does a new service instance authenticate to get its first secret?); and the audit and revocation path.
</details>

## 7. Notification Fan-Out With Rate Limits and Preferences

> Design the notification system for a social product: 1B users, 10B notifications/day across push, email, and in-app. Users have per-channel preferences and quiet hours; the product has per-user rate limits; a celebrity post can trigger 50M notifications in a minute.

<details><summary>Dominant constraint</summary>
The fan-out spike. 50M in a minute is ~1M/s, 100x the average. The design must absorb it (queue and spread) without delaying the ordinary notifications behind it: priority lanes, and the celebrity case is *deliberately* slower.
</details>
<details><summary>The trap</summary>
Deduplication and idempotency across channel providers (push services and email providers are at-least-once *and* have their own retries); and preference evaluation at fan-out time versus send time (preferences change during the minute it takes to fan out).
</details>
<details><summary>Deep dive here</summary>
The two-stage pipeline (fan-out to per-user inboxes, then per-user delivery with preferences, rate limits, and quiet hours evaluated at send time); per-provider circuit breakers and quota management; and the "was this delivered?" tracking that product will ask for and that costs more than sending.
</details>

## 8. Inventory and Reservation Under Contention

> Design inventory management for flash sales: 10K units of an item, 5M users trying to buy in the first minute. No overselling. Fairness is desirable but not required. The rest of the catalog (100M items) must be unaffected.

<details><summary>Dominant constraint</summary>
A hard invariant (stock ≥ 0) on a single hot key. This cannot be multi-master, cannot be eventually consistent, and cannot be cached. The design is about *shielding* the single serialization point, not about eliminating it.
</details>
<details><summary>The trap</summary>
Trying to make the hot key scale horizontally. The correct move is to admit only ~10K-50K requests to the serialization point (a lottery or a queue at the edge) and reject the other 4.95M cheaply and honestly. Also: reservations that expire, and the reservation-to-purchase saga.
</details>
<details><summary>Deep dive here</summary>
Edge admission control (a token or queue that admits a bounded number to the checkout path); the reservation as a lease with expiry and a fencing token; the single-key serialization point's implementation (a single-partition counter with conditional decrement) and its failover; and isolating the hot item from the catalog's normal path.
</details>

## 9. Data Deletion Across a Company (Privacy Compliance)

> Design the system that deletes a user's data across 400 services, 50 data stores, backups, caches, logs, and analytics pipelines within 30 days of a request, with proof.

<details><summary>Dominant constraint</summary>
"With proof." This is an orchestration and verification problem: every store must implement a deletion contract, report completion, and be *audited* for completeness. The orchestrator is a durable saga across hundreds of participants with retries and a deadline.
</details>
<details><summary>The trap</summary>
Backups and derived data. You cannot delete from an immutable backup; you use crypto-shredding (per-user keys, delete the key) or accept a documented retention window. Derived data (aggregates, ML features) needs a policy for what "deleted" means.
</details>
<details><summary>Deep dive here</summary>
The participant contract (idempotent delete-by-user-ID with completion reporting); the discovery problem (how do you *know* every store that has this user's data? a data catalog with lineage, and scanning as verification); and the proof (a signed completion record per participant, aggregated, retained).
</details>

## 10. API Gateway and Edge Platform

> Design the edge platform fronting every public API: 2M RPS, authentication, rate limiting, routing to 1,000 backends, request transformation, and protection against abuse. The edge must add < 5 ms p99 and must never be the reason the product is down.

<details><summary>Dominant constraint</summary>
"Never the reason the product is down." The edge's own dependencies (auth service, rate-limit store, config, routing table) must all be cached and fail-open or fail-static. And the edge is the largest blast radius in the company, so it deploys slower than anything else.
</details>
<details><summary>The trap</summary>
Synchronous calls to the auth service per request. Tokens are validated locally by public key; revocation is a pushed denylist with a bounded staleness. And exact global rate limiting: approximate, local buckets with periodic sync, deliberately.
</details>
<details><summary>Deep dive here</summary>
The fail-open matrix (for each edge dependency, what happens when it is down: auth → validate locally; rate limit store → local buckets; config → last known; routing → last known); the deploy model (canary per PoP, slow waves, automatic rollback); and abuse detection that does not add latency (async scoring feeding a synchronous denylist).
</details>

## 11. ML Feature Store With Training/Serving Consistency

> Design the feature store for a recommendation system: 10K features, 1B entities, online serving at p99 < 10 ms for 500K lookups/sec, and a guarantee that a feature's value at serving time matches what the model saw at training time for the same point in history.

<details><summary>Dominant constraint</summary>
Point-in-time correctness: training must see the feature value *as of* each training example's timestamp, not the current value. That requires a time-versioned offline store and a pipeline that produces both online and offline views from the same computation, or training/serving skew makes the model wrong silently.
</details>
<details><summary>The trap</summary>
Computing features twice (once in a batch pipeline for training, once in a streaming pipeline for serving) with subtly different logic. The two implementations diverge within a quarter. One definition, one computation, two materializations.
</details>
<details><summary>Deep dive here</summary>
The online store's data model (entity → feature vector, versioned, with TTL); the streaming materialization path and its freshness SLI; the offline point-in-time join at training time and its cost; and backfilling a new feature's history without recomputing every other feature.
</details>

## 12. Incident Response Platform (You Design the Thing You Use at 3am)

> Design the on-call and incident management platform for the company: paging, escalation, incident channels, status pages, and post-mortems. It must work when everything else is down, including the identity provider, the chat tool, and the cloud region.

<details><summary>Dominant constraint</summary>
Independence. This system's dependency set must be disjoint from production's, more so even than the <a href="../examples/observability-pipeline.md">observability pipeline</a>: separate cloud account or provider, separate DNS, phone-based paging that does not depend on the internet, break-glass auth. The design is 70% "what we do not depend on."
</details>
<details><summary>The trap</summary>
Building it on the company's own platform because it is convenient. The first regional outage takes down the tool used to respond to it. Also: alert fatigue as a *design* problem (deduplication, grouping, and burn-rate alerts as the default template) rather than a discipline problem.
</details>
<details><summary>Deep dive here</summary>
The paging delivery path and its escalation state machine (durable, with acknowledgment and timeouts, surviving the platform's own failover); the break-glass access model; and the alert deduplication and grouping logic that turns 5,000 alerts into one incident.
</details>

---

## Writing Your Own Prompts

The best prompts for principal-level practice have three properties:
1. **A constraint that dominates** and that a careless read would miss
2. **A trap**: a plausible design that is wrong for a specific, articulable reason
3. **A sub-problem where mechanism matters**: where "we'd handle that" is not an answer

Take a system you have actually built. Write the prompt as your manager would have. Then write the three hidden sections. If you cannot fill in the trap, the prompt is not yet principal-level.
