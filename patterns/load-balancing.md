# Load Balancing

Load balancing distributes incoming traffic across multiple servers to improve throughput, reduce latency, and ensure no single server becomes a bottleneck.

## Load Balancing Algorithms

### Static Algorithms

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| **Round Robin** | Requests distributed sequentially across servers | Homogeneous servers with similar request costs |
| **Weighted Round Robin** | Servers with higher weights get proportionally more traffic | Heterogeneous servers (different capacities) |
| **IP Hash** | Hash the client IP to determine the server | Session affinity without sticky sessions |
| **URL Hash** | Hash the request URL to determine the server | Maximizing cache hits on each server |

### Dynamic Algorithms

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| **Least Connections** | Route to the server with fewest active connections | Varying request durations |
| **Weighted Least Connections** | Like least connections but accounts for server capacity | Mixed server sizes with variable request costs |
| **Least Response Time** | Route to the server with lowest avg response time | Latency-sensitive workloads |
| **Random** | Pick a random server | Simple, statistically balanced at scale |
| **Power of Two Choices** | Pick two random servers, choose the one with fewer connections | Good balance with minimal coordination |

## Consistent Hashing

Standard hash-based routing breaks when servers are added or removed (all mappings change). Consistent hashing minimizes remapping.

**How it works:**
1. Servers and keys are mapped onto a hash ring (0 to 2^32)
2. A key is routed to the first server clockwise from its hash position
3. When a server is added/removed, only keys in the adjacent range are remapped

**Virtual nodes:** Each physical server gets multiple positions on the ring to ensure even distribution.

**Used in:** Memcached, Cassandra, DynamoDB, CDN routing.

## L4 vs. L7 Load Balancing

| Aspect | L4 (Transport) | L7 (Application) |
|--------|----------------|-------------------|
| Layer | TCP/UDP | HTTP/HTTPS/gRPC |
| Inspects | IP addresses, ports | URLs, headers, cookies, body |
| Speed | Faster (less processing) | Slower (full HTTP parsing) |
| Features | Basic routing | Content-based routing, SSL termination, header rewriting |
| Use case | High-throughput TCP traffic | Web applications, API routing |
| Example | AWS NLB | AWS ALB, Nginx, HAProxy (L7 mode) |

## Health Checks

Load balancers monitor server health to avoid routing traffic to failed instances.

| Type | Mechanism | Typical Interval |
|------|-----------|-----------------|
| **TCP check** | Can a TCP connection be established? | 5-10 sec |
| **HTTP check** | Does `/health` return 200? | 10-30 sec |
| **Deep check** | Does the app report all dependencies healthy? | 30-60 sec |

**Unhealthy threshold:** Mark a server as down after N consecutive failures (typically 2-3).
**Healthy threshold:** Mark a server as up after N consecutive successes.

## SSL/TLS Termination

The load balancer decrypts HTTPS traffic and forwards plain HTTP to backend servers.

**Pros:**
- Offloads CPU-intensive encryption from application servers
- Centralized certificate management
- Backend-to-backend traffic within a VPC can be unencrypted (lower latency)

**Cons:**
- Traffic between LB and backend is unencrypted (acceptable within a trusted network)
- LB becomes a potential bottleneck for TLS processing

## Global vs. Regional Load Balancing

| Level | Mechanism | Purpose |
|-------|-----------|---------|
| **Global (DNS-based)** | GeoDNS / Anycast | Route users to the nearest region |
| **Regional (L4/L7)** | Network or application LB | Distribute within a datacenter/region |
| **Service mesh** | Sidecar proxy (Envoy) | Load balance between microservices |

## Session Persistence (Sticky Sessions)

Route all requests from a user to the same backend server.

**Methods:**
- Cookie-based (LB inserts a cookie identifying the server)
- IP-based (hash client IP)
- Header-based (custom session ID)

**Trade-off:** Sticky sessions prevent even load distribution and complicate failover. **Prefer stateless backends with external session stores** (Redis).

## Key Interview Talking Points

- Round robin is a starting point; least connections handles variable-cost requests better
- Consistent hashing is essential for distributed caches and sharded databases
- L7 load balancing enables content-based routing (e.g., route `/api` to backend A, `/static` to backend B)
- Health checks prevent routing traffic to failed servers — always mention them
- Avoid sticky sessions — externalize state to Redis or a database
- In production, use multiple layers: DNS (global) → L4 (regional) → L7 (application)
