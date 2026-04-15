# CAP Theorem — Architecture

## Overview

CAP Theorem states that a distributed data store can guarantee at most **two** of the following three properties simultaneously:

- **C**onsistency — every read receives the most recent write or an error
- **A**vailability — every request receives a (non-error) response, without guarantee it's the most recent write
- **P**artition Tolerance — the system continues to operate despite network partitions (messages dropped between nodes)

CAP is one of the most frequently misquoted concepts in system design. Understanding what it actually means — and where it's often misapplied — separates junior from senior candidates.

---

## 1. The Three Properties Defined Precisely

### C — Consistency (Linearizability)
In CAP, "Consistency" means **linearizability** — not the C in ACID. It means:
- After a write completes, all subsequent reads from any node return that value
- The system behaves as if there is a single copy of the data

```
Without CAP Consistency:
  Client writes X=5 to Node A
  Client reads X from Node B → gets X=3 (stale)   ← NOT consistent

With CAP Consistency:
  Client writes X=5 to Node A
  Client reads X from Node B → gets X=5 ✅ OR gets an error (not a stale response)
```

### A — Availability
Every request made to a non-failing node must return a response — and it must not be an error.

```
With Availability guaranteed:
  Client sends request to Node B (even during partition)
  Node B MUST respond: 200 OK with some data (even if stale)

Without Availability:
  Client sends request to Node B during partition
  Node B returns: 503 Service Unavailable (or times out)
  ← This is acceptable in a CP system
```

### P — Partition Tolerance
The system continues operating even if network messages between nodes are lost or delayed.

```
Network partition:
  Node A ←─✂─→ Node B    ← messages lost between them

Partition tolerant: both nodes continue operating despite the cut
Not partition tolerant: the system stops (shuts down) when it can't communicate
```

**The critical insight:** In a real distributed system, network partitions WILL happen. Therefore, **P is not optional** — you must tolerate partitions. This makes CAP really a choice between **C and A when a partition occurs**.

---

## 2. CP vs AP Systems — The Real Choice

Since P is required, the theorem reduces to: **when a partition happens, do you sacrifice C or A?**

### CP System — Sacrifice Availability for Consistency

During a partition, the system refuses to serve requests rather than risk returning stale data.

```
Normal operation:
  [ Node A ] ←──────→ [ Node B ]    All clients → consistent reads ✅

Partition:
  [ Node A ] ←──✂──→ [ Node B ]

CP behavior:
  Node A: continues serving (it's the primary/majority)
  Node B: returns ERROR or stops accepting writes ← sacrifices availability
  → Clients routed to Node B see errors until partition heals
  → No stale data ever served
```

**Use when:** Data correctness is more important than always being available. Financial transactions, inventory management, distributed locks.

**Examples:** HBase, ZooKeeper, etcd, Spanner, MongoDB (with majority write concern), PostgreSQL (sync replication)

### AP System — Sacrifice Consistency for Availability

During a partition, the system continues serving requests from all nodes, but some nodes may return stale data.

```
Partition:
  [ Node A ] ←──✂──→ [ Node B ]

AP behavior:
  Node A: serves requests (may have latest data)
  Node B: serves requests ← may return stale data
  → No errors, always responds
  → After partition heals: nodes reconcile (eventual consistency)
```

**Use when:** Always responding is more important than guaranteed freshness. Social feeds, DNS, product catalogs, shopping carts.

**Examples:** Cassandra, DynamoDB (default), Riak, CouchDB, DNS

---

## 3. CAP is a Spectrum, Not a Binary

The modern view (Eric Brewer, 2012) acknowledges that CAP oversimplifies:

1. **Partitions are rare** — in practice, you can optimize for both C and A during normal operation. Only when a partition occurs do you choose.

2. **Consistency is tunable** — Cassandra's consistency levels (ONE, QUORUM, ALL) let you slide from AP to CP per-operation.

3. **PACELC** is a more complete model (Daniel Abadi):
   - If Partition → choose between A and C (same as CAP)
   - Else (no partition) → choose between Latency and Consistency

```
PACELC captures the everyday trade-off:
  Not just "what happens during a disaster"
  But: "what do you optimize for normally?"

  Low latency AND high consistency → impossible: cross-node consensus takes time
  Low latency: accept eventual consistency (tolerate stale reads)
  High consistency: accept higher latency (wait for consensus)
```

---

## 4. Real-World System Classification

### CP Systems

