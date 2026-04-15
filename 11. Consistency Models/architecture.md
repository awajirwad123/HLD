# Consistency Models — Architecture

## Overview

A consistency model is a **contract between a distributed system and its clients** that defines what values a read operation is allowed to return. Different models provide different guarantees about what you can see after a write, and each carries a different cost in latency, availability, or implementation complexity.

Understanding these models precisely is what separates junior candidates who say "eventual consistency" as a magic answer from senior engineers who can articulate exactly which anomalies each model prevents and at what cost.

---

## 1. The Consistency Spectrum

From strongest to weakest:

```
 STRONG ──────────────────────────────────────────────────────► WEAK

Linearizability  →  Sequential  →  Causal  →  Read-Your-Writes  →  Monotonic Reads  →  Eventual
    │                    │             │               │                    │                │
Highest cost         Lower cost    Practical      Per-session           Per-session      Cheapest
Zero anomalies    Small anomaly   No causal     No own stale         No time-travel    Stale ok
                    possible      anomalies       data seen             allowed
```

---

## 2. Strong Consistency (Linearizability)

**Definition:** After a write completes, all subsequent reads from any client, on any node, return that written value. The system appears as if it has a single copy of the data.

**Formally:** Operations appear to take effect instantaneously at a single point in real time between their invocation and their completion. All observers agree on the ordering of all operations.

```
Timeline:
  t=0ms:  Write X=5 starts
  t=5ms:  Write X=5 completes (acknowledged)
  t=6ms:  Any client reading X from any node → MUST see X=5
          (No node may return X=3 or "not found" after t=5ms)
```

**Anomalies prevented:** All of them. No stale reads, no causal violations, no write conflicts visible to readers.

**Cost:** Every read must verify it has the latest value — this requires coordination across replicas (consensus or synchronous replication). This adds latency equal to the network RTT between nodes.

**Implemented by:**
- PostgreSQL (sync replication + read from primary)
- Spanner (TrueTime)
- ZooKeeper, etcd (Raft consensus — all reads linearizable by default)
- Redis (single-node, no replication delay)
- DynamoDB (strongly consistent reads, 2× RCU cost)

---

## 3. Sequential Consistency

**Definition:** All operations appear to execute in some sequential order, and every client sees the same sequential order — but that order does not have to correspond to real-time ordering.

```
Two clients:
  Client A:  write X=1 at t=0  →  write X=2 at t=5
  Client B:  reads at t=3 and t=7

Sequential consistency ALLOWS:
  Client B at t=3: sees X=1 (the earlier write) ← fine
  Client B at t=7: sees X=2 ← fine
  Both clients agree on the order: X=1 happened before X=2

Sequential consistency DISALLOWS:
  Client B at t=7: sees X=1 (reverting back after already seeing X=2) ← violation
                   ← This would be a "time travel" anomaly
```

**Difference from linearizability:** Sequential consistency does not require the agreed-upon order to match real-world clock time. If Client A writes X=1 at t=0 and X=2 at t=10, clients might see X=2 "appear" before X=1 logically — as long as all clients see the same order.

**Implemented by:** Multi-processor CPU memory models (historical), some distributed systems with logical clocks.

---

## 4. Causal Consistency

**Definition:** Operations that are causally related (one happened-before the other) must be seen in that order by all clients. Concurrent (causally unrelated) operations may be seen in any order.

**Happens-before relationship:**
- If A happens before B on the same process → A causally precedes B
- If A sends a message received by B → A causally precedes B
- Transitivity: if A → B and B → C, then A → C

```
Example — reply before post (causal violation):
  Alice: posts "Anyone going to the meetup?" (event A)
  Bob:   reads Alice's post, replies "Yes, I'll be there!" (event B — causally after A)

Causal consistency GUARANTEES:
  Any client that sees Bob's reply MUST also see Alice's post
  You cannot see B without having seen A — that would be causally impossible

Eventual consistency (without causal) ALLOWS:
  Client sees Bob's reply before Alice's post → confusing, violates causal order
```

