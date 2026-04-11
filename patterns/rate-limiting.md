# Rate Limiting & Throttling

Rate limiting controls how many requests a client can make in a given time window. It protects services from abuse, ensures fair usage, and prevents cascading failures.

## Rate Limiting Algorithms

### Token Bucket

A bucket holds tokens. Each request consumes one token. Tokens are added at a fixed rate. If the bucket is empty, the request is rejected.

```
Bucket capacity: 10 tokens
Refill rate: 1 token/second

Time 0: 10 tokens (full)
Burst of 10 requests: 0 tokens remaining
1 second later: 1 token available
```

**Pros:** Allows bursts up to the bucket size; smooth rate limiting after burst.
**Cons:** Two parameters to tune (bucket size, refill rate).
**Used by:** AWS API Gateway, Stripe.

### Leaky Bucket

Requests enter a FIFO queue (bucket). The queue drains at a fixed rate. If the queue is full, new requests are rejected.

```
Queue capacity: 10
Drain rate: 1 request/second

Burst of 15 requests → 10 queued, 5 rejected
Processed at steady 1/sec rate
```

**Pros:** Perfectly smooth output rate; no bursts.
**Cons:** Bursts are queued (increased latency) or dropped; less flexible.
**Used by:** Network traffic shaping.

### Fixed Window Counter

Divide time into fixed windows (e.g., 1-minute intervals). Count requests per window. Reject if count exceeds the limit.

```
Window: 12:00:00 - 12:00:59 → limit 100
12:00:45: 95 requests so far → allowed
12:00:50: 5 more → at limit
12:01:00: counter resets to 0
```

**Pros:** Simple, low memory (one counter per window).
**Cons:** Burst at window boundary — a client can send 100 requests at 12:00:59 and 100 more at 12:01:00 (200 in 2 seconds).

### Sliding Window Log

Track the timestamp of every request. Count requests in the last N seconds. Reject if count exceeds the limit.

**Pros:** No boundary burst problem; precise.
**Cons:** High memory usage (storing every timestamp).

### Sliding Window Counter

A hybrid: combine the current window's count with a weighted portion of the previous window's count.

```
Previous window: 80 requests
Current window (30% through): 20 requests
Weighted count: 80 × 0.7 + 20 = 76
Limit: 100 → allowed
```

**Pros:** Low memory, smooth rate limiting, no boundary burst.
**Cons:** Approximate (but good enough in practice).

## Where to Rate Limit

| Layer | Tool | Granularity |
|-------|------|-------------|
| **Client** | Client-side throttling | Per-client (cooperative) |
| **CDN/Edge** | Cloudflare, AWS WAF | Per-IP, per-region |
| **API Gateway** | Kong, AWS API Gateway | Per-API key, per-route |
| **Application** | Middleware (e.g., Express rate limiter) | Per-user, per-endpoint |
| **Service** | In-process or distributed counter | Per-operation |

**Best practice:** Rate limit at multiple layers. Edge catches abuse early; application-level rate limiting handles per-user fairness.

## Distributed Rate Limiting

When running multiple application instances, a local counter isn't enough — you need a shared counter.

| Approach | How It Works | Trade-off |
|----------|-------------|-----------|
| **Redis counter** | Atomic INCR with TTL in Redis | Accurate, adds Redis dependency |
| **Redis + Lua script** | Atomic check-and-increment | Eliminates race conditions |
| **Local + sync** | Each instance tracks locally, periodically syncs | Less accurate, lower latency |
| **Token bucket in Redis** | Store token count and last refill time in Redis | Handles bursts well |

### Redis Sliding Window Example

```
MULTI
ZADD rate_limit:<user_id> <timestamp> <request_id>
ZREMRANGEBYSCORE rate_limit:<user_id> 0 <timestamp - window_size>
ZCARD rate_limit:<user_id>
EXEC
```

## Response to Rate-Limited Requests

**HTTP 429 Too Many Requests** with headers:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1672531260
```

**Options for rejected requests:**
- **Drop:** Return 429 immediately
- **Queue:** Accept but process later (for async workloads)
- **Degrade:** Serve a simplified/cached response

## Throttling Patterns

### Graceful Degradation
As load increases, progressively reduce functionality:
1. Full service
2. Disable non-essential features (recommendations, analytics)
3. Serve cached/stale data
4. Return maintenance page

### Priority-Based Throttling
Differentiate between request types:
- **Critical:** Payment processing — never throttle
- **Important:** User-facing reads — throttle last
- **Background:** Analytics, batch jobs — throttle first

## Key Interview Talking Points

- Token bucket is the most common algorithm — allows bursts, simple to implement
- Sliding window counter is the best balance of accuracy and memory
- Rate limit at multiple layers: edge, gateway, and application
- Distributed rate limiting requires a shared store (Redis is the go-to)
- Always return proper 429 responses with Retry-After headers
- Differentiate between rate limiting (protecting the system) and throttling (graceful degradation)
