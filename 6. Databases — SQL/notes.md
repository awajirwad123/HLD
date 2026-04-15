# Databases — SQL — Notes & Cheat Sheets

## ACID — One-Liner Per Property

| Property    | One-liner                                     | Mechanism                        |
|-------------|-----------------------------------------------|----------------------------------|
| Atomicity   | All-or-nothing — no partial transactions      | Rollback / UNDO log              |
| Consistency | DB goes valid state → valid state             | Constraints, triggers            |
| Isolation   | Concurrent txns don't corrupt each other      | MVCC, locks                      |
| Durability  | Committed data survives crashes               | WAL (Write-Ahead Log)            |

---

## Transaction Isolation Levels Cheat Sheet

| Level             | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------------------|------------|---------------------|--------------|-------------|
| Read Uncommitted  | ✅ yes     | ✅ yes              | ✅ yes       | Fastest     |
| Read Committed    | ❌ no      | ✅ yes              | ✅ yes       | Good        |
| Repeatable Read   | ❌ no      | ❌ no               | ✅ yes       | Moderate    |
| Serializable      | ❌ no      | ❌ no               | ❌ no        | Slowest     |

**Phenomena definitions:**
- **Dirty read:** Read uncommitted data from another transaction
- **Non-repeatable read:** Same row read twice in one transaction returns different values (because another transaction committed between reads)
- **Phantom read:** Same range query returns different row counts (because another transaction inserted/deleted rows between reads)

**PostgreSQL defaults:** Read Committed  
**MySQL defaults:** Repeatable Read  
**What to use in practice:** Read Committed is correct for 95% of apps.

---

## Index Selection Guide

| Scenario                                           | Index type           |
|----------------------------------------------------|----------------------|
| Equality or range on single column                 | B-tree               |
| Equality only, high read throughput                | Hash                 |
| Full-text search                                   | GIN (tsvector)       |
| JSONB key lookups                                  | GIN                  |
| Geospatial queries                                 | GiST (PostGIS)       |
| Large append-only table, sequential column (ts/id) | BRIN                 |
| Multi-column WHERE clause                          | Composite B-tree     |
| Index small subset of rows                         | Partial index        |
| Avoid heap fetch (cover SELECT columns)            | Covering index (INCLUDE) |

**Column order in composite index:**
- Put the highest-cardinality (most selective) column first
- Put the equality column before the range column
- `(user_id, created_at)` → works for `WHERE user_id = ? AND created_at > ?`
- `(created_at, user_id)` → does NOT work well for the same query

---

## Connection Pool Sizing Formula

```
pool_size = (core_count * 2) + effective_spindle_count

Common starting point for PostgreSQL:
  pool_size     = 10   (per application instance)
  max_overflow  = 20   (burst capacity)
  Total per app = 30 connections

For 20 app instances:
  Without pooler: 20 × 30 = 600 connections  (borderline for PostgreSQL)
  With PgBouncer:             20 connections  (PgBouncer holds 20, serves 600 app connections)
```

**PgBouncer transaction mode formula:**
```
DB connections = (avg_transaction_duration_ms / avg_interval_between_transactions_ms) × desired_concurrency
```
In practice: start with 20 DB connections through PgBouncer for most workloads, tune up.

---

## Replication Lag — Key Numbers

| Scenario                    | Typical Lag   | Action                              |
|-----------------------------|---------------|-------------------------------------|
| Same datacenter, async      | < 10ms        | Fine for most reads                 |
| Cross-AZ, async             | 10–50ms       | Fine for most reads                 |
| Cross-region, async         | 50–200ms      | Cache or read from primary if stale |
| Synchronous replication     | = network RTT | Only for financial systems          |

---

## Vertical Scaling Thresholds (PostgreSQL)

| Metric                | Threshold to Add Resources       | Action                          |
|-----------------------|----------------------------------|---------------------------------|
| CPU > 70% sustained   | Often queries, not CPU problem   | First: check slow query log     |
| RAM (buffer pool) full| Cache hit rate < 95%             | Add RAM or reduce dataset       |
| Connections > 300     | Connection wait times rising     | Add PgBouncer                   |
| Disk I/O > 80%        | Seq scans on large tables        | Add index or add SSDs           |
| Replication lag > 100ms | Replica reads becoming stale  | Route reads to primary or cache |
| Write TPS > 10K       | Heading toward single-node limit | Plan for read replicas + sharding |

---

## PostgreSQL Useful Commands

```sql
-- Show running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state = 'active' AND duration > interval '5 seconds'
ORDER BY duration DESC;

-- Kill a slow query
SELECT pg_cancel_backend(pid);  -- graceful
SELECT pg_terminate_backend(pid);  -- forceful

-- Check index usage
SELECT relname, indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;  -- low idx_scan = unused index, candidate for removal

-- Check table bloat / vacuum status
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

-- Show cache hit rate (should be > 95%)
SELECT
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
FROM pg_statio_user_tables;

-- Table sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC LIMIT 20;
```

---

## When to Choose SQL vs NoSQL

| Signal → Choose SQL                               | Signal → Consider NoSQL                          |
|---------------------------------------------------|--------------------------------------------------|
| You need ACID transactions across multiple tables | Each entity accessed independently, no JOINs    |
| Data is relational (users, orders, products)      | Document structure varies per record (JSONB exists too) |
| Complex queries, aggregations, reporting          | Extreme write throughput (>100K TPS per entity) |
| Team knows SQL                                    | Horizontal scale is primary requirement from day 1 |
| Schema is well-defined                            | Time-series / event-stream use case             |
| Compliance requires audit trails                  | Geographic distribution / multi-region writes   |

**Honest take:** Start with PostgreSQL. It handles more than most teams expect. Switch when you hit a specific, measurable limit.

---

## Key Numbers to Memorize

| Fact                                     | Number                   |
|------------------------------------------|--------------------------|
| PostgreSQL page size                     | 8KB                      |
| Typical max safe connections (no pooler) | 300–500                  |
| B-tree depth for 1B rows                 | 3–4 levels               |
| WAL write throughput (SSD)               | ~500MB/s                 |
| Index scan vs seq scan break-even        | ~5–10% of rows           |
| PgBouncer transaction mode overhead      | < 1ms                    |
| Async replication lag, same AZ           | < 10ms                   |
| PostgreSQL recommended `shared_buffers`  | 25–40% of total RAM      |
| VACUUM reclaims dead rows after          | On autovacuum trigger    |
| Connection setup cost                    | 2–10ms                   |

---

## Interview Answer Checklist — DB Questions

When asked any SQL DB design question, cover these:
1. **Schema design:** normalization level, FK constraints, data types
2. **Index strategy:** which columns, composite vs single, partial if applicable
3. **Read/write ratio:** read replicas for read-heavy, write path to primary
4. **Connection pooling:** PgBouncer, pool size formula
5. **Transaction boundaries:** which operations need to be atomic
6. **Failure modes:** replica lag, primary failover, pool exhaustion
7. **Scale-out trigger:** when to shard (don't shard prematurely)
8. **Observability:** slow query log, `pg_stat_activity`, cache hit rate
