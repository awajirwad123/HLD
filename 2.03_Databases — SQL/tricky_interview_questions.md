# Databases — SQL — Tricky Interview Questions

---

## Q1: "ACID sounds great — is there a cost?"

**What they're testing:** Do you understand the performance trade-offs of guarantees?

**Strong answer:**

Yes — every ACID property has a cost:

- **Atomicity:** The DB must maintain an undo log (or rollback segments) to revert partial writes. This takes memory and I/O.
- **Consistency:** Constraint checks (FK, UNIQUE) require reads or index probes on every write — adds latency.
- **Isolation:** MVCC (PostgreSQL) keeps multiple row versions alive — dead tuple bloat, requiring periodic VACUUM. Strict isolation (Serializable) adds SSI overhead.
- **Durability:** Every committed transaction requires a WAL flush to disk (fsync). On a spinning disk, this is the main write bottleneck. SSDs mitigate but don't eliminate this.

The cost of Serializable isolation is so high that PostgreSQL defaults to Read Committed — a pragmatic trade-off most teams accept.

**Bottom line:** ACID = correctness + measurable overhead. The right trade-off depends on your consistency requirements. A social feed tolerates Read Committed. A payment system may require Serializable or application-level compensating transactions.

---

## Q2: "Should you index every column for maximum query speed?"

**What they're testing:** Understanding of write amplification, index maintenance cost.

**Strong answer:**

No — each index is a write-time tax:

- On every `INSERT`, `UPDATE`, or `DELETE`, PostgreSQL must update every index on the affected table
- 10 indexes = 10 B-tree updates per write
- Indexes also consume disk space and shared buffer pages (competes with the data itself)

**Rules for index discipline:**
1. Index columns that appear in `WHERE`, `JOIN ON`, or `ORDER BY` clauses of slow queries
2. Verify with `EXPLAIN ANALYZE` that the index is actually used
3. Check `pg_stat_user_indexes` to find unused indexes (low `idx_scan`) and drop them
4. Prefer composite indexes over multiple single-column indexes for multi-column queries

**Interviewer follow-up:** "An index exists but the planner won't use it — why?"
- The data distribution estimate is stale → run `ANALYZE`
- The index is on a low-selectivity column → the planner correctly prefers seq scan
- The query has a function on the indexed column: `WHERE LOWER(email) = ...` → the B-tree index on `email` can't be used; needs a functional index `CREATE INDEX ON users (LOWER(email))`

---

## Q3: "Your API response times suddenly spiked. Queries take 10s. What do you investigate?"

**What they're testing:** Systematic DB performance triage.

**Investigation flow:**

```
1. Is it connection-related?
   → pg_stat_activity: any queries in 'waiting' state?
   → Connection count near max? → pool exhaustion

2. Is it lock-related?
   → SELECT * FROM pg_locks WHERE NOT granted; → any blocked queries?
   → Long-running transaction holding a lock?

3. Is it query performance regression?
   → Check slow query log (log_min_duration_statement = 1000ms)
   → EXPLAIN ANALYZE the slow query
   → Was a new deploy + schema change done? Missing migration index?

4. Is it resource saturation?
   → CPU: vacuuming? autovacuum running on a hot table?
   → Disk I/O: cache hit rate dropped? Buffer pool full?
   → Replication: replica lag spiked? Queries routed back to primary?
```

**Most common root causes in production:**
- A new code deploy introduced an N+1 query
- An index was accidentally dropped in a migration
- A long-running transaction (analytics query, batch job) is blocking row-level locks
- Connection pool exhausted under traffic spike (add PgBouncer)

---

## Q4: "How do you handle read-your-writes consistency when using read replicas?"

**What they're testing:** Practical understanding of replication lag and consistency patterns.

**The problem:**
```
User writes a post → write goes to PRIMARY
User immediately fetches their feed → read goes to REPLICA
Replica hasn't received the WAL yet (5ms lag) → user sees empty feed
→ User thinks the write failed
```

**Solutions (in order of preference):**

1. **Session stickiness for the writing user:** For 1–2 seconds after a write, route that user's reads to the primary. Implemented via a timestamp in the session cookie (`wrote_at: 1703001234`), and routing logic checking `now - wrote_at < 2s`.

2. **Read-from-primary for critical paths:** After any state-mutating request, the immediate response redirect/page fetch goes to primary. All other reads go to replica.

3. **Wait for replica to catch up:** PostgreSQL provides `pg_current_wal_lsn()` on primary and `pg_last_wal_apply_lsn()` on replica — you can poll until the replica has applied the primary's LSN. Expensive, rarely worth it.

