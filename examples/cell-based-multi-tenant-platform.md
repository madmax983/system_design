# Cell-Based Multi-Tenant SaaS Platform

> **Prompt:** Design the serving infrastructure for a B2B SaaS product with 50,000 tenants, ranging from 5-seat startups to enterprises with 100,000 users. A single tenant's traffic spike, bad data, or abusive workload must not affect other tenants. Enterprise tenants demand isolation guarantees, predictable performance, and the ability to be moved between regions.

The naive answer is "one big shared cluster with per-tenant rate limits." The naive answer works until the day a large tenant's batch import takes down the product for everyone, and that day is the one in the post-mortem. The principal answer is *cells*: bounded, independently-deployed replicas of the whole stack, each serving a subset of tenants, with a thin control plane above them.

## 1. Problem Framing

**What the prompt gets right:** blast radius is the core requirement. "Must not affect other tenants" is a statement about *failure isolation*, and failure isolation is an architectural property, not a rate limit.

**What the prompt gets wrong:**
- "Isolation" is treated as one thing. It is at least four: performance isolation (noisy neighbor), failure isolation (blast radius), data isolation (a bug cannot leak tenant A's data to tenant B), and operational isolation (a deploy or migration for tenant A does not touch tenant B). Each has a different mechanism and a different cost. A principal engineer separates them and prices them.
- "Enterprise tenants demand isolation" invites a design with dedicated infrastructure per enterprise. That is the correct answer for a *few* tenants and a catastrophe for 50,000. The design needs a spectrum: shared cells for the many, dedicated cells for the few, and a mechanism to move a tenant along that spectrum without a migration project.
- The prompt does not mention the control plane, and the control plane is where cell-based architectures fail: it is the one shared component, it is on the path for tenant routing, and it must be *less* available-critical than the cells it manages.

**The reframe to state out loud:** "I'm going to design a cell-based architecture: each cell is a complete, independently deployable stack serving a bounded number of tenants. A thin, read-mostly control plane routes tenants to cells. I'll separate the four kinds of isolation, define cell sizing from blast-radius targets, use shuffle sharding for the shared-cell tier, and design tenant migration as a routine operation rather than a project."

## 2. Requirements & SLOs

### Functional
- Tenants are routed to their cell by a stable identifier; every request is tenant-scoped
- A tenant can be moved between cells (for rebalancing, upgrading tier, or region change) with < 1 minute of write unavailability and zero data loss
- Tiers: shared (many tenants per cell), reserved (few tenants, guaranteed capacity), dedicated (one tenant per cell)
- Per-tenant quotas on request rate, storage, background job concurrency, and export bandwidth
- Per-cell deploys: a new version rolls out cell by cell with automatic rollback

### Non-Functional
| Requirement | Target | Note |
|-------------|--------|------|
| Blast radius of a cell failure | ≤ 2% of tenants (shared tier); 1 tenant (dedicated) | Sets cell size: 50,000 / 50 cells = 1,000 tenants per shared cell |
| Blast radius of a bad deploy | ≤ 1 cell before automatic halt | Cells deploy sequentially with SLO gates |
| Performance isolation | A tenant at 100x its quota degrades others in its cell by < 10% p99 | Quotas enforced at ingress, plus per-tenant resource pools |
| Data isolation | Provably zero cross-tenant reads | Enforced at the storage layer, not the application |
| Availability | 99.95% shared; 99.99% dedicated | Dedicated cells get dedicated on-call attention; that is part of what they pay for |
| Control plane dependency | Cells serve traffic for ≥ 24 h with the control plane down | The control plane is not on the hot path |
| Tenant migration | < 1 min write pause; routine operation, runs weekly | Migration is a feature, not an incident |

### Non-Goals
- Not designing the product's application logic; the cell contains it
- Not multi-region active-active *within* a tenant (a tenant lives in one region; see the [multi-region example](multi-region-active-active.md) for why)
- Not designing billing; quotas are the input to billing, not the output

**The dominant requirement:** *blast radius*. It drives cell size, drives the control-plane-off-the-hot-path rule, and drives per-cell deploys. It also sets the cost floor, because every cell carries fixed overhead.

## 3. Estimation

| Quantity | Estimate | Derivation |
|----------|----------|------------|
| Tenants | 50,000 | Given |
| Users | ~5M | Long tail: 49,000 tenants × ~20 users + 1,000 tenants × ~4,000 users |
| Peak request rate | ~500K RPS | 5M users × 10% concurrent × 1 req/s |
| Shared cells | ~50 | 1,000 tenants each, from the 2% blast-radius target |
| Reserved cells | ~20 | ~10 tenants each, for the ~200 largest non-dedicated tenants |
| Dedicated cells | ~30 | The largest enterprises |
| Cell capacity (shared) | ~10K RPS with 2x headroom | 500K / 50 |
| Per-cell fixed overhead | ~10-20% of a cell's cost | Load balancer, gateway, monitoring, minimum replica counts |
| Data per tenant | 100 MB median; 1 TB max | ~50 TB total, heavily skewed |
| Control plane read rate | ~500K/s (one lookup per request) | Must be cached at the edge; this is the number that puts the control plane off the hot path |

### Constraints the numbers force

1. **~100 cells means per-cell fixed overhead is a real cost line.** Cells must be sized large enough that overhead is < 20%, and small enough that blast radius is acceptable. 1,000 tenants per shared cell is the compromise; the number is derived from the blast-radius SLO, and if the SLO changes, the cell size changes.
2. **500K routing lookups/s cannot hit a database.** The tenant → cell mapping (50,000 × ~50 bytes = 2.5 MB) is replicated to every edge node and refreshed on change. The control plane publishes; the edge caches. The control plane could be down for a day and routing would continue.
3. **1 TB tenants in a shared cell are a noisy-neighbor problem by existence, not by behavior.** A single tenant with 1,000x the median data skews every per-cell operation (backups, index rebuilds, migrations). Data size is a tiering signal, not just request rate.
4. **~100 cells × sequential deploys × 20 minutes per cell = 33 hours per rollout.** That is too slow. Deploys go to cells in *waves* (1, then 5, then 20, then the rest) with SLO gates between waves. Blast radius is bounded per wave, not per cell.

## 4. Architecture

```
                  ┌────────────────────── Control Plane ────────────────────────┐
                  │  Tenant Registry     Cell Registry      Placement Engine    │
                  │  (tenant → cell,     (cell → region,    (balancing, tiering │
                  │   tier, quotas)       capacity, health)  decisions)         │
                  │                                                             │
                  │  Migration Orchestrator      Deploy Orchestrator            │
                  │  (move tenant A: cell 3→7)   (wave rollout, SLO gates)      │
                  └───────────┬─────────────────────────────────────────────────┘
                              │ publishes routing table (2.5 MB) to edge; push on change
                              ▼
                  ┌───────────────────────┐
     Clients ────▶│  Global Edge / Router │  tenant ID from token/subdomain → cell address
                  │  (cached routing;     │  per-tenant quota enforcement (token bucket keyed by tenant)
                  │   works if CP is down)│
                  └──┬────────┬────────┬──┘
                     │        │        │
          ┌──────────▼──┐ ┌───▼────────┐ ┌──▼──────────┐
          │  Cell S-01  │ │  Cell R-07 │ │  Cell D-22  │   ... ~100 cells
          │  (shared,   │ │  (reserved,│ │  (dedicated,│
          │   1000 ten.)│ │   10 ten.) │ │   1 tenant) │
          │             │ │            │ │             │
          │  API tier   │ │  API tier  │ │  API tier   │
          │  Workers    │ │  Workers   │ │  Workers    │
          │  Cache      │ │  Cache     │ │  Cache      │
          │  DB (tenant-│ │  DB        │ │  DB         │
          │   scoped    │ │            │ │             │
          │   schemas)  │ │            │ │             │
          │  Queue      │ │  Queue     │ │  Queue      │
          │  Object     │ │  Object    │ │  Object     │
          │   storage   │ │   storage  │ │   storage   │
          │   prefix    │ │   prefix   │ │   prefix    │
          └─────────────┘ └────────────┘ └─────────────┘
              nothing shared between cells except the control plane and the edge
```

**A cell is the whole stack.** API servers, workers, cache, database, queue, and an object-storage prefix. Nothing inside a cell talks to anything inside another cell. If it did, a cell failure would propagate, and the blast-radius guarantee would be a lie.

### The request path

```
1. Request arrives at edge with tenant ID (from the auth token or subdomain)
2. Edge looks up tenant → cell in its local routing cache (hit rate ~100%)
3. Edge checks the tenant's quota (token bucket, keyed by tenant, local to the edge node
   with periodic sync; slight over-admission is acceptable)
4. Edge forwards to the cell's ingress
5. Cell ingress re-validates the tenant belongs to this cell (defense in depth: a stale
   routing cache during migration must not let a request land on the wrong cell)
6. Cell serves the request; every storage access is tenant-scoped (Deep Dive 2)
```

## 5. Deep Dives

### Deep Dive 1: Shuffle sharding for the shared tier

Putting 1,000 tenants in a cell bounds the blast radius of a *cell* failure to 2%. It does not bound the blast radius of a *tenant*: a single abusive tenant degrades all 1,000 in its cell. Quotas help, but quotas are a rate limit on a dimension you anticipated; the abuse that takes down a cell is always on a dimension you did not.

**Shuffle sharding** changes the question. Instead of assigning each tenant to one cell, assign each tenant to a *combination* of nodes within a cell (or across a small set of cells). With, say, 8 worker nodes per cell and each tenant assigned to 2 of them, there are 28 possible pairs. Two tenants share *both* nodes with probability 1/28; they share *one* node with probability ~50%; and a tenant whose two nodes are both saturated by an abuser is rare and, with client-side retry to the other node, rarer.

At the cell level: 50 shared cells, each tenant assigned to a primary cell plus a *standby* cell with a copy of its data. If the primary cell is degraded (not down; degraded, which is the common case), the edge fails the tenant over to its standby. The set of tenants on any (primary, standby) pair is small, so a degraded cell affects 2% of tenants, and the fraction whose standby is *also* degraded is 2% of 2%.

**The cost:** data is replicated to the standby cell (2x storage for the shared tier), and the standby must be kept in sync (CDC, like the [multi-region example](multi-region-active-active.md)). For 100 MB median tenants that is cheap. For the 1 TB tenants, it is the reason they belong in the reserved tier, where isolation comes from having few neighbors rather than from statistical dilution.

**The principal-level point:** quotas are the *first* line of defense and are necessary. Shuffle sharding is the defense against the failure you did not predict, and it converts "an abusive tenant takes down a cell" into "an abusive tenant degrades a statistically small, random set of neighbors, most of whom fail over automatically."

### Deep Dive 2: Data isolation that does not depend on application code

"Every query includes `WHERE tenant_id = ?`" is an application-level convention, and conventions have exceptions. The first cross-tenant data leak in a SaaS company's history is almost always a missing `WHERE` clause in a reporting query or a background job. Data isolation must be enforced *below* the application.

**Mechanisms, in order of strength:**

| Mechanism | How | Strength | Cost |
|-----------|-----|----------|------|
| Row-level security in the database | A session variable sets the tenant; database policies filter every table automatically; the application cannot see other tenants' rows even with a bug | Strong: enforced by the DB engine | Session setup on every connection; policy maintenance; ~5% query overhead |
| Schema-per-tenant | Each tenant's tables are in its own schema; the connection is scoped to the schema | Strong; also simplifies per-tenant export and deletion | 1,000 schemas per cell database; migrations run 1,000 times; connection pooling is per-schema |
| Database-per-tenant | Each tenant has its own database instance | Strongest; also gives performance isolation | Expensive; only for dedicated cells |
| Encryption per tenant | Each tenant's data encrypted with a tenant-specific key; a leaked row is unreadable | Defense in depth; also enables crypto-shredding for deletion | Key management complexity |

**The chosen combination:** row-level security in shared and reserved cells (the connection sets the tenant at checkout from the pool, and a test suite asserts every table has a policy), database-per-tenant in dedicated cells, and per-tenant encryption keys everywhere for deletion and defense in depth.

**Verification:** a *canary tenant* in every cell with known synthetic data. A continuous job queries every API and background-job output for the canary's markers appearing under any other tenant's context. This turns "we believe isolation holds" into a monitored SLI.

### Deep Dive 3: Tenant migration as a routine operation

Tenants move between cells for rebalancing (a cell is full), tiering (a tenant grows into reserved), region change (data residency), or evacuation (a cell is being retired). At 50,000 tenants, this happens weekly. It must be boring.

```
Migrate tenant T from cell A to cell B:

1. Orchestrator marks T as MIGRATING in the registry; edge continues routing to A
2. Snapshot T's data in A (tenant-scoped export: the reason schema-per-tenant or RLS matters)
3. Restore into B; begin CDC from A to B for T's rows (tenant-filtered change stream)
4. Wait for CDC lag < 1 s
5. **Write pause**: edge returns 503 with Retry-After: 5 for T's writes (reads continue from A)
6. Drain in-flight requests in A (bounded, ~10 s); final CDC catch-up
7. Flip registry: T → B; push routing update to edge (propagation ~seconds)
8. Cell A's ingress now rejects T (step 5 of the request path); any stragglers retry and land on B
9. Un-pause writes; T is now served by B
10. Verify: sampled row compare A vs B for T; A's copy is retained 7 days, then purged
    (purge is a compliance requirement, not housekeeping)

Rollback before step 7: abort, delete B's copy. After step 7: reverse the procedure; B is
authoritative, so it is a new migration, not a rollback.
```

The write pause is the honest cost. It is tens of seconds, it is announced via `Retry-After`, and client SDKs handle it transparently. Product teams will ask for zero-pause migration; the answer is that zero-pause requires dual-write with conflict resolution for a window, which is a multi-master problem, and tens of seconds is cheaper than that problem.

**The step everyone skips:** step 8. During routing-cache propagation, a request can arrive at cell A for a tenant that now belongs to B. Cell A must *know* it does not own T and reject the request, or A and B will both accept writes for the same tenant for a few seconds. Cell-side ownership validation is a correctness requirement.

### Deep Dive 4: The control plane, and keeping it off the hot path

The control plane holds the tenant and cell registries, makes placement decisions, and orchestrates migrations and deploys. It is the only shared component, so it is the largest blast radius, so it must be the *least* critical to serving traffic.

**Rules:**
- **Cells never call the control plane to serve a request.** They cache their tenant list; the edge caches the routing table. The control plane *pushes* changes; nobody pulls on the hot path.
- **The control plane is read-mostly and its writes are rare.** Registry changes happen on tenant creation, migration, and tiering: thousands per day, not per second. It can be a single-region, strongly consistent, boring relational database with a replica in another region for disaster recovery.
- **Every cell can start from its last-known configuration.** A cell restarting while the control plane is down loads its cached tenant list from local storage and serves. A *brand-new* cell cannot be provisioned without the control plane, which is acceptable.
- **Deploys and migrations halt safely when the control plane is unavailable.** They are orchestrated as state machines with checkpoints; an outage mid-migration leaves the tenant in a defined state (before step 7: served by A; after: served by B) and resumes.

**The failure that this prevents:** a control-plane database failover that takes 30 seconds becomes a 30-second outage for every tenant if routing lookups are on the hot path. With pushed caches, it is a 30-second delay in the *next* tenant creation. That is the difference between a P0 and a footnote.

## 6. Invariants

| Invariant | Where enforced | Verification |
|-----------|---------------|--------------|
| Every tenant is served by exactly one cell at any moment | Registry is the source of truth; cell-side ownership validation rejects misrouted requests | Count of ownership rejections per cell (nonzero only during migrations) |
| No request reads or writes another tenant's data | Row-level security / schema scoping at the DB; per-tenant object-storage prefixes with IAM conditions | Canary tenant leak detection; DB policy coverage test in CI |
| No component inside a cell depends on another cell | Network policy: cells cannot reach each other's address space | Periodic connectivity audit; any cross-cell flow is a build failure |
| A cell serves traffic without the control plane | Pushed configuration; local persistence of tenant list | Quarterly game day: control plane taken down for 1 hour in production |
| A deploy cannot proceed past a cell whose SLO is burning | Deploy orchestrator gates on per-cell SLIs | Deploys are the most common cause of outages; this gate is exercised weekly by normal deploys |
| Per-tenant quotas are enforced before the request reaches the cell | Edge token bucket keyed by tenant | Throttle metrics per tenant; sum of admitted ≤ sum of quotas + tolerance |

**Deliberately relaxed:** quota enforcement at the edge is approximate (each edge node has a local bucket, synced periodically). A tenant can briefly exceed its quota by the number of edge nodes × bucket size. Exact global rate limiting would put a shared counter on the hot path; approximate is the right trade and is documented as such.

## 7. Failure Modes

| Failure | Blast radius | Behavior | Mitigation |
|---------|-------------|----------|------------|
| Shared cell fails | 2% of tenants | Edge fails over to standby cells (shuffle sharding) | Standby CDC; failover is automatic on cell health, human-confirmed for failback |
| Dedicated cell fails | 1 tenant | That tenant is down until the cell recovers or is rebuilt from backup | Dedicated tenants can pay for a warm standby cell; it is a tier feature |
| Abusive tenant in shared cell | Its cell, partially | Quotas throttle known dimensions; shuffle sharding limits neighbors affected by unknown ones | Placement engine detects sustained per-tenant resource skew and *proposes* a tiering change |
| Control plane down | Tenant creation, migrations, deploys | Serving continues from caches | 24 h target; game day tested |
| Routing table corruption pushed to edge | Every tenant misrouted | Cell-side ownership validation rejects everything → 100% errors | Routing table pushes are versioned, validated (every tenant maps to a live cell; no tenant maps to two), and canaried to one edge node first |
| Bad deploy | 1 cell (wave 1) | SLO gate halts the rollout; automatic rollback of the affected cell | Wave 1 is a cell of internal/canary tenants |
| Cell database fills (a 1 TB tenant grows) | Its cell | Writes fail for the whole cell | Per-tenant storage quotas; placement engine flags tenants > 10% of a cell's capacity for tiering *before* they fill it |
| Migration stuck mid-way | 1 tenant | Defined state per checkpoint | Orchestrator is a durable state machine; every step is idempotent and resumable |
| **Correlated: shared dependency outside the cells** (identity provider, DNS, certificate authority, the container registry all cells pull images from) | Everything | Full outage | Enumerated in a "shared dependencies" register with an owner and an SLO each; the container registry is mirrored per region; identity tokens are validated locally by public key, not by calling the IdP |
| **Correlated: same bug in every cell** (a data-dependent crash triggered by a date) | Everything, but *not simultaneously* if cells run mixed versions | Cells on the older version survive | Deploy waves are deliberately slow (days, not hours) so that at any time some cells run the previous version |

**Degradation order:**
1. Halt migrations and deploys
2. Throttle background jobs (imports, exports, reports) per tenant, then globally
3. Shed non-essential features per cell (search, analytics dashboards)
4. Fail over degraded shared cells to standbys
5. Serve read-only from standbys if a cell's primary database is lost

## 8. Evolution Path

**v1: one cell.** Build the application as a cell from day one: tenant-scoped everything, RLS, an ingress that validates tenant ownership (trivially true), a routing layer with a table of one entry. The cost is near zero and the payoff is that "add a second cell" is a configuration change instead of an architecture change.

**v2: N shared cells, manual placement.** Add cells as capacity requires. Placement is a spreadsheet. Migrations are a runbook. Build the control plane registry as the first real component.

**v3: tiers and automated placement.** Reserved and dedicated tiers. The placement engine proposes moves; a human approves. Migration orchestrator is a durable state machine.

**v4: shuffle sharding and standby cells.** Once cell count is large enough for statistical isolation to mean something (~20+ cells).

**v5: regional cells and residency.** Cells are placed in regions; tenants are assigned by residency; the multi-region concerns of the [multi-region example](multi-region-active-active.md) apply to the control plane's replication only.

**The one-way door:** tenant-scoped storage from v1. Retrofitting RLS onto a schema that was not designed for it means touching every table and every query, and that is a migration of its own.

## 9. Cost Model

| Component | Dominant driver | The knob |
|-----------|----------------|----------|
| Per-cell fixed overhead (~100 cells) | Cell count | Cell size: fewer, larger cells cost less and have larger blast radius. The SLO sets it |
| Headroom for failover | Standby capacity | Shuffle-sharded standbys share headroom statistically; dedicated warm standbys do not |
| Storage duplication (shared tier standbys) | 2x for shared tenants | Which tiers get standbys; shared tenants are small, so this is cheap |
| Dedicated cells | Per-tenant | Priced into the tier; a dedicated cell at 10% utilization is the tenant's cost, not the platform's |
| Control plane | Negligible | N/A |

**The insight:** the cost conversation is *cell size versus blast radius*, and it should be had with the business explicitly: "a 2% blast radius costs X; a 5% blast radius costs 0.6X; here is what a 5% incident looks like in customer count." Engineering does not own that decision; engineering owns making it legible.

## 10. What Breaks at 10x

At 500,000 tenants, 5M RPS, ~1,000 cells:

**First: the operational model.** 1,000 cells cannot be operated by humans looking at dashboards. Fix: cells become fully homogeneous and immutable; every operation (deploy, scale, migrate, retire) is driven by the orchestrators with no per-cell human action; on-call sees *aggregates* and *outliers*, never individual cells.

**Second: deploy duration.** 1,000 cells in waves is days. Fix: waves scale geometrically (1, 10, 100, 889) and the last wave is parallel within a region; the SLO gate is on the aggregate of the wave.

**Third: the routing table.** 500,000 entries is 25 MB; still fine to push, but the push fan-out to thousands of edge nodes on every change is not. Fix: the edge pulls deltas from a versioned, CDN-distributed snapshot; the control plane publishes to the CDN.

## 11. Rejected Alternatives

| Alternative | Why it loses |
|-------------|-------------|
| **One shared cluster with per-tenant rate limits** | Rate limits cover anticipated dimensions; the outage is always on the unanticipated one. Blast radius is 100% |
| **Dedicated infrastructure per tenant, for all tenants** | 50,000 stacks; fixed overhead dominates; deploys take weeks; the operational model does not exist |
| **Cells that share a database cluster for efficiency** | The database becomes the cross-cell dependency; a slow query in one cell's tenant affects every cell. If cells share storage, they are not cells |
| **Routing lookups on the hot path against the control plane** | Every control-plane hiccup is a global outage; covered in Deep Dive 4 |
| **Application-level tenant filtering only** | The first missing `WHERE` clause is a data-leak incident. Enforcement must be below the application |
| **Zero-pause tenant migration via dual-write** | Turns a migration into a multi-master conflict problem for a window; tens of seconds of write pause is the cheaper trade |
| **Fast, parallel deploys to all cells** | Removes the version-diversity defense against data-dependent bugs, and makes the bad-deploy blast radius 100% |

## 12. Level Signals

**A senior answer** designs a shared multi-tenant stack with tenant IDs on every table, per-tenant rate limits, and maybe a dedicated deployment option for large customers.

**A staff answer** introduces cells, sizes them from a blast-radius target, keeps the control plane off the hot path, uses row-level security, and designs a tenant migration procedure with a write pause.

**A principal answer** does all of that, and additionally:
- Separates the four kinds of isolation and gives each its own mechanism and cost
- Uses shuffle sharding to bound the blast radius of the *unanticipated* abuse, and explains why quotas alone are insufficient
- Treats data isolation as a monitored SLI (canary tenant) rather than a code convention
- Identifies cell-side ownership validation as the correctness requirement that makes migration safe during routing propagation
- Makes deploy waves deliberately slow *on purpose*, to preserve version diversity against data-dependent bugs
- Maintains a shared-dependencies register and names the ones that actually cause correlated outages
- Turns cell size versus blast radius into a business decision with a price tag

**Interviewer follow-ups to expect:**
- "A tenant is at 9% of a shared cell's storage and growing 20% a month. What happens?" (The placement engine flags it at 10%; a tiering proposal goes to the account team; the migration to a reserved cell is scheduled during the tenant's low-traffic window; it is routine.)
- "How do you deploy a database schema change across 100 cells?" (Expand/contract, as its own wave-gated deploy, ahead of the code that uses it. Cells run mixed schema versions for the duration; the code must tolerate both.)
- "The control plane's database is corrupted. What do you do?" (Serving continues from caches. Restore the registry from the replica or backup. Reconcile against what each cell believes it owns; the cells' local tenant lists are the recovery source of truth. Freeze migrations until reconciled.)
- "Why not just use Kubernetes namespaces per tenant?" (Namespaces are a *scheduling* boundary, not a failure or performance boundary; they share the control plane, the network, and usually the database. They can implement a cell's internals; they are not a substitute for cells.)
