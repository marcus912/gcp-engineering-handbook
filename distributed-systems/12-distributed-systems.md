# Chapter 12: Distributed Systems Concepts

## 12.1 Why Distributed Systems?

```
Single Server:           Distributed System:
┌─────────┐              ┌─────────┐ ┌─────────┐ ┌─────────┐
│ Server  │              │ Server  │ │ Server  │ │ Server  │
│         │              │    A    │ │    B    │ │    C    │
└─────────┘              └─────────┘ └─────────┘ └─────────┘
     │                        │           │           │
Single point of              └───────────┴───────────┘
failure                              │
                              Shared state, coordination

Benefits: Scalability, Availability, Fault tolerance
Challenges: Complexity, Consistency, Partial failures
```

---

## 12.2 CAP Theorem

**You can only have 2 of 3**:

```
         Consistency
             △
            / \
           /   \
          /     \
         /   X   \
        /         \
       /───────────\
Availability ───── Partition Tolerance
```

**Consistency (C)**: All nodes see the same data at the same time

**Availability (A)**: Every request receives a response

**Partition Tolerance (P)**: System works despite network partitions

### In Practice

Since network partitions happen, you must choose:
- **CP**: Consistent + Partition Tolerant (sacrifice availability)
- **AP**: Available + Partition Tolerant (sacrifice consistency)

| System | Type | Example |
|--------|------|---------|
| Traditional RDBMS | CA | Single-node MySQL |
| Cloud Spanner | CP | Global transactions |
| Cassandra | AP | High availability NoSQL |
| DynamoDB | AP | Eventually consistent |

---

## 12.3 Consistency Models

```
Strongest                                          Weakest
    │                                                  │
    ▼                                                  ▼
Linearizable → Sequential → Causal → Eventual

Linearizable:
  All operations appear instantaneous
  All clients see same order

Sequential:
  Operations from one client maintain order
  Different clients may see different orders

Causal:
  Causally related operations maintain order
  Concurrent operations may be reordered

Eventual:
  If no new updates, eventually all clients see same value
  No ordering guarantees
```

### Eventual Consistency in Practice

```
Write: key = "value1"
       │
       ▼
  ┌─────────┐
  │ Server A │ ──replication──▶ Server B ──replication──▶ Server C
  │value1   │    (async)        │ value1     (async)       │ ???
  └─────────┘                   └──────────────────────────┘

Read from A: "value1" ✓
Read from B: "value1" ✓ (if replication complete)
Read from C: "old value" or "value1" (depends on timing)

Eventually, all servers have "value1"
```

---

## 12.4 Distributed Consensus

### The Problem

How do distributed nodes agree on a value?

```
Node A: "Let's write X"
Node B: "Let's write Y"
Node C: ???

Need: Agreement, Validity, Termination
```

### Raft Consensus (simplified)

```
1. Leader Election
   - Nodes start as followers
   - If no heartbeat, become candidate
   - Request votes from other nodes
   - Majority wins → becomes leader

2. Log Replication
   - Leader receives client requests
   - Appends to log, sends to followers
   - Followers append and acknowledge
   - Leader commits when majority confirm

3. Safety
   - Only nodes with up-to-date logs can become leader
   - Committed entries never lost
```

---

## 12.5 Replication Strategies

### Synchronous vs Asynchronous

```
Synchronous:
Client → Primary → Replicas (wait for all)
                 ↓
              Response

+ Strong consistency
- Higher latency
- Lower availability (all must be up)

Asynchronous:
Client → Primary → Response
              ↓
          Replicas (background)

+ Lower latency
+ Higher availability
- May lose recent writes on failure
- Consistency challenges
```

### Leader-Follower vs Multi-Leader

```
Leader-Follower:
┌────────┐     ┌────────────┐     ┌────────────┐
│ Leader │────▶│ Follower 1 │     │ Follower 2 │
│  (RW)  │────▶│   (Read)   │     │   (Read)   │
└────────┘     └────────────┘     └────────────┘

+ Simple
+ No conflicts
- Leader is bottleneck
- Failover complexity

Multi-Leader:
┌────────┐     ┌────────┐
│Leader A│◀───▶│Leader B│
│  (RW)  │     │  (RW)  │
└────────┘     └────────┘

+ Better write throughput
+ Cross-region writes
- Conflict resolution needed
- More complex
```

---

## 12.6 Partitioning (Sharding)

### Strategies

```
Range Partitioning:
Keys A-M → Shard 1
Keys N-Z → Shard 2

+ Range queries easy
- Hot spots (uneven distribution)

Hash Partitioning:
hash(key) % num_shards → Shard N

+ Even distribution
- Range queries difficult

Consistent Hashing:
           ┌────────────────┐
           │    Hash Ring   │
      Node A              Node B
           │      Keys     │
           │  distributed  │
      Node D              Node C
           └────────────────┘

+ Adding/removing nodes moves few keys
+ Better load distribution
```

### Cross-Shard Operations