4. **Monotonic reads:** Route a given user's reads always to the same replica. That replica stays ahead of others' stale reads for that user's writes.

**In interviews:** Options 1 or 2 are the practical answers. Mention that you'd also set alerts on replication lag (`pg_stat_replication.write_lag`) and page if it exceeds a threshold.

---

## Q5: "When would you move from PostgreSQL to a NoSQL database?"

**What they're testing:** Maturity — you shouldn't jump to NoSQL immediately, but you should know exactly when the trade-off tips.

**Strong answer:**

I'd switch when I hit a *specific, measurable limit* that PostgreSQL can't address:

| Trigger                                     | Better fit            | Why                                         |
|---------------------------------------------|-----------------------|---------------------------------------------|
| Write throughput > 50K TPS on one logical partition | Cassandra / DynamoDB | Multi-master writes, no single leader bottleneck |
| Schema is truly dynamic per document        | MongoDB               | JSONB in PG works for moderate variability  |
| Time-series, append-only, IoT data          | InfluxDB / TimescaleDB | Optimized for time-based compression       |
| Full-text search is the primary access pattern | Elasticsearch       | Inverted index, relevance scoring           |
| Global active-active writes, multi-region   | CockroachDB / Spanner | Distributed SQL — not traditional NoSQL    |
| Graph traversal is the primary query        | Neo4j                 | Joins become O(n²) in SQL for deep graphs  |

**What I would NOT do:**
- Switch because "SQL doesn't scale" — it does, up to significant throughput
- Sacrifice ACID without understanding the consistency implications

**What I always say in this answer:** "PostgreSQL with proper indexing, PgBouncer, and read replicas can handle hundreds of thousands of QPS for most applications. I'd validate the bottleneck is actually in the DB layer before changing the data store."

---

## Q6: "Describe a deadlock scenario and how you'd prevent it."

**What they're testing:** Understanding of lock ordering and practical deadlock prevention.

**Scenario:**
```
Transfer service: TxA transfers from Account 1 → Account 2
Transfer service: TxB transfers from Account 2 → Account 1

TxA: LOCK account_1 → waits for account_2 (held by TxB)
TxB: LOCK account_2 → waits for account_1 (held by TxA)
→ Deadlock: both wait forever, PostgreSQL auto-detects and aborts one.
```

**Prevention strategies:**

1. **Lock ordering** (most robust): Always acquire locks in a globally consistent order (e.g., ascending primary key). Both transactions would lock `account_1` first — one succeeds, the other waits → no deadlock possible.

2. **Optimistic locking** (good for low-contention scenarios): Add a `version` column. Read version, update, check version still matches. Retry on conflict. No pessimistic locking at all.

3. **Short transactions**: The longer a transaction holds locks, the higher the deadlock probability. Keep transactions tight — no external API calls inside a transaction.

4. **`lock_timeout`**: Set `SET lock_timeout = '5s'` to abort rather than wait indefinitely on a lock. Prevents cascading waits.

**Interviewer follow-up:** "If a deadlock does occur despite prevention, what's the behavior?"
PostgreSQL automatically detects deadlock cycles and aborts the youngest transaction with error code `40P01`. Your application must retry the transaction.

---

## Q7: "How would you shard a users table? What are the trade-offs?"

**What they're testing:** Understanding that sharding is complex and often avoidable.

**Strong answer:**

First: **why are we sharding?** If it's for read scale, I'd add read replicas first. Sharding only makes sense when write throughput or storage has outgrown a single primary.

**Sharding strategies:**

| Strategy      | How                                         | Pro                          | Con                             |
|---------------|---------------------------------------------|------------------------------|---------------------------------|
| Range sharding | `shard = id < 1M → DB1`, etc.              | Range scans stay on one shard| Hotspot if new IDs go to one DB |
| Hash sharding | `shard = user_id % N → DB[N]`             | Even distribution            | Range queries span all shards  |
| Directory     | Lookup table maps user_id → shard_id       | Flexible reassignment        | Directory is a bottleneck/SPOF  |

**Problems that come with sharding:**
1. **Cross-shard JOINs** become application-level (fetch from both shards, join in code)
2. **Cross-shard transactions** require 2PC (Two-Phase Commit) or eventual consistency
3. **Resharding**: adding a new shard requires data migration — expensive, requires careful orchestration
4. **Global uniqueness**: auto-increment IDs no longer work — need UUID or distributed ID generator (Snowflake)

**My recommendation:** For most apps, try this in order:
1. Vertical scale (bigger instance)
2. Read replicas for read scale
3. Table partitioning (PostgreSQL native, transparent to app)
4. Application-level sharding only when nothing else suffices
