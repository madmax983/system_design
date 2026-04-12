# API Design

## REST (Representational State Transfer)

The most common API style for web services. Resources are identified by URLs, manipulated with HTTP methods.

### Design Principles

1. **Resource-oriented URLs:** Nouns, not verbs
   - Good: `GET /users/123/orders`
   - Bad: `GET /getUserOrders?userId=123`

2. **Use HTTP methods correctly:**
   - `GET` — read (idempotent, cacheable)
   - `POST` — create
   - `PUT` — replace entirely (idempotent)
   - `PATCH` — partial update
   - `DELETE` — remove (idempotent)

3. **Use HTTP status codes properly:** 201 for created, 404 for not found, 429 for rate limited

4. **Stateless:** Each request contains all information needed to process it

### Pagination

| Strategy | Mechanism | Pros | Cons |
|----------|-----------|------|------|
| **Offset-based** | `?offset=20&limit=10` | Simple, supports jumping to page | Slow for large offsets; inconsistent with concurrent writes |
| **Cursor-based** | `?cursor=abc123&limit=10` | Consistent, performant | Can't jump to arbitrary page |
| **Keyset-based** | `?after_id=500&limit=10` | Efficient with indexed columns | Must sort by indexed column |

### Versioning

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URL path** | `/v1/users`, `/v2/users` | Explicit, easy to route | URL pollution |
| **Query parameter** | `/users?version=2` | Easy to default | Easy to forget |
| **Header** | `Accept: application/vnd.api.v2+json` | Clean URLs | Less discoverable |

**Best practice:** URL path versioning is the most common and explicit.

### HATEOAS

Hypermedia As The Engine Of Application State — responses include links to related resources and available actions.

```json
{
  "id": 123,
  "name": "Alice",
  "links": {
    "orders": "/users/123/orders",
    "profile": "/users/123/profile"
  }
}
```

Rarely implemented fully, but including "next page" links for pagination is common.

## GraphQL

A query language for APIs. The client specifies exactly what data it needs.

```graphql
query {
  user(id: 123) {
    name
    orders(last: 5) {
      total
      status
    }
  }
}
```

**Pros:**
- No over-fetching or under-fetching — client gets exactly what it requests
- Single endpoint, single round trip for complex queries
- Strong typing with a schema
- Great for mobile (minimize payload size)

**Cons:**
- Complex queries can be expensive (N+1 problem without DataLoader)
- Caching is harder (every query is different)
- Rate limiting is harder (query cost varies)
- Overkill for simple CRUD APIs

**When to use:** Multiple client types (web, mobile, third-party) with different data needs; complex data graphs.

## gRPC

A high-performance RPC framework using Protocol Buffers (protobuf) for serialization and HTTP/2 for transport.

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
}
```

**Pros:**
- Binary serialization (smaller payloads, faster parsing than JSON)
- HTTP/2 multiplexing and streaming
- Strong typing with code generation
- Bidirectional streaming

**Cons:**
- Not human-readable (binary format)
- Limited browser support (needs gRPC-Web proxy)
- Less tooling than REST

**When to use:** Internal service-to-service communication, high-throughput low-latency scenarios, streaming data.

## Comparison

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| Format | JSON | JSON | Protobuf (binary) |
| Transport | HTTP/1.1 or 2 | HTTP | HTTP/2 |
| Schema | OpenAPI (optional) | Required (SDL) | Required (proto) |
| Caching | Easy (HTTP caching) | Complex | Complex |
| Streaming | SSE, WebSocket | Subscriptions | Native bidirectional |
| Best for | Public APIs, CRUD | Flexible queries, mobile | Internal services, performance |

## API Gateway

A single entry point that sits in front of backend services. Handles cross-cutting concerns.

**Responsibilities:**
- **Routing:** Forward requests to the correct backend service
- **Authentication/Authorization:** Validate tokens, API keys
- **Rate limiting:** Enforce quotas per client
- **Request/Response transformation:** Convert between formats, aggregate responses
- **SSL termination:** Offload TLS from backends
- **Caching:** Cache responses at the edge
- **Monitoring:** Log requests, track metrics

**Examples:** AWS API Gateway, Kong, Nginx, Envoy.

**Trade-off:** Adds latency (one extra hop) and is a potential single point of failure. Mitigate with redundancy and caching.

## Idempotency in APIs

An operation is idempotent if calling it multiple times produces the same result as calling it once.

**Why it matters:** Network failures cause retries. Without idempotency, retries create duplicates.

**Implementation:**
- Client generates a unique `Idempotency-Key` header
- Server stores the key and result
- On retry with the same key, server returns the stored result without re-executing

```
POST /payments
Idempotency-Key: abc-123
{ "amount": 50.00 }

→ First call: process payment, store result with key abc-123
→ Retry:      return stored result for key abc-123
```

## Key Interview Talking Points

- REST for public APIs and standard CRUD; GraphQL for flexible queries; gRPC for internal high-performance communication
- Always discuss pagination for list endpoints — cursor-based is the most robust
- API gateways centralize cross-cutting concerns but add a hop — it's a trade-off
- Idempotency keys are essential for any API that handles money or state changes
- Version your APIs from day one — breaking changes are inevitable
