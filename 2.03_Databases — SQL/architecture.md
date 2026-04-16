# Databases — SQL — Architecture

## Overview

Relational databases (RDBMS) are the backbone of most production systems. PostgreSQL and MySQL power the majority of transactional workloads at companies of every scale. Knowing their internals — indexes, ACID, replication, connection pooling — is mandatory for senior-level interviews.

---

## 1. RDBMS Internals

### How Data is Stored on Disk

PostgreSQL and MySQL store data in **heap files** — pages of 8KB each. Every table row lives in a page. When you query, the DB reads pages from disk into the **buffer pool** (shared RAM cache).

```
Disk:
  Table file = [Page 1][Page 2]...[Page N]
  Each page = 8KB = ~400 rows

Buffer Pool (RAM):
  Hot pages cached here → reads served from memory
  Cold pages evicted when pool is full (LRU)

Index files:
  Separate file per index, also stored as pages
```

**Why this matters in interviews:**
- A query that fits in the buffer pool = fast (sub-millisecond)
- A full table scan that can't fit = slow (disk I/O, milliseconds to seconds)
- DB sizing: buffer pool should hold your hot dataset. Rule of thumb: 25–40% of RAM for `shared_buffers` in PostgreSQL.

---

### B-Tree Index — The Most Important Data Structure

Almost every index in PostgreSQL/MySQL is a B-tree.

```
               [ Root: 50 ]
              /              \
      [ 20 | 40 ]          [ 70 | 90 ]
      /    |    \           /    |    \
 [10,15] [25,30] [45,48] [60,65] [75,82] [92,99]
   ↓        ↓       ↓       ↓       ↓       ↓
 heap    heap    heap    heap    heap    heap
 pages   pages   pages   pages   pages   pages
```

- **B-tree = balanced tree** — all leaf nodes at same depth → O(log N) lookup
- Each node is a disk page (8KB) — minimizes disk I/O per level
- For a table with 1B rows: depth = log₁₆₀₀(1B) ≈ 3–4 levels → 3–4 page reads to find any row
- B-tree supports: `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE 'prefix%'`
- B-tree does NOT efficiently support: `LIKE '%suffix'`, full-text search

### Index Types

| Index Type    | Best For                                      | Example                                |
|---------------|-----------------------------------------------|----------------------------------------|
| B-tree        | Equality, range, ordering (default)           | `WHERE id = 5`, `WHERE age > 30`       |
| Hash          | Equality only (faster than B-tree for `=`)    | `WHERE session_id = 'abc'`             |
| GIN           | Full-text search, array containment           | `WHERE tags @> ARRAY['python']`        |
| GiST          | Geometric, range types, fuzzy search          | `WHERE location @@ point(lat,lng)`     |
| BRIN          | Naturally ordered data (timestamps, IDs)      | `WHERE created_at BETWEEN x AND y`     |
| Partial index | Index only a subset of rows                   | `CREATE INDEX ON orders(id) WHERE status='pending'` |
| Composite     | Multi-column queries                          | `WHERE user_id = 1 AND status = 'active'` |

### Index Selectivity

An index is only useful if it's **selective** — narrows down the result set significantly.

```
Low selectivity (bad):  gender = 'M'  → 50% of rows → full scan faster
High selectivity (good): email = 'alice@...' → 1 row → index perfect
```

Rule of thumb: index is beneficial if it filters to < 5–10% of rows.

---

## 2. ACID Properties

Every transaction in PostgreSQL/MySQL guarantees ACID:

### A — Atomicity
All operations in a transaction succeed or all are rolled back. No partial updates.
```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- If either UPDATE fails, both are rolled back
```

### C — Consistency
The DB moves from one valid state to another. Constraints (FK, UNIQUE, CHECK) are enforced.
```sql
-- Constraint prevents invalid state:
ALTER TABLE orders ADD CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id);
```

### I — Isolation
Concurrent transactions don't see each other's uncommitted changes (at the configured isolation level).

| Isolation Level    | Dirty Read | Non-repeatable Read | Phantom Read |
|--------------------|------------|---------------------|--------------|
| Read Uncommitted   | ✅ possible| ✅ possible         | ✅ possible  |
| Read Committed     | ❌ prevented | ✅ possible       | ✅ possible  |
| Repeatable Read    | ❌         | ❌ prevented        | ✅ possible  |
| Serializable       | ❌         | ❌                  | ❌ prevented |

