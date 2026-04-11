# Caches

Caching stores frequently accessed data in a faster storage layer to reduce latency and database load. This page covers cache infrastructure; see [Caching Patterns](../patterns/caching.md) for strategies.

## Cache Types

### Application-Level (In-Process) Cache

Data stored in the application's memory space (e.g., a HashMap, Guava Cache, Caffeine).

| Aspect | Detail |
|--------|--------|
| **Latency** | ~nanoseconds (no network hop) |
| **Scope** | Single process only |
| **Capacity** | Limited by process memory |
| **Consistency** | Inconsistent across instances |
| **Failure mode** | Lost on process restart |

**Use when:** Hot-path data that's read thousands of times per second and tolerance for per-instance inconsistency.

### Distributed Cache

A shared cache cluster accessible by all application instances.

| Aspect | Detail |
|--------|--------|
| **Latency** | ~1ms (network hop) |
| **Scope** | Shared across all instances |
| **Capacity** | Scales horizontally |
| **Consistency** | Single source of truth |
| **Failure mode** | Survives app restarts; lost on cache cluster failure |

### Multi-Tier Cache

Combine both for optimal performance:

```
Request → L1 (in-process) → L2 (distributed) → Database
```

L1 catches the hottest data with zero latency; L2 catches the rest with ~1ms latency; database is the fallback.

## Redis

The most popular distributed cache. An in-memory data structure store.

### Key Features

| Feature | Detail |
|---------|--------|
| **Data structures** | Strings, hashes, lists, sets, sorted sets, streams, bitmaps, HyperLogLog |
| **Persistence** | RDB snapshots, AOF (append-only file), or both |
| **Replication** | Async primary-replica replication |
| **Clustering** | Redis Cluster — automatic sharding across nodes |
| **Pub/Sub** | Built-in publish-subscribe messaging |
| **Lua scripting** | Atomic server-side operations |
| **TTL** | Per-key expiration |

### Redis Use Cases

| Use Case | Data Structure | Example |
|----------|---------------|---------|
| **Caching** | String (key-value) | Cache DB query results |
| **Session store** | Hash | `HSET session:abc user_id 123` |
| **Rate limiting** | String + INCR | `INCR rate:user:123` with TTL |
| **Leaderboard** | Sorted Set | `ZADD leaderboard 1000 "alice"` |
| **Distributed lock** | String + NX | `SET lock:order NX PX 30000` |
| **Queue** | List | `LPUSH queue task` / `BRPOP queue` |
| **Unique counting** | HyperLogLog | `PFADD visitors user123` |
| **Real-time analytics** | Sorted Set / Stream | Time-windowed aggregations |

### Redis Deployment Modes

| Mode | Description | Trade-off |
|------|-------------|-----------|
| **Standalone** | Single node | Simple, no HA |
| **Sentinel** | Primary + replicas + sentinel processes for failover | HA, automatic failover |
| **Cluster** | Data sharded across multiple primaries, each with replicas | Horizontal scaling + HA |

### Redis Limitations

- **Memory-bound:** All data must fit in RAM (or use Redis on Flash)
- **Single-threaded** for commands: one CPU core per instance (I/O threads in Redis 6+)
- **Async replication:** Potential data loss on primary failure
- **No built-in query language:** No JOINs, no ad-hoc queries

## Memcached

A simpler, multi-threaded in-memory cache.

| Aspect | Redis | Memcached |
|--------|-------|-----------|
| Data structures | Rich (lists, sets, sorted sets, etc.) | Simple key-value only |
| Persistence | Yes (RDB, AOF) | No |
| Replication | Built-in | Not built-in |
| Threading | Single-threaded (core) | Multi-threaded |
| Memory efficiency | Higher overhead per key | More memory-efficient for simple caching |
| Use case | Multi-purpose (cache, queue, lock, etc.) | Pure caching |

**When to choose Memcached:** Simple caching with high throughput needs and no requirement for persistence or data structures.

## Cache Sizing

### Estimate Memory Needs

```
Memory = number_of_items × average_item_size × overhead_factor
```

**Overhead factor:** ~1.5-2x for Redis (internal data structure overhead, fragmentation).

### Example

Caching 10M user profiles, 1 KB each:
```
10,000,000 × 1 KB × 1.5 = ~15 GB
```

A single Redis instance can handle this. For higher availability, use a cluster with replicas.

### Cache Hit Rate

Target: **>95% hit rate** for a well-tuned cache.

```
Hit rate = cache hits / (cache hits + cache misses)
```

Monitor hit rate continuously. Low hit rate means:
- Cache is too small (keys evicted too quickly)
- TTLs are too short
- Access patterns are not cache-friendly (high cardinality, random access)

## Key Interview Talking Points

- Redis is the default choice for distributed caching — mention specific data structures for your use case
- Multi-tier caching (in-process + distributed) gives the best latency profile
- Always size the cache based on working set size, not total data size
- Monitor cache hit rates — a miss-heavy cache adds latency without benefit
- Redis persistence (AOF + RDB) provides durability but don't treat Redis as a primary database
- For simple key-value caching at extreme throughput, Memcached may outperform Redis
