# Databases — NoSQL — Notes & Cheat Sheets

## The Four NoSQL Families — One-Line Each

| Family          | Model                         | Primary Access       | Representatives                    |
|-----------------|-------------------------------|----------------------|------------------------------------|
| Key-Value       | Opaque value by key           | GET/SET by key only  | Redis, DynamoDB (KV), Memcached    |
| Document        | JSON/BSON documents           | Any indexed field    | MongoDB, Firestore, DocumentDB     |
| Column-Family   | Rows → sorted columns         | Partition key (+ clustering key range) | Cassandra, HBase, Bigtable |
| Graph           | Nodes + Edges + Properties    | Traversal by relationship | Neo4j, Neptune, JanusGraph    |

---

## B-Tree vs LSM Tree — Write vs Read Optimization

| Property           | B-Tree (PostgreSQL, MySQL)      | LSM Tree (Cassandra, RocksDB)       |
|--------------------|----------------------------------|-------------------------------------|
| Write path         | In-place update (random I/O)     | Append to Memtable → SSTable (sequential) |
| Read path          | O(log N) — single index lookup   | Check Memtable + multiple SSTables (read amplification) |
| Write throughput   | Moderate (bounded by random I/O) | Very high (sequential append)       |
| Read latency       | Low for random access            | Higher for random access; Bloom filters help |
| Space efficiency   | Efficient                        | Write amplification from compaction |
| Best for           | OLTP read-heavy                  | Write-heavy, append-only workloads  |

---

## Cassandra Consistency Levels

| Level       | Write must reach  | Read must agree | Use when                              |
|-------------|-------------------|-----------------|---------------------------------------|
| ONE         | 1 node            | 1 node          | Fastest; occasional stale reads ok    |
| QUORUM      | RF/2 + 1 nodes    | RF/2 + 1 nodes  | Strong consistency with W+R > RF      |
| LOCAL_QUORUM| Quorum in local DC| Local DC quorum | Multi-region; avoid cross-region RTT  |
| ALL         | All replicas      | All replicas    | Strongest; any node failure = failure |
| ANY         | 1 (can be hinted) | N/A             | Highest availability; stale possible  |

**Strong consistency rule:** Write CL + Read CL > Replication Factor

With RF=3: QUORUM (2) + QUORUM (2) = 4 > 3 → Strong ✅

---

## Cassandra Partition Key Design Rules

```
Rule 1: Partition key = access pattern anchor
  → What is the main WHERE clause? That's your partition key.

Rule 2: Avoid hot partitions
  → If one key gets >> others (e.g., "trending topic"), add a time bucket suffix
  → Bad:  chat_id = "popular_group"  (one node = all traffic)
  → Good: (chat_id, bucket) = ("popular_group", "2024-01")

Rule 3: Avoid unbounded partitions
  → A partition that grows forever eventually kills the node
  → Cap by time: messages per chat PER MONTH, not all messages ever

Rule 4: Clustering key = sort order within partition
  → Use TIMESTAMP or UUID (time-ordered) as clustering key for time-series
  → Queries become: WHERE partition_key = ? AND clustering_key > ?
```

---

## MongoDB vs Cassandra vs DynamoDB — Cheat Sheet

| Question                     | MongoDB          | Cassandra        | DynamoDB             |
|------------------------------|------------------|------------------|----------------------|
| Ad-hoc queries?              | ✅ Yes           | ❌ Only by PK    | ❌ Only by PK+SK     |
| Flexible schema?             | ✅ Per document  | ⚠️ Per row       | ✅ Per item          |
| ACID multi-doc?              | ✅ (v4+)         | ⚠️ Single partition LWT | ✅ TransactWrite (25 items) |
| Write scale?                 | Good             | Excellent        | Excellent (managed)  |
| Operational overhead?        | Medium (self-hosted) | High (cluster mgmt) | Zero (managed) |
| Query language?              | MQL + aggregation pipeline | CQL (SQL-like) | DynamoDB API / PartiQL |
| Multi-region?                | Global Clusters (Atlas) | Multi-DC natively | Global Tables |
| Full-text search built-in?   | ✅ Text indexes  | ❌ (use Elasticsearch) | ❌ (use OpenSearch) |
| Best for?                    | Flexible document workloads | Time-series, chat, IoT | Serverless, AWS-native |

