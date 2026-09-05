# Principal Engineer Signals

What interviewers are actually listening for at Staff (IC6) and Principal (IC7) level, and how to demonstrate it in 45 minutes. This is the rubric behind the [worked examples](../examples/).

## The One-Sentence Version

> A senior engineer designs a system that works. A staff engineer designs a system that keeps working when things go wrong. A principal engineer designs a system that keeps working when things go wrong, can be built by the teams that exist, can be changed after it ships, and is the right system to build at all.

## What Changes at Each Level

### Senior (IC5): Correct and Complete

- Clarifies requirements, estimates scale, draws a coherent architecture
- Picks reasonable technologies and can justify them
- Handles the obvious failure modes (a server dies, a database fails over)
- Knows the patterns in this repository and applies them correctly

**Typical failure mode in interviews:** the design is fine, but every decision is presented as the only option. No trade-offs are named because none were considered.

### Staff (IC6): Trade-offs and Failure

- Names the non-goals explicitly and defends them
- Derives design constraints from the estimate ("at this write rate a single leader cannot keep up, so we shard on X")
- Reasons about partial failure: retries, idempotency, timeouts, backpressure
- Identifies the single points of failure *including the non-obvious ones* (the config service, the DNS resolver, the certificate expiry)
- Discusses SLOs and what gets sacrificed when they are under threat
- Can go deep on any component the interviewer picks

**Typical failure mode in interviews:** depth without prioritization. Every component is discussed at equal depth, so the 15 minutes that should have gone to the hardest problem are spread across the easy ones.

### Principal (IC7): Judgment, Evolution, and Leverage

Everything above, plus:

- **Reframes the problem.** Identifies the requirement that dominates the design, and the requirement in the prompt that is subtly wrong or unnecessary. ("You said strongly consistent. Reads of the balance need to be; the transaction history feed does not, and that difference is 80% of the design.")
- **Designs the evolution, not just the end state.** Knows what v1 looks like, why you cannot build v3 first, and how to migrate between them under live traffic.
- **Reasons from invariants.** States what must always be true, and points to the exact place in the design where each invariant is enforced. Knows where an invariant is deliberately relaxed and what the compensating mechanism is.
- **Treats operations as part of the design.** Deploy safety, rollback, on-call load, capacity planning cadence, and runbooks are design outputs, not afterthoughts.
- **Models cost.** Knows the dominant cost driver, the knob that controls it, and the point at which the architecture must change rather than scale.
- **Matches system boundaries to team boundaries.** Knows that a service boundary is also an ownership boundary, an on-call boundary, and a deploy boundary.
- **Knows what they do not know.** Says "I would want to measure X before committing to this" and names the experiment.

**Typical failure mode in interviews:** talking about organizational and strategic concerns *instead of* the technical design rather than *in addition to* it. The technical bar does not lower; it rises.

## The Principal Questions

Ask yourself these about every design. In an interview, ask the interviewer the ones that matter.

### About the requirements
- Which single requirement, if removed, would make this design 10x simpler? Is that requirement real?
- Which requirement is the design most sensitive to? What if it is wrong by 10x in either direction?
- What is the *unit of consistency*? (An account? A user? An order? A tenant?) Everything outside that unit can be eventually consistent.
- What is the *unit of failure* the business can tolerate? (One user? One tenant? One region?)

### About the data
- What is the source of truth for each piece of data, and is there exactly one?
- What is the write path's idempotency key, and who generates it?
- What happens to in-flight data during a deploy? During a failover? During a schema change?
- How do you know the data is correct? (Not "how do you keep it correct" — how do you *detect* when it is not?)

### About failure
- What is the blast radius of the worst single failure? Can it be made smaller?
- What is the *correlated* failure? (Shared dependency, shared config, shared deploy, shared certificate, shared AZ.)
- In what order does the system degrade? Who decides? Is it automatic?
- What is the dependency that everyone forgot is a dependency? (DNS, NTP, the secrets manager, the feature flag service, the container registry.)
- If this system is down for an hour, what is lost? For a day?

