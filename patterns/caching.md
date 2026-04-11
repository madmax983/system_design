# Caching Patterns

Caching stores copies of frequently accessed data in a faster storage layer to reduce latency and load on the origin.

## Cache Strategies

### Cache-Aside (Lazy Loading)

The application manages the cache explicitly.

```
Read:
1. Check cache
2. If hit → return cached data
3. If miss → read from DB → write to cache → return data

Write:
1. Write to DB
2. Invalidate (or update) cache entry
```

**Pros:** Only requested data is cached; cache failure doesn't break the system.
**Cons:** Cache miss penalty (three steps); data can become stale if the DB is updated outside the app.
**Use when:** Read-heavy workloads with unpredictable access patterns.

### Write-Through

Every write goes to both the cache and the DB synchronously.

```
Write:
1. Write to cache
2. Cache writes to DB
3. Return success
```

**Pros:** Cache is always consistent with DB; no stale reads.
**Cons:** Write latency increases (two writes); cache may store data that's never read.
**Use when:** Data consistency is critical and write latency is acceptable.

### Write-Behind (Write-Back)

Writes go to the cache immediately; the cache asynchronously flushes to the DB.

```
Write:
1. Write to cache → return success immediately
2. Cache batches and writes to DB asynchronously
```

**Pros:** Very low write latency; batching reduces DB load.
**Cons:** Risk of data loss if cache fails before flushing; complex to implement.
**Use when:** Write-heavy workloads where some data loss risk is acceptable (e.g., analytics, counters).

### Read-Through

Similar to cache-aside, but the cache itself is responsible for loading data from the DB on a miss.

```
Read:
1. Application always reads from cache
2. On miss, cache loads data from DB transparently
```

**Pros:** Simpler application code; cache logic is centralized.
**Cons:** First request for each key is slow; less control over what gets cached.

### Refresh-Ahead

The cache proactively refreshes entries before they expire, based on predicted access patterns.

**Pros:** Eliminates cache miss latency for frequently accessed data.
**Cons:** Wastes resources refreshing data that may not be read again.

## Cache Eviction Policies

| Policy | How It Works | Best For |
|--------|-------------|----------|
| **LRU (Least Recently Used)** | Evict the entry not accessed for the longest time | General purpose, most common |
| **LFU (Least Frequently Used)** | Evict the entry accessed the fewest times | Workloads with stable hot sets |
| **FIFO (First In, First Out)** | Evict the oldest entry | Simple, predictable eviction |
| **TTL (Time To Live)** | Evict entries after a fixed time | Data with known staleness tolerance |
| **Random** | Evict a random entry | When access patterns are uniform |

**In practice:** Use TTL + LRU together. TTL bounds staleness; LRU manages memory pressure.

## Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

| Strategy | How It Works | Trade-off |
|----------|-------------|-----------|
| **TTL expiration** | Data auto-expires after N seconds | Simple but stale during TTL window |
| **Event-driven invalidation** | Publish invalidation events on writes | Real-time but adds infrastructure complexity |
| **Write-through invalidation** | Invalidate on every write | Consistent but high cache churn |
| **Version-based** | Store version number; check on read | Flexible but adds read-time logic |

## Cache Topologies

### Local (In-Process) Cache
Data lives in application memory (e.g., a HashMap, Guava cache).

- Ultra-fast (no network hop)
- Limited to a single instance
- Inconsistency across instances in a cluster

### Distributed Cache
A shared cache cluster accessible by all application instances (e.g., Redis, Memcached).

- Consistent view across all instances
- Survives application restarts
- Network hop adds latency (~1ms)

### Multi-Tier Cache
Local cache → Distributed cache → Database. Each layer catches misses from the layer above.

## Cache Stampede (Thundering Herd)

When a popular cache entry expires, many requests simultaneously hit the DB to reload it.

**Mitigations:**
- **Locking:** Only one request reloads; others wait for the lock
- **Early expiration (jitter):** Add random jitter to TTLs so entries don't expire at the same time
- **Stale-while-revalidate:** Serve stale data while one background request refreshes

## Key Interview Talking Points

- Cache-aside is the most common and flexible strategy
- Write-through gives consistency; write-behind gives performance — pick based on requirements
- Always discuss cache invalidation — it's where bugs hide
- Distributed caches (Redis) are the default choice; mention local caches for hot-path optimization
- Cache stampede is a real production problem — mention mitigation strategies
- Set TTLs deliberately: too short → no benefit; too long → stale data