**Causality tracking via vector clocks / version vectors:**
```
Each node maintains a version vector [A: n, B: m, C: k]
When a client reads and writes:
  Read from node A → client receives version vector from A
  Client writes to node B → includes the read version vector
  Node B applies the write only after seeing the causally preceding writes

→ Implements "don't show a reply before the post"
```

**Implemented by:** DynamoDB (internal MVCC tracks causal order per item), Riak (version vectors), MongoDB sessions (causal consistency mode with session tokens), some CRDT-based systems.

---

## 5. Read-Your-Writes (Read-Your-Own-Writes)

**Definition:** A client always sees its own previous writes. If a client writes X=5, any subsequent read by that same client returns a value ≥ 5 (at least as new as its last write).

**This is a per-session guarantee** — other clients may still see older values.

```
Client A:
  Writes profile: bio = "Engineer at ACME"   ← their own write
  Reads  profile: MUST see bio = "Engineer at ACME"  ← read-your-writes guarantee
  
Client B (different session):
  Reads  profile: MAY see bio = "Software Developer" (old value) ← still converging
```

**Why this matters:** Without read-your-writes, users who save their settings and immediately view them see the old settings — appearing as if the save failed. This is among the most noticeable consistency failures in practice.

**How to implement:**

```python
# Option 1: Route reads to primary for N seconds after a write
# (covered in Replication topic — cookie-based primary routing)

# Option 2: Version-stamped reads
# After write: client receives a version token
# Read request includes the token: "give me data at least as new as version V"
# Only a replica that has applied V (or later) can respond

# Option 3: Sticky sessions
# After a write, bind the user's reads to the replica that processed the write
# That replica has the latest data for this user

# DynamoDB implementation:
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('users')

# Write and capture the version token
response = table.update_item(
    Key={'user_id': 'alice'},
    UpdateExpression='SET bio = :bio',
    ExpressionAttributeValues={':bio': 'Engineer at ACME'},
    ReturnValues='ALL_NEW'
)

# Subsequent read with strong consistency to guarantee read-your-writes
response = table.get_item(
    Key={'user_id': 'alice'},
    ConsistentRead=True   # Force read from primary → always sees latest write
)
```

---

## 6. Monotonic Reads

**Definition:** Once a client has seen a value for a data item, it will never see an older value in subsequent reads. Time only moves forward.

```
Client reads X:
  First read:  X = 7   (from Replica A)
  Second read: X = 7 or X = 8 (any newer value)  ← OK
  Second read: X = 5 (an older value)              ← VIOLATION — time travel!
```

**Why this matters without monotonic reads:**
```
Scenario: replica A is current, replica B is 30 seconds behind
  User reads from A: sees their post "Hello World" → exists
  Load balancer routes next request to B: doesn't have the post yet
  User sees: post is GONE → user panics, submits bug report

With monotonic reads:
  Guaranteed not to route to a replica older than the one the user last saw
  Implementation: hash user to replica OR track minimum observed version
```

**Implementation:**
```python
# Track minimum version seen per session
class MonotonicReadClient:
    def __init__(self, replicas):
        self.replicas = replicas
        self.min_version: dict = {}  # key → minimum version seen
    
    def read(self, key):
        min_ver = self.min_version.get(key, 0)
        
        # Only query replicas that are at least as current as our last read
        for replica in self.replicas:
            result = replica.read(key)
            if result and result['version'] >= min_ver:
                self.min_version[key] = result['version']
                return result['value']
        
        # Fallback: read from primary (always current)
        return self.replicas[0].read_primary(key)['value']
```

---

## 7. Eventual Consistency

**Definition:** If no new writes are made to a data item, all replicas will eventually converge to the same value. No guarantees about how long convergence takes or what intermediate values are visible.

**What eventual consistency does NOT guarantee:**
- Reads will NOT see stale data (they may)
- Causal order will be preserved (it may not)
- Your own writes will be visible to you immediately (they may not)
- Two clients will see writes in the same order (they may not)

**What eventual consistency DOES guarantee:**
- Given enough time with no new writes, all replicas converge
- No permanent divergence (data is not permanently lost)

```
Timeline:
  t=0ms: Write X=5 on Node A
  t=5ms: Node A → Node B replication in progress
  t=6ms: Client reads X from Node B → MAY see X=3 (old value)  ← allowed
  t=10ms: Replication complete
  t=11ms: Client reads X from Node B → MUST see X=5 ← eventually converged
```

