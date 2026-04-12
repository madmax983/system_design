# Scalability

Scalability is the ability of a system to handle increased load by adding resources. A well-designed system scales gracefully — performance degrades linearly (or not at all) as load increases.

## Vertical Scaling (Scale Up)

Adding more power to a single machine: faster CPU, more RAM, larger disks.

**Pros:**
- Simple — no code changes needed
- No distributed systems complexity
- Strong consistency is straightforward

**Cons:**
- Hardware limits create a hard ceiling
- Single point of failure
- Expensive at the top end (superlinear cost)
- Downtime during upgrades

**When to use:** Early-stage products, databases that are hard to shard, workloads that need strong single-node performance (e.g., in-memory analytics).

## Horizontal Scaling (Scale Out)

Adding more machines to a pool of resources.

**Pros:**
- Near-unlimited scaling potential
- Fault tolerance through redundancy
- Cost-effective with commodity hardware
- Can scale incrementally

**Cons:**
- Distributed systems complexity (network partitions, consistency)
- Requires stateless design or external state management
- Data partitioning and rebalancing overhead
- Harder to debug

**When to use:** Web servers, stateless application tiers, read-heavy workloads with replicas, systems that need high availability.

## Stateless vs. Stateful Services

| Aspect | Stateless | Stateful |
|--------|-----------|----------|
| Scaling | Easy — any instance handles any request | Hard — sessions are pinned to instances |
| Failure recovery | Instant — redirect to another instance | Complex — state must be recovered |
| Example | REST API server | WebSocket server with in-memory sessions |

**Key principle:** Push state out of application servers into dedicated stores (databases, caches, object storage) so the application tier can scale horizontally.

## Auto-Scaling

Automatically adjusting the number of instances based on load metrics.

**Common triggers:**
- CPU utilization > threshold
- Request queue depth
- Custom metrics (requests per second, latency percentiles)

**Scaling policies:**
- **Target tracking** — maintain a target metric value (e.g., keep average CPU at 60%)
- **Step scaling** — add/remove N instances when metric crosses thresholds
- **Scheduled scaling** — pre-scale for known traffic patterns (e.g., Black Friday)

**Cooldown periods** prevent thrashing — after a scaling action, wait before evaluating again.

## Database Scaling Strategies

| Strategy | How it works | Trade-off |
|----------|-------------|-----------|
| Read replicas | Write to primary, read from replicas | Replication lag (eventual consistency) |
| Sharding | Partition data across multiple databases | Cross-shard queries are expensive |
| Caching layer | Cache frequent reads in Redis/Memcached | Cache invalidation complexity |
| Connection pooling | Reuse DB connections across requests | Pool exhaustion under high concurrency |

## Key Interview Talking Points

- Start with vertical scaling, move to horizontal when you hit limits
- Stateless application tiers are a prerequisite for horizontal scaling
- Scaling reads is easier than scaling writes (replicas vs. sharding)
- Auto-scaling handles variable load but has lag — pre-scale for predictable spikes
- Every scaling strategy introduces trade-offs; name them explicitly
