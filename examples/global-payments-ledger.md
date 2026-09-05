# Global Payments Ledger

> **Prompt:** Design the ledger and money-movement service for a payments company. It records every transfer between accounts, must never lose or double-count money, and serves balance reads for 200M accounts across three continents.

This is the canonical "exactly-once" problem. The trap is treating it as a distributed-transactions problem. It is an idempotency and reconciliation problem with a database in the middle.

## 1. Problem Framing

**What the prompt gets right:** money must never be lost or duplicated. That is the invariant everything else bends around.

**What the prompt gets wrong (or leaves ambiguous):**
- "Never lose money" is usually heard as "strong consistency everywhere." In practice only the *balance check and debit* need to be serializable. The transaction history feed, the notification, the analytics event, and the fraud score can all be eventually consistent. Separating those is most of the design.
- "Across three continents" invites a multi-region active-active ledger. That is the wrong default. An account lives in exactly one region (its *home region*). Cross-region transfers are the rare case and are modeled as two single-region operations with a settlement step between them.
- "Serves balance reads" hides the real read pattern: the *authorization* read ("can this account spend $50 right now?") is on the write path and must be exact; the *display* read ("what's my balance?") tolerates a second of lag.

**The reframe to state out loud:** "I'm going to design this as a single-writer-per-account system with a home region, an append-only double-entry ledger as the source of truth, idempotent commands, and reconciliation as a first-class component rather than a batch job. I'll treat cross-region transfers as the exception, not the default."

## 2. Requirements & SLOs

### Functional
- `Transfer(from, to, amount, idempotency_key)` moves money atomically between two accounts
- `Authorize(account, amount)` holds funds; `Capture` / `Release` finalizes or cancels the hold
- `GetBalance(account)` returns available and pending balances
- `GetHistory(account, cursor)` returns a paginated ledger of entries
- Every entry is immutable and auditable: who, what, when, why, and the causal link to its counterpart entry

### Non-Functional
| Requirement | Target | Why this number |
|-------------|--------|-----------------|
| Durability | Zero committed-transfer loss, ever | Regulatory and existential |
| Write availability | 99.99% per region | ~52 min/year; each minute of ledger downtime is measured in lost revenue and regulator attention |
| Read availability (display) | 99.999% | Display reads can fall back to replicas and caches; there is no excuse for them to fail |
| Transfer latency | p99 < 500 ms within region | Card networks time out around 1-2 s; leave headroom for fraud checks |
| Authorization latency | p99 < 100 ms | On the critical path of every purchase |
| Consistency | Serializable per account; causal across accounts | The invariant is per-account; the cross-account guarantee is "no money appears or disappears" |
| Auditability | Any balance reconstructable from the ledger alone | The ledger is the truth; the balance is a cache of it |

### Non-Goals (say these)
- Not a general-purpose distributed transaction system
- Not designing fraud detection; it is a synchronous dependency with a timeout and a fail-closed policy
- Not designing the card-network integration; it is a client of this service
- Not doing multi-currency FX in v1; every account has one currency and cross-currency transfers are a separate service

**The dominant requirement:** *no lost or duplicated money*. It is why the ledger is append-only, why every command is idempotent, and why reconciliation exists.

## 3. Estimation

| Quantity | Estimate | Derivation |
|----------|----------|------------|
| Accounts | 200M | Given |
| Transfers/day | 500M | ~2.5 per account per day, weighted toward a minority of active accounts |
| Average transfer rate | ~6K/s | 500M / 86,400 |
| Peak transfer rate | ~30K/s | 5x average (Black Friday, salary days) |
| Ledger entries/day | 1B | Double-entry: 2 entries per transfer |
| Entry size | ~200 bytes | IDs, amount, currency, timestamps, metadata reference |
| Ledger growth | ~200 GB/day, ~73 TB/year | 1B × 200 B |
| Authorization reads | ~100K/s peak | Every purchase, plus retries |
| Display balance reads | ~50K/s | Lower than authorization; people check balances less than they spend |

### Constraints the numbers force

