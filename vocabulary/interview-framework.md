# System Design Interview Framework

A step-by-step approach for tackling any system design question. Aim to spend roughly the following time per section in a 45-minute interview.

> **Interviewing at staff or principal level?** This framework is the baseline. See [Principal Engineer Signals](principal-engineer-signals.md) for how the pacing and emphasis shift, and the [Worked Examples](../examples/) for what a complete principal-level answer looks like.

## Step 1: Clarify Requirements (5 minutes)

**Goal:** Understand what you're building and define the scope.

### Functional Requirements
Ask the interviewer to confirm:
- What are the core features? (List 3-5 most important)
- Who are the users? (End users, internal services, third-party developers)
- What are the key use cases? (Walk through the user journey)

### Non-Functional Requirements
- **Scale:** How many users? How many requests per second?
- **Availability:** What uptime is required? (99.9%? 99.99%?)
- **Latency:** What's the acceptable response time? (< 200ms? < 1s?)
- **Consistency:** Can we tolerate eventual consistency, or do we need strong consistency?
- **Durability:** Can we lose data? (Usually no for user data, yes for caches)

### Scope Boundaries
- What features are out of scope?
- Is this a greenfield design or an extension of an existing system?
- Any specific technology constraints?

**Output:** A short list of functional and non-functional requirements agreed upon with the interviewer.

## Step 2: Back-of-the-Envelope Estimation (3-5 minutes)

**Goal:** Quantify the scale to inform design decisions.

Calculate:
1. **QPS** (queries per second) — average and peak
2. **Storage** — total data over the retention period
3. **Bandwidth** — inbound and outbound
4. **Cache size** — using the 80/20 rule

See the [Estimation Cheatsheet](estimation-cheatsheet.md) for formulas and reference numbers.

**Output:** Key numbers that justify your design choices (e.g., "at 50K QPS we need sharding" or "500 GB fits in a single Redis instance").

## Step 3: High-Level Design (10 minutes)

**Goal:** Draw the architecture — the major components and how they interact.

### Start with the Data Flow
1. What does a write path look like? (Client → API → processing → storage)
2. What does a read path look like? (Client → API → cache → storage)

### Core Components
Draw these (even if some will be refined later):
- **Clients** (web, mobile, API consumers)
- **API layer** (REST/gRPC endpoints)
- **Application services** (business logic)
- **Data stores** (database, cache, blob storage)
- **Async processing** (queues, workers)

### API Design
Define the key endpoints:
```
POST /api/v1/resource    — create
GET  /api/v1/resource/id — read
PUT  /api/v1/resource/id — update
DELETE /api/v1/resource/id — delete
```

Include request/response shapes for the most critical endpoints.

**Output:** A diagram showing the major components, data flow arrows, and key APIs.

## Step 4: Deep Dive into Core Components (15 minutes)

**Goal:** Design the most critical parts in detail. The interviewer may steer you toward specific areas.

### Database Design
- Schema design (key tables/collections, relationships)
- SQL vs. NoSQL choice with justification
- Indexing strategy
- Sharding key (if needed)

### Scaling Strategy
- Read vs. write ratio — drives caching and replication decisions
- Caching layer (what to cache, eviction policy, invalidation)
- Database scaling (replicas for reads, sharding for writes)

### Critical Algorithms
- How does the core feature work? (e.g., news feed ranking, URL shortening hash, rate limiting)
- Trade-offs in the algorithm choice

### Data Flow for Key Operations
Walk through the most important operations step by step:
1. Request arrives at the API gateway
2. Authentication/authorization check
3. Business logic in the service layer
4. Data written to primary store
5. Cache updated/invalidated
6. Event published for async processing
7. Response returned to client

## Step 5: Address Non-Functional Requirements (5-7 minutes)

**Goal:** Show that your design handles real-world challenges.

### Scalability
- Which components need horizontal scaling?
- Where are the bottlenecks? How do you address them?

### Reliability & Availability
- What are the single points of failure? How do you add redundancy?
- What happens when [component X] goes down?
- How does failover work?

### Consistency
- Which operations need strong consistency?
- Where can you tolerate eventual consistency?
- How do you handle conflicts?

### Monitoring & Observability
- Key metrics to monitor (latency percentiles, error rates, queue depth)
- Alerting strategy
- Logging and tracing

### Security
- Authentication (OAuth, JWT, API keys)
- Authorization (RBAC, per-resource permissions)
- Data encryption (at rest, in transit)
- Rate limiting and DDoS protection

## Step 6: Review & Trade-offs (3-5 minutes)

**Goal:** Demonstrate mature engineering judgment.

### Summarize Key Trade-offs
For each major decision, state:
- What you chose
- What you traded away
- Why it's the right call for this system

### Identify Limitations
- What doesn't scale well in this design?
- What would break first under 10x growth?
- What would you do differently with unlimited time?

### Future Improvements
- What features or improvements would you add next?
- How would the architecture evolve?

## Common Pitfalls to Avoid

| Pitfall | Better Approach |
|---------|----------------|
| Jumping into the solution | Spend 5 min on requirements first |
| Not estimating scale | A few numbers justify every design decision |
| Single database with no caching | Always consider read optimization |
| Ignoring failure modes | Discuss what happens when things break |
| Over-engineering | Start simple, add complexity only when justified by requirements |
| Not discussing trade-offs | Every decision has a trade-off — name it |
| Monologuing | Check in with the interviewer regularly |
| Trying to cover everything | Go deep on 2-3 areas rather than shallow on everything |

## Beyond the Framework: The Principal-Level Additions

At senior level, executing the six steps well is the bar. At staff and principal level, interviewers listen for additions the framework does not prompt:

| Addition | Where it fits | One-line version |
|----------|--------------|------------------|
| **Reframe the problem** | Step 1 | "The requirement that dominates is X; the requirement in the prompt that is subtly wrong is Y." |
| **Derive constraints, not just numbers** | Step 2 | "This number forces this decision." |
| **State invariants and where they are enforced** | Step 4 | "This must always be true; it is enforced here; it is verified by this." |
| **Name the deliberately relaxed invariant** | Step 4 | "This is temporarily false during X; the compensating mechanism is Y." |
| **Correlated failure and degradation order** | Step 5 | "The failure that hits everything is Z. Under overload we shed A, then B, and never C." |
| **Evolution path and one-way doors** | Step 6 | "v1 is this; the path to v3 under live traffic is this; the decisions we cannot undo are these." |
| **The cost knob** | Step 6 | "The dominant cost is X and the knob is Y." |

The [scorecard](../practice/scorecard.md) turns these into an 18-dimension self-assessment.

## Cheat Sheet: Components to Consider

Use this as a mental checklist. Not everything applies to every problem.

```
[ ] DNS / CDN
[ ] Load Balancer
[ ] API Gateway
[ ] Application Servers
[ ] Cache (Redis/Memcached)
[ ] Primary Database (SQL/NoSQL)
[ ] Read Replicas
[ ] Search Index (Elasticsearch)
[ ] Message Queue (Kafka/SQS)
[ ] Object Storage (S3)
[ ] Notification Service
[ ] Monitoring / Logging
[ ] Rate Limiter
```
