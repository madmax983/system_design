# Prompt Card: Global Payments Ledger

**Worked example:** [examples/global-payments-ledger.md](../../examples/global-payments-ledger.md). Do not open it until step 4 of the [session protocol](../README.md#the-session-protocol).

## The Prompt

> Design the ledger and money-movement service for a payments company. It records every transfer between accounts, must never lose or double-count money, and serves balance reads for 200M accounts across three continents.

## What This Trains

Idempotency reasoning under every failure mode; separating the invariant that needs serializability from everything that does not; designing reconciliation as a component; making the durability-versus-availability trade explicit and per-scope.

## Pacing (45 min)

| Minutes | Do |
|---------|-----|
| 0-7 | Framing. Which requirement dominates? What in the prompt is ambiguous or wrong? What is the unit of consistency? |
| 7-10 | Estimate transfers/s, ledger growth, authorization read rate. Which number forces what? |
| 10-17 | Architecture and the critical write path, step by step, including where the idempotency key is checked |
| 17-35 | Deep dives. Pick two of: idempotency end-to-end; cross-shard transfers; region failover and data loss |
| 35-42 | Failure modes with blast radius; degradation order; the correlated failure |
| 42-45 | Evolution from a single-region v1; one-way doors; rejected alternatives |

## Before You Look: Questions to Answer in Your Attempt

- What exactly must be strongly consistent, and what may not be?
- Who generates the idempotency key, what is its scope, what is stored with it, and how long does it live?
- A transfer touches two accounts on two shards. How, without 2PC, and what does the client see?
- A region goes down. Do you promote the replica? What is lost if you do?
- How do you *detect* that a balance is wrong, as opposed to keeping it right?

## Hints (reveal one at a time, only if stuck)

<details><summary>Hint 1: the reframe</summary>
Only the balance check and debit need serializability. History, notifications, fraud scoring, and analytics can be eventually consistent. Say this in the first five minutes and the design becomes tractable.
</details>
<details><summary>Hint 2: the data model</summary>
Append-only double-entry ledger as the source of truth; balance as a materialized value updated in the same transaction. Five row writes per transfer, nothing external inside the transaction.
</details>
<details><summary>Hint 3: cross-shard</summary>
A saga using the ledger's own structure: debit with a PENDING_CREDIT entry, credit idempotently on the other shard, complete. Money in transit is an account. The system-wide sum is off by exactly the in-transit amount, and that is checkable.
</details>
<details><summary>Hint 4: region failover</summary>
Async cross-region replication plus auto-promotion loses committed money. The honest choice is: no auto-failover; a region outage is an availability event, human-gated promotion with acknowledged loss, and per-shard sync replication as an opt-in for accounts that prefer latency cost over outage risk.
</details>

## Interviewer Follow-Ups (answer out loud before revealing)

<details><summary>Q1: A client sends two different transfers with the same idempotency key. What happens?</summary>
Reject with a specific error. Store a hash of the request body with the key so reuse with a different payload is detected. Silently honoring the first payload hides a client bug.
</details>
<details><summary>Q2: Walk me through what a user sees during a region outage.</summary>
Reads served from the async replica with a "balance as of T" marker. Writes fail with a retryable error. The client's persisted idempotency key means it can safely retry hours later. Cross-region transfers *into* the failed region's accounts sit in the in-transit state and the reconciler alerts on their age.
</details>
<details><summary>Q3: How would you add multi-currency?</summary>
Every ledger entry already carries a currency. A cross-currency transfer is two same-currency transfers plus an FX pair of entries against a house account. Double-entry absorbs it; the ledger schema does not change.
</details>
<details><summary>Q4: The reconciler finds a balance that does not match the ledger. Now what?</summary>
Page. Freeze the account. The ledger is the truth; the balance is corrected to match it, never the reverse. Then root-cause, because a single drift means invariant enforcement has a hole, and holes do not stay single.
</details>
<details><summary>Q5: Why is the idempotency insert first in the transaction?</summary>
So that a concurrent retry blocks on the unique constraint's row lock and then sees the committed result, rather than racing the balance check and potentially both passing it.
</details>

## Scoring Focus

Use the full [scorecard](../scorecard.md). For this prompt, weight dimensions 5 (critical path), 8 (invariants), 9 (relaxed invariants), and 11 (degradation order) most heavily. A principal answer names the in-transit relaxation and the fraud-check fail-open/fail-closed threshold unprompted.