1. **30K transfers/s peak is within a single-primary relational database's write budget only if the transaction is tiny.** A transfer must be: two ledger inserts, two balance updates, one idempotency record. Five row writes, one transaction, no external calls inside it. Anything else (notifications, fraud, analytics) is outside the transaction.
2. **73 TB/year of immutable ledger cannot live in the hot database.** The hot store holds the current balance and recent entries; older entries tier to cold storage. The ledger is the truth, but "the ledger" is a logical concept spanning hot and cold tiers.
3. **200M accounts across three home regions is ~70M per region.** That fits a sharded relational cluster per region: ~16-32 shards, each a few million accounts. Shard key is account ID. A transfer touches two accounts, and therefore potentially two shards. That is the hardest problem in the design (see Deep Dive 2).
4. **100K/s authorization reads must not hit the primary.** But they must be exact. This forces the *balance* to be maintained as a materialized value in the same transaction as the ledger write, and served from a synchronously-replicated read path or from the primary with a very small working set.

## 4. Architecture

```
                         ┌────────────────────────────────────────────────┐
                         │                 Home Region (one of 3)          │
                         │                                                │
  Client ──▶ API GW ──▶  │  Ledger API ──▶ Command Router ──▶ Shard 0..N   │
  (idempotency_key)      │       │              │              (Postgres,   │
                         │       │              │               sync replica│
                         │       │              │               in 2nd AZ)  │
                         │       │              ▼                    │      │
                         │       │      Cross-Shard Coordinator       │      │
                         │       │      (saga log, own shard)         │      │
                         │       │                                    ▼      │
                         │       │                              Outbox table │
                         │       │                                    │      │
                         │       ▼                                    ▼      │
                         │  Balance Read Path            CDC ──▶ Event Log   │
                         │  (sync replica / cache)              (Kafka)      │
                         │                                         │         │
                         └─────────────────────────────────────────┼─────────┘
                                                                   │
                     ┌───────────────┬───────────────┬─────────────┴────────┐
                     ▼               ▼               ▼                      ▼
              History Service   Notifications   Analytics / Cold      Reconciler
              (read model)                      Ledger Tier (S3)      (continuous)
```

### Critical write path: intra-shard transfer

```
1. Client sends Transfer with idempotency_key (client-generated UUID, persisted client-side before send)
2. API validates schema, authenticates, resolves both accounts' home region and shard
3. If both accounts are on the same shard → single database transaction:
     a. INSERT idempotency(key, status=IN_PROGRESS)     -- unique constraint; conflict means "already seen"
     b. SELECT balance FROM accounts WHERE id=from FOR UPDATE
     c. Check available >= amount, else abort with INSUFFICIENT_FUNDS
     d. INSERT ledger_entry (from, -amount, transfer_id, seq)
     e. INSERT ledger_entry (to,   +amount, transfer_id, seq)
     f. UPDATE accounts SET balance = balance - amount WHERE id=from
     g. UPDATE accounts SET balance = balance + amount WHERE id=to
     h. INSERT outbox (event=TransferCompleted, payload)
     i. UPDATE idempotency SET status=DONE, response=...
   COMMIT
4. Return response. Every retry with the same key returns the stored response.
5. CDC tails the outbox → Kafka → downstream consumers.
```

The order matters: the idempotency insert is first so a concurrent retry blocks on the unique constraint rather than racing the balance check.

## 5. Deep Dives

### Deep Dive 1: Idempotency that survives everything

The idempotency key is the single most important piece of data in the system. The design has to answer four questions:

**Who generates it?** The client, before sending, persisted durably on the client side. A server-generated key is useless because the failure case is "the client doesn't know if the server received the request."

**What is its scope?** `(account_id, idempotency_key)`. Scoping to the account means the key lives on the same shard as the balance, so the idempotency check and the debit are in one transaction. A global key store would require a cross-shard lookup on every write and would become the availability bottleneck.

**What is stored with it?** The full response, including the transfer ID and final status. A retry after success must return the *same* transfer ID, not create a new one.

