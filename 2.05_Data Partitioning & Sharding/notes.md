# Data Partitioning & Sharding — Notes & Reference

## Sharding Strategy Cheat Sheet

| Strategy | Routing | Range queries | Add/remove node | Hotspot risk |
|---|---|---|---|---|
| Range-based | Range map | Fast (single shard) | Easy but creates hotspot | High (latest range) |
| Hash-mod | `hash(key) % N` | Scatter-gather | Hard (80% migration) | Low |
| Consistent hashing | Ring lookup | Scatter-gather | Easy (~1/N migration) | Low |
| Directory-based | Shard map lookup | Depends on index | Easy (update map) | Lookup service |

---

## Shard Key Selection Rules

```
Good shard key:
  ✓ High cardinality (thousands+ distinct values)
  ✓ Uniform distribution (no one value >> others)
  ✓ Query alignment (most queries filter by this key)
  ✓ Immutable (never changes after creation)

Bad shard key:
  ✗ Low cardinality (gender, status, boolean)
  ✗ Monotonically increasing without randomness (auto-increment int + range sharding)
  ✗ Skewed distribution (country: 40% "US")
  ✗ Mutable (email, username — people change these)
```

---

## Hotspot Shard Solutions

| Problem | Solution |
|---|---|
| Celebrity write hotspot | Add random prefix: `shard_key = rand(1,100):entity_id` |
| Recent data hotspot | Consistent hashing + many vnodes; separate hot shard |
| Large tenant in multi-tenant SaaS | Dedicated physical node for large tenant |
| Geographic skew | Geo-based routing (North America, Europe, Asia shards) |

---

## Cross-Shard Query Patterns

```
1. Always include shard key in WHERE clause
   → Single shard lookup: O(1) shards

2. Fan-out and merge (scatter-gather)
   → Query all N shards in parallel, merge results
   → Latency = slowest shard, cost = N × query cost

3. Secondary index shard
   → Separate node maps secondary key → primary shard
   → 2 hops: index lookup → target shard

4. Offline analytical DB
   → Replicate all shards into a single analytics DB (BigQuery, ClickHouse)
   → Cross-shard joins happen in analytical layer, not OLTP layer
```

---

## Virtual Nodes (Vnodes) Explained

```
Without vnodes: 3 nodes → each owns a contiguous 1/3 of ring
  Problem: one large arc = one node handles all keys in that range

With 150 vnodes per node: 3 nodes → 450 positions on ring
  Each node owns ~150 scattered mini-arcs
  → Better load balance even with unequal machines
  → Removing a node: its 150 mini-arcs are distributed among ALL remaining nodes
  → Adding a node: steals mini-arcs from ALL existing nodes
```

---

## Resharding Strategies Compared

| Method | Downtime | Complexity | When to use |
|---|---|---|---|
| Hash-mod change | High (full migration) | Low | Never in production |
| Double shards (N → 2N) | Low (dual-write) | Medium | Planned growth |
| Consistent hashing add node | Zero | Low | Always preferred |
| Virtual shard reassignment | Zero | Medium | Cassandra/DynamoDB style |

---

## Real-World Shard Architectures

### Cassandra
- Partition key → MurmurHash → token → vnode → physical node
- 256 vnodes per node by default
- No central routing node — all nodes know the ring (gossip protocol)

### DynamoDB
- AWS manages sharding transparently
- Partition key is your "shard key"
- Hot partition → DynamoDB auto-splits if metrics exceed threshold

### Vitess (MySQL sharding)
- Pre-splits into many tablets (virtual shards) — e.g., 4096 tablet shards
- Each tablet shard = MySQL instance
- VTGate routes queries based on tablet-to-shard mapping
- Used by YouTube, Slack, GitHub

### MongoDB
- Config servers store chunk metadata (shard ranges)
- Mongos is the query router
- Supports both range and hash sharding per collection

---

## Data Locality and Co-location

For `JOIN`-heavy workloads, shard related data by the same key:

```
User posts:     shard by user_id
User comments:  shard by user_id
User followers: shard by user_id

→ All data for user_id=42 on the same shard
→ "Give me all posts + comments for user 42" = single-shard query, no cross-shard join
```

**Cassandra compound partition key** achieves this:
```
PRIMARY KEY ((user_id), created_at)
→ All rows for user_id are always on the same node
→ created_at is a clustering key for ordering within the partition
```

---

## Key Numbers

| Metric | Value |
|---|---|
| Single PostgreSQL practical write limit | ~20K–50K writes/sec |
| Single Cassandra node throughput | ~50K–100K writes/sec |
| Consistent hashing migration on +1 node | ~1/N of keys |
| Virtual nodes per physical node (Cassandra) | 256 (default) |
| Typical initial shard count | 8–64 (with vnodes you can grow) |
| Cross-shard scatter-gather overhead | N × per-shard query time |
| Directory-based sharding lookup latency | 1 extra hop (~1ms) |
| Hot partition DynamoDB threshold | 1,000 WCU/sec per partition |