**Where eventual consistency is acceptable:**
- Social media feeds (a few ms of staleness is invisible)
- Product catalog (prices updated at human speed, not microsecond speed)
- DNS (TTL-based eventual consistency is by design)
- Shopping cart (adding items to cart doesn't require global coordination)
- View counts, like counts (approximate is fine in real-time)

**Where eventual consistency is NOT acceptable:**
- Account balances (stale read can enable overdraft)
- Inventory reservation (stale read can enable oversell)
- Authentication state (session revocation must be immediate)
- Distributed locks (stale lock state causes safety violations)

---

## 8. Consistency Models in Real Systems

### How Consistency is Selected Per-Operation

```python
# PostgreSQL — choose per query
# Strong (primary):
conn_primary.execute("SELECT balance FROM accounts WHERE id = %s", (account_id,))

# Eventual (replica — may be stale):
conn_replica.execute("SELECT balance FROM accounts WHERE id = %s", (account_id,))

# ---

# Cassandra — choose per query  
session.execute("SELECT * FROM users WHERE id = %s", (user_id,),
                consistency_level=ConsistencyLevel.QUORUM)         # strong-ish

session.execute("SELECT * FROM users WHERE id = %s", (user_id,),
                consistency_level=ConsistencyLevel.ONE)             # eventual

# ---

# DynamoDB — choose per read
table.get_item(Key={'id': user_id}, ConsistentRead=True)   # strong
table.get_item(Key={'id': user_id}, ConsistentRead=False)  # eventual (default)

# ---

# MongoDB — choose per session (causal consistency)
with client.start_session() as session:
    session.start_transaction()
    # All operations in this session are causally consistent
    col.find_one({'_id': user_id}, session=session)
```

### Consistency Model per System (Summary)

| System                    | Default Model           | Strongest Available           |
|---------------------------|-------------------------|-------------------------------|
| PostgreSQL (primary)      | Linearizable            | Linearizable                  |
| PostgreSQL (async replica)| Eventual                | N/A (replica doesn't upgrade) |
| Redis (single node)       | Linearizable            | Linearizable                  |
| Cassandra (CL=ONE)        | Eventual                | Strong (CL=QUORUM, W+R>N)    |
| DynamoDB (default)        | Eventual                | Strong (ConsistentRead=True)  |
| MongoDB (default)         | Read-your-writes (session) | Linearizable (w:majority + j:true) |
| ZooKeeper                 | Linearizable            | Linearizable                  |
| etcd                      | Linearizable            | Linearizable                  |
| Spanner                   | Linearizable (external) | Linearizable                  |

---

## 9. Common Anomalies — What Each Model Prevents

| Anomaly                  | Description                                              | Prevented by           |
|--------------------------|----------------------------------------------------------|------------------------|
| Stale read               | Read returns old value after newer one committed         | Linearizability        |
| Time-travel read         | Client sees newer, then older value                      | Monotonic reads        |
| Lost write visibility    | Client's own write not visible on immediate re-read      | Read-your-writes       |
| Causal violation         | Reply visible before the post it responds to             | Causal consistency     |
| Write reorder            | Two clients see writes in different orders               | Sequential consistency |
| Phantom read             | Range query returns different rows in same transaction   | Serializable isolation |

---

## 10. Consistency vs Isolation — Don't Confuse Them

| Dimension     | Consistency Models                          | Isolation Levels (ACID)                    |
|---------------|---------------------------------------------|---------------------------------------------|
| Scope         | Distributed systems, replicas across nodes  | Single-database, concurrent transactions    |
| Concern       | What value do you see across replicas?      | What do concurrent transactions see?        |
| Examples      | Linearizable, eventual, causal              | Read committed, repeatable read, serializable |
| Violated by   | Replication lag, network partition          | Concurrent writes, phantom reads            |

They are orthogonal concerns. A strongly consistent distributed database can still suffer from dirty reads between concurrent transactions (isolation issue). A database with serializable isolation can still serve stale data from a replica (consistency issue).
