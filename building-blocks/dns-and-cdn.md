# DNS & CDN

## DNS (Domain Name System)

DNS translates domain names into IP addresses. It's the first step in every request and a critical building block in system design.

### Architecture

```
Client → Recursive Resolver → Root NS → TLD NS (.com) → Authoritative NS → IP Address
```

Each layer caches results based on TTL (Time To Live).

### DNS in System Design

| Use Case | How It Works |
|----------|-------------|
| **Load distribution** | Return multiple IP addresses (round-robin DNS) |
| **Geographic routing** | GeoDNS returns IPs for the nearest datacenter |
| **Failover** | Health-checked DNS removes unhealthy endpoints |
| **Blue-green deployments** | Switch DNS to point to the new environment |
| **Canary deployments** | Weighted DNS routes a percentage of traffic to the new version |

### DNS Trade-offs

**TTL considerations:**
- Low TTL (30-60s): Fast failover, but more DNS queries (increased latency and load on nameservers)
- High TTL (hours/days): Fewer queries, but slow propagation of changes

**Limitations:**
- Not suitable for real-time failover (TTL caching delays propagation)
- Limited health checking compared to load balancers
- Clients may cache beyond TTL

### Managed DNS Services
AWS Route 53, Cloudflare DNS, Google Cloud DNS — provide health checks, GeoDNS, weighted routing, and latency-based routing.

## CDN (Content Delivery Network)

A globally distributed network of edge servers that cache content close to users, reducing latency and offloading origin servers.

### How CDNs Work

```
User in Tokyo → Edge server in Tokyo (cache hit) → return content (5ms)
User in Tokyo → Edge server in Tokyo (cache miss) → Origin in US (200ms) → cache + return
```

### CDN Content Types

| Type | Examples | Caching |
|------|---------|---------|
| **Static** | Images, CSS, JS, fonts, videos | Long TTL (hours to days) |
| **Dynamic** | API responses, personalized content | Short TTL or no-cache |
| **Streaming** | Live video, audio | Chunked delivery from edge |

### Push vs. Pull CDN

| Model | How It Works | Best For |
|-------|-------------|----------|
| **Pull (origin-pull)** | CDN fetches from origin on first request, then caches | Most websites; content accessed unevenly |
| **Push (origin-push)** | You upload content to CDN proactively | Known content (software releases, videos); predictable access |

### Cache Invalidation

| Method | How It Works |
|--------|-------------|
| **TTL expiration** | Content auto-expires after a set time |
| **Purge** | Explicitly remove specific URLs from the cache |
| **Versioned URLs** | `style.v2.css` or `style.css?v=abc123` — new URL = new cache entry |
| **Stale-while-revalidate** | Serve stale content while fetching fresh content in the background |

**Best practice:** Use versioned URLs for static assets (infinite TTL, instant updates on deploy).

### CDN Architecture Considerations

**Multi-tier caching:**
```
User → Edge POP → Regional Shield → Origin
```
Shield nodes reduce origin load by aggregating cache misses from multiple edge nodes.

**Origin failover:** CDNs can route to a backup origin if the primary is down.

**DDoS protection:** CDNs absorb volumetric attacks at the edge, protecting the origin.

### Major CDN Providers

| Provider | Key Features |
|----------|-------------|
| **CloudFront** | Deep AWS integration, Lambda@Edge for edge compute |
| **Cloudflare** | Large network, DDoS protection, Workers for edge compute |
| **Akamai** | Largest network, enterprise-focused |
| **Fastly** | Real-time purging, Compute@Edge |

## Key Interview Talking Points

- DNS is the entry point for all traffic — mention it when discussing global load balancing
- CDNs reduce latency for static content from 200ms+ to single-digit ms
- Use versioned URLs for cache-busting static assets (avoids TTL-based invalidation complexity)
- CDNs are also a DDoS mitigation layer — absorb attacks at the edge
- Pull CDNs are simpler and work for most use cases; push CDNs for known, large content
- CDN + DNS together enable global traffic management: DNS routes to the nearest edge, CDN serves cached content
