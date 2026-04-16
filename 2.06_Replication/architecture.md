# Replication — Architecture

## Overview

Replication means keeping a copy of the same data on multiple machines connected via a network. It serves three goals:
1. **High availability** — if one node fails, others take over
2. **Durability** — data survives hardware failures
3. **Read scalability** — distribute reads across replicas

Replication is everywhere: PostgreSQL streaming replication, MySQL binlog, Redis Sentinel, Cassandra, MongoDB replica sets, Kafka. Understanding the models and their trade-offs is mandatory for senior interviews.

---

## 1. Synchronous vs Asynchronous Replication

### Asynchronous Replication (Default in most systems)

```
Client
  │
  ▼ write
[ Primary ]──────────────────────────────────────────────────► ack to client ✅
     │
     │  WAL/binlog (async — after ack)
     ▼
[ Replica 1 ]   [ Replica 2 ]
   (applying)     (applying)
```

- Primary acknowledges the write **before** waiting for replicas
- **Advantage:** write latency is unaffected by replica speed, network, or replica failures
- **Risk:** if the primary crashes between the ack and replica applying the write → **data loss** (the replica doesn't have the latest write)
- Used by: PostgreSQL (default), MySQL (default), MongoDB (default write concern `w:1`)

### Synchronous Replication

```
Client
  │
  ▼ write
[ Primary ]──────── wait ────────────────────────────────────► ack to client ✅
     │                                                               ▲
     │  WAL (sync — before ack)                                     │
     ▼               ▼                                              │
[ Replica 1 ] ←─ confirm   [ Replica 2 ] (async)     ─────────────┘
```

- Primary waits for at least one replica to confirm before replying to client
- **Advantage:** zero data loss — committed data is on ≥ 2 machines
- **Risk:** write latency = network RTT to replica; if the sync replica is slow/down → primary blocks
- Semi-sync (MySQL) or `synchronous_standby_names` (PostgreSQL) achieve "sync to at least one"
- Used by: AWS RDS Multi-AZ, Google Spanner, financial systems

### Trade-off Summary

| Property               | Async                       | Sync                              |
|------------------------|-----------------------------|-----------------------------------|
| Write latency          | Low (no replica wait)       | Higher (+ network RTT to replica) |
| Data durability        | ⚠️ Data loss window on crash | ✅ No data loss                   |
| Replica failure impact | None on primary writes      | Blocks writes if sync replica down |
| Use case               | Social apps, read scale     | Payments, audit logs, critical data |

---

## 2. Leader-Follower Replication (Single-Leader)

The most common model. One node accepts all writes (the **leader/primary**); all writes are replicated to zero or more **followers/replicas**, which only accept reads.

```
                    ┌─────────────┐
   Write ──────────►│   Primary   │◄────── Reads (optional)
                    └──────┬──────┘
                           │
              WAL/binlog streaming
              ┌────────────┴────────────┐
              ▼                         ▼
       [ Replica A ]             [ Replica B ]
    Read replicas only         Read replicas only
```

### How the Replication Log Works

**PostgreSQL — WAL Streaming:**
```
Primary:
  1. Write log record to WAL (sequential disk file)
  2. Apply to data pages (heap, indexes)
  3. WAL sender process streams WAL records to replica

Replica:
  1. WAL receiver receives records
  2. WAL apply process replays them on local disk
  → Replica is a physical byte-for-byte copy of the primary
```

**MySQL — Binlog:**
```
Primary:
  1. Execute the transaction
  2. Write an event to the binary log (statement-based OR row-based)
  3. Committed

Replica:
  1. IO thread: fetches binlog events from primary → relay log
  2. SQL thread: replays events from relay log → applies changes
  → Logical replication (easier to use for cross-version, cross-platform)
```

### Replication Lag

The replica is always slightly behind the primary. This window is called **replication lag**.

```
t=0: Primary commits row X with value A
t=5ms: Replica applies the change → has value A
t=0..5ms window: A read from replica returns stale value (or missing row)
```

**Replication lag becomes a problem when:**
- User writes something, then immediately reads it from a replica — sees stale data ("read-your-writes" problem)
- Replica is far behind (seconds or minutes) during high write load
- A batch job replays large amounts of data

**Monitoring:**
```sql
-- PostgreSQL: check replication lag (on primary)
SELECT
    client_addr,
    state,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;

-- MySQL
SHOW REPLICA STATUS\G
-- Look at Seconds_Behind_Source
```

---

## 3. Multi-Leader Replication

Multiple nodes can accept writes simultaneously. Each node replicates its writes to all other leaders.

```
  DC East                           DC West
┌──────────┐  ◄── replicate ──► ┌──────────┐
│ Leader A │                     │ Leader B │
└──────────┘                     └──────────┘
     │                                │
  followers                        followers
```

**Use cases:**
- Multi-datacenter active-active (users in each DC write locally, low latency)
- Offline-capable clients (e.g., Google Docs — your local doc is a "leader" while offline)
- Collaborative editing

**The big problem: Write Conflicts**
```
User 1 (connected to Leader A): sets title = "Hello"
User 2 (connected to Leader B): sets title = "Hi"
Both changes replicated → conflict: what is the final title?
```

**Conflict resolution strategies:**
1. **Last Write Wins (LWW):** Use a timestamp; keep the latest. Simple but loses data
2. **Causality tracking:** Use version vectors to detect concurrent writes; surface conflict to application
3. **Application-level merge:** CRDTs (Conflict-free Replicated Data Types) — used by Riak, Redis CRDTs
4. **Avoid conflicts:** Route all writes for a given record to the same leader (conflicts can't occur)

---

## 4. Leaderless Replication (Dynamo-style)

No designated leader. Any replica can accept reads and writes. Used by Cassandra, DynamoDB (internally), Riak.

```
                         Client
                        /   |   \
                       /    |    \
                      W     W     W
                      ▼     ▼     ▼
              [Node 1] [Node 2] [Node 3]   ← RF=3
```

### Quorums

**W** = number of nodes a write must succeed on  
**R** = number of nodes a read must query  
**N** = replication factor (total copies)

```
Strong consistency condition: W + R > N

Common with N=3:
  W=2, R=2 → 2+2=4 > 3 → reads always overlap with at least one written node ✅
  W=1, R=1 → 1+1=2 < 3 → possible to read from a node that doesn't have latest write ❌

Availability trade-off:
  W=3, R=1 → every write must reach all 3 nodes → any node down = writes fail
  W=1, R=1 → any single node available = reads and writes succeed (may be stale)
```

### Read Repair and Anti-Entropy

When a read finds different values on different replicas, it uses version numbers to pick the newest and writes the newer value back to outdated replicas. This is **read repair**.

**Anti-entropy process:** Background job that compares replicas using Merkle trees and synchronizes divergent data.

### Hinted Handoff

If a target replica is down during a write, another node stores the write temporarily with a "hint" that it belongs to the down node. When the node recovers, the hint is replayed.

---

## 5. Failover

### Automatic Failover (Leader-Follower)

```
Normal:
  [ Primary ] ← all writes

Primary crashes:

Failover process:
  1. Detect failure (heartbeat timeout, typically 30s–2min)
  2. Elect new primary from replicas (replica with most up-to-date WAL wins)
  3. Update load balancer / DNS to point writes to new primary
  4. Old primary (if it comes back) must be told it's now a replica

PostgreSQL: pg_auto_failover, Patroni (etcd-backed)
MySQL: Group Replication, Orchestrator
MongoDB: Replica set auto-election (Raft-like, < 30s)
```

### Split-Brain Problem

If the network partition divides the cluster and both halves think the other is dead, both elect a new primary → **two primaries accepting writes simultaneously** → data divergence.

**Prevention:**
- Require **majority quorum** for election (odd number of nodes + majority vote)
- A primary must surrender its role if it loses quorum (STONITH — Shoot The Other Node In The Head)
- Use a dedicated **witness node** (e.g., etcd) to break ties

---

## 6. Read Scaling with Replicas

The primary motivation for adding replicas is distributing read load:

```
Without replicas:
  Primary: 80% CPU (reads + writes), single bottleneck

With 3 replicas (80/20 read/write split):
  Primary: handles writes + 20% reads = ~36% CPU
  Each replica: 20% reads = ~16% CPU
  Total: 4x read capacity
```

**Routing strategy in FastAPI/SQLAlchemy:**
```python
# Route writes to primary, reads to replica
write_engine = create_async_engine("postgresql+asyncpg://primary:5432/db")
read_engine  = create_async_engine("postgresql+asyncpg://replica:5432/db",
                                   execution_options={"postgresql_readonly": True})

# For critical reads-after-writes: read from primary
# For eventually-consistent reads (dashboards, lists): read from replica
```

### What You Cannot Solve with More Replicas

Adding read replicas scales **reads only**. The write path is still a single primary. To scale writes, you need:
- **Sharding** (partition data across multiple primaries)
- **CQRS** (event-sourced write model; replicas serve the read model)
- **Eventually-consistent leaders** (Cassandra multi-leader, DynamoDB)

---

## 7. Replication in Specific Systems

| System        | Model            | Protocol          | Default mode | Auto-failover  |
|---------------|------------------|-------------------|--------------|----------------|
| PostgreSQL    | Leader-follower  | WAL streaming     | Async        | Patroni / pg_auto_failover |
| MySQL         | Leader-follower  | Binlog (row-based)| Async/semi-sync | Group Replication, Orchestrator |
| MongoDB       | Leader-follower  | Oplog             | Async (w:1)  | ✅ Replica set election |
| Cassandra     | Leaderless       | Gossip + Merkle   | Async (ONE)  | Automatic (no election needed) |
| Redis Sentinel| Leader-follower  | RDB/AOF streaming | Async        | ✅ Sentinel quorum |
| Redis Cluster | Leaderless-ish   | RDB/AOF per shard | Async        | ✅ Per-shard primary |
| Kafka         | Leader-per-partition | ISR (In-Sync Replicas) | Configurable acks | ✅ Controller election |

---

## 8. Common Failure Modes

| Failure                         | Root Cause                                   | Mitigation                                              |
|---------------------------------|----------------------------------------------|---------------------------------------------------------|
| Data loss on primary crash      | Async replication, write not yet replicated  | Use sync replication for critical data; increase replica count |
| Replica falls behind (lag spike)| Burst of writes, replica CPU/disk bottleneck | Monitor `replay_lag`; alert on > 1s; scale replica vertically |
| Split-brain                     | Network partition, broken failover logic     | Majority quorum for election; STONITH; witness node    |
| Read-your-writes violation      | Routing reads to stale replica               | Sticky routing after write; read from primary after write |
| Replication slot bloat (PG)     | Replica offline while slot held              | Set `max_slot_wal_keep_size`; alert on slot lag         |
| Cascading replica lag           | Chained replication (replica of replica)     | Don't use chain replication in production; fan-out from primary |