```
Challenge: Transaction spanning multiple shards

Solutions:
1. Two-Phase Commit (2PC)
   - Prepare phase: All shards ready?
   - Commit phase: All shards commit
   - Blocking if coordinator fails

2. Saga Pattern
   - Series of local transactions
   - Compensating transactions for rollback
   - Eventually consistent

3. Avoid cross-shard transactions
   - Design data model to keep related data together
```

---

## 12.7 Common Patterns

### Circuit Breaker

```
         ┌──────────────────────────────────────────────┐
         │                  States                      │
         │                                              │
         │   CLOSED ────(failures)────▶ OPEN           │
         │     ▲                          │            │
         │     │                     (timeout)         │
         │     │                          │            │
         │  (success)                     ▼            │
         │     │                     HALF-OPEN         │
         │     └──────(test call)────────┘            │
         └──────────────────────────────────────────────┘

CLOSED: Normal operation, count failures
OPEN: Fail immediately, don't call downstream
HALF-OPEN: Allow one test call
```

### Retry with Exponential Backoff

```python
def retry_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception:
            if attempt == max_retries - 1:
                raise
            sleep_time = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(sleep_time)

# Attempts: 1s, 2s, 4s, 8s, 16s (with jitter)
```

### Idempotency

```
Idempotent: f(f(x)) = f(x)

Example - Non-idempotent:
POST /transfer {"amount": 100}
→ Retrying could transfer 200!

Example - Idempotent:
POST /transfer {"idempotency_key": "abc123", "amount": 100}
→ Server tracks key, ignores duplicates
```

### Bulkhead Pattern

```
┌─────────────────────────────────────────────────┐
│                  Application                     │
│  ┌─────────────┐  ┌─────────────┐              │
│  │ Thread Pool │  │ Thread Pool │              │
│  │  Service A  │  │  Service B  │              │
│  │   (10)      │  │   (10)      │              │
│  └─────────────┘  └─────────────┘              │
│                                                 │
│  If Service A is slow, only its pool exhausted │
│  Service B continues working                    │
└─────────────────────────────────────────────────┘
```

---

## 12.8 Caching Strategies

```
Cache-Aside (Lazy Loading):
1. Check cache
2. If miss, read from DB
3. Write to cache

+ Only requested data cached
- Cache miss penalty
- Data can become stale

Write-Through:
1. Write to cache
2. Cache writes to DB
3. Return

+ Cache always has latest
- Write latency higher

Write-Behind:
1. Write to cache
2. Return
3. Cache writes to DB (async)

+ Fast writes
- Data loss risk
- Complexity
```

### Cache Invalidation

```
"There are only two hard things in Computer Science:
 cache invalidation and naming things." - Phil Karlton

Strategies:
1. TTL (Time To Live)
   - Simple, eventual consistency

2. Event-based invalidation
   - Pub/Sub on data changes

3. Write-through/Write-behind
   - Cache always updated with writes
```

---

## 12.9 Message Queues

### Patterns

```
Point-to-Point:
Producer → Queue → Consumer
           │
           Only one consumer gets message

Pub/Sub:
Producer → Topic ─▶ Consumer A
                 ─▶ Consumer B
                 ─▶ Consumer C
           All subscribers get message
```

### Delivery Guarantees

| Guarantee | Description | Example |
|-----------|-------------|---------|
| **At-most-once** | May lose messages | UDP |
| **At-least-once** | May duplicate | Most queues |
| **Exactly-once** | No loss, no duplicates | Hard to achieve |

### GCP Messaging

```
Cloud Pub/Sub:
- Global, scalable pub/sub
- At-least-once delivery
- Ordering optional

Cloud Tasks:
- Task queues
- Scheduled execution
- Rate limiting
```

---

## 12.10 GCP Distributed Services

### Cloud Spanner

```
Global distributed SQL with strong consistency

How it achieves this:
1. TrueTime API
   - GPS + atomic clocks
   - Global time synchronization

2. Paxos consensus
   - Distributed agreement

3. Two-phase commit
   - Atomic transactions

Trade-off: Higher latency than eventually consistent systems
```

### Cloud Bigtable

```
Wide-column NoSQL, eventually consistent

Scaling:
- Data auto-sharded by row key
- Add nodes for more throughput
- Linear scaling

Design considerations:
- Row key design critical
- Avoid hotspots
```

---

## Chapter 12 Review Questions

1. Explain CAP theorem. Give examples of CP and AP systems.

2. What's the difference between strong and eventual consistency?

3. Why is the circuit breaker pattern useful?

4. How does Cloud Spanner achieve strong consistency globally?

5. Explain the difference between synchronous and asynchronous replication.

6. What is idempotency and why is it important in distributed systems?

---

## Key Takeaways

1. **Trade-offs are inevitable** - CAP theorem forces choices
2. **Eventual consistency is common** - Design for it
3. **Failures are normal** - Design for resilience
4. **Patterns help** - Circuit breaker, retry, bulkhead
5. **Idempotency is crucial** - Safe retries
6. **Test failure scenarios** - Chaos engineering

---

[Next Chapter: Go Programming Essentials →](../programming/13-go.md)
