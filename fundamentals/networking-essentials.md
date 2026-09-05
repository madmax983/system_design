# Networking Essentials

## The Network Stack (Simplified)

| Layer | Protocol | System Design Relevance |
|-------|----------|------------------------|
| Application | HTTP, gRPC, WebSocket, DNS | API design, data format |
| Transport | TCP, UDP | Reliability vs. speed trade-off |
| Network | IP | Routing, addressing |
| Link | Ethernet, Wi-Fi | Rarely discussed in interviews |

## TCP vs. UDP

| Aspect | TCP | UDP |
|--------|-----|-----|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery, ordering, retransmission | Best-effort, no guarantees |
| Overhead | Higher (headers, ACKs, flow control) | Lower |
| Latency | Higher (handshake + retransmissions) | Lower |
| Use cases | HTTP, database connections, file transfer | Video streaming, DNS lookups, gaming |

## DNS (Domain Name System)

Translates human-readable domain names to IP addresses.

### Resolution Flow
1. Browser cache → OS cache → Resolver cache
2. Recursive resolver → Root nameserver → TLD nameserver → Authoritative nameserver
3. Result is cached at each level according to TTL

### Record Types

| Type | Purpose | Example |
|------|---------|---------|
| **A** | Domain → IPv4 address | `example.com → 93.184.216.34` |
| **AAAA** | Domain → IPv6 address | `example.com → 2606:2800:220:1:...` |
| **CNAME** | Alias to another domain | `www.example.com → example.com` |
| **MX** | Mail server | `example.com → mail.example.com` |
| **NS** | Nameserver delegation | `example.com → ns1.provider.com` |
| **TXT** | Arbitrary text (verification, SPF) | `v=spf1 include:...` |
| **SRV** | Service discovery | `_sip._tcp.example.com → ...` |

### DNS in System Design
- **Global load balancing:** DNS can route users to the nearest datacenter (GeoDNS)
- **Failover:** Update DNS records to point away from failed endpoints
- **Limitation:** TTL-based caching means changes propagate slowly (minutes to hours)

## HTTP/HTTPS

### HTTP Methods

| Method | Idempotent | Safe | Use |
|--------|-----------|------|-----|
| GET | Yes | Yes | Retrieve a resource |
| POST | No | No | Create a resource / trigger an action |
| PUT | Yes | No | Replace a resource entirely |
| PATCH | No* | No | Partial update |
| DELETE | Yes | No | Remove a resource |

*PATCH can be idempotent depending on implementation.

### HTTP Status Codes

| Range | Meaning | Key Codes |
|-------|---------|-----------|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Permanent, 302 Found, 304 Not Modified |
| 4xx | Client error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Server error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

### HTTP/1.1 vs. HTTP/2 vs. HTTP/3

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Multiplexing | No (one req/connection) | Yes (streams over one TCP) | Yes (streams over QUIC) |
| Header compression | No | HPACK | QPACK |
| Transport | TCP | TCP | QUIC (UDP-based) |
| Head-of-line blocking | Connection level | TCP level | None |
| Server push | No | Yes (limited adoption) | Limited / often disabled in clients |

### TLS Handshake

1. **Client Hello** — supported cipher suites, TLS version
2. **Server Hello** — chosen cipher suite, server certificate
3. **Key Exchange** — establish shared secret (Diffie-Hellman)
4. **Finished** — both sides verify, encrypted communication begins

**Latency cost:** 1-2 round trips. TLS 1.3 reduces this to 1 round trip (0-RTT resumption for repeat connections).

## WebSockets

Full-duplex communication channel over a single TCP connection.

**Lifecycle:**
1. Client sends HTTP upgrade request
2. Server responds with `101 Switching Protocols`
3. Both sides can send messages at any time
4. Either side can close the connection

**When to use:** Chat, live notifications, collaborative editing, real-time dashboards.

**When NOT to use:** Standard request/response patterns, infrequent updates (use SSE or long polling instead).

## Long Polling vs. SSE vs. WebSockets

| Technique | Direction | Connection | Complexity | Use Case |
|-----------|-----------|------------|------------|----------|
| **Short polling** | Client → Server | New connection each poll | Low | Simple status checks |
| **Long polling** | Client → Server | Held open until data ready | Medium | Notifications with broad compatibility |
| **SSE (Server-Sent Events)** | Server → Client | Persistent, one-way | Low | Live feeds, stock tickers |
| **WebSocket** | Bidirectional | Persistent, full-duplex | High | Chat, gaming, collaboration |

## Key Interview Talking Points

- TCP guarantees delivery and ordering; UDP trades reliability for speed
- DNS is the first step in every request — its caching behavior affects failover speed
- HTTP/2 multiplexing reduces the need for domain sharding and many connection-pooling workarounds
- TLS adds 1-2 round trips; TLS 1.3 reduces this to 1 (or 0 for resumption)
- Choose WebSockets only when you need true bidirectional communication; SSE is simpler for server-to-client pushes
