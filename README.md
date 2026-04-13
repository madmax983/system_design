# System Design Patterns & Vocabulary

A comprehensive reference for system design interviews. Covers core fundamentals, reusable design patterns, infrastructure building blocks, and a vocabulary glossary with back-of-the-envelope estimation guides.

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

---

### [Examples](examples/)
Worked examples that combine fundamentals, patterns, and vocabulary into interview-ready answers.

| Example | Description |
|---------|-------------|
| [Question 1: Distributed Metrics Logging & Aggregation](examples/question-1-distributed-metrics-logging-and-aggregation.md) | First-principles breakdown with Mermaid diagrams, APIs, and trade-offs |

---

## How to Use This Repository

1. **Before an interview**: Read through the [Fundamentals](fundamentals/) to solidify core concepts
2. **During practice**: Use [Patterns](patterns/) as building blocks when designing systems
3. **Quick reference**: Keep the [Glossary](vocabulary/glossary.md) and [Estimation Cheatsheet](vocabulary/estimation-cheatsheet.md) handy
4. **Mock interviews**: Follow the [Interview Framework](vocabulary/interview-framework.md) step-by-step
