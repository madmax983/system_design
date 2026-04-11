# Database Scaling

## Replication

Copying data across multiple database servers to improve read performance, availability, and durability.

### Single-Leader (Primary-Replica)

```
Writes → Primary → replicates to → Replica 1, Replica 2, ...
Reads  → Primary or any Replica
```

**Synchronous replication:** Primary waits for replica acknowledgment. Strong consistency, higher write latency.
**Asynchronous replication:** Primary doesn't wait. Lower write latency, potential data loss on primary failure.
**Semi-synchronous:** Wait for at least one replica. Balances durability and latency.

**Failover:** When the primary fails, promote a replica to primary.
- Automated failover can cause split-brain (two primaries) if not implemented carefully
- Data loss possible if the failed primary had un-replicated writes

### Multi-Leader

Multiple nodes accept writes. Each leader replicates to the others.

**Use cases:** Multi-datacenter writes, offline-capable applications (each device is a "leader").
**Challenge:** Write conflicts when two leaders modify the same data simultaneously. Requires conflict resolution (LWW, merge functions, CRDTs).

### Leaderless

All replicas accept reads and writes. Clients write to and read from multiple replicas (quorum-based).

**Examples:** Cassandra, DynamoDB.
**Consistency:** Tunable via quorum parameters (W + R > N for strong consistency).

## Partitioning (Sharding)

Splitting data across multiple databases so each holds a subset.

### Partitioning Strategies

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Range-based** | Partition by key range (e.g., A-M, N-Z) | Good for range queries | Hot spots if data is skewed |
| **Hash-based** | Hash the key, mod by number of shards | Even distribution | Range queries require scatter-gather |
| **Directory-based** | A lookup service maps keys to shards | Flexible, supports rebalancing | Lookup service is a SPOF and bottleneck |
| **Geographic** | Partition by region/country | Data locality, compliance | Cross-region queries are complex |

### Shard Key Selection

The shard key determines how data is distributed. A good shard key:

1. **Distributes evenly** — avoids hot shards
2. **Aligns with query patterns** — queries that access a single shard are fast
3. **Is immutable** — changing a shard key requires re-partitioning

**Examples:**
- User ID — good for user-scoped queries, even if user activity varies
- Timestamp — bad alone (all writes go to the latest shard); combine with another field
- Country code — works for geo-partitioning but may create uneven shards

### Rebalancing

When shards become uneven or you add/remove nodes:

- **Fixed number of partitions:** Create many more partitions than nodes; reassign partitions to new nodes
- **Dynamic partitioning:** Split large partitions, merge small ones
- **Consistent hashing:** Minimizes data movement when nodes change

### Cross-Shard Queries

Queries that span multiple shards are expensive:

- **Scatter-gather:** Send query to all shards, merge results. Latency = slowest shard.
- **Denormalization:** Duplicate data across shards to avoid cross-shard joins
- **Global indexes:** Maintain a secondary index that spans all shards (adds write overhead)

## Indexing

Indexes speed up reads at the cost of slower writes and additional storage.

| Index Type | Use Case | Notes |
|------------|----------|-------|
| **B-tree** | General-purpose, range queries | Default in most RDBMS |
| **Hash index** | Exact-match lookups | O(1) lookups, no range support |
| **Composite index** | Multi-column queries | Column order matters (leftmost prefix) |
| **Covering index** | Queries satisfied entirely by the index | Avoids table lookup |
| **Full-text index** | Text search | Inverted index structure |
| **Geospatial index** | Location-based queries | R-tree, geohash |

**Index trade-offs:** Every index slows writes (the index must be updated) and consumes storage. Only index columns that appear in WHERE, JOIN, or ORDER BY clauses.

## Read Optimization Patterns

| Pattern | Technique | Trade-off |
|---------|-----------|-----------|
| **Read replicas** | Route reads to replicas | Replication lag |
| **Caching** | Cache query results in Redis | Invalidation complexity |
| **Materialized views** | Pre-computed query results stored as tables | Storage cost, refresh lag |
| **Denormalization** | Store redundant data to avoid joins | Write complexity, data inconsistency risk |
| **CQRS** | Separate read/write models and stores | Architectural complexity |

## Write Optimization Patterns

| Pattern | Technique | Trade-off |
|---------|-----------|-----------|
| **Batching** | Group multiple writes into one | Higher per-write latency |
| **Write-ahead log (WAL)** | Append to log first, then apply | Recovery time on crash |
| **Sharding** | Distribute writes across shards | Cross-shard complexity |
| **Async writes** | Queue writes, apply later | Durability risk |
| **Append-only storage** | Never update in place, always append | Compaction needed |

## Key Interview Talking Points

- Start with replication for read scaling; move to sharding when write volume or data size demands it
- Sharding is a one-way door — it's hard to undo. Exhaust simpler options first
- The shard key choice is the most critical decision — it affects query patterns, data distribution, and rebalancing
- Cross-shard joins are expensive — design your schema so most queries hit a single shard
- Always mention the consistency trade-offs when discussing replication (sync vs. async)
- Indexes are not free — discuss the read/write trade-off
