# Databases — NoSQL — Architecture

## Overview

NoSQL ("Not Only SQL") is a family of databases that trade the relational model and full ACID guarantees for specific performance, scale, or flexibility properties. Understanding the four major NoSQL families — and which one to reach for in each scenario — is a core competency at senior-level system design interviews.

---

## 1. The Four NoSQL Families

### Family 1: Key-Value Stores

**Model:** Simple hash map — a key maps to an opaque blob (string, bytes, JSON).

**Characteristics:**
- O(1) read/write by key
- No query language — only GET/SET/DELETE by key
- Values are opaque (the DB doesn't inspect them)
- Optimal for: sessions, caches, feature flags, leaderboards

**Representatives:** Redis, DynamoDB (in KV mode), Memcached, etcd

```
key                     → value
"session:abc123"        → {"user_id": 42, "role": "admin", "exp": 1703001234}
"rate:user:42:2024-01"  → "47"
"config:feature:dark_mode" → "true"
```

**Where it shines:**
- Session storage (sub-millisecond access, TTL support)
- Distributed locks (Redis `SET NX PX`)
- Real-time counters and leaderboards (Redis sorted sets)
- Caching layer in front of SQL (cache-aside pattern)

---

### Family 2: Document Stores

**Model:** Semi-structured documents (JSON/BSON) grouped into collections. Each document has a flexible schema.

**Characteristics:**
- Each document is a self-contained unit (no JOINs needed within a document)
- Documents in the same collection CAN have different fields
- Rich query language (filter, aggregate, project)
- Indexes on any field
- Optimal for: catalogs, user profiles, content management, event data

**Representatives:** MongoDB, CouchDB, Firestore, Amazon DocumentDB

```json
// users collection
{
  "_id": "user_123",
  "email": "alice@example.com",
  "profile": {
    "name": "Alice",
    "bio": "Engineer",
    "skills": ["python", "system-design"]
  },
  "address": {
    "city": "Seattle",
    "country": "US"
  },
  "created_at": "2024-01-15T10:00:00Z"
}
```

**Why no JOINs? — Embedding vs Referencing:**

```
Embed when: child data is always read with parent, bounded size
  → user document embeds address and skills

Reference when: child data is large, shared across documents, or queried independently
  → user document references orders by order_id (orders in separate collection)
```

---

### Family 3: Column-Family Stores (Wide-Column)

**Model:** Data organized by row key → sorted columns. Rows can have different columns. Designed for massive-scale writes and time-series patterns.

**Characteristics:**
- Write path is extremely fast (append-only LSM tree, not B-tree)
- Data is sorted by row key — efficient range scans by key
- Schema is defined logically but columns can vary per row
- Query only by partition key (+ optional clustering key range)
- No JOINs, no ad-hoc queries
- Optimal for: time-series, IoT, message history, activity feeds, analytics write path

**Representatives:** Apache Cassandra, HBase, Google Bigtable, Amazon Keyspaces

```
Table: messages
Row Key (partition_key + clustering_key)    │ Columns
────────────────────────────────────────────┼──────────────────────────────
chat_123 │ 2024-01-15T10:00:00 │ msg_1001  │ text: "hello", sender: user_A
chat_123 │ 2024-01-15T10:01:00 │ msg_1002  │ text: "hi!", sender: user_B
chat_456 │ 2024-01-16T09:00:00 │ msg_2001  │ text: "good morning", sender: user_C
```

**Cassandra's key design rule:** Model your tables around your query patterns, not your data structure. One query pattern → one table.

---

### Family 4: Graph Databases

**Model:** Nodes (entities) and Edges (relationships) with properties on both.

**Characteristics:**
- Traversals follow edges natively — O(1) per hop, regardless of data size
- JOINs in SQL become O(N²) for deep traversals; graph DBs do them in O(depth)
- Specialized query languages: Cypher (Neo4j), Gremlin
- Optimal for: social networks (friends-of-friends), recommendation engines, fraud detection, knowledge graphs

**Representatives:** Neo4j, Amazon Neptune, JanusGraph, ArangoDB

```
(Alice) -[FOLLOWS]→ (Bob) -[FOLLOWS]→ (Charlie)
               ↓
           [LIKED]
               ↓
          (Post: "HLD Notes")

Query: Find all people Alice follows who liked this post
  MATCH (alice:User {name: 'Alice'})-[:FOLLOWS]->(:User)-[:LIKED]->(post:Post {title: "HLD Notes"})
  RETURN post
```

---

## 2. LSM Tree — Why Column-Family Stores Write So Fast

PostgreSQL uses B-trees (read-optimized). Cassandra/HBase use **LSM trees** (write-optimized). Understanding this is a differentiating insight in interviews.

```
Write path (LSM tree):
  1. Write to in-memory buffer (Memtable) — extremely fast
  2. Write to append-only WAL (for durability)
  3. Memtable periodically flushed to disk as immutable SSTable
  4. Background compaction merges SSTables, removes tombstones

Read path (LSM tree):
  1. Check Memtable (newest data)
  2. Check SSTables newest → oldest (use Bloom filter to skip irrelevant files)
  3. Merge results
  → Reads are slower than B-tree for random access

Trade-off:
  LSM tree: writes are O(1) sequential appends — ideal for high-write throughput
  B-tree:   reads are O(log N) random access — ideal for OLTP reads
```

**Bloom filter:** A probabilistic structure that tells you if a key is DEFINITELY NOT in an SSTable (skip it) or MIGHT be in it (check it). Reduces read amplification significantly.

---

## 3. MongoDB Architecture

### Data Model
```
Database → Collection(s) → Document(s) → Fields
MongoDB   →  "users"     → JSON docs   → {"name": ..., "age": ...}
SQL       →  schema      → table       → rows → columns
```

### BSON (Binary JSON)
MongoDB stores documents as BSON — extends JSON with additional types: `Date`, `ObjectId`, `Binary`, `Decimal128`. ObjectId is a 12-byte, time-ordered unique ID (contains timestamp, machine ID, process ID, counter).

### MongoDB Replica Set
```
[ Primary ] ←── writes
     │
     │  Oplog streaming (similar to WAL)
     │
┌────┴────┐
[Secondary 1] [Secondary 2]
← reads (optional) ← reads (optional)
```
- A replica set is 3+ nodes (1 primary + 2+ secondaries)
- Primary elected via Raft-like consensus if current primary fails
- **Read preference:** `primary` (default), `primaryPreferred`, `secondary`, `nearest`

### Indexing in MongoDB
```javascript
// Single field
db.users.createIndex({ email: 1 })  // 1=ascending, -1=descending

// Compound
db.orders.createIndex({ user_id: 1, created_at: -1 })

// Text (full-text search)
db.posts.createIndex({ content: "text" })

// TTL index (auto-delete expired documents)
db.sessions.createIndex({ created_at: 1 }, { expireAfterSeconds: 3600 })
```

---

## 4. Cassandra Architecture

### Cassandra's Distributed Design (No Single Point of Failure)
```
Cluster ring (consistent hashing):
  Node A  (token range 0–25%)
  Node B  (token range 25–50%)
  Node C  (token range 50–75%)
  Node D  (token range 75–100%)

Data with hash(partition_key) = 30% → goes to Node B
Replication Factor = 3 → also stored on Node C and Node D
```

No master, no leader — any node can serve any read or write. This is the key difference from PostgreSQL replication.

### Consistency Levels (Tunable)
```
Write with QUORUM:  write must succeed on (RF/2 + 1) replicas before ack
Read  with QUORUM:  read from (RF/2 + 1) replicas, return newest

With RF=3:
  QUORUM = 2 replicas for both read and write
  → Strong consistency (write + read quorums overlap)

With ONE:
  Write/read from just 1 replica
  → Fastest, but may return stale data (eventual consistency)
```

**Tunable consistency** means you can choose per-query: ALL, QUORUM, LOCAL_QUORUM, ONE. This lets you trade consistency for availability/latency depending on the use case.

### Partition Key Design — Critical
```sql
-- Schema for message history
CREATE TABLE messages (
    chat_id    UUID,
    sent_at    TIMESTAMP,
    message_id UUID,
    text       TEXT,
    sender_id  UUID,
    PRIMARY KEY ((chat_id), sent_at, message_id)
    -- chat_id        = partition key  → determines which node stores the data
    -- sent_at + message_id = clustering keys → determines sort order within partition
);

-- Efficient query (uses partition key):
SELECT * FROM messages WHERE chat_id = ? AND sent_at > ? LIMIT 50;

-- Inefficient query (no partition key → full cluster scan):
SELECT * FROM messages WHERE sender_id = ?;  -- ← requires an index or ALLOW FILTERING
```

### Cassandra Failure Modes to Know

| Failure | Impact | Behavior |
|---------|--------|----------|
| One node down | Reads/writes route to other replicas | Hinted handoff: stores the missed writes temporarily for the failed node |
| Network partition | Nodes on both sides still serve requests (AP by default) | May serve stale reads; repairs on reconnect (read repair, anti-entropy) |
| Hot partition | One partition key gets all traffic | Data skew → that node overloaded; re-design partition key |

---

## 5. DynamoDB Architecture

DynamoDB is AWS's fully managed Key-Value + Document store. It occupies a unique position: managed service (no ops), serverless pricing, and can scale to single-table designs.

### Data Model
```
Table:  Orders
PK  (Partition Key): user_id
SK  (Sort Key):      created_at

Item:
{
  "user_id":    "user_123",            ← PK
  "created_at": "2024-01-15T10:00",   ← SK
  "order_id":   "ord_abc",
  "total":      4999,
  "items":      [{"sku": "BOOK-001", "qty": 2}]
}
```

### Key Design Patterns

**Composite Sort Key for range queries:**
```
PK: user_123
SK: ORDER#2024-01-15 → ORDER#2024-01-20 range query → returns all orders in range
SK: PROFILE         → returns user profile
SK: FRIEND#user_456 → returns friendship with user_456
```

**Single-Table Design**: Multiple entity types in one table, differentiated by SK prefix. Eliminates JOINs by co-locating related data under the same PK.

### DynamoDB vs Cassandra vs MongoDB

| Dimension        | DynamoDB                    | Cassandra                    | MongoDB                    |
|------------------|-----------------------------|------------------------------|----------------------------|
| Hosting          | Fully managed AWS           | Self-hosted / Astra          | Atlas / self-hosted        |
| Data model       | KV + Document               | Wide-column                  | Document                   |
| Primary access   | PK + optional SK            | Partition key + clustering   | Any indexed field          |
| Consistency      | Eventually consistent (default), strong option | Tunable (ONE/QUORUM/ALL) | Tunable read preference |
| Transactions     | ✅ Transact writes (up to 25 items) | ✅ Lightweight transactions (LWT) single partition | ✅ Multi-doc ACID (4.0+) |
| Schema           | Schema-less (item-level)    | Table schema required        | Schema-less per document   |
| Best for         | Serverless, AWS-native apps | Extreme write scale, time-series | Flexible document workloads |
| Avoid when       | Complex queries, reporting  | Multi-partition transactions | Full ACID across documents |

---

## 6. When to Choose NoSQL vs SQL

```
Start with PostgreSQL. Switch when you hit a specific, measurable limit.
```

| Use Case                                          | Recommendation             | Reason                                               |
|---------------------------------------------------|----------------------------|------------------------------------------------------|
| User profiles, orders, products (relational)      | PostgreSQL                 | JOINs, ACID, complex queries                         |
| Session storage, caching, rate-limit counters     | Redis                      | Sub-ms access, TTL, atomic operations                |
| Chat message history, activity feeds              | Cassandra                  | Time-series write scale, range queries by time       |
| Product catalog with varying attributes           | MongoDB                    | Flexible schema per document, rich queries           |
| IoT sensor data at millions of events/sec         | Cassandra / TimescaleDB    | Append-only write scale, time-based partitioning     |
| Friend graph, recommendation engine               | Neo4j / Neptune            | Graph traversal at depth                             |
| Serverless, unpredictable load, AWS ecosystem     | DynamoDB                   | Auto-scaling, pay-per-request, managed               |
| Reporting, analytics aggregations                 | PostgreSQL / Redshift       | GROUP BY, window functions, SQL familiarity          |
| Full-text search as primary access pattern        | Elasticsearch              | Inverted index, relevance scoring                    |

---

## 7. Common Failure Modes

| Failure                          | System     | Root Cause                                  | Fix                                             |
|----------------------------------|------------|---------------------------------------------|-------------------------------------------------|
| Hot partition                    | Cassandra  | Partition key has low cardinality           | Add time bucket or UUID suffix to partition key  |
| Unbounded document growth        | MongoDB    | Array in document keeps growing             | Cap array size; use separate collection instead  |
| Missing index causes COLLSCAN     | MongoDB    | Query has no matching index                 | Run `explain("executionStats")`, add index      |
| Read amplification               | Cassandra  | Reading many SSTables before compaction     | Tune compaction strategy; add Bloom filter size |
| WCU/RCU throttling               | DynamoDB   | Traffic spike exceeds provisioned capacity  | Switch to on-demand; or add DAX (cache)         |
| Stale read after write           | Any NoSQL (AP) | Eventual consistency window              | Use strong/quorum reads for critical paths      |
