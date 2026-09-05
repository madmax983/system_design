# System Design Patterns & Vocabulary

A comprehensive reference for system design interviews, from the fundamentals through Principal / IC7-level worked designs. Covers core fundamentals, reusable design patterns, infrastructure building blocks, a vocabulary glossary with back-of-the-envelope estimation guides, seven fully worked principal-level examples, and a deliberate-practice program built around them.

## Where to Start

| If you are... | Start here |
|---------------|-----------|
| Building foundations | [Fundamentals](fundamentals/) → [Patterns](patterns/) → [Building Blocks](building-blocks/) |
| Preparing for senior interviews | [Interview Framework](vocabulary/interview-framework.md) + [Estimation Cheatsheet](vocabulary/estimation-cheatsheet.md) |
| Preparing for staff / principal interviews | [Principal Engineer Signals](vocabulary/principal-engineer-signals.md) → [Worked Examples](examples/) → [Practice](practice/) |
| Running mocks with a partner | [Interviewer Guide](practice/interviewer-guide.md) + any [prompt card](practice/prompts/) |

## Repository Structure

### [Fundamentals](fundamentals/)
Core concepts that underpin every system design discussion.

| Topic | Description |
|-------|-------------|
| [Scalability](fundamentals/scalability.md) | Horizontal vs. vertical scaling, stateless design, auto-scaling |
| [Reliability & Availability](fundamentals/reliability-and-availability.md) | Fault tolerance, redundancy, SLAs, SLOs, SLIs |
| [CAP Theorem & Consistency](fundamentals/cap-theorem-and-consistency.md) | CAP trade-offs, consistency models (strong, eventual, causal) |
| [Latency & Throughput](fundamentals/latency-and-throughput.md) | Performance measurement, bottleneck analysis, Little's Law |
| [Networking Essentials](fundamentals/networking-essentials.md) | TCP/UDP, HTTP/HTTPS, DNS resolution, TLS handshakes |

### [Patterns](patterns/)
Reusable architectural patterns for solving common system design problems.

| Pattern | Description |
|---------|-------------|
| [Caching](patterns/caching.md) | Cache-aside, write-through, write-behind, eviction policies |
| [Load Balancing](patterns/load-balancing.md) | Round-robin, least connections, consistent hashing |
| [Database Scaling](patterns/database-scaling.md) | Replication, sharding, partitioning, read replicas |
| [Message Queues & Async Processing](patterns/message-queues.md) | Pub/sub, event-driven architecture, backpressure |
| [API Design](patterns/api-design.md) | REST, GraphQL, gRPC, API gateways, versioning |
| [Microservices](patterns/microservices.md) | Service decomposition, saga pattern, CQRS, event sourcing |
| [Rate Limiting & Throttling](patterns/rate-limiting.md) | Token bucket, leaky bucket, sliding window |
| [Consistency Patterns](patterns/consistency-patterns.md) | Two-phase commit, consensus algorithms, conflict resolution |

### [Building Blocks](building-blocks/)
Infrastructure components you'll reference in every design.

| Component | Description |
|-----------|-------------|
| [DNS & CDN](building-blocks/dns-and-cdn.md) | Name resolution, content distribution, edge caching |
| [Load Balancers & Reverse Proxies](building-blocks/load-balancers-and-reverse-proxies.md) | L4 vs. L7 balancing, health checks, SSL termination |
| [Databases](building-blocks/databases.md) | SQL vs. NoSQL, ACID, BASE, indexing, query optimization |
| [Caches](building-blocks/caches.md) | Redis, Memcached, local vs. distributed caches |
| [Message Brokers](building-blocks/message-brokers.md) | Kafka, RabbitMQ, SQS — when to use which |
| [Blob & Object Storage](building-blocks/blob-and-object-storage.md) | S3-style storage, data lakes, content-addressable storage |
| [Search Engines](building-blocks/search-engines.md) | Elasticsearch, inverted indexes, full-text search |

### [Vocabulary](vocabulary/)
Quick-reference material for interviews.

