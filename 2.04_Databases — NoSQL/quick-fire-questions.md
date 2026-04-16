# Databases — NoSQL — Quick-Fire Questions

**Format:** Answer out loud, then check.

---

**Q1: Name the four NoSQL database families.**
Key-Value, Document, Column-Family (Wide-Column), Graph.

---

**Q2: What is Redis's primary data structure and what makes it fast?**
In-memory store. Data lives in RAM, so all reads/writes skip disk entirely — sub-millisecond latency. Redis also uses single-threaded I/O with an event loop, avoiding lock contention.

---

**Q3: What does LSM tree stand for, and which databases use it?**
Log-Structured Merge Tree. Used by Cassandra, HBase, RocksDB, LevelDB. Optimized for write-heavy workloads via sequential appends (Memtable → SSTable).

---

**Q4: Why is Cassandra described as "AP" in CAP theorem?**
By default, Cassandra prioritizes Availability and Partition Tolerance over Consistency. With consistency level ONE, any available node responds even if others are down — but may return stale data.

---

**Q5: What is a hot partition in Cassandra, and how do you fix it?**
A hot partition is when one partition key receives disproportionately more traffic than others, overwhelming the node that owns it. Fix: add a time bucket or random suffix to the partition key to spread load across multiple nodes.

---

**Q6: What does QUORUM mean in Cassandra?**
QUORUM = (Replication Factor / 2) + 1 nodes must acknowledge a write or agree on a read. With RF=3, QUORUM=2. When both write and read use QUORUM, the overlap guarantees strong consistency.

---

**Q7: What is the MongoDB aggregation pipeline?**
A sequence of stages that transform documents — similar to SQL's GROUP BY, JOIN, and projection. Common stages: `$match` (filter), `$group` (aggregate), `$sort`, `$project` (reshape), `$lookup` (join from another collection).

---

**Q8: What is a DynamoDB Global Secondary Index (GSI)?**
A GSI is an alternate index with a different partition key (and optional sort key), allowing queries on non-primary-key attributes. You define which attributes to project into the GSI. GSIs have their own capacity and are maintained automatically.

---

**Q9: What is the DynamoDB single-table design pattern?**
Storing multiple entity types in one table, differentiating them by SK prefix (e.g., `PROFILE`, `ORDER#date`, `SESSION#id`). Items sharing the same PK can be fetched together in a single query, eliminating joins and reducing round trips.

---

**Q10: What is the document size limit in MongoDB?**
16MB per document.

---

**Q11: What is a TTL index in MongoDB?**
An index on a date field with `expireAfterSeconds` set. MongoDB automatically deletes documents once the date field value + expiry duration = past. Common use case: session documents, OTP records, temporary carts.

---

**Q12: What does the Redis `SET key value NX PX ttl` command do?**
Sets `key` to `value` only if the key does Not eXist (`NX`), with expiry in milliseconds (`PX`). This is the atomic primitive for distributed locks — only one caller succeeds; all others get NULL.

---

**Q13: What is read amplification in LSM trees?**
Because data is spread across multiple SSTable files (Memtable + L0..LN), a read must check multiple files until it finds the key. Bloom filters mitigate this by quickly ruling out SSTables that don't contain the key.

---

**Q14: What is the Cassandra hinted handoff?**
When a target replica is down, the coordinator node stores a "hint" (the write that failed to reach the replica). When the replica comes back online, the coordinator replays the hint to bring it up to date.

---

**Q15: When would you use DynamoDB over Cassandra?**
When you want zero operational overhead (fully managed), run on AWS, need auto-scaling with pay-per-request pricing, or have serverless/unpredictable traffic. Cassandra is preferred when you need multi-DC control, very high write throughput, or are not on AWS.

---

**Q16: What is a Bloom filter in Cassandra?**
A probabilistic data structure that tells you with certainty that a key is NOT in an SSTable (skip it) or might be in it (check it). Bloom filters have a small false-positive rate (~1%) but dramatically reduce read amplification.

---

**Q17: What is BASE?**
Basically Available — the system always responds (may be stale). Soft state — state may change without input (due to replication). Eventually consistent — the system will converge to a consistent state given enough time.

---

**Q18: What is the difference between embedding and referencing in MongoDB?**
Embedding: sub-documents are nested inside the parent document — fast reads (one fetch), but the document grows with every update. Referencing: store only an ID reference, fetch the related document separately — avoids unbounded growth but requires two round trips or `$lookup`.

---

**Q19: What Redis data structure would you use for a real-time leaderboard?**
Sorted Set (`ZSET`). `ZADD leaderboard score member` to update; `ZREVRANGE leaderboard 0 9 WITHSCORES` to get top 10; `ZREVRANK leaderboard member` for a user's rank — all O(log N).

---

**Q20: Name two Cassandra failure modes and how Cassandra handles them.**
1. **Node failure**: Reads/writes are routed to other replicas holding the same data (replication factor ensures redundancy). Hinted handoff stores missed writes for replay when node recovers.
2. **Hot partition**: One node overloaded while others are idle. Cassandra detects this imperfectly; prevention requires good schema design (time-bucketed partition keys).
