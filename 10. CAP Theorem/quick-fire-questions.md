# CAP Theorem — Quick-Fire Questions

**Format:** Answer out loud, then check.

---

**Q1: What does CAP stand for?**
Consistency, Availability, Partition Tolerance.

---

**Q2: Which of the three CAP properties is not optional in a distributed system?**
Partition Tolerance. Network partitions will happen in any real distributed system, so you cannot design a system that doesn't tolerate them. The real choice is between C and A when a partition occurs.

---

**Q3: What does "Consistency" mean in CAP — and how is it different from ACID Consistency?**
CAP Consistency means linearizability: after a write completes, every subsequent read from any node returns that written value — or returns an error, never stale data. ACID Consistency means the database enforces its integrity constraints (FKs, uniqueness rules). They are completely different concepts.

---

**Q4: What does "Availability" mean in CAP precisely?**
Every request made to a non-failing node receives a response — and that response must not be an error. The response may be stale, but it must never be a 503 or timeout.

---

**Q5: What does a CP system do during a network partition?**
It sacrifices availability to preserve consistency. Nodes in the minority partition (those that can't reach a quorum) refuse to serve read or write requests — they return errors rather than risk serving or accepting stale/inconsistent data.

---

**Q6: What does an AP system do during a network partition?**
It sacrifices consistency to preserve availability. All nodes continue serving requests. Nodes in the minority partition respond with potentially stale data. After the partition heals, nodes reconcile their state (eventual consistency).

---

**Q7: Is ZooKeeper CP or AP? Why?**
CP. ZooKeeper uses the ZAB (ZooKeeper Atomic Broadcast) protocol which requires quorum. If fewer than a majority of nodes can communicate, the cluster stops accepting writes and the minority nodes serve reads only or go unavailable.

---

**Q8: Is Cassandra CP or AP by default?**
AP by default (consistency level ONE). Any node responds with its local data. Under a partition, all nodes remain available but may return stale data. With CL=QUORUM on N=3, it becomes effectively CP because W+R=4 > N=3.

---

**Q9: What is PACELC? Why is it more complete than CAP?**
PACELC extends CAP: if Partition → choose A vs C (same as CAP); Else (normal operation) → choose Latency vs Consistency. It captures the everyday trade-off that applies even when there's no partition — most latency vs consistency decisions happen in normal operation, not just during failures.

---

**Q10: Can a distributed system be both Consistent and Available in CAP terms?**
Only when there are no partitions — which you cannot guarantee in a distributed system. During normal operation (no partition), you CAN have both. The theorem says you must choose when a partition occurs.

---

**Q11: MongoDB with w:majority — is it CP or AP?**
CP. A write with w:majority requires acknowledgment from a majority of replica set members before confirming. During a partition, if the client is connected to a minority node, writes will fail rather than succeed without majority confirmation.

---

**Q12: What is the quorum formula that makes a system strongly consistent (CP-like)?**
W + R > N, where W = write quorum, R = read quorum, N = replication factor. This ensures at least one node is guaranteed to have the latest write when you read.

---

**Q13: DynamoDB uses eventual consistency by default. How do you get strong reads?**
Set `ConsistentRead=True` on the read request. This reads from the primary node for that key, not an eventually-consistent replica. It costs 2× read capacity units (RCUs) vs eventual reads.

---

**Q14: etcd is classified as CP. What happens if 2 of 5 etcd nodes fail?**
The cluster still has 3 nodes alive — a majority (3 of 5) — so it continues operating normally (Raft requires majority). If 3 of 5 fail, the remaining 2 nodes cannot form a majority → the cluster stops accepting writes → Kubernetes (which uses etcd) cannot update its state.

---

**Q15: DNS is classified as AP. What consistency anomaly does it allow?**
DNS servers respond with cached records even after the authoritative record has changed. A user may resolve an old IP address until the TTL expires. This is eventual consistency: the stale response is served rather than returning an error.

---

**Q16: What is a "CA system" in CAP terms, and why is it misleading?**
A CA system is Consistent and Available but doesn't tolerate Partitions. This is only possible for a single-node system — which isn't distributed. Any truly distributed system must tolerate partitions. Labeling a distributed system "CA" means it will fail silently during a partition.

---

**Q17: What is linearizability?**
A consistency model where operations appear to take effect atomically at a single point in time, and all observers see a consistent ordering of those operations. This is what CAP's C refers to. After a linearizable write, all subsequent reads from any node see that write.

---

**Q18: How does Cassandra achieve "AP" — what mechanism allows it to always respond?**
Leaderless architecture: any node can act as coordinator and any replica can serve reads/writes. With CL=ONE, the coordinator returns as soon as one node responds. No single point of failure exists, so no partition can make the whole cluster unavailable.

---

**Q19: Why do microservices that make synchronous cross-service calls tend toward CP behavior?**
If Service A calls Service B synchronously and B is unreachable (partition), A's request fails with an error — it sacrifices availability to avoid acting on incomplete/stale information. Async messaging (queues) inverts this to AP: A accepts the request and processes it eventually even if B is unavailable.

---

**Q20: A user updates their profile. The response says "saved." Seconds later they reload and see the old data. Which CAP property was violated and which system model permits this?**
CAP Consistency (linearizability) was violated: after a write, the read returned a stale value. This is permitted in an AP system — the write may have gone to one node and the read returned from a replica that hadn't yet applied the update. This is expected behavior in an eventually-consistent (AP) system and must be handled at the application layer (e.g., optimistic UI update while the backend converges).
