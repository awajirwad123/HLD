# Data Partitioning & Sharding — Architecture

## Why Partition Data?

A single database node has limits:
- Storage: ~10–20TB practical maximum
- Write throughput: ~10K–50K writes/sec for a typical RDBMS
- Read throughput: saturates with concurrent connections

Partitioning splits data across multiple nodes so each node handles a subset, scaling both storage and throughput horizontally.

---

## Partitioning vs Sharding

| Term | Meaning |
|---|---|
| **Partitioning** | Dividing data within a single DB (e.g., table partitions by date) |
| **Sharding** | Distributing data across multiple DB nodes/servers |

In practice, "sharding" usually means horizontal partitioning across nodes — the industry uses both terms interchangeably for cross-node splitting.

---

## Horizontal vs Vertical Partitioning

### Horizontal Partitioning (the main focus)

Split **rows** across nodes. Each shard has the same schema, different rows.

```
Users table (10M rows):
  Shard 1: user_id 1 – 2,500,000
  Shard 2: user_id 2,500,001 – 5,000,000
  Shard 3: user_id 5,000,001 – 7,500,000
  Shard 4: user_id 7,500,001 – 10,000,000
```

### Vertical Partitioning

Split **columns** across nodes. Different tables (or column groups) live on different servers.

```
User profile service DB:  user_id, name, email, bio
User activity service DB: user_id, last_login, post_count, follower_count
```

Vertical partitioning is essentially microservice decomposition at the DB layer. It reduces the columns-per-query each node handles but doesn't solve the row-count scaling problem.

---

## Sharding Strategies

### 1. Range-Based Sharding

Each shard owns a contiguous range of the shard key.

```
Shard key: created_at (date)
  Shard 1: Jan 2020 – Jun 2020
  Shard 2: Jul 2020 – Dec 2020
  Shard 3: Jan 2021 – Jun 2021
```

**Pros:** Simple, range queries are fast (scan one or a few shards).

**Cons:** Hotspot problem — recent data all lands on the latest shard ("last shard gets all writes"). Uneven storage growth.

**Use case:** Time-series data where old data is cold and recent data is hot (requires special handling).

---

### 2. Hash-Based Sharding

```
shard_id = hash(shard_key) % num_shards
```

```
shard_id = SHA256("user_12345") % 4  →  shard 2
```

**Pros:** Uniform distribution (no hotspots), simple to implement.

**Cons:** Adding or removing shards changes the modulo → massive data migration required. Range queries touch all shards (scatter-gather).

**Mitigation:** Consistent hashing avoids the migration problem (see T20).

---

### 3. Directory-Based Sharding

A lookup service (shard map) maps keys to shards.

```
ShardDirectory table:
  key_range / user_id → shard_node

Query: "Which shard has user 42?" → lookup → Shard 3
```

**Pros:** Most flexible — can rebalance shards arbitrarily, move data with no key recalculation.

**Cons:** Lookup service is a single point of failure + bottleneck. Extra network hop per query. Complex to maintain.

**Use case:** MongoDB config servers, Vitess's vtgate/vtctld.

---

### 4. Consistent Hashing

(See T20 for full detail.) A ring of 2^32 virtual positions. Nodes own segments of the ring. Adding/removing a node only migrates ~1/N of the data.

```
Ring positions: 0 ──────────────── 2^32
Node A: owns [0, 2^30)
Node B: owns [2^30, 2^31)
Node C: owns [2^31, 3×2^30)
Node D: owns [3×2^30, 2^32)

key = SHA256("user_42") % 2^32 = 1.5×2^30 → lands in Node B's range
```

Real systems use virtual nodes (vnodes): each physical node gets 100–200 ring positions to achieve even balance.

---

## Choosing a Shard Key

The shard key is the most critical design decision. A bad shard key causes hotspots and kills performance.

| Property | Goal |
|---|---|
| High cardinality | Many distinct values → even distribution |
| Even distribution | No value much more frequent than others |
| Query alignment | Queries that filter by this key hit one shard, not all |
| Immutable | Shard key value should never change (or resharding = data migration) |

### Bad shard keys