**How long does it live?** 24 hours in the hot store, then archived. A client retrying after 24 hours gets `UNKNOWN_KEY` and must reconcile through the history API. This is a business decision stated as an engineering constraint: "if you have not resolved a transfer within a day, you have a bigger problem than idempotency."

**The failure cases that make this hard:**

| Failure | What happens | Why it is safe |
|---------|-------------|----------------|
| Client times out, server committed | Retry hits unique constraint on key, returns stored response | Key inserted in same transaction as debit |
| Server crashes after commit, before responding | Same as above | Same |
| Server crashes mid-transaction | Transaction rolls back including the key row; retry proceeds normally | Atomicity of the local transaction |
| Two concurrent requests, same key | Second blocks on the row lock from the first's key insert, then sees the committed key | Unique constraint + row lock |
| Client reuses a key with a *different* payload | Reject with `IDEMPOTENCY_KEY_REUSED`; store a hash of the request with the key | Detects client bugs instead of silently honoring the first payload |
| Region failover during the request | Client retries against the new primary; the key is either there (synchronously replicated) or not (transaction never committed) | Synchronous replication to a second AZ; asynchronous cross-region is handled in Deep Dive 3 |

### Deep Dive 2: Cross-shard transfers without 2PC

Roughly 20% of transfers cross shards (two accounts, two shards). Two-phase commit would work and is the wrong answer: it holds locks across a network round trip, the coordinator is a single point of failure during the window, and at 6K/s it is a latency and availability tax on every cross-shard transfer.

**The approach: a two-entry saga with a durable coordinator log, using the ledger's own double-entry structure as the saga state.**

```
Cross-shard transfer A (shard 1) → B (shard 2), amount X:

Step 1 (shard 1, one transaction):
   idempotency insert
   debit A by X
   insert ledger_entry(A, -X, transfer_id, state=PENDING_CREDIT)
   insert outbox(CreditRequested, transfer_id, B, X)
   commit

Step 2 (shard 2, one transaction, driven by the outbox consumer):
   idempotency insert on (B, transfer_id)      -- makes the credit idempotent
   credit B by X
   insert ledger_entry(B, +X, transfer_id, state=COMPLETE)
   insert outbox(CreditCompleted, transfer_id)
   commit

Step 3 (shard 1, driven by CreditCompleted):
   update ledger_entry state PENDING_CREDIT → COMPLETE
   insert outbox(TransferCompleted)
   commit
```

