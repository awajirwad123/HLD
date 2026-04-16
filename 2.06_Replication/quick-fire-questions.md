# Replication — Quick-Fire Questions

**Format:** Answer out loud, then check.

---

**Q1: What are the three goals of replication?**
High availability (survive node failures), durability (data survives hardware failures), and read scalability (distribute reads across replicas).

---

**Q2: What is the difference between synchronous and asynchronous replication?**
Async: the primary acknowledges the write to the client before replicas confirm — low write latency, possible data loss on crash. Sync: the primary waits for at least one replica to confirm before acknowledging — zero data loss, higher write latency equal to network RTT.

---

**Q3: What is replication lag?**
The delay between when a write is committed on the primary and when it is applied on a replica. Typically milliseconds in the same AZ, but can spike under heavy write load.

---

**Q4: What is the "read-your-writes" consistency problem?**
A user writes data, then immediately reads it back. If the read is routed to a replica that hasn't yet applied the write, the user sees stale data — appearing as if their write was lost.

---

**Q5: How do you fix the read-your-writes problem without always reading from the primary?**
After each write, set a short-lived indicator (e.g., a cookie with timestamp). For a configurable window (e.g., 2 seconds), route that user's reads to the primary. After the window, resume reading from replicas.

---

**Q6: What is monotonic reads and why does it matter?**
Monotonic reads guarantee that once you've seen a value, you won't see an older value in a subsequent read. Violation: you read a value from Replica A, then the next request hits Replica B which is further behind — you see a rollback in time. Fix: consistently route a user to the same replica.

---

**Q7: What is the quorum formula for strong consistency in leaderless replication?**
W + R > N, where W = write quorum, R = read quorum, N = replication factor. This ensures the read set always overlaps the write set by at least one node.

---

**Q8: With N=5, W=3, R=3 — is this strongly consistent? How many node failures can it tolerate?**
Yes, strongly consistent: W + R = 6 > 5. It can tolerate 2 node failures (since quorum of 3 is still satisfied on the remaining 3 nodes).

---

**Q9: What is split-brain in leader-follower replication?**
When a network partition causes both the old primary and a newly-elected replica to believe they are the sole leader, both accepting writes simultaneously. This leads to data divergence that is hard to reconcile.

---

**Q10: How do you prevent split-brain?**
Require a majority quorum for election (odd-numbered cluster + more than half must vote). A node that cannot reach a majority surrenders its leader role. STONITH (Shoot The Other Node In The Head) forcefully fences the old primary.

---

**Q11: What is hinted handoff in Cassandra?**
When a write cannot reach its target replica (node is down), the coordinator stores the write temporarily as a "hint." When the target replica recovers, the coordinator replays the hint to bring it up to date.

---

**Q12: What is anti-entropy in leaderless replication?**
A background process that compares data across replicas using Merkle trees and synchronizes any divergent data. Ensures eventually-consistent convergence even for data that was never accessed after a write (unlike read repair which only fixes during reads).

---

**Q13: What is the PostgreSQL command to check replication lag?**
`SELECT client_addr, state, write_lag, flush_lag, replay_lag FROM pg_stat_replication;` — run on the primary. `replay_lag` is the most relevant: how far behind the replica is in applying WAL records.

---

**Q14: What is a replication slot in PostgreSQL, and what can go wrong?**
A replication slot tracks the WAL position a replica has consumed up to. The primary retains WAL that the slot hasn't consumed yet. If a replica goes offline and the slot isn't dropped, the primary accumulates WAL indefinitely → disk exhaustion. Fix: set `max_slot_wal_keep_size`, monitor slot lag, drop slots for permanently-offline replicas.

---

**Q15: What is the write amplification problem in multi-leader replication?**
In multi-leader, every write must be replicated to all other leaders. With 5 leaders, each write triggers 4 replication messages. Under high write volume, this creates significant network traffic and processing overhead on each leader.

---

**Q16: What does "last write wins" (LWW) mean, and what is wrong with it?**
LWW resolves write conflicts by keeping the write with the highest timestamp. The problem: clocks are not perfectly synchronized across distributed nodes. A write with a slightly lower timestamp (but that happened logically later) can be silently discarded, causing data loss.

---

**Q17: What is a version vector?**
A data structure that tracks, per node, how many writes that node has applied. It enables determining causality: if one version vector dominates another, one write causally follows the other. If neither dominates, the writes are concurrent (a true conflict).

---

**Q18: Can you scale writes by adding read replicas?**
No. Read replicas only scale reads. All writes still go to the single primary. To scale writes you need sharding (multiple primaries), multi-leader replication, or a leaderless system.

---

**Q19: What is the RPO and RTO for AWS RDS Multi-AZ?**
RPO = ~0 seconds (synchronous replication to standby in another AZ — no data loss). RTO = ~1–2 minutes (automatic failover to standby, DNS cutover).

---

**Q20: In what scenario does async replication become dangerous even for non-financial data?**
During a primary failover under heavy write load. If the replica is significantly behind (e.g., 30 seconds of lag due to a write burst), promoting that replica means 30 seconds of committed writes are permanently lost. Mitigation: alert on lag > threshold; delay promotion until lag is low; or require sync for critical tables only.
