# Message Brokers

Message brokers enable asynchronous communication between services. See [Message Queues & Async Processing](../patterns/message-queues.md) for patterns; this page covers the infrastructure itself.

## Apache Kafka

A distributed event streaming platform. Log-based architecture.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Topic** | A named log of events, partitioned across brokers |
| **Partition** | An ordered, append-only log; unit of parallelism |
| **Producer** | Writes events to topics |
| **Consumer** | Reads events from topics |
| **Consumer Group** | A set of consumers that share the work of reading a topic |
| **Offset** | The position of a consumer in a partition |
| **Broker** | A Kafka server that stores partitions |

### Architecture

```
Producer → Topic (3 partitions) → Consumer Group A (3 consumers, 1 per partition)
                                → Consumer Group B (2 consumers)
```

Each partition is read by exactly one consumer within a group. Multiple groups can independently consume the same topic.

### Key Properties

| Property | Detail |
|----------|--------|
| **Ordering** | Guaranteed within a partition (not across partitions) |
| **Retention** | Time-based (e.g., 7 days) or size-based; messages persist after consumption |
| **Replay** | Consumers can reset their offset to re-read messages |
| **Throughput** | Millions of messages/sec (sequential disk writes + zero-copy) |
| **Durability** | Configurable replication factor (typically 3) |

### When to Use Kafka

- High-throughput event streaming (clickstream, logs, metrics)
- Event sourcing and CQRS read model population
- Real-time data pipelines (ETL, change data capture)
- Decoupling microservices via event-driven architecture

### When NOT to Use Kafka

- Simple task queues (use SQS or RabbitMQ instead)
- Low-latency point-to-point messaging (Kafka has higher base latency)
- Small-scale applications (operational overhead not justified)

## RabbitMQ

A traditional message broker implementing AMQP (Advanced Message Queuing Protocol).

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Exchange** | Receives messages from producers and routes to queues |
| **Queue** | Stores messages until consumed |
| **Binding** | Rules that link exchanges to queues |
| **Consumer** | Reads and acknowledges messages from queues |

### Exchange Types

| Type | Routing Logic | Use Case |
|------|--------------|----------|
| **Direct** | Route by exact routing key match | Task distribution by type |
| **Fanout** | Broadcast to all bound queues | Notifications to all subscribers |
| **Topic** | Pattern matching on routing key (`*.error`, `order.#`) | Flexible pub/sub |
| **Headers** | Route by message headers | Complex routing rules |

### Key Properties

| Property | Detail |
|----------|--------|
| **Ordering** | Per-queue FIFO |
| **Acknowledgment** | Consumer must ACK; unACKed messages are redelivered |
| **Priority** | Supports priority queues |
| **TTL** | Per-message and per-queue TTL |
| **Dead Letter** | Built-in DLQ support |

### When to Use RabbitMQ

- Complex routing requirements (topic exchanges, header-based routing)
- Task queues with priority and TTL
- Low-latency messaging (lower base latency than Kafka)
- When you need per-message acknowledgment and redelivery

## Amazon SQS

A fully managed message queue service.

### Variants

| Variant | Ordering | Deduplication | Throughput |
|---------|----------|--------------|------------|
| **Standard** | Best-effort | At-least-once | Nearly unlimited |
| **FIFO** | Strict per message group | Exactly-once | 3,000 msg/sec (with batching) |

### Key Features

- **Visibility timeout:** Message is hidden from other consumers while being processed
- **Dead letter queue:** Failed messages automatically moved after N retries
- **Long polling:** Reduces empty responses and API costs
- **Delay queues:** Postpone delivery of new messages

### When to Use SQS

- AWS-native workloads needing a simple, zero-ops queue
- Decoupling services without Kafka's operational complexity
- Lambda integration (SQS triggers Lambda functions)

## Comparison

| Feature | Kafka | RabbitMQ | SQS |
|---------|-------|----------|-----|
| **Model** | Log-based pub/sub | Exchange + queue | Managed queue |
| **Ordering** | Per-partition | Per-queue | Best-effort (FIFO optional) |
| **Throughput** | Very high | Moderate | High |
| **Replay** | Yes (offset reset) | No (consumed = gone) | No |
| **Latency** | Low-medium | Low | Medium |
| **Operational burden** | High (manage cluster) | Medium | None (managed) |
| **Message retention** | Days to forever | Until consumed | Up to 14 days |
| **Best for** | Event streaming, logs | Task routing, RPC | Simple queues, AWS |

## Choosing a Message Broker

| Requirement | Recommendation |
|-------------|---------------|
| High-throughput event streaming | Kafka |
| Event replay / audit trail | Kafka |
| Complex routing rules | RabbitMQ |
| Priority queues | RabbitMQ |
| Zero operational overhead | SQS |
| AWS Lambda integration | SQS |
| Real-time analytics pipeline | Kafka |
| Simple background job queue | SQS or RabbitMQ |

## Key Interview Talking Points

- Kafka for event streaming and log-based architectures; RabbitMQ for complex routing; SQS for zero-ops simplicity
- Kafka retains messages after consumption (replayable); RabbitMQ and SQS delete after consumption
- Kafka ordering is per-partition — choose partition keys carefully
- SQS FIFO provides exactly-once + ordering at the cost of throughput
- In production, many systems use multiple brokers: Kafka for the event bus + SQS for individual task queues
