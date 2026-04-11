# Back-of-the-Envelope Estimation Cheatsheet

Quick reference numbers for system design interviews. These are approximations — the goal is order-of-magnitude correctness, not precision.

## Powers of Two

| Power | Exact Value | Approximate | Unit |
|-------|------------|-------------|------|
| 2^10 | 1,024 | ~1 Thousand | 1 KB |
| 2^20 | 1,048,576 | ~1 Million | 1 MB |
| 2^30 | 1,073,741,824 | ~1 Billion | 1 GB |
| 2^40 | 1,099,511,627,776 | ~1 Trillion | 1 TB |
| 2^50 | | ~1 Quadrillion | 1 PB |

## Common Unit Conversions

| Unit | Equivalent |
|------|-----------|
| 1 byte | 8 bits |
| 1 KB | 1,000 bytes (or 1,024 exact) |
| 1 MB | 1,000 KB |
| 1 GB | 1,000 MB |
| 1 TB | 1,000 GB |
| 1 ASCII character | 1 byte |
| 1 Unicode character (UTF-8) | 1-4 bytes |

## Time Conversions

| Duration | Seconds |
|----------|---------|
| 1 minute | 60 |
| 1 hour | 3,600 |
| 1 day | 86,400 (~10^5) |
| 1 month | 2,592,000 (~2.5 × 10^6) |
| 1 year | 31,536,000 (~3 × 10^7) |

**Handy shortcut:** ~100K seconds/day, ~2.5M seconds/month, ~30M seconds/year.

## Latency Numbers

| Operation | Time |
|-----------|------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Main memory reference | 100 ns |
| Send 1 KB over 1 Gbps network | 10 μs |
| SSD random read | 150 μs |
| Read 1 MB from SSD | 1 ms |
| Datacenter round trip | 0.5 ms |
| Read 1 MB from HDD | 20 ms |
| HDD seek | 10 ms |
| Cross-continent round trip | 150 ms |

**Key ratios:** Memory is ~1000x faster than SSD. SSD is ~100x faster than HDD. Network round trips dominate in distributed systems.

## Throughput Numbers

| Component | Throughput |
|-----------|-----------|
| HDD sequential write | ~100 MB/s |
| SSD sequential write | ~500 MB/s - 3 GB/s |
| 1 Gbps network | 125 MB/s |
| 10 Gbps network | 1.25 GB/s |
| Redis (single instance) | ~100K ops/sec |
| Memcached (single instance) | ~200K ops/sec |
| PostgreSQL (single instance) | ~10K-50K QPS (read) |
| MySQL (single instance) | ~10K-50K QPS (read) |
| Kafka (per broker) | ~1M messages/sec |
| Nginx (static content) | ~100K RPS |
| Single web server (API) | ~1K-10K RPS |

## Typical Data Sizes

| Data | Size |
|------|------|
| UUID | 16 bytes (128 bits) |
| Snowflake ID | 8 bytes (64 bits) |
| MD5 hash | 16 bytes (128 bits) |
| SHA-256 hash | 32 bytes (256 bits) |
| IPv4 address | 4 bytes |
| IPv6 address | 16 bytes |
| Timestamp (epoch seconds) | 4 bytes (32-bit) or 8 bytes (64-bit) |
| Average tweet | ~300 bytes |
| Average email (text) | ~50 KB |
| Average web page | ~2 MB |
| Smartphone photo | ~3-5 MB |
| 1 minute of HD video | ~100-150 MB |
| 1 minute of 4K video | ~350-400 MB |

## Scale References

| Service | Approximate Scale |
|---------|------------------|
| Monthly active users (MAU) of major social platform | ~2-3 billion |
| Daily active users (DAU) | ~50-70% of MAU |
| Average tweets per day | ~500 million |
| Google searches per day | ~8.5 billion |
| YouTube video uploads per minute | ~500 hours |
| WhatsApp messages per day | ~100 billion |

## Estimation Formulas

### QPS (Queries Per Second)

```
QPS = Daily Active Users × Avg Queries per User / Seconds per Day

Example:
  100M DAU × 10 queries/user / 100K sec/day = 10,000 QPS

Peak QPS ≈ 2-3× average QPS
```

### Storage

```
Storage = Users × Data per User × Retention Period

Example: Store user profiles for 5 years
  500M users × 1 KB/profile × 5 years = 500M × 1KB × 5
  = 2.5 TB total (manageable on a single machine)

Example: Store messages for 5 years
  100M DAU × 50 messages/day × 300 bytes × 365 days × 5 years
  = 100M × 50 × 300 × 1825
  ≈ 2.7 PB (needs distributed storage)
```

### Bandwidth

```
Bandwidth = QPS × Average Request/Response Size

Example:
  10,000 QPS × 10 KB avg response = 100 MB/s ≈ 1 Gbps
```

### Number of Servers

```
Servers = Peak QPS / QPS per Server

Example:
  30,000 peak QPS / 5,000 QPS per server = 6 servers
  With redundancy: 6 × 1.5 = 9 servers
```

### Cache Size (80/20 Rule)

```
Cache size = 20% of daily read data

Example:
  10,000 read QPS × 1 KB × 86,400 sec/day = 864 GB/day
  Cache = 20% × 864 GB ≈ 170 GB
```

## Estimation Walkthrough Example

**Question:** Design a URL shortener that handles 100M new URLs per month.

**Writes:**
```
100M URLs/month ÷ 2.5M sec/month ≈ 40 writes/sec
```

**Reads (assume 100:1 read-to-write ratio):**
```
40 × 100 = 4,000 reads/sec
Peak: 4,000 × 3 = 12,000 reads/sec
```

**Storage (5-year retention):**
```
100M URLs/month × 12 months × 5 years = 6B URLs
Average URL: 100 bytes (short code + original URL)
6B × 100 bytes = 600 GB
```

**Bandwidth:**
```
Writes: 40/sec × 500 bytes = 20 KB/s (negligible)
Reads: 4,000/sec × 500 bytes = 2 MB/s (easily handled)
```

**Cache (80/20):**
```
4,000 reads/sec × 500 bytes × 86,400 sec = ~170 GB/day
Cache 20% = ~34 GB (fits in a single Redis instance)
```

## Key Interview Tips

- Round aggressively — 86,400 sec/day ≈ 100,000. Precision doesn't matter.
- Show your work — the process matters more than the answer
- Use the 80/20 rule for cache sizing (20% of data serves 80% of requests)
- Peak traffic ≈ 2-3× average traffic
- Always check if numbers are reasonable: a single server handles ~1K-10K RPS for API workloads
- State assumptions explicitly ("assuming 100:1 read-to-write ratio")