**PostgreSQL default: Read Committed.** Most apps are fine with this.

### D — Durability
Once committed, a transaction survives crashes. Achieved via WAL (Write-Ahead Log).

```
Write path:
  1. Write to WAL (sequential, fast)
  2. Acknowledge the commit to client
  3. Apply to heap file (async, background)

On crash:
  WAL replay restores all committed transactions
```

---

## 3. Query Execution & Optimization

### EXPLAIN ANALYZE — The Most Important Tool

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';

-- Output:
Index Scan using idx_orders_user_status on orders
  (cost=0.56..8.58 rows=1 width=152)
  (actual time=0.021..0.023 rows=1 loops=1)
  Index Cond: ((user_id = 123) AND (status = 'pending'))
Planning Time: 0.5ms
Execution Time: 0.1ms
```

**Key nodes to understand:**

| Plan Node         | Meaning                                           | Good or Bad?              |
|-------------------|---------------------------------------------------|---------------------------|
| Index Scan        | Walks B-tree, fetches heap pages                  | ✅ Good                   |
| Index Only Scan   | B-tree only, no heap fetch (covering index)       | ✅✅ Best                 |
| Bitmap Index Scan | Batches heap reads (good for many rows)           | ✅ Good for bulk           |
| Seq Scan          | Full table scan                                   | ⚠️ OK for small tables, bad for large ones |
| Hash Join         | Builds hash table from smaller relation           | ✅ Fast for large joins    |
| Nested Loop       | For each row in outer, probe inner                | ✅ Fast when inner is tiny |

### N+1 Query Problem

```python
# BAD: N+1 queries
orders = db.query("SELECT * FROM orders LIMIT 100")
for order in orders:
    user = db.query(f"SELECT * FROM users WHERE id = {order.user_id}")
    # 1 query for orders + 100 queries for users = 101 queries total
```

```python
# GOOD: JOIN or IN clause
orders = db.query("""
    SELECT o.*, u.name, u.email
    FROM orders o
    JOIN users u ON u.id = o.user_id
    LIMIT 100
""")
# 1 query total
```

### Covering Index (Index Only Scan)

```sql
-- Query: SELECT user_id, status FROM orders WHERE user_id = 123
-- Without covering index: B-tree lookup → fetch heap page for status column
-- With covering index:
CREATE INDEX idx_covering ON orders (user_id) INCLUDE (status);
-- B-tree lookup → return status from index directly, no heap fetch
-- → 2x faster, eliminates random I/O
```

---

## 4. Connection Pooling

**The Problem:** Opening a new DB connection costs 2–10ms (TCP handshake + auth + memory allocation). With 100 API servers × 20 connections each = 2,000 DB connections. PostgreSQL struggles above 500–1000 connections.

### Without Pooling
```
100 API instances × 20 connections = 2,000 connections to PostgreSQL
PostgreSQL: each connection forks a process (~5MB) → 10GB overhead
Max practically: ~300–500 before degradation
```

### With PgBouncer (Connection Pooler)
```
100 API instances × 20 connections = 2,000 app connections → PgBouncer
PgBouncer: maintains 20 real DB connections → multiplexes 2,000 through them

PostgreSQL: sees only 20 connections ← happy, fast
```

**PgBouncer modes:**

| Mode          | How it works                               | Limitation                         |
|---------------|--------------------------------------------|-------------------------------------|
| Session       | One DB connection per client session       | 1:1 mapping, no pooling benefit     |
| Transaction   | DB connection held only during transaction | ✅ Best for most apps               |
| Statement     | Released after each statement              | Cannot use `SET`, prepared statements |

> **Use transaction mode** with FastAPI/SQLAlchemy. It's the sweet spot.

---

## 5. Replication

### Master-Replica (Leader-Follower)

```
Writes → [ Primary / Master ]
              │
              │  WAL streaming (async by default)
              │
         ┌────┴─────┐
    [ Replica 1 ] [ Replica 2 ]
         │
    Read-only queries routed here
