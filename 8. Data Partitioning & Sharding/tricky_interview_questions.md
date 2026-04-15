# Data Partitioning & Sharding — Tricky Interview Questions

## Q1: "You shard your user table by user_id. Now your product team wants a feature: show all users who signed up on a specific date. How do you implement this efficiently?"

**What they're testing:** Cross-shard query problem, secondary indexing, trade-offs between OLTP and analytics.

**The problem:** `SELECT * FROM users WHERE signup_date = '2024-01-15'` — `signup_date` is not the shard key. This query must scatter to all N shards and aggregate, which is O(N × per-shard cost).

**Solutions (in order of complexity):**

1. **Scatter-gather + accept the cost:** If this query is rare (e.g., a daily admin report), fanning out to 8 shards and getting back a few hundred rows is fine. Add a limit/pagination to avoid massive responses.

2. **Secondary index shard:** Maintain a separate small service (or DB table on a dedicated "index node") that maps `signup_date → list of user_ids`. This index is small (365 rows/year) and easy to query. Use it to get user IDs, then fan-out to only the relevant shards to fetch full rows.

3. **Event-driven denormalization:** On user signup, publish an event to Kafka. A consumer writes to an analytics DB (ClickHouse, BigQuery) separate from the OLTP shards. Date-based queries hit the analytics DB — no scatter-gather on the OLTP layer.

4. **Dual-write to a reporting store:** Write every new user both to the sharded OLTP DB (optimized for user_id lookups) and to a non-sharded read replica (optimized for full-table analytics). Common in data warehousing.

**Strong answer:** "I'd keep the OLTP shards optimized for the 99% use case (lookup by user_id), and build a separate analytics pipeline for date-based queries. Mixing analytics workloads with OLTP shards causes query interference and is a known anti-pattern."

---

## Q2: "Your system uses hash-mod sharding with 4 nodes. Traffic grows and you need to add a 5th. Walk me through a zero-downtime migration."

**What they're testing:** Operational knowledge of resharding, dual-write, consistency during migration.

**The naive approach:** Change `% 4` to `% 5`, restart, migrate. This causes 80% data movement + downtime. Don't do this.

**Zero-downtime dual-write migration:**

```
Phase 1: Setup (no traffic change yet)
  - Provision 5th shard node (empty)
  - Deploy code that can write to BOTH old (4-shard) and new (5-shard) layout

Phase 2: Dual-write begins
  - All new writes go to both old shard layout AND new shard layout
  - Reads still use old layout (4 shards)
  - Gap: data written before dual-write started only exists in old layout

Phase 3: Backfill
  - Background job reads all data from old shards
  - For each record: compute new shard → write to correct new shard (if not already there)
  - Track backfill progress by key range

Phase 4: Verification
  - Compare row counts and checksums between old and new layouts
  - Run shadow reads: send read traffic to both layouts, compare results

Phase 5: Read cutover
  - Gradually shift reads to new layout (10% → 50% → 100%)
  - Monitor error rates

Phase 6: Write cutover
  - Stop dual-writing to old layout
  - Old shards become read-only, then decommission

Phase 7: Cleanup
  - Remove old shard infrastructure
  - Remove dual-write code
```

**Key safely:** During dual-write, a crash between the two writes creates a brief inconsistency. Accept this: reads will use the old layout (which is always consistent) until Phase 5 cutover.

---

## Q3: "In a multi-tenant SaaS with PostgreSQL, when should you shard by tenant_id vs use a shared schema?"

**What they're testing:** Multi-tenancy models, cost vs isolation trade-offs, when sharding is premature.

**Three multi-tenancy models:**

| Model | Isolation | Cost | Use case |
|---|---|---|---|
| Shared schema (row-level `tenant_id`) | Low | Very low | Early-stage SaaS, < 10K tenants |
| Schema-per-tenant | Medium | Low-medium | 10K tenants, each with moderate data |
| DB-per-tenant (sharding by tenant) | High | High | Enterprise customers with compliance/data residency needs |

**When to shard by tenant_id:**

1. **Data residency requirements:** EU tenant must store data in EU. US tenant in US. Shard = geographic placement.
2. **Performance isolation:** One large tenant doing heavy analytics shouldn't degrade all other tenants. Dedicated DB = dedicated resources.
3. **Per-tenant customization:** Some enterprise tenants need custom schemas, extra columns, or custom indexes.
4. **Security/compliance:** Tenant data must never be mixed (HIPAA, FedRAMP). Schema separation or DB separation required.