---

## DynamoDB Single-Table Design Pattern

```
The goal: put all related data under the same PK to enable
single-query reads of a parent entity + all its children.

Entity    PK              SK
User      USER#u123       PROFILE
Order     USER#u123       ORDER#2024-01-15#ord_abc
OrderItem USER#u123       ORDER#2024-01-15#ord_abc#ITEM#0
Session   USER#u123       SESSION#sess_xyz

Query: "Give me user profile + all orders"
  → KeyConditionExpression: PK = 'USER#u123'
  → Returns PROFILE + all ORDER# rows in one request

GSI design (access patterns that cross entities):
  GSI1PK = STATUS#pending → find all pending orders across all users
```

---

## ACID vs BASE

| Property  | ACID (SQL, mostly)                     | BASE (NoSQL, mostly)                          |
|-----------|----------------------------------------|-----------------------------------------------|
| Focus     | Correctness — strong guarantees        | Availability — always respond                 |
| Consistency | Immediate, strong                    | Eventual — converges over time                |
| Isolation | Transactions isolated                  | No isolation across entities (usually)        |
| Failure mode | Reject the write if can't guarantee| Accept the write, reconcile later             |
| Trade-off | Lower availability during partitions   | May serve stale reads                         |

**BASE** = Basically Available, Soft state, Eventually consistent

---

## NoSQL Use Case Reference Card

| Use Case                         | Store          | Key Reason                                    |
|----------------------------------|----------------|-----------------------------------------------|
| User sessions                    | Redis          | TTL, sub-ms, simple GET/SET                   |
| Distributed locks                | Redis          | SET NX PX, Lua scripts, atomic                |
| Real-time leaderboard            | Redis          | Sorted sets (ZADD/ZREVRANGE)                  |
| Rate limiting counters           | Redis          | INCR + EXPIRE, atomic operations              |
| Pub/sub fan-out                  | Redis          | PubSub or Streams                             |
| Product catalog, user profiles   | MongoDB        | Flexible schema, rich query                   |
| CMS, event metadata              | MongoDB        | Nested documents, text search                 |
| Chat message history             | Cassandra      | Time-ordered partition, LSM writes           |
| IoT sensor data                  | Cassandra      | Append-only writes, time-bucketed partition   |
| Activity feeds (write volume)    | Cassandra      | Fan-out writes, read by user+time             |
| Simple KV, serverless, AWS       | DynamoDB       | Managed, auto-scale, single-table design      |
| Multi-entity transactions, AWS   | DynamoDB       | TransactWrite up to 25 items                  |
| Social graph, recommendations    | Neo4j          | Edge traversal O(depth) not O(N²)             |

---

## Key Numbers to Memorize

| Fact                                              | Number                    |
|---------------------------------------------------|---------------------------|
| Redis GET/SET latency (single node)               | < 0.5ms                   |
| Cassandra write throughput (single node, SSD)     | ~50–100K writes/sec       |
| Cassandra replication factor (typical)            | 3                         |
| DynamoDB item size limit                          | 400KB                     |
| DynamoDB TransactWrite max items                  | 25                        |
| MongoDB document size limit                       | 16MB                      |
| LSM tree levels (typical)                         | 4–7                       |
| Bloom filter false-positive rate (typical)        | ~1%                       |
| Cassandra partition size warning threshold        | 100MB (soft limit)        |

---

## Interview Answer Checklist — NoSQL Questions

When asked any NoSQL design question, structure your answer around:
1. **Access patterns first** — what queries will run? This determines the schema
2. **Partition key design** — cardinality, hotspot risk, time-bucketing if needed
3. **Consistency requirement** — eventual ok? Or strong needed for specific path?
4. **ACID vs compensating** — does the operation span multiple entities? How to handle failure?
5. **Scale-out model** — replication factor, cross-region, read/write split
6. **Failure modes** — what if a node is down? What is client behavior?
7. **Observability** — what metrics to monitor (replication lag, partition size, cache hit rate)
