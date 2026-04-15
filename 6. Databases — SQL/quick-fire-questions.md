# Databases — SQL — Quick-Fire Questions

**Format:** Read the question, answer out loud, then check.

---

**Q1: What does ACID stand for?**
Atomicity, Consistency, Isolation, Durability.

---

**Q2: What is the default isolation level in PostgreSQL?**
Read Committed.

---

**Q3: What is a dirty read?**
Reading uncommitted data from another transaction — data that may later be rolled back.

---

**Q4: What isolation level prevents phantom reads?**
Serializable.

---

**Q5: What data structure does PostgreSQL use for its default index?**
B-tree.

---

**Q6: What is the PostgreSQL page size?**
8KB.

---

**Q7: What does `EXPLAIN ANALYZE` tell you that `EXPLAIN` alone does not?**
`EXPLAIN ANALYZE` actually executes the query and shows real runtime statistics (actual rows, actual time), while `EXPLAIN` shows only the planner's estimate.

---

**Q8: When would a Seq Scan be faster than an Index Scan?**
When the query returns a large percentage (> 5–10%) of the table. Fetching many random heap pages (index scan) can be slower than reading the table sequentially.

---

**Q9: What is the N+1 query problem?**
Fetching a list of N items, then executing 1 additional query per item to fetch related data — resulting in N+1 total queries. Fix with JOIN or batched IN clause.

---

**Q10: What is a covering index?**
An index that includes all columns needed by a query, allowing an Index Only Scan — no heap page fetch required. Created with `INCLUDE (col1, col2)` in PostgreSQL.

---

**Q11: What is PgBouncer and why is it used?**
PgBouncer is a connection pooler that sits between application servers and PostgreSQL. It multiplexes thousands of app connections through a small pool of real DB connections, preventing connection overhead and overload.

---

**Q12: What are the three PgBouncer pooling modes?**
Session (1 DB connection per client session), Transaction (DB connection held only during a transaction — recommended), Statement (released after each statement — very restrictive).

---

**Q13: What is WAL in PostgreSQL?**
Write-Ahead Log — a sequential log written before any data page changes. On crash, WAL is replayed to restore committed state. Also the basis for streaming replication.

---

**Q14: How does streaming replication work in PostgreSQL?**
The primary writes WAL records and streams them to replicas in real time. Replicas replay the WAL to stay in sync. By default it is asynchronous — the primary doesn't wait for replica confirmation.

---

**Q15: What is the difference between async and sync replication?**
Async: primary commits and replies to client without waiting for replica — faster, possible data loss on primary crash. Sync: primary waits for at least one replica to confirm — no data loss, higher write latency.

---

**Q16: What is connection pool exhaustion?**
All connections in the pool are in use and new requests must wait. Manifests as "too many clients" errors or high client wait times. Fix: add PgBouncer, reduce pool size per app instance, or scale the app horizontally.

---

**Q17: What is a partial index? Give an example.**
An index on a subset of rows matching a WHERE condition. Example: `CREATE INDEX ON orders (created_at) WHERE status = 'pending'` — indexes only pending orders. Smaller, faster, used by queries that include the same WHERE clause.

---

**Q18: What is the difference between a master-replica setup and sharding?**
Master-replica (replication): full copy of data on each node; scales reads only by adding replicas. Sharding: data is partitioned across nodes; each shard holds a subset; scales both reads and writes but adds cross-shard complexity.

---

**Q19: When should you NOT add an index?**
On low-cardinality columns (e.g., boolean, gender), on columns that are rarely queried, or when the table is small enough that a seq scan is fast. Also: every index slows down writes and takes disk space.

---

**Q20: What PostgreSQL query shows you the current cache hit rate?**
```sql
SELECT
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
FROM pg_statio_user_tables;
```
Should be > 95%. If lower, increase `shared_buffers` or RAM.
