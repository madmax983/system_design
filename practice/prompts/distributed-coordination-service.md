# Prompt Card: Distributed Coordination Service

**Worked example:** [examples/distributed-coordination-service.md](../../examples/distributed-coordination-service.md). Do not open it until step 4 of the [session protocol](../README.md#the-session-protocol).

## The Prompt

> Design the internal coordination service that every other system at the company uses for leader election, distributed locks, service discovery, and small-configuration storage. Think ZooKeeper, etcd, or Chubby. It must be correct under partitions, and it will be the single most depended-upon system in the company.

## What This Trains

Recognizing that a service cannot make its clients correct and designing the API so that client incorrectness is detectable (fencing); reasoning about leases, clocks, and partitions precisely; treating dependents as a risk to reduce; an operational posture as part of the design.

## Pacing (45 min)

| Minutes | Do |
|---------|-----|
| 0-7 | Framing. What can a lock service *not* guarantee? What is "the most depended-upon system" a risk of, and how do you reduce it? |
| 7-10 | Sessions, renewals (the dominant load), watches (the dominant fan-out), writes, shard count |
| 10-17 | Raft groups sharded by prefix; the data model with revisions; the SDK's responsibilities |
| 17-35 | Deep dives. Pick two of: fencing tokens; lease expiry and clocks; watches that do not lose events; guardrails |
| 35-42 | Failure modes including majority loss and the SDK release as a correlated failure; degradation order ending with renewals |
| 42-45 | The one-way doors (API semantics); operational posture |

## Before You Look: Questions to Answer in Your Attempt

- A client acquires a lock, pauses for 15 seconds, and resumes. What happens, and what makes it safe?
- Who decides a lease has expired, using which clock, and why can a partitioned old leader not expire anyone?
- A watcher falls behind the log's retention. What does it receive and what does the SDK do?
- What stops a team from using this as a database? What stops it from being on every request's hot path?
- Under overload, what is the last thing you stop doing, and why?

## Hints (reveal one at a time, only if stuck)

<details><summary>Hint 1: the reframe</summary>
The service cannot make clients correct. It can give every lock grant a monotonic revision (the fencing token) that the *protected resource* checks. Rename "lock" to "lease" in the API and explain the GC-pause scenario in the first paragraph of the docs.
</details>
<details><summary>Hint 2: leases</summary>
The leader expires client leases on its monotonic clock, and it can only commit an expiry while it holds a majority. Clients renew with margin and assume expiry conservatively. Neither compares absolute timestamps; a clock jump makes the leader step down.
</details>
<details><summary>Hint 3: watches</summary>
Watch-from-revision. Read at R, watch from R+1, no gap. Followers serve watches from their replicated log. Past the compaction boundary, the client gets an explicit error and the SDK does a full resync. Test that path; it will happen on every follower restart.
</details>
<details><summary>Hint 4: degradation</summary>
Reject scans, then new watches, then new sessions, then over-quota writes. Never stop processing renewals for existing sessions while quorum holds: losing renewals turns your incident into a company-wide leader-election storm.
</details>

## Interviewer Follow-Ups (answer out loud before revealing)

<details><summary>Q1: A client holds a lease and wants to write to object storage. How is that fenced?</summary>
Conditional put on a version or ETag the client read first, with the fencing token stored in object metadata and compared. If the store cannot do conditional writes, it cannot be safely fenced; make the operation idempotent instead.
</details>
<details><summary>Q2: Quorum is lost on a shard for ten minutes. What happened to the ephemeral keys?</summary>
Nothing; they are frozen. Clients' SDKs already assumed their own leases expired and stopped acting. On quorum return, the leader expires sessions whose clients did not reconnect. Discovery was stale for ten minutes, which is why consumers have their own health checks.
</details>
<details><summary>Q3: Migrate a prefix to a new shard with live watches on it.</summary>
Watchers receive a "moved" error with the new shard and the revision of the move; the SDK re-establishes there from that revision. The new shard's revisions start above the old shard's last revision for that prefix, preserving monotonicity for fencing.
</details>
<details><summary>Q4: Why should the SDK release slower than the service?</summary>
The service is 250 nodes you control; the SDK is 2,000 services you do not. A service rollback takes minutes; an SDK rollback takes weeks and every team's redeploy.
</details>
<details><summary>Q5: Why not one global Raft group?</summary>
Company-wide blast radius and a leader-throughput cap. Sharding by prefix bounds both.
</details>

## Scoring Focus

Weight dimensions 7 (deep dive depth: mechanisms, not descriptions), 8 (invariants), 11 (degradation order), and 13 (operations). A principal answer leads with fencing and ends with "reduce the number of dependents."