| System       | Why CP                                               | What it sacrifices                           |
|--------------|------------------------------------------------------|----------------------------------------------|
| ZooKeeper    | Leader must have quorum; minority partitions go read-only | Availability on minority partition side  |
| etcd         | Raft consensus; minority nodes refuse writes         | Minority nodes unavailable for writes        |
| HBase        | Writes to single HMaster; region server with partition rejects writes | Availability during partition |
| PostgreSQL (sync) | Primary won't ack until replica confirms      | Write availability if sync replica is down   |
| Google Spanner | TrueTime + 2PC ensures global linearizability    | Higher write latency (cross-region consensus)|

### AP Systems

| System          | Why AP                                              | What it sacrifices                              |
|-----------------|-----------------------------------------------------|-------------------------------------------------|
| Cassandra       | Any node accepts writes (W=1); hinted handoff       | Consistency (may read stale data at low CL)     |
| DynamoDB        | Any AZ serves reads/writes by default               | Default eventual consistency for most reads     |
| Riak            | Leaderless; always responds                         | Conflicts reconciled via vector clocks post-partition |
| CouchDB         | Multi-master; always available                      | Conflicts must be resolved by application       |
| DNS             | Cached responses served even if authoritative is down | Stale records until TTL expires               |
| Caching (Redis) | Serves cached value if DB is unreachable            | May serve stale data                            |

### CA — "Consistent AND Available, gives up Partition Tolerance"

This category is frequently cited but almost never real in a distributed system. A single-node RDBMS (no replication) is "CA" — but it's not a distributed system. Any system that is NOT distributed (single node, single datacenter with no replication) doesn't need to tolerate partitions.

**Practical take:** Don't design a distributed system and label it "CA" — that means it will fail silently when a partition occurs.

---

## 5. Cassandra — Tunable from AP to CP

Cassandra is unique: it can behave as AP or CP depending on your configured consistency levels.

```
N = 3 replicas

Consistency ONE  (AP):
  Write: succeeds if 1 node acks           → available during partition, stale reads possible
  Read:  returns result from 1 node        → fast, but may be stale

Consistency QUORUM  (CP-like):
  Write: succeeds if 2 of 3 nodes ack      → if 2 nodes are on the partition-isolated side: FAIL
  Read:  returns result agreed by 2 nodes  → always the latest written value

Consistency ALL  (full CP):
  Write: succeeds only if all 3 ack        → any single node failure = write failure
  Read:  all 3 must agree                  → any single node failure = read failure
```

This tunability is why Cassandra is described as "AP by default, can be made CP per-operation."

---

## 6. CAP in Microservices

In a microservices architecture, each service has independent data — but the system as a whole experiences CAP properties too.

**Network partition between services:**
```
Order Service ←────✂────→ Inventory Service

CP approach (synchronous call):
  Order Service calls Inventory Service for stock check
  If partition: Order Service FAILS the order request
  → No orders placed on guesses, but orders may be blocked during partition

AP approach (asynchronous / eventual):
  Order Service accepts order immediately, emits event
  Inventory Service eventually processes the event
  If it turns out stock was 0: compensating transaction (cancel + refund)
  → Orders always accepted, correctness ensured eventually
```

**The tradeoff restated for microservices:**
- Synchronous cross-service calls = CP (fail if dependency unavailable)
- Async messaging (Kafka, queues) = AP (accept immediately, reconcile later)

---

## 7. Common Misconceptions to Correct in Interviews

| Misconception                                   | Correction                                                 |
|-------------------------------------------------|------------------------------------------------------------|
| "CAP's C is the same as ACID's C"               | No. CAP C = linearizability. ACID C = constraint invariants. |
| "You choose any 2 of 3"                         | P is required in distributed systems. Real choice: C vs A during partitions |
| "CA systems exist in distributed systems"       | A truly distributed CA system collapses during any partition |
| "Cassandra is pure AP"                          | Cassandra is tunable; QUORUM makes it CP-like              |
| "MongoDB is CP"                                 | MongoDB with w:majority is CP. With w:1, it's AP.          |
| "My single-datacenter system doesn't need CAP"  | Even same-DC has network partitions (switch failure, cable)  |
| "CAP means you can't have both C and A"         | Only during partitions. In normal operation, you can have both. |

---

## 8. Failure Modes Mapped to CAP Choices

| Scenario                                    | CP behavior                     | AP behavior                              |
|---------------------------------------------|---------------------------------|------------------------------------------|
| Network partition during peak traffic       | Partial outage (minority errors)| Full availability, some stale reads      |
| Replica falls behind (lag spike)            | Reads blocked or error          | Stale reads served from replica          |
| Primary failure                             | Writes blocked during election  | Writes accepted by any node immediately  |
| Conflicting writes from two partitioned nodes | Impossible (one side blocked)  | Both accepted; conflict resolved later   |
| Client sees stale data                      | Should never happen             | Expected; handled by application         |
