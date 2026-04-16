# Data Partitioning & Sharding — Interview Simulator

## Scenario 1: "Design the sharding strategy for Instagram at scale"

**Prompt:** "Instagram has 1B users, 100B photos, 50B likes, 2B comments. Design how you would shard the data layer."

---

**Clarifying questions to ask:**
1. What are the primary access patterns? (User profile, photo feed, photo detail, search?)
2. Do we need cross-user analytics in real-time, or is eventual consistency OK?
3. What geographic distribution is needed (CDN? Multi-region DB)?
4. Are there regulatory data residency requirements?

---

**Model answer:**

**Entity → shard key mapping:**

| Entity | Shard key | Why |
|---|---|---|
| Users | user_id | High cardinality, uniform |
| Photos | owner_user_id | Co-located with user data |
| Likes | photo_id | Optimizes "get likes for photo X" |
| Comments | photo_id | Co-located with likes |
| Follow graph | follower_user_id | Optimizes "who does user X follow?" |

**Architecture:**

```
PostgreSQL cluster (sharded by user_id):
  - Shard 1: user_ids 1–250M
  - Shard 2: user_ids 250M–500M
  - Shard 3: user_ids 500M–750M
  - Shard 4: user_ids 750M–1B
  (Managed via Vitess or PgBouncer + shard router)

Object storage (S3/GCS):
  - Photos are NOT in the DB: stored in S3, keyed by photo_id
  - DB stores metadata only (owner, timestamp, location, caption)

Feed service (Redis + Cassandra):
  - User's feed stored in Redis Sorted Set, keyed by user_id
  - Historical feed in Cassandra, partitioned by user_id

Analytics (separate from OLTP):
  - All events (likes, comments, views) → Kafka → ClickHouse
  - "Top photos by likes this week" runs in ClickHouse, not in sharded OLTP
```

**Celebrity hotspot handling:**
- Celebrity user_id lands on one shard. Millions of followers read their photos.
- Solution: CDN caches photos + metadata for celebrity accounts (95% read hit rate).
- Writes (new celebrity post): fan-out to Redis feeds of all followers via async worker.

**Cross-shard query ("show global trending photos"):**
- Not served from OLTP shards.
- Dedicated trending service consumes Kafka events → computes per-photo engagement score → serves from Redis sorted set `trending:global`.

**Trade-off to mention:** Co-locating photos with owner (`shard by user_id`) makes "get all photos by user X" fast. But "get all photos liked by user Y across accounts" requires scatter-gather. Accept this trade-off: user's own content is the primary access pattern.

---

## Scenario 2: "After 3 years, shard 2 is running out of disk space. How do you migrate with zero downtime?"

**Prompt:** "You have 8 shards based on hash-mod. Shard 2 has grown to 90% capacity. You need to relieve it."

---

**Investigation first:**
- Is it uniform data growth, or a few large tenants/users on shard 2?
- Can you delete soft-deleted data or move archived data to cold storage first?
- Is this a storage problem or also a write throughput problem?

**If a few large entities drive the problem:**
- Move N largest entities on shard 2 to a new dedicated shard
- Update shard directory to route their IDs to the new node
- Much simpler than resharding everything

**If uniform growth (resharding required):**

```
Phase 1: Provision shard 2a and 2b (replace shard 2)
  - 2a takes lower half of shard 2's key space
  - 2b takes upper half

Phase 2: Dual-write
  - All writes for keys in shard 2 go to both shard 2 (old) AND shard 2a/2b (new)
  - Reads still from shard 2

Phase 3: Backfill
  - Background job copies all existing shard 2 data to 2a/2b
  - Mark each batch as "backfilled" with a timestamp

Phase 4: Read cutover
  - Shadow reads: run queries on both shard 2 and 2a/2b, compare
  - Once checksums match: switch reads to 2a/2b
  - Monitor error rates for 30 minutes

Phase 5: Decommission shard 2
  - Stop dual-writes
  - Archive shard 2 for 7 days as safety window
  - Delete
```

**Operational safeguards:**
- Track migration progress in a `migration_state` table
- If anything goes wrong during Phase 4: fall back to shard 2 reads (still valid)
- The "shadow read" phase is non-negotiable — don't assume backfill is correct without verification

---

## Scenario 3: "Debugging — some users complain of stale data; recent changes aren't appearing"

**Prompt:** "After a routine rebalance of your consistent hash ring, some users are getting stale data for up to 30 minutes. Why?"

---

**Root cause analysis:**

Consistent hash rebalance: a new node was added → some keys migrated to the new node. The migration copied data from the old node to the new node. But what if the migration had a lag?

**Scenario:**
```
t=0: New node D added to ring. Key range [3000–4000] moves from Node B → Node D.
t=0 to t=30min: Background migration job copies keys [3000–4000] from B to D.
t=5: User writes user_id=3500 to Node D (correctly routed to new node).
t=10: Another request reads user_id=3500 → routes to Node D → data is there. ✓
t=15: Migration job overwrites user_id=3500 on Node D with the OLD value from Node B. ✗
```

The migration job blindly copied from B to D without checking if D already had newer data.

**Fixes:**

1. **Version-aware migration:** Before overwriting a key in D, compare timestamps. Only copy if source is newer: `if source.updated_at > dest.updated_at: copy`.

2. **Pause writes to migrating key range during migration:** Route writes to migrating range to a holding queue. Drain queue after migration completes. Brief increased latency but no stale data.

3. **Copy-then-cutover:** Migration job writes to D, but the ring doesn't advertise D as the owner of [3000–4000] until migration is 100% complete. All writes still go to B during migration → D starts with fully up-to-date data → ring updated → B drains remaining writes.

4. **Idempotent writes with monotonic version number:** Every write includes a version number. Migration copies with the version number intact. Application layer discards any write with a lower version than what it already has.

**Best practice:** Option 3 (copy-then-cutover) is the safest because it avoids concurrent write races entirely. The downside is slightly delayed node activation. Option 1 is fine if writes have reliable monotonic timestamps (not wall-clock time — use Lamport timestamps or Snowflake IDs).