**Why this is correct:**
- Money is debited from A before it is credited to B, so the system-wide sum of balances is *never higher* than it should be. It can be *lower* for the duration of the saga; that money is in the `PENDING_CREDIT` entry, which is a ledger account in its own right ("in-transit"). Sum of all balances + sum of in-transit = constant. That is the invariant, and it is checkable.
- Step 2 is idempotent on `(B, transfer_id)`, so the outbox consumer can deliver at-least-once.
- If step 2 fails permanently (B's account was closed), a compensating step credits A back and marks the transfer `REVERSED`. The debit entry is never deleted; the reversal is a new entry. The ledger is append-only.

**What the client sees:** the API returns `PENDING` for cross-shard transfers and `COMPLETE` for intra-shard ones. Product teams push back on this. Hold the line: pretending a cross-shard transfer is synchronous means either 2PC or lying. The p99 for the saga to complete is well under a second; a client that needs synchronous semantics polls or subscribes.

**The reconciler's role:** any `PENDING_CREDIT` entry older than a threshold (say, 60 seconds) is an alert. Older than 10 minutes is a page. The reconciler is not fixing anything; it is detecting a stuck saga so that a human or an automated retry can drive it forward. This is the difference between "eventually consistent" and "eventually consistent with a deadline."

### Deep Dive 3: Region failover without losing committed transfers

Each region is primary for its home accounts. Within a region, Postgres uses synchronous replication to a replica in a second availability zone; a commit is acknowledged only when the replica has the WAL. That gives zero data loss for AZ failure at the cost of ~1-2 ms of write latency.

Cross-region replication is asynchronous, with a lag of typically 100-500 ms. This means a *region* failure can lose the last few hundred milliseconds of commits. That is unacceptable for a ledger. Three options:

| Option | Data loss on region failure | Write latency | Verdict |
|--------|---------------------------|---------------|---------|
| Async cross-region replication, promote on failure | Up to replication lag (~500 ms of transfers, ~3K transfers) | Unchanged | Rejected: loses committed money |
| Sync cross-region replication (quorum of 2 of 3 regions) | Zero | +80-150 ms per write | Rejected for the general case: doubles p99 |
| Async replication + **do not auto-failover**; treat region failure as an availability event, not a durability event | Zero (the region is down, not gone; commits are in its storage) | Unchanged | **Chosen**, with the escape hatch below |

**The chosen policy:** a region outage means that region's accounts are unavailable for writes until the region returns or until a human authorizes promotion with acknowledged data loss. Reads continue from the async replica with a "balance as of T" marker. This is the honest trade: 99.99% write availability with zero loss, versus 99.999% with a small probability of loss. For a ledger, durability wins.

**The escape hatch:** for accounts that opt into it (large merchants that would rather have a rare small reconciliation event than an outage), synchronous cross-region replication is available per shard. Those shards pay the latency. This is the "unit of consistency is per account" principle applied to the availability/durability trade.

**Rehearsed failover runbook** (state that one exists and what is in it):
1. Confirm the region is unreachable from two independent vantage points, not just the health checker
2. Freeze writes for the region's shards at the API layer (fail with `REGION_UNAVAILABLE`, retryable)
3. If the outage exceeds the RTO (say 30 min), a named on-call lead may promote the async replica; the replication lag at promotion time is recorded, and every transfer committed in the lost window is enumerated from the failed region's storage when it returns and replayed through the reconciler
4. When the original region returns, it becomes a replica; it never resumes as primary without a full ledger diff

## 6. Invariants

State them and point to where they are enforced.

| Invariant | Where enforced | How verified |
|-----------|---------------|--------------|
| Sum of all balances + in-transit = sum of all deposits - sum of all withdrawals | Double-entry: every transfer inserts exactly two entries summing to zero | Reconciler recomputes per shard hourly, globally daily |
| An account's balance = sum of its ledger entries | Balance update and ledger insert in one transaction | Reconciler recomputes for a random 1% of accounts continuously; any drift is a P0 |
| No transfer is applied twice | `(account_id, idempotency_key)` unique constraint, inserted first in the transaction | Duplicate-detection metric on unique violations should be nonzero (retries happen) but responses must be identical |
| No balance goes below zero (for non-overdraft accounts) | `SELECT ... FOR UPDATE` then check, in the transaction | Database `CHECK (balance >= 0)` constraint as the last line of defense |
| Ledger entries are never updated or deleted, only appended | Database role permissions: the application role has INSERT but not UPDATE/DELETE on `ledger_entry` (state transitions on the saga column are the one exception, on a separate, narrow column) | Audit log on DDL and role changes |
| Every outbox event is delivered at least once | Outbox in the same transaction; CDC tails the committed WAL | Consumer lag and outbox-age alerts |

**Where an invariant is deliberately relaxed:** the system-wide balance sum is *temporarily* off by the in-transit amount during cross-shard sagas. The compensating mechanism is the in-transit account and the reconciler's staleness alert.

## 7. Failure Modes

| Failure | Blast radius | Behavior | Mitigation |
|---------|-------------|----------|------------|
| One shard primary fails | 1/32 of a region's accounts | Sync replica promotes in ~10-30 s; writes fail retryably during the window | Client retries with same idempotency key; no loss |
| One AZ fails | Half the sync replicas | Primaries in the other AZ continue; the failed replicas' primaries now run without sync replication until a new replica catches up | Alert; temporarily degraded durability is visible on a dashboard, and an operator can choose to block writes on those shards |
| Region fails | 1/3 of accounts unavailable for writes | See Deep Dive 3 | Runbook, human-authorized promotion |
| Kafka / CDC fails | Downstream consumers (history, notifications, analytics) go stale | Core transfers unaffected; outbox rows accumulate | Outbox depth alert; the outbox table is the buffer, sized for hours of backlog |
| Fraud service slow or down | Every transfer (it is on the synchronous path) | Timeout at 200 ms, then **fail closed** for transfers over a threshold and **fail open** under it | Business-defined thresholds; this is the degradation order, decided in advance |
| Cross-shard saga stalls | The individual transfer | Money sits in in-transit | Reconciler alerts at 60 s, pages at 10 min; retry is safe because step 2 is idempotent |
| Idempotency store fills (retention job fails) | Writes slow as the index grows | Gradual | Age-based alert on the oldest key |
| Clock skew | Ledger `seq` ordering within a shard | None: `seq` is a database sequence, not a wall clock | Wall-clock timestamps are informational only; never used for ordering or expiry logic |
| Correlated: bad deploy of the ledger service to all regions | Everything | Full outage | Regional canary with automated rollback on error-rate SLO burn; regions deploy at least 30 minutes apart |
| Correlated: certificate expiry on the database connection | Everything, simultaneously | Full outage | Certificate expiry is a monitored SLI with a 30-day warning; this is the outage that actually happens to real companies |

**Degradation order (decided in advance, in writing):**
1. Shed analytics and notification consumers (they are async; the outbox buffers)
2. Serve display balances from replicas with a staleness marker
3. Raise fraud-check timeout thresholds (fail open for small amounts)
4. Reject new cross-shard transfers while continuing intra-shard (cross-shard needs the outbox pipeline healthy)
5. Freeze all writes for a shard, preserving reads
6. Freeze the region

## 8. Evolution Path

**v1 (month 0-6): one region, one Postgres primary, no sharding.**
Everything is intra-shard by definition. The idempotency table, double-entry ledger, outbox, and reconciler exist from day one because they are cheap to build now and impossible to retrofit later. This v1 handles ~3-5K transfers/s, which is a real business for a long time.

**v2 (month 6-18): shard within the region.**
Introduce the command router and the cross-shard saga. Migrate accounts to shards using the [zero-downtime migration playbook](zero-downtime-database-migration.md). The shard key was account ID from day one, so the routing layer is the only new component. Cross-shard transfers go from 0% to ~20%; the saga path gets real traffic for the first time. The `PENDING` status appears in the API; announce it a quarter early.

**v3 (month 18+): multi-region with home regions.**
Add regions. Every account gets a home region at creation. Existing accounts stay in the original region until a migration tool moves them (rare; driven by data residency or latency). Cross-region transfers reuse the cross-shard saga: the mechanism is identical, only the latency differs.

**The one-way doors:**
- The shard key. Account ID is chosen in v1 and never changes.
- The ledger schema's immutability. Once auditors have seen "entries are never updated," you cannot walk it back.
- The `PENDING` API status. Once clients depend on synchronous cross-shard transfers, you cannot make them async.

## 9. Cost Model

| Component | Dominant driver | Rough share | The knob |
|-----------|----------------|-------------|----------|
| Hot database (3 regions × 32 shards × primary + sync replica) | Instance count and IOPS | ~55% | Shard count; vertical scaling delays sharding but the cost curve is steeper |
| Cold ledger tier (object storage, ~73 TB/year growing) | Storage volume | ~5% | Compression (ledger entries compress ~5x) and tiering age |
| Kafka + CDC | Throughput and retention | ~10% | Retention window; 7 days is plenty because the outbox is the durable buffer |
| Read replicas and cache | Read QPS | ~15% | Cache TTL for display balances (1 s vs 10 s is a 10x difference in replica load) |
| Compute (API, coordinator, reconciler) | Request volume | ~15% | Mostly linear; the reconciler's sampling rate is the one tunable |

**The insight:** cost is dominated by the hot database, and the hot database is sized by *peak* write throughput with sync replication. The single biggest lever is keeping the transaction tiny (five row writes) so that each shard's write budget is spent on transfers, not on side effects. Every column added to the hot path costs real money.

## 10. What Breaks at 10x

At 300K transfers/s peak, 5B ledger entries/day:

**First to break: the cross-shard saga pipeline.** At 10x, cross-shard transfers are ~60K/s. The outbox consumers and Kafka can handle it; the problem is that each saga is three transactions across two shards, and the coordination overhead is now a large fraction of total database load. Fix: co-locate accounts that frequently transact (a merchant and its customers) on the same shard via a *placement* service that observes transfer graphs. This turns the 20% cross-shard rate into ~5%.

**Second to break: the reconciler.** Recomputing balances from the ledger for 1% of 2B accounts continuously is 20M full ledger scans per cycle. Fix: incremental reconciliation from CDC (maintain a shadow balance from the event stream and compare), with full recompute only for accounts that show drift.

**Third: the hot database's write amplification.** Five row writes plus indexes plus WAL plus sync replication is ~20 physical writes per transfer. At 300K/s that is 6M IOPS across the fleet. Fix: this is where you move the balance and idempotency tables to a purpose-built store (or partition the ledger by time so that indexes stay small), which is a v4 migration and another instance of the migration playbook.

## 11. Rejected Alternatives

| Alternative | Why it loses |
|-------------|-------------|
| **Two-phase commit for cross-shard transfers** | Holds locks across a network round trip; coordinator failure blocks participants; at scale it caps throughput and availability. The saga gets the same correctness with an explicit in-transit state that is auditable. |
| **Globally replicated ledger (Spanner-style, sync across regions)** | Zero-loss failover is attractive. The cost is 80-150 ms on every write and a much higher price per transaction. Offered as a per-shard opt-in instead. |
| **Event-sourced ledger with balances computed on read** | Elegant on paper. Authorization reads at 100K/s cannot afford to fold a ledger. Balances must be materialized, in the same transaction as the entry. |
| **Single global idempotency store (Redis/Dynamo)** | Makes idempotency a cross-shard dependency on every write and an availability SPOF. Co-locating the key with the account's shard keeps the check inside the local transaction. |
| **Server-generated idempotency keys** | Cannot solve the "client doesn't know if it was received" case, which is the only case that matters. |
| **Deleting or updating ledger entries for corrections** | Destroys auditability. Corrections are new entries. |
| **Auto-failover across regions** | Trades a rare small data loss for availability. For a ledger, that is the wrong trade; made explicit and human-gated instead. |

## 12. Level Signals

**A senior answer** designs a relational database with a transfers table, adds an idempotency key column, uses a distributed transaction or "the database handles it" for cross-account transfers, and adds read replicas for balance reads. Correct enough to ship a v1.

**A staff answer** introduces double-entry, keeps the transaction tiny, uses the outbox pattern for side effects, and handles cross-shard transfers with a saga. Discusses idempotency failure cases specifically. Names the retention window as a decision.

**A principal answer** does all of that, and additionally:
- Reframes "strongly consistent" into "serializable per account, causal across accounts" and shows that this distinction is what makes the design tractable
- Treats the reconciler as a component of the design rather than a batch job, and uses it to give "eventual consistency" a deadline
- Makes the region-failover durability trade explicit, chooses zero-loss over availability, and offers a per-shard opt-in rather than a global policy
- Lists the one-way doors and identifies which decisions must be made in v1 because they cannot be retrofitted
- Says which invariant is deliberately relaxed and what the compensating mechanism is
- Knows that the certificate-expiry outage is the one that actually happens

**Interviewer follow-ups to expect:**
- "What if the same client sends two different transfers with the same idempotency key?" (Reject; store a request hash.)
- "Walk me through what the user sees during a region outage." (Reads with a staleness marker; writes fail retryably; the client's persisted idempotency key means it can retry hours later.)
- "How would you add multi-currency?" (Every ledger entry already has a currency; a cross-currency transfer is two same-currency transfers plus an FX entry against a house account. The double-entry model absorbs it.)
- "The reconciler finds a drift. Now what?" (Page. Freeze the account. Reconstruct from the ledger. The ledger is the truth; the balance is corrected to match it, never the other way around. Root-cause the drift because a single drift means the invariant enforcement has a hole.)