| Resource | Description |
|----------|-------------|
| [Glossary](vocabulary/glossary.md) | A-to-Z definitions of system design terms |
| [Estimation Cheatsheet](vocabulary/estimation-cheatsheet.md) | Powers of two, latency numbers, back-of-the-envelope math |
| [Interview Framework](vocabulary/interview-framework.md) | Step-by-step approach to tackling any system design question |
| [Principal Engineer Signals](vocabulary/principal-engineer-signals.md) | What separates senior, staff, and principal answers; the questions to ask; a design review rubric; principal-level pacing |

### [Worked Examples](examples/) (IC7 / Principal Level)
Complete designs under real constraints, each following a twelve-section template: problem reframing, SLO-driven requirements, estimation that forces decisions, architecture, deep dives on the hardest sub-problems, invariants, failure modes with blast radius and degradation order, evolution path, cost model, what breaks at 10x, rejected alternatives, and level signals.

| Example | Core Tension |
|---------|-------------|
| [Global Payments Ledger](examples/global-payments-ledger.md) | Exactly-once money movement vs. availability; idempotency, sagas, reconciliation |
| [Multi-Region Active-Active](examples/multi-region-active-active.md) | Local latency vs. a source of truth; per-class conflict resolution, residency, failover |
| [Event Streaming Platform](examples/event-streaming-platform.md) | 10M events/sec; the honest boundary of exactly-once, schema governance, isolated replay |
| [Zero-Downtime Database Migration](examples/zero-downtime-database-migration.md) | Changing the storage layer under live traffic with rollback at every stage |
| [Cell-Based Multi-Tenant Platform](examples/cell-based-multi-tenant-platform.md) | Tenant isolation vs. cost; shuffle sharding, control plane off the hot path, routine migration |
| [Distributed Coordination Service](examples/distributed-coordination-service.md) | Lock correctness under partitions; fencing tokens, leases, watches, guardrails |
| [Observability Pipeline](examples/observability-pipeline.md) | Cardinality and cost vs. debugging an outage; a pipeline that is up when nothing else is |

Three further IC6/IC7 examples take a first-principles, build-the-component angle (product contract, capacity model, APIs, failure-first write path, roadmap) and pair naturally with the platform-level designs above:

| Example | Pairs with |
|---------|-----------|
| [Distributed Metrics Logging & Aggregation](examples/question-1-distributed-metrics-logging-and-aggregation.md) | [Observability Pipeline](examples/observability-pipeline.md) |
| [Distributed Stream Processing like Kafka](examples/question-2-distributed-stream-processing-like-kafka.md) | [Event Streaming Platform](examples/event-streaming-platform.md) |
| [Globally Distributed Key-Value Store](examples/question-3-design-a-key-value-store.md) | [Multi-Region Active-Active](examples/multi-region-active-active.md) |

### [Practice](practice/)
A self-directed practice program: attempt before reading, score against a rubric, reflect, re-attempt on a spaced schedule.

| Resource | Description |
|----------|-------------|
| [Practice Method](practice/README.md) | The session protocol, spacing schedule, and how to pick a focus |
| [Prompt Cards](practice/prompts/) | One per worked example: prompt, pacing, hidden hints, interviewer follow-ups with hidden answers |
| [Prompt Bank](practice/prompt-bank.md) | Twelve additional principal-level prompts with hidden hints |
| [Scorecard](practice/scorecard.md) | Eighteen-dimension self-assessment with level thresholds |
| [Interviewer Guide](practice/interviewer-guide.md) | How a partner runs a mock from any card |
| [Reflection Template](practice/reflection-template.md) | Post-session journal |

---

## How to Use This Repository

1. **Before an interview**: Read through the [Fundamentals](fundamentals/) to solidify core concepts
2. **During practice**: Use [Patterns](patterns/) as building blocks when designing systems
3. **Quick reference**: Keep the [Glossary](vocabulary/glossary.md) and [Estimation Cheatsheet](vocabulary/estimation-cheatsheet.md) handy
4. **Mock interviews**: Follow the [Interview Framework](vocabulary/interview-framework.md) step-by-step
5. **Principal-level preparation**: Read the [Signals](vocabulary/principal-engineer-signals.md), then work through the [Practice](practice/) program using the [Worked Examples](examples/) as the answer key. Attempt every prompt before reading its solution; the learning is in the gap.
