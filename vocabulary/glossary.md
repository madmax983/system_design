# System Design Glossary

A-to-Z reference of terms commonly used in system design interviews.

---

### A

**ACID** — Atomicity, Consistency, Isolation, Durability. Properties that guarantee reliable database transactions.

**API Gateway** — A single entry point for client requests that handles routing, authentication, rate limiting, and other cross-cutting concerns.

**Async Processing** — Handling work in the background (via queues or events) rather than synchronously in the request path.

**Auto-scaling** — Automatically adjusting the number of compute instances based on load metrics.

**Availability** — The proportion of time a system is operational. Measured in "nines" (e.g., 99.99% = four nines).

### B

**Backpressure** — A mechanism where a downstream system signals upstream to slow down when it can't keep up with the incoming rate.

**BASE** — Basically Available, Soft state, Eventually consistent. The NoSQL counterpart to ACID.

**Bloom Filter** — A probabilistic data structure that tests whether an element is a member of a set. Can have false positives but never false negatives.

**Blue-Green Deployment** — Running two identical production environments. Traffic is switched from blue (current) to green (new) for zero-downtime deploys.

**Bulkhead** — Isolating components so that a failure in one doesn't cascade to others. Named after ship hull compartments.

### C

**Cache Stampede** — When a popular cache entry expires and many concurrent requests hit the database simultaneously to reload it.

**Canary Deployment** — Gradually rolling out a change to a small percentage of users before full deployment.

**CAP Theorem** — In a distributed system, you can guarantee at most two of: Consistency, Availability, Partition tolerance.

**CDC (Change Data Capture)** — Tracking changes in a database and streaming them to other systems (e.g., updating a search index).

**Circuit Breaker** — A pattern that detects failures and prevents cascading failures by short-circuiting requests to a failing service.

**Consistent Hashing** — A hashing technique that minimizes key redistribution when nodes are added or removed.

**CQRS** — Command Query Responsibility Segregation. Separating the write model from the read model.

**CRDT** — Conflict-Free Replicated Data Type. Data structures that can be merged across replicas without coordination.

### D

**Data Lake** — A centralized repository for storing structured, semi-structured, and unstructured data at scale.

**Data Partition** — Splitting data across multiple storage nodes. Also called sharding.

**Dead Letter Queue (DLQ)** — A queue for messages that fail processing after multiple retries.

**Denormalization** — Storing redundant data to optimize read performance at the cost of write complexity.

**Distributed Lock** — A lock mechanism that works across multiple processes or servers.

**DNS** — Domain Name System. Translates domain names to IP addresses.

### E

**Edge Computing** — Processing data near the source (at the network edge) rather than in a centralized datacenter.

**Error Budget** — The allowed amount of unreliability (e.g., if SLO is 99.9%, the error budget is 0.1% downtime).

**Eventual Consistency** — A consistency model where all replicas will converge to the same value given enough time with no new updates.

**Event Sourcing** — Storing every state change as an immutable event, rather than storing current state.

### F

**Failover** — Switching to a backup system when the primary system fails.

**Fan-out** — Distributing a message or request to multiple recipients (e.g., fan-out on write for social feeds).

**Fencing Token** — A monotonically increasing token used with distributed locks to prevent stale lock holders from corrupting data.

**Follower / Replica** — A read-only copy of a database that receives updates from the leader/primary.

### G

**GeoDNS** — DNS routing that returns different IP addresses based on the geographic location of the requester.

**gRPC** — A high-performance RPC framework using Protocol Buffers and HTTP/2.

### H

**Heartbeat** — A periodic signal sent between systems to indicate they are alive and operational.

**Hedged Request** — Sending the same request to multiple replicas and using the first response. Reduces tail latency.

**Horizontal Scaling (Scale Out)** — Adding more machines to handle increased load.

**Hot Spot** — A single node or partition receiving a disproportionate amount of traffic.

### I

**Idempotency** — The property where performing an operation multiple times produces the same result as performing it once.

**Index** — A data structure that improves the speed of data retrieval at the cost of additional storage and slower writes.

**Inverted Index** — A mapping from content (words) to locations (documents). The core data structure behind full-text search.

### K

**Kafka** — A distributed event streaming platform. Log-based architecture with topics and partitions.

### L

**Latency** — The time between a request being sent and the response being received.

**Leader Election** — The process of selecting a single node to coordinate actions in a distributed system.

