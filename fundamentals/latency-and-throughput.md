# Latency & Throughput

## Definitions

**Latency** — the time it takes for a single request to travel from the client to the server and back. Measured in milliseconds (ms).

**Throughput** — the number of requests (or amount of data) a system can handle per unit of time. Measured in requests per second (RPS), queries per second (QPS), or megabytes per second (MB/s).

**Bandwidth** — the maximum theoretical throughput of a network link.

These are related but distinct: a system can have low latency but low throughput (a single fast thread), or high throughput but high latency (batch processing).

## Latency Numbers Every Engineer Should Know

| Operation | Approximate Time |
|-----------|-----------------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Main memory reference | 100 ns |
| SSD random read | 150 μs |
| HDD random read | 10 ms |
| Send 1 KB over 1 Gbps network | 10 μs |
| Round trip within same datacenter | 0.5 ms |
| Round trip cross-continent | 150 ms |
| Read 1 MB sequentially from SSD | 1 ms |
| Read 1 MB sequentially from HDD | 20 ms |
| Read 1 MB sequentially from network (1 Gbps) | 10 ms |
| Disk seek | 10 ms |

**Key takeaway:** Memory is ~100x faster than SSD, SSD is ~100x faster than HDD, and network round-trips dominate latency in distributed systems.

## Measuring Latency

### Percentiles (Not Averages)

Averages hide outliers. Use percentiles:

| Metric | Meaning |
|--------|---------|
| **p50 (median)** | 50% of requests are faster than this |
| **p95** | 95% of requests are faster; 5% are slower |
| **p99** | 99% of requests are faster; the "tail" latency |
| **p99.9** | The worst 0.1% of requests |

**Why tail latency matters:** Your most active users (who make the most requests) are the most likely to experience p99 latency. In microservices, tail latency compounds — if a request fans out to 10 services, the slowest service determines overall latency.

### Tail Latency Amplification

If a single service has p99 = 100ms and a request fans out to N services in parallel:

- N=1 → 1% chance of >100ms
- N=10 → ~10% chance that at least one is >100ms
- N=50 → ~40% chance

**Mitigation:**
- Hedged requests — send duplicate requests, use the first response
- Speculative execution — retry after a timeout shorter than expected latency
- Partition data to reduce fan-out

## Throughput Analysis

### Little's Law

```
L = λ × W
```

- **L** = average number of concurrent requests in the system
- **λ** = arrival rate (requests per second)
- **W** = average time each request spends in the system

**Example:** If your server processes requests with average latency of 200ms and you have 100 concurrent connections, your throughput is:

```
λ = L / W = 100 / 0.2 = 500 RPS
```

### Bottleneck Analysis

Throughput is limited by the slowest component in the pipeline:

```
System throughput ≤ min(component throughputs)
```

Common bottlenecks:
1. **CPU-bound:** Encryption, compression, serialization, complex business logic
2. **I/O-bound:** Database queries, disk reads, network calls
3. **Memory-bound:** Large working sets, garbage collection pauses
4. **Network-bound:** Bandwidth saturation, connection limits

### Amdahl's Law

The speedup from parallelization is limited by the serial portion:

```
Speedup = 1 / (S + (1 - S) / N)
```

Where S = serial fraction, N = number of processors. If 10% of work is serial, max speedup with infinite cores is 10x.

## Optimization Strategies

| Strategy | Helps With | Example |
|----------|-----------|---------|
| Caching | Latency + throughput | Redis for hot data |
| Connection pooling | Throughput | Reuse DB connections |
| Async processing | Latency (perceived) | Queue work, return immediately |
| Batching | Throughput | Batch DB writes |
| Compression | Network latency | gzip HTTP responses |
| CDN | Latency | Serve static assets from edge |
| Indexing | Latency | Database indexes for frequent queries |
| Denormalization | Read latency | Pre-join data at write time |
| Sharding | Throughput | Distribute data across DBs |

## Key Interview Talking Points

- Always discuss latency in percentiles, not averages
- Tail latency matters — it affects your best customers and compounds across microservices
- Use Little's Law to estimate required concurrency for a target throughput
- Identify the bottleneck before optimizing — CPU, I/O, memory, or network
- Latency and throughput are often at odds — optimizing for one may hurt the other (e.g., batching improves throughput but increases individual request latency)
