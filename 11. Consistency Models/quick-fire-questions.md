# Consistency Models — Quick-Fire Q&A

**Format:** Answer each question in ≤2 sentences before checking the answer.

---

**Q1. What is linearizability?**
Every operation appears to execute atomically at a single point in real-time between its invocation and completion — as if there were a single global copy of the data.

---

**Q2. What is the difference between linearizability and sequential consistency?**
Both guarantee a single consistent order of operations, but sequential consistency only requires that each client's operations appear in program order — it doesn't require that order to match real-time wall clock order. Linearizability is strictly stronger.

---

**Q3. What does "read-your-writes" guarantee?**
After a write, any subsequent read by the same client (session) will return that write or a later value. It makes no guarantees for other clients.

---

**Q4. What anomaly does monotonic reads prevent?**
Time-travel: a client reading the same key multiple times will never see a version older than what it has already read. Each successive read is at least as fresh as the last.

---

**Q5. What does eventual consistency actually guarantee?**
If all writes stop, all replicas will converge to the same value. It guarantees convergence but says nothing about when, the order of visible updates, or preventing stale reads during the convergence window.

---

**Q6. What is a causal violation? Give a concrete example.**
An effect is visible before its cause. Classic example: a user sees a reply to a post on a replica that hasn't yet received the original post. The reply is the effect; the post is the cause.

---

**Q7. How do vector clocks help enforce causal consistency?**
Each write carries a vector clock timestamp. A replica only delivers a message if all causally prior messages (indicated by the clock) have already been delivered — otherwise the message is buffered.

---

**Q8. What is the ordering of consistency models from strongest to weakest?**
Linearizability > Sequential > Causal > Read-Your-Writes > Monotonic Reads > Eventual Consistency.

---

**Q9. Is Read-Your-Writes a subset of Causal Consistency?**
Yes. RYW is the single-client, single-session view of causality. Causal consistency extends this across multiple clients: any client that observes an effect must also observe its cause.

---

**Q10. What is Cassandra's default consistency level and what model does it provide?**
The default is `ONE` — read/write succeeds on acknowledgement from any single replica. This provides eventual consistency: reads may return stale data if the acknowledged replica is behind.

---

**Q11. How does DynamoDB achieve linearizable reads?**
Setting `ConsistentRead=True` routes the read to the partition's primary node, bypassing eventual-consistent replicas. The trade-off is approximately 2× read capacity unit consumption.

---

**Q12. What does MongoDB's `causal_consistency=True` session option do?**
It uses a logical session clock to ensure reads within the session always reflect previous writes in that session, providing both read-your-writes and monotonic reads guarantees across primary and secondaries.

---

**Q13. How can you implement read-your-writes without always hitting the primary?**
Return the WAL/LSN position (a version token) to the client after a write. On subsequent reads, route to any replica whose applied LSN is ≥ the token. If no replica qualifies, fall back to primary.

---

**Q14. What is a "stale read" and under what model is it acceptable?**
A stale read returns an older value than the current committed state of the system. It is acceptable under eventual consistency for use cases where brief incongruence is tolerable — e.g., social media like counts, CDN-cached pages, DNS TTL caches.

---

**Q15. Why is ZooKeeper considered linearizable?**
All reads and writes are served through the elected leader, and writes are only acknowledged after being committed to a quorum of nodes via the ZAB (ZooKeeper Atomic Broadcast) protocol, ensuring a single, real-time-ordered view.

---

**Q16. Can a system be both AP (from CAP) and linearizable?**
No. Linearizability requires consistency (C in CAP) — it cannot be maintained during a network partition if the system must remain available. An AP system sacrifices C, which means it cannot provide linearizability during partitions.

---

**Q17. What is Last Write Wins (LWW) and what problem does it have?**
LWW resolves conflicts by keeping the write with the highest timestamp. The problem: concurrent writes on different partitioned nodes can have clock skew, and the "losing" write is silently discarded — data loss without any error.

---

**Q18. What is a G-Counter CRDT and why is it conflict-free?**
A G-Counter lets each node increment only its own counter slot and merges by taking the element-wise maximum. Because increments are only ever added (never subtracted) and merge is idempotent, any merge order produces the correct total.

---

**Q19. What is the difference between consistency (the C in CAP) and consistency (the C in ACID)?**
CAP consistency means linearizability — all nodes see the same up-to-date value. ACID consistency means the database moves from one valid state to another (referential integrity, constraints). They are unrelated concepts sharing a name.

---

**Q20. What Google Spanner feature achieves external consistency (stronger than linearizability)?**
TrueTime — GPS and atomic clocks expose a bounded uncertainty interval `[earliest, latest]`. Spanner waits out this uncertainty interval before committing a transaction, ensuring its commit timestamp is globally after all previously committed transactions. External consistency = linearizability across globally distributed shards.