| Key | Problem |
|---|---|
| `status` (active/inactive) | Low cardinality: 2 shards max |
| `country_code` | Uneven: "US" gets 40% of traffic |
| `created_at` (range) | Hotspot on "today" shard |
| `user_type` (free/paid) | Most users are free → imbalanced |

### Good shard keys

| Key | Why it works |
|---|---|
| `user_id` (UUID or Snowflake) | High cardinality, uniform distribution |
| `tenant_id` (for SaaS) | Tenant-level isolation, each tenant is a "shard" |
| `order_id` (hash) | Uniform, queries are usually by exact order ID |

---

## The Hotspot Problem

Even with a good shard key, hotspots occur:

**Celebrity problem:** User with 50M followers has their user_id on one shard. Every follower action hits that shard.

**Time-of-day burst:** All users log in at 9am → shard 3 (whose range includes IDs 1M–2M) handles the "popular user" load spike.

**Solutions:**
1. **Add prefix to key:** `shard_key = random(1,100) + ":" + user_id` — spreads one logical key across 100 sub-shards. Application must query all 100 and merge.
2. **Dedicated shard:** Move hot users/tenants to their own shard.
3. **Caching layer:** Front the hot user's data with Redis to absorb reads before they hit the DB shard.

---

## Cross-Shard Queries: The Big Trade-off

With hash-based sharding, a query without the shard key hits **all shards** (scatter-gather):

```sql
-- FAST (shard key in WHERE clause):
SELECT * FROM orders WHERE user_id = 42;
-- → hits exactly 1 shard

-- SLOW (no shard key, scatter-gather all N shards):
SELECT * FROM orders WHERE status = 'pending' AND total > 1000;
-- → queries all N shards, merge results in application
```

**Solutions:**
1. **Design queries to always include the shard key** — most lookups go by entity ID.
2. **Maintain a secondary index** — a separate index node maps `status → list of shard locations`.
3. **Denormalize into a query-optimized data store** — write to Elasticsearch or a reporting DB for cross-shard analytic queries.
4. **Fan-out and merge** — scatter to all shards, merge results. Acceptable for low-traffic analytical queries.

---

## Resharding: What Happens When Shards Fill Up?

With simple hash-mod sharding:
- Add shard 5 → change `% 4` to `% 5` → 80% of data maps to wrong shard → massive migration.

**Approaches to resharding:**

1. **Consistent hashing:** Adding a node only migrates ~20% of data (1/N).
2. **Double the shards:** Go from 4 → 8 shards. Each existing shard splits into 2. Half its data migrates to the new sibling shard.
3. **Use virtual shards:** Preallocate 1024 virtual shards, mapped to N physical nodes. When you add a node, move some virtual shards (not all data) to it. Cassandra/DynamoDB use this pattern.
4. **Online migration:** Use dual-write (write to old+new shard simultaneously) + background backfill + cutover. Zero-downtime resharding.

---

## Consistent Hashing for Dynamic Shard Count

```
Physical  Virtual ring positions
Node A  → positions 15, 180, 320, ...  (100 vnodes)
Node B  → positions 45, 200, 360, ...
Node C  → positions 90, 240, 400, ...

Add Node D: it gets 100 new virtual positions, ~25% of ring
→ Each of A, B, C gives up ~8% of their positions to D
→ Only ~25% of total keys migrate (vs 80% with hash-mod)
```

---

## Real-World Sharding Examples

| System | Sharding strategy | Shard key |
|---|---|---|
| Instagram | PostgreSQL logical sharding | user_id |
| Twitter (older) | MySQL hash sharding | tweet_id, user_id |
| Cassandra | Consistent hashing (vnodes) | partition key hash |
| DynamoDB | Internal consistent hashing | partition key |
| MongoDB | Range or hash sharding | configured by user |
| Vitess (MySQL) | Virtual shards (pre-split) | user_id or tenant_id |

---

## Key Numbers

| Metric | Value |
|---|---|
| Practical single-node DB limit | 5–20TB, 50K writes/sec |
| Consistent hashing migration on +1 node | ~1/N of data |
| Typical virtual nodes per physical node | 100–256 |
| Cross-shard fan-out penalty | O(N shards) latency × serial or parallel |
| Resharding downtime (dual-write approach) | 0 if done correctly |
| Shard count typical starting point | 8–64 (can grow with vnodes) |
