# CAP Theorem & Consistency Models

## CAP Theorem

In a distributed system, you can only guarantee two of three properties simultaneously:

- **C — Consistency:** Every read receives the most recent write (or an error)
- **A — Availability:** Every request receives a (non-error) response, without guarantee it contains the most recent write
- **P — Partition Tolerance:** The system continues to operate despite network partitions between nodes

### The Real Trade-off

Network partitions **will** happen in distributed systems. So "P" is not optional — you're really choosing between:

- **CP (Consistency + Partition Tolerance):** During a partition, the system refuses requests to avoid stale reads. Example: ZooKeeper, HBase, MongoDB (with majority write concern).
- **AP (Availability + Partition Tolerance):** During a partition, the system serves requests but may return stale data. Example: Cassandra, DynamoDB, CouchDB.

When there is **no partition**, a system can be both consistent and available. The trade-off only kicks in during failures.

### PACELC Extension

A more nuanced view: if there's a **P**artition, choose **A** or **C**; **E**lse (normal operation), choose **L**atency or **C**onsistency.

| System | Partition: A or C | Else: L or C |
|--------|-------------------|--------------|
| DynamoDB | A | L |
| Cassandra | A | L |
| MongoDB | C | C |
| PostgreSQL | C | C |
| Cosmos DB | Configurable | Configurable |

## Consistency Models

### Strong Consistency
Every read returns the most recent write. Behaves as if there's a single copy of data.

- **Linearizability:** The strongest model. Operations appear to take effect at a single instant between invocation and response.
- **Serializability:** Transactions appear to execute one at a time, in some serial order.

**Cost:** Higher latency (consensus required), lower throughput, reduced availability during partitions.

### Eventual Consistency
If no new updates are made, all replicas will **eventually** converge to the same value. No guarantee on when.

**Suitable for:**
- Social media feeds (a few seconds of staleness is fine)
- DNS propagation
- Shopping cart "last-write-wins"

**Challenge:** Conflicts when concurrent writes occur on different replicas.

### Causal Consistency
Causally related operations are seen in the same order by all nodes. Concurrent (unrelated) operations may be seen in different orders.

- Stronger than eventual, weaker than strong
- Preserves "happens-before" relationships
- Example: if User A posts, then User B replies, every node sees the post before the reply

### Read-Your-Writes Consistency
A user always sees their own writes, even if they hit a different replica on the next read.

**Implementation:** Route reads to the same replica that handled the write, or use session tokens / version vectors.

### Monotonic Reads
Once a user reads a value, subsequent reads never return an older value.

**Implementation:** Pin a user to a replica, or track the last-seen version.

## Conflict Resolution Strategies

When replicas diverge, you need a strategy to reconcile:

| Strategy | How it works | Trade-off |
|----------|-------------|-----------|
| **Last-write-wins (LWW)** | Timestamp determines winner | Simple but loses data; clock skew issues |
| **Vector clocks** | Track causal history per replica | Detects conflicts but doesn't resolve them |
| **CRDTs** | Data structures that merge automatically | Limited to certain data types (counters, sets) |
| **Application-level merge** | Custom logic resolves conflicts | Most flexible but most complex |

## Quorum-Based Consistency

With **N** replicas, **W** write acknowledgments, **R** read acknowledgments:

- **W + R > N** → strong consistency (at least one node overlaps)
- **W + R <= N** → eventual consistency

Common configurations:
- **N=3, W=2, R=2** — strong consistency, tolerates 1 failure
- **N=3, W=1, R=1** — eventual consistency, highest availability and lowest latency
- **N=3, W=3, R=1** — fast reads, slower writes, strong consistency

## Key Interview Talking Points

- CAP is about trade-offs during partitions — when things are healthy you can have both C and A
- Most real systems are **tunable** — you pick consistency levels per operation
- Strong consistency is expensive — use it only where correctness requires it (bank transfers, inventory counts)
- Eventual consistency is fine for many features (timelines, analytics, recommendations)
- Always mention the specific consistency model you're choosing and **why**
