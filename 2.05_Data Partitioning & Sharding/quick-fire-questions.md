# Data Partitioning & Sharding — Quick-Fire Questions

**Q1: What is the difference between horizontal and vertical partitioning?**

Horizontal: split rows across nodes — each shard has the same schema but different rows. Vertical: split columns — different tables or column groups live on different nodes (essentially DB-level service decomposition). Sharding usually means horizontal partitioning.

---

**Q2: What is the "hotspot" problem in range-based sharding?**

With range-based sharding by timestamp, all new writes go to the "latest" shard. This shard handles 100% of writes while older shards are cold. The system is not horizontally scaled for writes — it just distributed storage. Fix: use hash-based or consistent hashing so new keys are distributed uniformly.

---

**Q3: Why is hash-mod sharding (`hash(key) % N`) problematic when you add a node?**

Changing N from 4 to 5 changes the modulo for approximately 4/5 = 80% of all keys. Those keys now hash to a different shard, requiring a massive data migration. Consistent hashing solves this: adding one node migrates only ~1/N of data.

---

**Q4: How does consistent hashing reduce data migration?**

Keys and nodes are mapped to positions on a 2^32 ring using a hash function. Each node owns the arc between its position and the next node clockwise. Adding a node only "steals" keys from the immediate neighbor, not the entire ring. With virtual nodes, each physical node gets 100–256 scattered positions, ensuring even distribution across the ring.

---

**Q5: What are virtual nodes and why are they needed?**

With only one ring position per physical node, adding/removing nodes causes uneven load splits (one arc may be much larger than another). Virtual nodes give each physical node many small, scattered positions on the ring, spreading its key range evenly. When a node is added or removed, its responsibility is divided across all other nodes, maintaining balance.

---

**Q6: What makes a bad shard key? Give examples.**

Low cardinality: `user_type` (free/paid) → only 2 possible shards. Skewed distribution: `country` → 40% of users are "US" → one shard is overloaded. Monotonically increasing `auto-increment id` with range sharding → all current writes hit the highest-range shard (same hotspot as timestamp). Mutable key: using `email` as shard key means when a user changes their email, data must move to a new shard.

---

**Q7: How do you handle cross-shard queries?**

Main patterns: (1) Scatter-gather — send the query to all N shards in parallel, merge results; cost = N × per-shard query. (2) Secondary index shard — a dedicated index node maps the non-shard-key column to primary shard locations; 2-hop lookup. (3) Denormalize to an analytics store (BigQuery, ClickHouse, Elasticsearch) for complex multi-dimensional queries. The best long-term solution is to design queries to always include the shard key.

---

**Q8: What is directory-based sharding?**

A separate lookup service (shard directory) maps keys or key ranges to specific shard nodes. Unlike hash-based sharding, routing is explicit and configurable — you can move data freely by updating the directory. Pros: maximum flexibility. Cons: the directory is a single point of failure + adds a network hop per query. MongoDB's config servers and Vitess's vtgate implement this pattern.

---

**Q9: What does "data co-location" mean in sharding?**

Storing related data on the same shard so that common JOINs (or reads) require only one shard lookup. Example: shard both `posts` and `comments` by `user_id` → fetching all posts and comments for a user is a single-shard query with no cross-shard join. Cassandra's compound partition key enables this explicitly.

---

**Q10: How do you reshard with zero downtime?**

Dual-write migration: (1) Start writing to both old and new shard layout simultaneously. (2) Background backfill copies existing data from old shards to new. (3) Gradually shift reads to new layout as data arrives. (4) Verify consistency, then cut over writes fully. (5) Decommission old shards. This avoids a big-bang migration and requires no downtime.

---

**Q11: What is the celebrity/hotspot problem in social media sharding and how do you solve it?**

A celebrity's user_id lands on one shard. Every action involving that user (followers looking at posts, likes, comments) hits that single shard, overloading it. Solutions: (1) Add a random salt prefix to the shard key (`rand(1,100):user_id`) — spreads reads across 100 sub-shards, but reads must query all 100 and aggregate. (2) Move hot entities to a dedicated "VIP shard" with more resources. (3) Cache-heavy: put Redis in front of the hot shard so most reads never reach the DB.

---

**Q12: Cassandra vs. DynamoDB — how does each handle sharding?**

Cassandra: uses consistent hashing with 256 virtual nodes per physical node. The partition key is hashed via MurmurHash to find the owning node. All nodes know the full ring (gossip protocol) — no central routing node. DynamoDB: AWS manages partitioning transparently based on partition key hash. DynamoDB automatically splits hot partitions if they exceed 1,000 WCU/sec. Users don't control shard placement directly.