### About change
- How is this deployed? How is it rolled back? How long does rollback take?
- What is the first schema change this design will need, and does the design survive it?
- What is the migration path from what exists today?
- Which decision here is a one-way door?

### About operations and cost
- What is the dominant cost? What knob controls it?
- What does on-call look like? What pages at 3am, and is each page actionable?
- How is capacity planned? What is the lead time to add capacity?
- What is the first thing that breaks at 10x? What is the second?

## Phrasings That Signal the Level

These are not scripts. They are the *shape* of sentences that principal engineers say naturally because they reflect how they think.

| Instead of | Say |
|-----------|-----|
| "We'll use Kafka." | "We need a durable, replayable log because consumers will need to reprocess after a bug. Kafka is the default; the alternative is a cloud-managed log, which I'd pick if the team is under five people." |
| "We'll add a cache." | "The read path is 100:1 over writes and the working set is ~50 GB, so a cache turns this from a sharding problem into a single-primary problem. The cost is a staleness window; let me define what that window is allowed to be." |
| "We'll make it idempotent." | "The idempotency key is the client-generated request ID, stored with the result in the same transaction as the side effect, with a 24-hour retention. Retries after that window are a business decision, not an engineering one." |
| "We'll use microservices." | "There are two teams. I'd draw two service boundaries, here and here, because those are the places where the deploy cadence and the on-call rotation differ." |
| "It should be highly available." | "The SLO is 99.95% on the write path and 99.99% on reads. The read path can serve stale data from replicas during a primary failover; that is why the two numbers differ." |
| "We'll shard the database." | "Sharding is a one-way door. Before that, I'd exhaust vertical scaling, read replicas, and moving the hot table out; that buys roughly 18 months. When we do shard, the key is tenant ID because every query is tenant-scoped and it keeps a tenant's blast radius to one shard." |
| "We'll monitor it." | "The SLI is the fraction of requests under 200ms at the load balancer. Alerting is on burn rate against the error budget, not on raw thresholds, so that a slow burn pages during business hours and a fast burn pages immediately." |

## Design Review Rubric

Principal engineers spend as much time reviewing designs as producing them. Use this rubric on your own designs before the interview, and on the [worked examples](../examples/).

| Criterion | Question | Red Flag |
|-----------|----------|----------|
| **Problem fit** | Does the design solve the stated problem, and only that problem? | Components with no requirement that justifies them |
| **Dominant constraint** | Is it named? Does the design visibly bend around it? | Every component gets equal attention |
| **Invariants** | Are they stated? Is enforcement located? | "The service ensures consistency" with no mechanism |
| **Idempotency** | Is every side effect safe to retry? Who holds the key? | Retries mentioned, deduplication not |
| **Failure modes** | Are correlated failures considered? Is degradation ordered? | Only "add a replica" for every failure |
| **Blast radius** | What is the worst single failure's scope? | A shared component with global scope and no cell/partition boundary |
| **Evolution** | Is there a v1? Can v1 become v2 without downtime? | Only the end state is designed |
| **Operability** | Deploy, rollback, on-call, runbooks | "We'll add monitoring" |
| **Cost** | Dominant driver known? Knob known? | Not mentioned, or mentioned without a number |
| **Alternatives** | Were they considered? Is the losing reason specific? | "Best practice" as a justification |

## Pacing a 45-Minute Principal Interview

The [interview framework](interview-framework.md) is the baseline. At principal level, shift the time:

| Phase | Senior Pacing | Principal Pacing | Why |
|-------|--------------|-----------------|-----|
| Requirements & framing | 5 min | 7 min | The reframing happens here; it pays for itself later |
| Estimation | 5 min | 3 min | Do less arithmetic, extract more constraints |
| High-level design | 10 min | 7 min | The interviewer has seen this diagram before |
| Deep dives | 15 min | 18 min | This is where the level is decided; pick the hardest problem, not the most familiar |
| Failure, operations, evolution | 5 min | 8 min | Correlated failure and migration paths are principal signals |
| Trade-offs & wrap-up | 5 min | 2 min | You have been stating trade-offs the whole time |

**The most common principal-level mistake:** spending deep-dive time on a component you know well instead of the one the design is most sensitive to. Interviewers notice.
