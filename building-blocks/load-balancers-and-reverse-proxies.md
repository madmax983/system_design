# Load Balancers & Reverse Proxies

## Reverse Proxy

A server that sits between clients and backend servers, forwarding client requests and returning server responses.

### Functions

| Function | Description |
|----------|-------------|
| **Load balancing** | Distribute traffic across backend servers |
| **SSL termination** | Decrypt HTTPS, forward HTTP to backends |
| **Compression** | Compress responses (gzip, Brotli) |
| **Caching** | Cache static content and API responses |
| **Security** | Hide backend topology, filter malicious requests |
| **Request routing** | Route based on URL path, headers, or cookies |

### Reverse Proxy vs. Forward Proxy

| Aspect | Forward Proxy | Reverse Proxy |
|--------|--------------|---------------|
| Sits in front of | Clients | Servers |
| Purpose | Client anonymity, content filtering | Load balancing, security, caching |
| Who configures it | Client | Server operator |
| Example | Corporate proxy, VPN | Nginx, HAProxy, AWS ALB |

## Load Balancer Types

### Layer 4 (Transport Layer)

Routes based on IP address and TCP/UDP port. Does not inspect request content.

```
Client → L4 LB → Server A (based on IP hash or round-robin)
```

**Pros:** Very fast (minimal processing), handles any TCP/UDP protocol.
**Cons:** Can't route based on URL, headers, or content.
**Examples:** AWS NLB, HAProxy (TCP mode), Linux IPVS.

### Layer 7 (Application Layer)

Routes based on HTTP content: URL path, headers, cookies, request body.

```
Client → L7 LB → /api/* → API Server Pool
                → /static/* → Static Server Pool
                → /ws/* → WebSocket Server Pool
```

**Pros:** Content-based routing, SSL termination, request manipulation, caching.
**Cons:** Higher latency (must parse HTTP), more resource-intensive.
**Examples:** AWS ALB, Nginx, HAProxy (HTTP mode), Envoy.

## High Availability for Load Balancers

The load balancer itself must not be a single point of failure.

### Active-Passive

Two LBs: one active, one standby. Heartbeat mechanism detects failure.

```
Active LB (handles traffic) ←heartbeat→ Passive LB (standby)
Active fails → Passive takes over (VIP floats to passive)
```

### Active-Active

Both LBs handle traffic simultaneously. DNS or an upstream LB distributes between them.

```
DNS → LB-1 (handles 50% traffic)
    → LB-2 (handles 50% traffic)
```

### Global Server Load Balancing (GSLB)

DNS-based routing across multiple regions, each with its own local load balancer.

```
User in Europe → DNS → European LB → European servers
User in Asia → DNS → Asian LB → Asian servers
```

## Software vs. Hardware Load Balancers

| Aspect | Software | Hardware |
|--------|----------|----------|
| Cost | Low (open source available) | High (specialized appliances) |
| Flexibility | Highly configurable | Vendor-dependent features |
| Performance | Good (handles millions of RPS) | Excellent (dedicated ASICs) |
| Scalability | Horizontal (add more instances) | Vertical (buy bigger box) |
| Examples | Nginx, HAProxy, Envoy | F5 BIG-IP, Citrix ADC |

**Modern default:** Software LBs (Nginx, Envoy) or cloud-managed (ALB, NLB).

## Connection Handling

### Connection Draining

When removing a server from the pool, allow existing connections to complete before stopping traffic.

```
1. Mark server as "draining"
2. Stop sending new connections
3. Wait for existing connections to finish (up to timeout)
4. Remove server
```

### Keep-Alive Connections

Reuse TCP connections between LB and backend to avoid the overhead of establishing new connections for every request.

### Health Checks

| Check Type | Mechanism | Speed |
|-----------|-----------|-------|
| **TCP** | Can we establish a connection? | Fast |
| **HTTP** | Does /health return 200? | Medium |
| **Custom** | Application-specific logic | Slow but thorough |

**Intervals:** Check every 5-30 seconds. Mark unhealthy after 2-3 consecutive failures. Mark healthy after 2-3 consecutive successes.

## Key Interview Talking Points

- L4 LB for raw performance and non-HTTP protocols; L7 LB for content-based routing and HTTP features
- Load balancers must be redundant (active-passive or active-active)
- Nginx and Envoy are the go-to software load balancers; cloud-managed LBs (ALB/NLB) reduce operational burden
- SSL termination at the LB simplifies certificate management and offloads crypto from backends
- Connection draining ensures graceful deployments without dropped requests
- Always mention health checks — they're how the LB knows which backends are alive
