# Prompt Card: Cell-Based Multi-Tenant Platform

**Worked example:** [examples/cell-based-multi-tenant-platform.md](../../examples/cell-based-multi-tenant-platform.md). Do not open it until step 4 of the [session protocol](../README.md#the-session-protocol).

## The Prompt

> Design the serving infrastructure for a B2B SaaS product with 50,000 tenants, from 5-seat startups to enterprises with 100,000 users. A single tenant's traffic spike, bad data, or abusive workload must not affect other tenants. Enterprise tenants demand isolation guarantees, predictable performance, and the ability to be moved between regions.

## What This Trains

Decomposing "isolation" into its four kinds with separate mechanisms and prices; deriving a structural parameter (cell size) from an SLO (blast radius); keeping a control plane off the hot path; making a risky operation (tenant migration) routine; turning an engineering trade-off into a business decision with a price tag.

## Pacing (45 min)

| Minutes | Do |
|---------|-----|
| 0-7 | Framing. What kinds of isolation are being asked for? What is the requirement that sets the architecture's shape? |
| 7-10 | Tenants, users, RPS, cell count from the blast-radius target, control-plane lookup rate, deploy duration |
| 10-17 | Cells as the whole stack; the request path with cell-side ownership validation; the tiers |
| 17-35 | Deep dives. Pick two of: shuffle sharding; data isolation below the application; tenant migration; the control plane off the hot path |
| 35-42 | Failure modes including the shared-dependency register; why deploys are deliberately slow; degradation order |
| 42-45 | Evolution from one cell; cost versus blast radius as a business decision |

## Before You Look: Questions to Answer in Your Attempt

- Name the kinds of isolation. Which mechanism serves each?
- Why are quotas necessary but insufficient? What handles the abuse you did not anticipate?
- Where is data isolation enforced, and how do you *know* it holds?
- During a tenant migration, what stops both cells from accepting writes for the same tenant?
- What must keep working when the control plane is down, for how long?

## Hints (reveal one at a time, only if stuck)

<details><summary>Hint 1: the reframe</summary>
Performance isolation, failure isolation, data isolation, operational isolation. Cells give failure and operational isolation structurally; quotas plus shuffle sharding give performance isolation; row-level security plus per-tenant keys give data isolation. Each has a price.
</details>
<details><summary>Hint 2: the derivation</summary>
Blast-radius SLO (≤ 2% of tenants per cell failure) → 50 shared cells → 1,000 tenants each. If the SLO changes, the cell size changes. Fixed overhead per cell is what stops you going smaller.
</details>
<details><summary>Hint 3: shuffle sharding</summary>
Each tenant maps to a primary cell plus a standby with a data copy. Two tenants sharing both is rare. An abusive tenant degrades a small random neighborhood, most of whom fail over automatically. It defends against the dimension you did not put a quota on.
</details>
<details><summary>Hint 4: the migration step everyone skips</summary>
After the registry flips, a stale routing cache can send a request to the old cell. The old cell must reject requests for tenants it no longer owns. Cell-side ownership validation is a correctness requirement.
</details>

## Interviewer Follow-Ups (answer out loud before revealing)

<details><summary>Q1: A tenant is at 9% of a shared cell's storage and growing 20% a month.</summary>
The placement engine flags it at 10%; a tiering proposal goes to the account team; migration to a reserved cell is scheduled in the tenant's low-traffic window with a tens-of-seconds write pause. It is routine, not an incident.
</details>
<details><summary>Q2: Deploy a database schema change across 100 cells.</summary>
Expand/contract as its own wave-gated deploy ahead of the code that uses it. Cells run mixed schema versions for the duration and the code tolerates both.
</details>
<details><summary>Q3: The control plane's database is corrupted.</summary>
Serving continues from pushed caches. Restore the registry from replica or backup. Reconcile against what each cell believes it owns; the cells' local tenant lists are the recovery source of truth. Freeze migrations until reconciled.
</details>
<details><summary>Q4: Why not Kubernetes namespaces per tenant?</summary>
A scheduling boundary, not a failure or performance boundary; namespaces share the control plane, network, and usually the database. Fine for a cell's internals; not a substitute for cells.
</details>
<details><summary>Q5: Why are deploys deliberately slow?</summary>
Version diversity: at any moment some cells run the previous version, so a data-dependent bug (a date, a payload shape) does not take every cell down simultaneously. Fast parallel deploys remove that defense.
</details>

## Scoring Focus

Weight dimensions 3 (derived constraints), 8 (invariants), 10 (failure modes, especially correlated), and 14 (cost). A principal answer prices blast radius for the business and maintains a shared-dependencies register.
