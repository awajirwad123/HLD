# Consensus Algorithms — Quick-Fire Questions

**Q1: What problem does consensus solve in distributed systems?**

Multiple nodes need to agree on a single value (who is the leader, what is the next committed log entry) despite failures. Without consensus, two nodes can independently believe they're the leader ("split brain"), causing divergent state. Consensus ensures at most one leader per term and that committed decisions are durable.

---

**Q2: What is a quorum and why is a majority specifically required?**

A quorum is `floor(N/2) + 1` — a majority of nodes. The majority requirement guarantees that any two quorums share at least one node. That shared node carries information from both groups, preventing two groups from making contradictory decisions independently. With a simple majority of N, you can tolerate up to `floor((N-1)/2)` node failures.

---

**Q3: Name the three Raft node states and the transition conditions.**

**Follower:** passive, receives heartbeats from the leader. Transitions to Candidate if election timeout fires (no heartbeat received). **Candidate:** requests votes from all peers. Transitions to Leader if a quorum grants votes; back to Follower if it discovers a higher term. **Leader:** replicates log, sends heartbeats. Transitions to Follower if it sees a higher term in any response.

---

**Q4: Why are Raft election timeouts randomized?**

If all followers had the same timeout, they would all become candidates simultaneously when the leader crashed → split vote → no majority → new election with same problem. Randomized timeouts (e.g., 150–300ms) mean one node times out first, sends vote requests, and often wins before others even start an election.

---

**Q5: How does Raft guarantee that a committed log entry is never lost?**

An entry is committed only after a quorum writes it to their logs. A candidate can only win an election if its log is at least as up-to-date as any voter's. Since any two quorums share at least one node, a new leader's log must contain all committed entries. Uncommitted entries may be overwritten by a new leader, but committed ones never are.

---

**Q6: What is the difference between Paxos and Raft?**

Both solve consensus. Paxos uses proposal numbers with a two-phase prepare/accept protocol for single-value consensus — it's correct but notoriously hard to implement for a full replicated log. Raft was explicitly designed for understandability: it decomposes consensus into three well-defined sub-problems (leader election, log replication, safety) and has a complete spec for the replicated log. Production systems (etcd, CockroachDB) use Raft; Paxos variants appear in ZooKeeper (ZAB) and Spanner.

---

**Q7: How is etcd used for leader election in a microservices cluster?**

Each service instance tries to create an etcd key with a lease (TTL). Only one write succeeds (etcd's linearizability guarantees this via Raft). The instance that successfully writes considers itself the primary. It must periodically renew the lease. If it crashes, the lease expires → other instances race to acquire the key → new primary elected.

---

**Q8: Why do consensus systems sacrifice availability during a network partition?**

Consensus systems are CP (CAP theorem: Consistent + Partition Tolerant). During a partition, the minority group can't reach quorum → it refuses reads and writes. This prevents the minority from making decisions that could conflict with the majority side. Accepting writes on both sides would lead to split brain. The trade-off: brief unavailability vs. permanent data inconsistency.

---

**Q9: What is FLP impossibility and why doesn't it stop practical systems?**

FLP (1985): in a purely asynchronous system where messages can be delayed indefinitely, no deterministic consensus algorithm is guaranteed to terminate if even one process can fail. In practice: systems assume partial synchrony — message delays are bounded in normal operation. Raft uses heartbeat timeouts to detect failures. If the network is well-behaved, Raft always makes progress. FLP assumes worst-case infinite message delays.

---

**Q10: Why is an even number of nodes a bad choice for a Raft cluster?**

With 4 nodes, quorum = 3. Failures tolerated = 1. Same as 3 nodes (quorum = 2, tolerates 1 failure). You pay for an extra node without gaining fault tolerance. With 4 nodes in a network partition of 2|2, neither side can reach quorum → cluster becomes unavailable. With 3 nodes in a 2|1 partition, the majority side stays available. Always use odd cluster sizes.

---

**Q11: What is "log compaction" in Raft and why is it needed?**

Over time, the Raft log grows unboundedly. A new node joining the cluster would need to replay the entire log from the beginning — impractical if the log is millions of entries. Log compaction (snapshotting) saves the current application state at a specific commit index and discards all log entries before that index. New nodes get the snapshot + only recent log entries to replay.

---

**Q12: How does Kafka's KRaft (Raft for Kafka) differ from using ZooKeeper?**

Old Kafka used ZooKeeper for controller election and metadata storage. Problems: separate ZooKeeper cluster to manage, extra operational burden, ZooKeeper became a bottleneck for partition metadata at large scale. KRaft (Kafka 2.8+) brings Raft directly into Kafka: one of the Kafka brokers becomes the active controller via Raft election. Metadata (partition assignments, configs) is stored in an internal Kafka topic (`__cluster_metadata`) replicated via Raft. Eliminates ZooKeeper dependency entirely.