**Load Balancer** — A component that distributes incoming traffic across multiple servers.

**Long Polling** — A technique where the server holds a client's request open until new data is available, then responds.

**LSM Tree** — Log-Structured Merge Tree. A write-optimized data structure used in databases like Cassandra and RocksDB.

### M

**Materialized View** — A pre-computed query result stored as a table, updated periodically or on changes.

**Message Broker** — Middleware that translates messages between producers and consumers (e.g., Kafka, RabbitMQ, SQS).

**Microservices** — An architectural style where an application is composed of small, independently deployable services.

**Monolith** — An application deployed as a single unit, as opposed to microservices.

### N

**Network Partition** — A failure where nodes in a distributed system cannot communicate with each other.

### O

**Optimistic Locking** — Allowing concurrent access but checking for conflicts at commit time (using version numbers).

**Orchestration** — A centralized coordinator directs the workflow across services (contrast with choreography).

### P

**Pagination** — Breaking large result sets into pages. Common strategies: offset-based, cursor-based, keyset-based.

**Partition Tolerance** — The system continues to operate despite network partitions between nodes.

**Pessimistic Locking** — Acquiring a lock before accessing data to prevent concurrent modification.

**Pre-signed URL** — A time-limited URL that grants temporary access to a private resource (e.g., S3 upload/download).

**Pub/Sub** — Publish-Subscribe. A messaging pattern where publishers send messages to topics and subscribers receive them.

### Q

**Quorum** — The minimum number of nodes that must agree for a distributed operation to succeed. Typically N/2 + 1.

**QPS** — Queries Per Second. A measure of throughput.

### R

**Raft** — A consensus algorithm designed to be understandable. Used in etcd, CockroachDB.

**Rate Limiting** — Controlling the number of requests a client can make in a given time window.

**Read Replica** — A read-only copy of a database that handles read traffic to reduce load on the primary.

**Rebalancing** — Redistributing data across nodes after adding or removing a node from the cluster.

**Redundancy** — Duplicating components to ensure the system can survive individual failures.

**Replication** — Copying data across multiple nodes for durability, availability, and read scalability.

**Reverse Proxy** — A server that sits between clients and backends, forwarding requests and handling cross-cutting concerns.

**RPS** — Requests Per Second. A measure of throughput.

### S

**Saga** — A pattern for managing distributed transactions through a sequence of local transactions with compensating actions.

**Service Discovery** — The mechanism by which services find and communicate with each other in a dynamic environment.

**Service Mesh** — Infrastructure layer that handles service-to-service communication (observability, retries, mTLS). Examples: Istio, Linkerd.

**Sharding** — Splitting data across multiple databases so each holds a subset. Also called data partitioning.

**SLA** — Service Level Agreement. A contract defining availability and performance guarantees.

**SLI** — Service Level Indicator. A measured metric (e.g., p99 latency).

**SLO** — Service Level Objective. A target for an SLI (e.g., p99 latency < 200ms).

**Snowflake ID** — A distributed unique ID scheme using timestamp + machine ID + sequence number.

**Split Brain** — When a network partition causes two parts of a system to independently believe they are the primary.

**SSE (Server-Sent Events)** — A one-way protocol for servers to push updates to clients over HTTP.

**Sticky Session** — Routing all requests from the same client to the same server. Generally discouraged in favor of stateless design.

**Strong Consistency** — Every read returns the most recent write. Behaves as if there's a single copy of data.

### T

**Throughput** — The number of operations a system handles per unit of time.

**Token Bucket** — A rate limiting algorithm that allows bursts up to a bucket size, with tokens refilling at a steady rate.

**TTL (Time To Live)** — The lifespan of data in a cache or DNS record before it expires.

### V

**Vertical Scaling (Scale Up)** — Adding more resources (CPU, RAM) to a single machine.

**Virtual Node (vnode)** — Multiple hash ring positions assigned to a single physical node in consistent hashing to improve distribution.

### W

**WAL (Write-Ahead Log)** — A log where changes are recorded before being applied to the database, ensuring durability.

**WebSocket** — A protocol for full-duplex communication over a single TCP connection.

**Write-Ahead Log** — See WAL.

### Z

**Zero-Copy** — A technique where data is transferred without copying between kernel and user space, improving I/O performance.

**ZooKeeper** — A distributed coordination service used for configuration management, leader election, and distributed locking.
