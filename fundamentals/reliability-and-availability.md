# Reliability & Availability

## Definitions

**Reliability** — the probability a system performs its intended function without failure over a given time period. A reliable system produces correct results even under adverse conditions.

**Availability** — the proportion of time a system is operational and accessible. Usually expressed as a percentage ("nines").

**Relationship:** A reliable system is available, but an available system is not necessarily reliable (it could return wrong answers while staying "up").

## The Nines of Availability

| Availability | Downtime/Year | Downtime/Month | Downtime/Week |
|-------------|---------------|----------------|---------------|
| 99% (two 9s) | 3.65 days | 7.31 hours | 1.68 hours |
| 99.9% (three 9s) | 8.77 hours | 43.83 min | 10.08 min |
| 99.99% (four 9s) | 52.60 min | 4.38 min | 1.01 min |
| 99.999% (five 9s) | 5.26 min | 26.30 sec | 6.05 sec |

**In series:** `A_total = A1 × A2 × ... × An` — total availability decreases as you add components.

**In parallel (redundancy):** `A_total = 1 - (1 - A1) × (1 - A2)` — total availability increases with redundancy.

## SLAs, SLOs, and SLIs

| Term | Definition | Example |
|------|-----------|---------|
| **SLI** (Service Level Indicator) | A measured metric | p99 latency = 200ms |
| **SLO** (Service Level Objective) | A target for an SLI | p99 latency < 300ms |
| **SLA** (Service Level Agreement) | A contract with consequences | 99.9% uptime or credits issued |

SLIs feed into SLOs. SLOs feed into SLAs. Set SLOs tighter than SLAs so you have an error budget to catch problems before breaching the contract.

## Fault Tolerance Strategies

### Redundancy
- **Active-active:** Multiple instances handle traffic simultaneously. No failover delay.
- **Active-passive:** One instance handles traffic; standby takes over on failure. Simpler but has failover latency.
- **N+1 / N+2:** Run more instances than needed so you can survive 1 or 2 failures.

### Failover
- **DNS failover:** Update DNS records to point to healthy endpoints. Slow (TTL-dependent).
- **Load balancer failover:** Health checks detect failures, traffic rerouted in seconds.
- **Database failover:** Promote a replica to primary. Risk of data loss if replication is async.

### Replication
- **Synchronous:** Primary waits for replica acknowledgment. Zero data loss, higher latency.
- **Asynchronous:** Primary doesn't wait. Lower latency, potential data loss on failure.
- **Semi-synchronous:** Wait for at least one replica. Balance of durability and performance.

## Design Principles for Reliability

1. **Design for failure** — assume every component will fail and plan for it
2. **Eliminate single points of failure (SPOF)** — redundancy at every layer
3. **Fail fast** — detect errors quickly, don't let them cascade
4. **Graceful degradation** — serve partial results rather than failing entirely
5. **Blast radius containment** — isolate failures so they don't take down the whole system (bulkheads)
6. **Chaos engineering** — proactively inject failures to find weaknesses

## Health Checks & Monitoring

| Type | Purpose | Example |
|------|---------|---------|
| **Liveness check** | Is the process alive? | HTTP 200 from `/health` |
| **Readiness check** | Can it serve traffic? | DB connection established |
| **Deep health check** | Are all dependencies healthy? | Check DB + cache + downstream services |

**Alerting hierarchy:** Pages (immediate action) > Alerts (investigate soon) > Logs (forensics).

## Key Interview Talking Points

- Always quantify availability: "We need four nines" is clearer than "highly available"
- Serial dependencies multiply failure probabilities — minimize the critical path
- Redundancy is the primary tool for availability; replication is the primary tool for durability
- Active-active is more complex but has zero failover latency
- Error budgets make reliability a feature that competes with velocity
