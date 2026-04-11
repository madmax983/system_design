# Message Queues & Async Processing

Message queues decouple producers (senders) from consumers (receivers), enabling asynchronous communication, load leveling, and fault tolerance.

## Core Concepts

**Producer** — sends messages to the queue.
**Consumer** — reads and processes messages from the queue.
**Broker** — the middleware that stores and routes messages (Kafka, RabbitMQ, SQS).

## Messaging Models

### Point-to-Point (Queue)

Each message is delivered to exactly one consumer. Multiple consumers compete for messages.

```
Producer → [Queue] → Consumer A (gets message 1)
                   → Consumer B (gets message 2)
                   → Consumer C (gets message 3)
```

**Use cases:** Task distribution, work queues, order processing.

### Publish-Subscribe (Pub/Sub)

Each message is delivered to all subscribers of a topic.

```
Producer → [Topic] → Subscriber A (gets all messages)
                   → Subscriber B (gets all messages)
                   → Subscriber C (gets all messages)
```

**Use cases:** Event notifications, fan-out (e.g., new user signup triggers email, analytics, welcome flow).

### Consumer Groups

A hybrid: messages are published to a topic, but within a consumer group, each message is delivered to only one consumer. Multiple consumer groups each get all messages.

```
Producer → [Topic] → Group 1: Consumer A (partition 0), Consumer B (partition 1)
                   → Group 2: Consumer X (all partitions)
```

**This is the Kafka model** — it combines pub/sub (across groups) with load balancing (within a group).

## Delivery Guarantees

| Guarantee | Meaning | Implementation |
|-----------|---------|----------------|
| **At-most-once** | Message may be lost, never duplicated | Don't retry failed deliveries |
| **At-least-once** | Message is never lost, may be duplicated | Retry with acknowledgment; consumer must be idempotent |
| **Exactly-once** | Message is delivered and processed exactly once | Requires idempotent writes + transactional processing |

**In practice:** At-least-once + idempotent consumers is the most common and practical approach. True exactly-once is very expensive.

### Idempotency

An operation is idempotent if performing it multiple times has the same effect as performing it once.

**Techniques:**
- Deduplicate using a unique message ID
- Use database upserts instead of inserts
- Track processed message IDs in a separate table

## Ordering

| Level | Guarantee | How |
|-------|-----------|-----|
| **No ordering** | Messages may arrive in any order | Default in most queue systems |
| **Partition ordering** | Messages with the same key are ordered within a partition | Kafka: messages with same key → same partition |
| **Total ordering** | All messages are globally ordered | Single partition (limits throughput) |

## Backpressure

When consumers can't keep up with producers:

| Strategy | How It Works |
|----------|-------------|
| **Buffering** | Queue absorbs the burst; consumers catch up later |
| **Dropping** | Discard messages when the queue is full (acceptable for metrics, logs) |
| **Rate limiting producers** | Reject or slow down producers when queue depth exceeds threshold |
| **Auto-scaling consumers** | Spin up more consumer instances based on queue depth |

## Dead Letter Queues (DLQ)

Messages that fail processing repeatedly are moved to a DLQ for investigation.

```
Main Queue → Consumer (fails) → Retry Queue → Consumer (fails again) → DLQ
```

**DLQ contents should be:**
- Monitored with alerts
- Investigated and reprocessed or discarded
- Never ignored

## Event-Driven Architecture

Systems communicate through events rather than direct calls.

**Event:** An immutable record of something that happened ("OrderPlaced", "PaymentProcessed").

### Event Sourcing

Store every state change as an event, rather than storing current state.

```
Events: [OrderCreated, ItemAdded, ItemAdded, OrderSubmitted, PaymentReceived]
Current state: derived by replaying events
```

**Pros:** Complete audit trail, temporal queries, easy debugging.
**Cons:** Complexity, event schema evolution, replay time for large histories.

### Choreography vs. Orchestration

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Choreography** | Services react to events independently | Loosely coupled, simple | Hard to track the overall flow |
| **Orchestration** | A central coordinator directs the workflow | Clear flow, easy to monitor | Coordinator is a SPOF; tighter coupling |

## Common Message Brokers

| Broker | Model | Ordering | Throughput | Key Feature |
|--------|-------|----------|------------|-------------|
| **Apache Kafka** | Log-based pub/sub | Per-partition | Very high (millions/sec) | Durable, replayable, streaming |
| **RabbitMQ** | AMQP, flexible routing | Per-queue | Moderate (tens of thousands/sec) | Flexible routing, priority queues |
| **Amazon SQS** | Managed queue | FIFO optional | High | Fully managed, no ops |
| **Amazon SNS** | Managed pub/sub | No | High | Fan-out, push-based |
| **Redis Streams** | Log-based | Per-stream | High | Low latency, built into Redis |

## Key Interview Talking Points

- Message queues decouple services — the producer doesn't need to know who consumes the message
- At-least-once + idempotent consumers is the pragmatic choice for most systems
- Kafka is the default for high-throughput event streaming; SQS/RabbitMQ for simpler task queues
- Always mention dead letter queues for handling poison messages
- Backpressure management is critical — queues are not infinite buffers
- Event sourcing is powerful but complex — use it when you need audit trails or temporal queries