```

**Replication modes:**

| Mode         | Durability                         | Write latency         | Use case              |
|--------------|------------------------------------|-----------------------|-----------------------|
| Async        | Data loss possible on primary crash| No overhead           | Default; social apps  |
| Sync         | Zero data loss                     | +RTT per write        | Financial, critical   |
| Semi-sync    | At least one replica confirms      | +RTT for one replica  | Balance               |

**AWS RDS Multi-AZ** uses synchronous replication between primary and standby in different AZs — zero data loss, automatic failover < 1 min.

### Read/Write Split Pattern

```python
# Route writes to primary, reads to replica
from sqlalchemy import create_engine

write_engine = create_engine("postgresql://primary:5432/db")
read_engine  = create_engine("postgresql://replica:5432/db")

def get_user(user_id: int) -> dict:
    with read_engine.connect() as conn:    # read replica
        return conn.execute("SELECT * FROM users WHERE id = ?", user_id).fetchone()

def create_user(data: dict):
    with write_engine.connect() as conn:   # primary
        conn.execute("INSERT INTO users ...")
```

---

## 6. Vertical Scaling vs Sharding

### Vertical Scaling (Scale Up) — Try First

```
Current: 4 vCPU, 32GB RAM, 1TB SSD
Upgrade: 32 vCPU, 256GB RAM, 10TB SSD
```

- Simple: no code changes, no routing logic
- Buffer pool now holds more hot data → fewer disk reads
- Handles most workloads up to ~50K TPS
- Hard limit: largest available instance

### Sharding (Horizontal Scale) — Complex, Use Only When Needed

Partition data across multiple DB instances, each owning a subset:

```
Shard 1: users where user_id % 4 = 0   → DB host A
Shard 2: users where user_id % 4 = 1   → DB host B
Shard 3: users where user_id % 4 = 2   → DB host C
Shard 4: users where user_id % 4 = 3   → DB host D
```

**Problems sharding introduces (always mention):**
- Cross-shard queries (JOIN across shards) become app-level operations — complex
- Transactions across shards require distributed transactions (2PC)
- Resharding when adding nodes is painful (data migration)
- Application must know the shard routing logic

**When to shard:**
- Vertical scale is maxed out
- Single DB write throughput is the bottleneck (not reads — add replicas for reads)
- Dataset size exceeds single-disk capacity

**Single-table sharding candidates:**
- High-volume tables that grow unbounded (events, messages, logs)
- Tables with clear partition key (user_id, tenant_id, created_at)

---

## 7. PostgreSQL vs MySQL — Key Differences for Interviews

| Feature               | PostgreSQL                           | MySQL (InnoDB)                    |
|-----------------------|--------------------------------------|-----------------------------------|
| ACID compliance       | Full                                 | Full (InnoDB engine)              |
| JSON support          | Superior (JSONB with indexing)       | JSON type, limited indexing       |
| Full-text search      | Built-in (tsvector/tsquery)          | Basic FULLTEXT indexes            |
| Replication           | Logical + physical streaming         | Binlog-based, GTID                |
| Concurrency control   | MVCC (Multi-version)                 | MVCC (InnoDB)                     |
| Window functions      | Excellent                            | Good (MySQL 8+)                   |
| Extension ecosystem   | Rich (PostGIS, pgvector, TimescaleDB)| Limited                           |
| Default isolation     | Read Committed                       | Repeatable Read                   |
| Performance           | Better for complex queries           | Faster for simple read-heavy      |
| Choose when           | Complex queries, JSON, geospatial    | Simple OLTP, well-known ecosystem |

---

## 8. Common Failure Modes

| Failure                    | Symptom                                 | Fix                                        |
|----------------------------|-----------------------------------------|--------------------------------------------|
| Missing index              | Seq Scan on large table, slow queries   | `EXPLAIN ANALYZE` → add targeted index    |
| N+1 queries                | 100s of similar queries per request     | Use JOIN or eager load with `IN` clause    |
| Connection pool exhaustion | "too many clients" errors               | Add PgBouncer, reduce pool size per app   |
| Replica lag                | Stale reads from replica                | Monitor lag; route critical reads to primary |
| Lock contention            | Queries waiting, timeouts               | Shorter transactions, optimistic locking   |
| Table bloat                | Table size grows despite deletes        | `VACUUM ANALYZE` / autovacuum tuning      |
| Long-running transactions  | Locks held, replica lag grows           | Set `statement_timeout`, optimize query   |
