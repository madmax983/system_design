# Consistency Patterns

Patterns for maintaining data consistency across distributed systems, especially when multiple services or databases need to agree on state.

## Two-Phase Commit (2PC)

A protocol for atomic distributed transactions. All participants either commit or abort together.

### Phases

**Phase 1 — Prepare:**
1. Coordinator sends "prepare" to all participants
2. Each participant executes the transaction locally (but doesn't commit)
3. Each participant votes "yes" (ready to commit) or "no" (abort)

**Phase 2 — Commit/Abort:**
- If all participants voted "yes" → Coordinator sends "commit"
- If any participant voted "no" → Coordinator sends "abort"

### Problems

| Problem | Description |
|---------|------------|
| **Blocking** | If the coordinator crashes after Phase 1, participants are stuck holding locks |
| **Latency** | Two round trips minimum; participants hold locks the entire time |
| **Single point of failure** | Coordinator failure blocks the entire transaction |
| **Not partition-tolerant** | Network partition between coordinator and participant causes indefinite blocking |

**Use when:** Strong consistency is absolutely required across a small number of participants (e.g., transferring between two databases in the same datacenter).

**Avoid when:** High-throughput systems, systems that span datacenters, or systems with more than a few participants.

## Three-Phase Commit (3PC)

Adds a "pre-commit" phase between prepare and commit to reduce blocking.

1. **Can-Commit:** Coordinator asks if participants can commit
2. **Pre-Commit:** If all say yes, coordinator sends pre-commit (participants know that everyone agreed)
3. **Do-Commit:** Coordinator sends final commit

**Improvement:** Participants can make a decision if the coordinator fails (they know everyone agreed in Phase 2).
**Limitation:** Still vulnerable to network partitions; rarely used in practice.

## Consensus Algorithms

Used to get a group of nodes to agree on a value, even if some nodes fail.

### Paxos

Theoretical foundation for consensus. Three roles: proposers, acceptors, learners.

1. **Prepare:** Proposer sends a proposal number to acceptors
2. **Promise:** Acceptors promise not to accept lower-numbered proposals
3. **Accept:** Proposer sends the value with its proposal number
4. **Accepted:** If a majority accepts, the value is chosen

**Complex to implement correctly.** Multi-Paxos extends this for a sequence of values.

### Raft

Designed to be more understandable than Paxos. Used in etcd, CockroachDB, TiKV.

**Key concepts:**
- **Leader election:** One node is elected leader; it handles all writes
- **Log replication:** Leader appends entries to its log and replicates to followers
- **Safety:** A committed entry is never lost (guaranteed by majority replication)

**Failure handling:**
- If the leader fails, a new leader is elected (via randomized timeouts)
- Followers with the most up-to-date logs are preferred

### ZAB (ZooKeeper Atomic Broadcast)

Used by Apache ZooKeeper. Similar to Raft but designed for primary-backup replication.

## Distributed Locking

Ensuring mutual exclusion across distributed systems.

### Redis-Based Lock (Redlock)

```
1. Acquire lock: SET lock_key unique_value NX PX 30000
2. Do work
3. Release lock: DELETE lock_key (only if value matches)
```

**Redlock (multi-node):**
1. Try to acquire the lock on N/2+1 Redis nodes
2. If acquired on a majority within a timeout → lock is held
3. If not → release all locks and retry

**Caveats:**
- Clock skew can cause issues
- GC pauses can cause a client to hold an expired lock
- **Fencing tokens** mitigate this: attach a monotonically increasing token to each lock; resources reject requests with older tokens

### ZooKeeper-Based Lock

1. Create an ephemeral sequential znode under `/locks/`
2. If your znode has the lowest sequence number → you hold the lock
3. Otherwise, watch the znode with the next-lowest sequence number
4. When it's deleted, re-check if you now have the lowest

**Pros:** Robust, handles client failures (ephemeral nodes auto-delete).
**Cons:** ZooKeeper dependency, higher latency than Redis.

## Conflict Resolution

When concurrent writes happen on different replicas:

### Last-Write-Wins (LWW)

The write with the latest timestamp wins. Simple but:
- Clock skew can cause "wrong" writes to win
- Silently discards conflicting writes

### Version Vectors

Each replica maintains a vector of version numbers. Detect conflicts by comparing vectors.

- `[A:2, B:1]` vs. `[A:1, B:2]` → conflict (neither dominates)
- `[A:2, B:1]` vs. `[A:1, B:1]` → first dominates (no conflict)

On conflict, present both versions to the application for resolution.

### CRDTs (Conflict-Free Replicated Data Types)

Data structures that can be merged automatically without conflicts.

| CRDT | Type | Use Case |
|------|------|----------|
| **G-Counter** | Grow-only counter | Like counts, view counts |
| **PN-Counter** | Positive-negative counter | Inventory counts |
| **G-Set** | Grow-only set | Tags, followers |
| **OR-Set** | Observed-remove set | Shopping carts |
| **LWW-Register** | Last-write-wins register | Simple key-value |

**Pros:** No coordination needed; always merge-able.
**Cons:** Limited to specific data types; can't model arbitrary business logic.

## Idempotent Operations

Ensuring that retries don't cause unintended side effects.

**Naturally idempotent:** `SET x = 5` (always produces the same result)
**Not naturally idempotent:** `INCREMENT x` (each retry adds 1)

**Making operations idempotent:**
1. **Idempotency keys:** Client sends a unique key; server deduplicates
2. **Conditional writes:** `UPDATE ... WHERE version = X` (optimistic locking)
3. **Deduplication table:** Store processed operation IDs

## Key Interview Talking Points

- 2PC provides strong consistency but is slow and fragile — avoid for high-throughput systems
- Sagas (from the microservices section) are the practical alternative to distributed transactions
- Raft is the consensus algorithm to mention — it's simpler to explain than Paxos
- Distributed locks are tricky — always mention fencing tokens to handle edge cases
- CRDTs enable conflict-free replication but are limited to specific data types
- Choose the consistency pattern based on your consistency requirements — not everything needs strong consistency