**When NOT to shard (row-level tenant_id is fine):**

- You have 100K small tenants each with < 100MB of data
- Total data fits in one PostgreSQL instance + replicas
- Tenants don't have compliance requirements
- Operational complexity of N databases is too high for your team

**Strong answer:** "Start with row-level tenant isolation using `tenant_id` column + row security policies (PostgreSQL RLS). Add a `tenant_id` index. Only shard when: (a) total data exceeds practical single-node scale, (b) you have an enterprise customer requiring data residency, or (c) noisy neighbor problems become unacceptable. Premature multi-DB sharding multiplies your operational burden significantly."

---

## Q4: "A hotspot develops on shard 3 because a single user suddenly goes viral (100M followers). Adding a cache layer didn't fully solve it. What next?"

**What they're testing:** Beyond-basic hotspot handling, application-level shard splitting.

**Layered approach:**

1. **Cache (already tried):** Redis read-cache for the hot user's profile and recent posts. Handles reads but not writes (likes, comments are still DB writes).

2. **Salted write keys:** For the hot user's like counter, instead of one DB row, maintain N counter rows: `likes_{user_id}_0` through `likes_{user_id}_99`. Each write randomly picks one of the 100 "salt buckets." Total likes = SUM of all 100 rows. This spreads write load across 100 rows (potentially across shards with consistent hashing). Cassandra's "counter" anti-pattern solution uses this.

3. **Async writes with buffering:** Don't write every like directly to DB. Write to Kafka → a consumer batches and flushes every 5 seconds. Absorbs the write burst into a queue. Rate of DB writes = 1/5 of like rate.

4. **Dedicated hot-entity shard:** Move user_id=viral_celebrity to its own physical shard (or even its own DB node). Update the shard directory to point to this dedicated node. This user no longer competes with regular users on shard 3.

5. **Shard splitting:** Split shard 3 into two: shard 3a and shard 3b. Migrate half its key range to 3b. The viral celebrity may land in 3a while other users go to 3b, halving the load. This is what Cassandra/DynamoDB do automatically when a partition gets too hot.

**Which to choose:** For an immediate crisis: async writes (2) + dedicated shard (4). For long-term: automatic shard splitting triggered by hot partition metrics.

---

## Q5: "Why does Cassandra's partition key design matter more than PostgreSQL's index design?"

**What they're testing:** Deep understanding of Cassandra's storage model vs PostgreSQL.

**PostgreSQL:** Indexes are separate B-tree structures. Any column can have an index. The DB engine figures out which index to use. Cross-table JOINs are possible. You can query by any column (with performance trade-offs).

**Cassandra's model is fundamental different:**

1. **All data for a partition key lives on exactly one node.** The partition key is your shard key — there's no secondary routing.

2. **No server-side JOINs.** Cassandra is a distributed system. Data from two partition keys may be on two different nodes. JOINs across partition keys would require cross-node coordination — Cassandra doesn't do this.

3. **Table design is query-driven.** You design your Cassandra table schema based on your access patterns. Different queries → different tables (denormalized). One query pattern = one table.

4. **Clustering keys for ordering within a partition.** Data within a partition is stored sorted by clustering keys (on disk, not just in index). This makes range scans within a partition blazing fast.

```
-- Bad: querying by non-partition key
SELECT * FROM users WHERE email = 'alice@example.com';
-- Requires full table scan or secondary index (expensive in Cassandra)

-- Good: query by partition key
SELECT * FROM messages WHERE conversation_id = '123' AND created_at > '2024-01-01';
-- hits exactly one node, data is sorted by created_at on disk
```

**Implication:** Bad Cassandra data modeling = queries that don't match the partition key → scatter-gather or secondary indexes → terrible performance. Bad PostgreSQL index = slow query but still correct. Cassandra punishes data model mistakes much harder.

**Strong answer:** "In Cassandra, the partition key is the fundamental routing mechanism, not a hint to the query planner. Every query must include the partition key to avoid a full cluster scan. You design tables around queries, not around normalized data models. This is why Cassandra data modeling is harder and more critical than PostgreSQL indexing."
