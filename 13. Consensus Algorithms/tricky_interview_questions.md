# Consensus Algorithms — Tricky Interview Questions

## Q1: "Your 5-node etcd cluster loses 3 nodes simultaneously. What happens to Kubernetes?"

**What they're testing:** Quorum, availability impact, recovery.

**Analysis:**
- 5-node cluster, quorum = 3
- 3 nodes down → remaining 2 nodes < quorum
- etcd cluster becomes **unavailable** (no writes, no reads for consistent data)

**Impact on Kubernetes:**
- **No new pods, services, deployments, or config changes** — all `kubectl apply` commands fail
- **Existing pods keep running** — kubelet on each worker node operates independently; it doesn't need etcd for runtime operations. Pods already scheduled continue as normal.
- **New scheduling fails** — kube-scheduler reads from etcd to assign new pods to nodes; can't do that without etcd

**Recovery procedure:**
1. Bring back the 3 failed nodes from backup (etcd data is durable on disk)
2. If data is lost: restore from a recent etcd snapshot (`etcdctl snapshot restore`)
3. Rejoin nodes to cluster

**Prevention:**
- Take etcd snapshots every 30 minutes (store in S3)
- Use 5-node cluster across 3 availability zones (no single AZ failure kills quorum)
- Monitor etcd quorum health: alerting when any node is down (before losing quorum)

**The key insight:** "Kubernetes is split into the control plane (needs etcd) and data plane (doesn't). Data plane keeps running, but you lose the ability to make changes."

---

## Q2: "Two etcd nodes think they're both leaders simultaneously. Is that possible? How does Raft prevent it?"

**What they're testing:** Raft safety guarantee, term numbers, how split-brain is prevented.

**Can it happen?**

Theoretically, for an instant after a network heals from a partition: Node A was leader in term 5, was partitioned, the rest of the cluster elected Node B as leader in term 7. When the partition heals, Node A still thinks it's the leader of term 5.

**Does this cause split brain?**

No, for two reasons:

1. **Before we even get there:** Node A (term 5 leader) rejected all writes during the partition because it couldn't reach quorum. No data was "committed" on node A's partition. Any writes that needed quorum were refused.

2. **When partition heals:** Node B sends heartbeats with term 7 to Node A. Raft rule: "if you see a higher term, immediately revert to Follower." Node A sees term 7 > its term 5 → immediately becomes Follower.

**In practice:** There's never a point where two leaders are simultaneously accepting committed writes. Leader A during the partition required quorum which it didn't have → no commits. When it discovers term 7, it steps down before making any new commits.

**The term number IS the safety mechanism:** Terms are monotonically increasing, can never go backwards, and a node immediately defers to higher terms. This is what makes Raft's safety properties provable.

---

## Q3: "Why is CockroachDB faster than a Redis master for writes? Isn't Raft adding more round-trips?"

**What they're testing:** Understanding what Raft adds vs. what it replaces.

**Clarification needed:** This comparison isn't apples-to-apples. Compare on what they're solving:

**Redis single master (no replication):**
- 1 write = just write to memory on one node
- No durability by default (unless AOF/RDB is enabled)
- If master crashes = data loss

**CockroachDB (Raft per-range):**
- 1 write = propose to Raft log → quorum of N=3 nodes write to WAL → committed → applied
- Fully durable across 3 nodes
- Zero data loss on single node crash

**Write latency comparison:**
- Redis in-memory: ~100µs
- CockroachDB (same datacenter, 3 replicas): ~2–5ms

CockroachDB is NOT faster; it's slower but safer. The Raft overhead is:
- 1 extra network RTT for the Propose → Quorum phase
- WAL flush on each replica

**When Raft is "free":** CockroachDB can process many concurrent Raft proposals in a single batch. At high throughput, the amortized cost per write drops significantly.

**The real trade-off:** Redis is fast but ephemeral; CockroachDB is slower but provides ACID guarantees + automatic failover + horizontal scalability. They solve different problems.

---

## Q4: "Design a distributed lock service using Raft. What guarantees does it provide?"

**What they're testing:** Applying consensus to a real problem, lease semantics, fencing tokens.

**Basic design:**

```
Lock service: 3-node Raft cluster
  Internal state (replicated log): {lock_name → {holder, lease_expiry}}

Acquire lock:
  Client → any node → forwarded to leader
  Leader proposes: "SET lock:X holder=client-A expire=now+30s"
  Quorum acks → committed → client-A has the lock

Release lock:
  Client → leader proposes: "DEL lock:X IF holder=client-A"

Automatic expiry:
  Leadership heartbeat checks for expired leases → proposes deletion
```

**What it guarantees:**
1. **Mutual exclusion:** At most one client holds the lock at any time (Raft ensures linearizability)
2. **Durability:** Lock state survives leader crashes (replicated to quorum)
3. **Liveness:** If lock holder crashes, lease expires → another client can acquire

**The fencing token problem (critical):**

Even with a perfect lock service, a client holding the lock can pause (GC, OS sleep) and "wake up" after its lease expired. Another client acquired the lock. Now both think they hold it.

**Solution:** Fencing tokens.
- Lock service assigns a monotonically increasing token on each successful acquire.
- Resource server (e.g., storage) tracks the highest token it's seen.
- Client must include token with every resource request.
- Resource server rejects requests with a token lower than the highest seen.

```
Client A acquires lock → token=5
Client A pauses (GC pause)
Lease expires
Client B acquires lock → token=6
Client A resumes → tries to write with token=5
Storage: "5 < 6, rejecting stale token"
Client B's writes with token=6: accepted
```

This is exactly how Google Chubby and etcd leases work in practice.

---

## Q5: "Kafka's old controller election used ZooKeeper. Why was this a problem at scale, and how did KRaft fix it?"

**What they're testing:** Real-world application of consensus, operational knowledge.

**Old architecture (ZooKeeper-based):**

```
ZooKeeper cluster (3–5 nodes) ←→ Kafka brokers
  - ZK stored: broker metadata, partition leadership, consumer group offsets (old)
  - Controller election: broker tries to create ephemeral /controller znode → wins if created first
```

**Problems at scale:**

1. **Metadata bottleneck:** For a Kafka cluster with 1M partitions, all partition state was in ZooKeeper. Broker startup required reading all 1M partition assignments from ZK → slow startup (minutes!).

2. **Operational complexity:** Two separate distributed systems to monitor, operate, and upgrade. ZooKeeper has its own quorum, separate from Kafka's replication quorum.

3. **Failover speed:** ZooKeeper session timeout (30–60s) → slow controller failover → partitions unavailable during failover.

4. **Scaling ceiling:** ZooKeeper's ZAB protocol limits write throughput. Large clusters with frequent partition reassignments could overwhelm ZooKeeper.

**KRaft solution:**

```
KRaft (Kafka 2.8+):
  - Dedicated "controller" nodes (subset of brokers) run Raft natively
  - Cluster metadata stored in an internal Kafka topic: __cluster_metadata
  - Controller election via Raft → no ZooKeeper needed
  - Active controller = Raft leader; other controllers = followers

Benefits:
  - Startup: metadata loaded from Kafka log (fast, efficient)
  - Failover: Raft controller election → 10–15s (vs 30–60s with ZK)
  - Single system to operate (no ZooKeeper cluster)
  - 10× more partitions supported (millions vs hundreds of thousands)
```

**Key lesson:** Moving consensus in-process (same team, same language) allows tighter integration, better performance, and simpler operations vs. an external consensus system.
