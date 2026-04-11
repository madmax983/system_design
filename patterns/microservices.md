# Microservices Patterns

## Monolith vs. Microservices

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| Deployment | Single unit | Independent per service |
| Scaling | Scale the whole app | Scale individual services |
| Technology | One stack | Polyglot (different languages/DBs per service) |
| Data | Shared database | Database per service |
| Complexity | In the codebase | In the infrastructure |
| Team structure | One team, one codebase | Small teams own individual services |

**Start with a monolith.** Extract microservices when you have a clear need: independent scaling, team autonomy, or different technology requirements.

## Service Decomposition

### How to Split

**By business domain (Domain-Driven Design):**
- Identify bounded contexts (e.g., Orders, Payments, Inventory, Users)
- Each bounded context becomes a service
- Services communicate through well-defined interfaces

**By subdomain:**
- **Core:** Your competitive advantage — invest heavily (e.g., recommendation engine)
- **Supporting:** Necessary but not differentiating (e.g., user management)
- **Generic:** Commodity (e.g., email sending) — buy or use SaaS

### Database Per Service

Each microservice owns its data. No direct database access between services.

**Pros:** Independent schema evolution, independent scaling, technology freedom.
**Cons:** Cross-service queries require API calls; distributed transactions are hard.

**Data sharing options:**
- API calls between services
- Event-driven data replication (service publishes events, others maintain local copies)
- Shared read-only views (materialized from events)

## Communication Patterns

### Synchronous

| Pattern | Mechanism | When to Use |
|---------|-----------|-------------|
| **Request/Response** | REST, gRPC | When the caller needs an immediate answer |
| **Service Mesh** | Sidecar proxy (Envoy/Istio) | When you need observability, retries, mTLS between services |

### Asynchronous

| Pattern | Mechanism | When to Use |
|---------|-----------|-------------|
| **Event-driven** | Kafka, SNS/SQS | When services need to react to state changes |
| **Command queue** | RabbitMQ, SQS | When you need to offload work for later processing |

**Prefer async when possible** — it reduces coupling and improves resilience (the caller doesn't block on the callee).

## Saga Pattern

Manages distributed transactions across multiple services without a global transaction coordinator.

### Choreography-Based Saga

Each service listens for events and publishes its own events.

```
OrderService: OrderCreated →
PaymentService: PaymentProcessed →
InventoryService: InventoryReserved →
ShippingService: ShipmentScheduled
```

**Compensating actions (rollback):**
```
ShippingService: ShipmentFailed →
InventoryService: InventoryReleased →
PaymentService: PaymentRefunded →
OrderService: OrderCancelled
```

**Pros:** No central coordinator, loosely coupled.
**Cons:** Hard to understand the overall flow, difficult to debug.

### Orchestration-Based Saga

A central orchestrator directs the workflow.

```
OrderOrchestrator:
  1. Tell PaymentService to charge
  2. Tell InventoryService to reserve
  3. Tell ShippingService to schedule
  If any step fails → execute compensating actions in reverse
```

**Pros:** Clear flow, easy to monitor and debug.
**Cons:** Orchestrator is a single point of failure; tighter coupling to the orchestrator.

## CQRS (Command Query Responsibility Segregation)

Separate the write model (commands) from the read model (queries).

```
Write path: Command → Command Handler → Write DB (normalized)
Read path:  Query → Query Handler → Read DB (denormalized, optimized for reads)
```

**Sync mechanism:** Events from the write side update the read side (eventually consistent).

**When to use:**
- Read and write workloads have very different characteristics
- You need different data models for reads vs. writes
- You need to scale reads and writes independently

**When NOT to use:** Simple CRUD applications where read and write models are the same.

## Event Sourcing

Store every state change as an immutable event instead of storing current state.

```
Events:
  1. AccountCreated { id: 123, name: "Alice" }
  2. MoneyDeposited { id: 123, amount: 100 }
  3. MoneyWithdrawn { id: 123, amount: 30 }

Current state (derived): { id: 123, name: "Alice", balance: 70 }
```

**Pros:**
- Complete audit trail
- Can reconstruct state at any point in time
- Natural fit with CQRS and event-driven architectures
- Easy to add new read models (replay events)

**Cons:**
- Event schema evolution is complex
- Rebuilding state from many events is slow (use snapshots)
- Steeper learning curve

## Service Discovery

How services find each other in a dynamic environment (instances come and go).

| Approach | How It Works | Example |
|----------|-------------|---------|
| **Client-side discovery** | Client queries a registry, picks an instance | Netflix Eureka |
| **Server-side discovery** | Load balancer queries registry, routes request | AWS ALB + ECS |
| **DNS-based** | Services register DNS records | Consul DNS, Kubernetes DNS |
| **Service mesh** | Sidecar proxy handles discovery transparently | Istio, Linkerd |

## Circuit Breaker

Prevents cascading failures when a downstream service is failing.

**States:**
1. **Closed:** Requests flow normally. Track failure rate.
2. **Open:** Failure rate exceeds threshold. All requests fail immediately (fast-fail). Start a timer.
3. **Half-Open:** Timer expires. Allow a few test requests. If they succeed → Closed. If they fail → Open.

```
Normal → failures increase → [Open] → timeout → [Half-Open] → success → [Closed]
                                                              → failure → [Open]
```

**Combine with:**
- **Retry with backoff:** Retry transient failures before tripping the circuit
- **Fallback:** Return cached/default data when the circuit is open
- **Bulkhead:** Isolate thread/connection pools per dependency

## Key Interview Talking Points

- Start with a monolith; extract microservices driven by concrete needs
- Database per service is the foundation — shared databases create tight coupling
- Sagas replace distributed transactions; always discuss compensating actions
- Prefer async communication to reduce coupling
- Circuit breakers prevent cascading failures — they're essential in microservice architectures
- CQRS and event sourcing are powerful but complex — use them when the problem demands it
