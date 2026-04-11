# Databases

## SQL (Relational) Databases

Store data in tables with rows and columns. Enforce schemas and relationships through foreign keys.

### ACID Properties

| Property | Meaning | Guarantee |
|----------|---------|-----------|
| **Atomicity** | All operations in a transaction succeed or none do | No partial updates |
| **Consistency** | Transactions move the database from one valid state to another | Constraints are always satisfied |
| **Isolation** | Concurrent transactions don't interfere with each other | Serializable by default |
| **Durability** | Committed data survives crashes | Written to disk/WAL |

### Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|-----------|-------------------|--------------|-------------|
| **Read Uncommitted** | Possible | Possible | Possible | Fastest |
| **Read Committed** | Prevented | Possible | Possible | Fast |
| **Repeatable Read** | Prevented | Prevented | Possible | Moderate |
| **Serializable** | Prevented | Prevented | Prevented | Slowest |

Most production systems use **Read Committed** (PostgreSQL default) or **Repeatable Read** (MySQL InnoDB default).

### When to Choose SQL

- Data has clear relationships (users → orders → items)
- You need ACID transactions (financial data, inventory)
- You need complex queries with JOINs, aggregations, subqueries
- Schema is well-defined and relatively stable

### Popular SQL Databases

| Database | Strengths |
|----------|----------|
| **PostgreSQL** | Feature-rich, extensible, strong consistency, JSONB support |
| **MySQL** | High performance reads, wide ecosystem, mature replication |
| **Amazon Aurora** | Cloud-native, MySQL/PostgreSQL compatible, auto-scaling storage |
| **CockroachDB** | Distributed SQL, global transactions, automatic sharding |
| **Vitess** | MySQL sharding middleware (used by YouTube, Slack) |

## NoSQL Databases

### Key-Value Stores

Simple key → value mapping. Extremely fast for point lookups.

| Database | Key Feature |
|----------|-------------|
| **Redis** | In-memory, sub-ms latency, data structures (lists, sets, sorted sets) |
| **DynamoDB** | Managed, auto-scaling, single-digit ms latency at any scale |
| **Memcached** | In-memory, simple caching, multi-threaded |

**Use when:** Caching, session storage, rate limiting, leaderboards, feature flags.

### Document Stores

Store semi-structured data as JSON/BSON documents. Flexible schema.

| Database | Key Feature |
|----------|-------------|
| **MongoDB** | Flexible schema, rich queries, horizontal scaling |
| **Couchbase** | Built-in caching, mobile sync, SQL-like query language |

**Use when:** Content management, user profiles, catalogs — data that varies per record and doesn't need complex joins.

### Wide-Column Stores

Store data in column families. Optimized for writes and large-scale time-series data.

| Database | Key Feature |
|----------|-------------|
| **Cassandra** | High write throughput, linear scalability, tunable consistency |
| **HBase** | Hadoop integration, strong consistency, large sequential reads |
| **ScyllaDB** | Cassandra-compatible, C++ implementation, lower latency |

**Use when:** Time-series data, IoT, activity logs, messaging — write-heavy workloads at massive scale.

### Graph Databases

Store data as nodes and edges. Optimized for traversing relationships.

| Database | Key Feature |
|----------|-------------|
| **Neo4j** | Mature, Cypher query language, ACID transactions |
| **Amazon Neptune** | Managed, supports Gremlin and SPARQL |

**Use when:** Social networks, recommendation engines, fraud detection — problems where relationships are the primary query pattern.

## BASE Properties

The NoSQL counterpart to ACID:

| Property | Meaning |
|----------|---------|
| **Basically Available** | The system guarantees availability (may serve stale data) |
| **Soft state** | State may change over time even without input (due to eventual consistency) |
| **Eventually consistent** | The system will become consistent given enough time |

## Indexing Deep Dive

### B-Tree Index

The default index in most RDBMS. Balanced tree structure, O(log n) lookups.

- Good for: equality and range queries, sorting
- Structure: root → internal nodes → leaf nodes pointing to data rows

### LSM Tree (Log-Structured Merge Tree)

Used by Cassandra, RocksDB, LevelDB. Optimized for writes.

1. Writes go to an in-memory buffer (memtable)
2. When full, memtable is flushed to an immutable SSTable on disk
3. Background compaction merges SSTables

- **Write performance:** Excellent (sequential writes)
- **Read performance:** May need to check multiple SSTables (mitigated with bloom filters)

### Hash Index

O(1) exact-match lookups. No range query support. Used in memory-based systems.

### Composite Indexes

Index on multiple columns: `CREATE INDEX ON orders(user_id, created_at)`

**Leftmost prefix rule:** The index supports queries on:
- `user_id` alone
- `user_id` AND `created_at`
- But NOT `created_at` alone

## Choosing a Database

| Requirement | Recommended |
|-------------|-------------|
| Strong consistency, complex queries | PostgreSQL, MySQL |
| High write throughput, time-series | Cassandra, ScyllaDB |
| Flexible schema, documents | MongoDB |
| Sub-ms caching, counters | Redis |
| Managed, auto-scaling key-value | DynamoDB |
| Relationship traversal | Neo4j |
| Full-text search | Elasticsearch |
| Global distribution, SQL | CockroachDB, Spanner |

## Key Interview Talking Points

- Default to a relational database unless you have a specific reason for NoSQL
- The choice of SQL vs. NoSQL depends on the data model and access patterns, not scale alone
- ACID is not optional for financial or inventory systems
- Understand the indexing strategy — it's the #1 lever for query performance
- Mention that many real systems use **polyglot persistence** — different databases for different needs (e.g., PostgreSQL for core data + Redis for caching + Elasticsearch for search)
