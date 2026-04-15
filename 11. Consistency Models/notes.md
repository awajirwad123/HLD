# Consistency Models — Notes & Reference

## The Consistency Spectrum

```
← STRONGER                                     WEAKER →
Linearizability → Sequential → Causal → Read-Your-Writes → Monotonic Reads → Eventual
     (cost: high)                                                     (cost: low)
```

**Stronger = fewer anomalies, higher latency/cost**
**Weaker = more anomalies possible, lower latency/cost**

---

## Model One-Liners

| Model | One-Line Definition | Key Invariant |
|-------|-------------------|---------------|
| **Linearizability** | Every op appears instantaneous, globally ordered | Reads always reflect latest write |
| **Sequential Consistency** | All nodes see same order, but order may lag real-time | Program order respected per-client |
| **Causal Consistency** | Causally related ops seen in causal order by all | Effect never seen before cause |
| **Read-Your-Writes** | You always see your own most recent write | Per-session: write then read = fresh |
| **Monotonic Reads** | Each successive read is at least as fresh as the last | Time never goes backward per session |
| **Eventual Consistency** | If writes stop, all replicas eventually agree | Convergence guaranteed (timing isn't) |

---

## What Each Model Prevents

| Anomaly | Linearizable | Sequential | Causal | RYW | Monotonic | Eventual |
|---------|:---:|:---:|:---:|:---:|:---:|:---:|
| Stale read (any) | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Time-travel (read older than prev read) | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ |
| Read-your-writes violation | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Causal violation (reply before post) | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Write reorder across clients | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## System Defaults Reference Card

| System | Default Model | Stronger Option |
|--------|--------------|-----------------|
| **PostgreSQL (primary)** | Linearizable | N/A — already strongest |
| **PostgreSQL (replica)** | Eventual (async lag) | `synchronous_commit=on` → Linearizable |
| **MySQL replica** | Eventual | `WAIT_FOR_EXECUTED_GTID_SET()` → RYW |
| **Redis** (single node) | Linearizable | N/A |
| **Redis Cluster** | Eventual per-slot | `WAIT` command → stronger |
| **MongoDB (primary)** | Linearizable with `"majority"` write concern | Default is eventual |
| **MongoDB (secondary reads)** | Eventual | Session with `causal_consistency=True` |
| **Cassandra** | Eventual (ONE) | QUORUM or ALL → Linearizable (quorum intersect) |
| **DynamoDB** | Eventual | `ConsistentRead=True` → Linearizable |
| **ZooKeeper / etcd** | Linearizable | N/A — always linearizable |
| **Google Spanner** | Linearizable (TrueTime) | N/A |

---

## Model Selection Decision Tree

```
Do you need the globally latest value?
├── YES → Linearizability
│          (PostgreSQL primary, ZooKeeper, etcd)
│          Cost: ~2–5ms+ per read, requires quorum
└── NO
    ├── Does the read causally depend on a prior write in THIS session?
    │   └── YES → Read-Your-Writes
    │              (cookie to primary, DynamoDB ConsistentRead, MySQL GTID routing)
    │              Cost: moderate — route to fresh replica
    ├── Must the user never see older data than they've already seen?
    │   └── YES → Monotonic Reads
    │              (version-stamped routing, consistent replica affinity)
    │              Cost: low — just track session version
    ├── Is there a cause-effect relationship between items?
    │   └── YES → Causal Consistency
    │              (MongoDB causal sessions, vector clocks, causal+ systems)
    │              Cost: moderate — track causal dependencies
    └── None of the above — Eventual Consistency
                 (DynamoDB default, Cassandra ONE, CDN edge caches)
                 Cost: lowest — read from nearest replica
```

---

## Key Design Questions Before Choosing a Model

1. **Who observes the inconsistency?** — A single user (RYW/Monotonic) vs multiple users (Causal/Linearizable)?
2. **What is the cost of a stale read?** — Display bug (tolerate) vs double-charge (never tolerate)?
3. **How long can inconsistency last?** — Milliseconds (replication lag) vs seconds/minutes (acceptable for counters)?
4. **Is the use case write-heavy or read-heavy?** — Stronger models penalise writes more (quorum, fsync).
5. **Can the client carry a token?** — Version tokens in headers/cookies enable RYW cheaply without schema changes.

---

## Consistency vs. Isolation — Not the Same Thing

| | **Consistency Models** | **Isolation Levels** |
|--|------------------------|----------------------|
| **Scope** | Distributed (multi-node) | Single database (transactions) |
| **Concern** | Replication currency | Concurrent transactions |
| **Examples** | Linearizable, Eventual | READ COMMITTED, Serializable |
| **Problem they solve** | Stale reads across replicas | Dirty reads, phantoms, lost updates |
| **Configured at** | Client/read policy | `SET TRANSACTION ISOLATION LEVEL` |

You can have low isolation + strong consistency or high isolation + eventual consistency. They are orthogonal axes.

---

## Anomaly Quick-Definitions

| Anomaly | Definition | Example |
|---------|-----------|---------|
| **Stale read** | Reading a value that has been overwritten | Reading `stock=1` when it was already set to 0 |
| **Time-travel** | Reading an older version than you read before | Seeing v3 after having seen v5 |
| **Lost write** | One of two concurrent writes disappears | LWW silently drops the earlier timestamp |
| **Causal violation** | Effect visible before cause | Reply visible before original post |
| **Write reorder** | Two writes appear in different order to different clients | A sees W1→W2, B sees W2→W1 |

---

## Key Numbers to Cite in Interviews

| Metric | Typical Value |
|--------|--------------|
| PostgreSQL async replication lag | 10–100ms under normal load |
| Cassandra cross-DC replication lag | 50–500ms |
| DynamoDB ConsistentRead extra cost | ~2× read capacity units |
| ZooKeeper linearizable read latency | 2–10ms (leader round-trip) |
| Redis replication lag | < 1ms (same DC) |
| Spanner TrueTime uncertainty | ±7ms |

---

## Cheat Sheet: 4 Interview Answer Frameworks

### "Why not always use linearizability?"
> Linearizability requires reading from the leader/primary or a quorum, adding round-trips and making the system unavailable during network partitions. For globally distributed systems, this can add 50–200ms of latency per read. Eventual consistency with monotonic reads or RYW gives 99% of the UX benefit at 10% of the cost.

### "How does Cassandra handle consistency?"
> Cassandra offers tunable consistency: ONE/TWO/THREE/QUORUM/ALL. With W+R > N (e.g., QUORUM writes + QUORUM reads on N=3), reads always overlap the latest write, approximating linearizability. The default is ONE — fastest but stale. QUORUM is the typical production setting.

### "What is eventual consistency actually guaranteed to do?"
> It guarantees that if no new writes occur, all replicas will converge to the same value. It does NOT guarantee when. It does NOT guarantee any ordering. It does NOT prevent temporary stale, time-travel, or causal violations during the convergence window.

### "Difference between causal and read-your-writes?"
> Read-your-writes is a per-client guarantee: you see your own writes. Causal consistency is a multi-client guarantee: if client A's write causally influenced client B's action, any third client C that sees B's action also sees A's write. RYW ⊆ Causal — causal is the stronger model.
