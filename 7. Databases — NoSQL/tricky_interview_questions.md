# Databases — NoSQL — Tricky Interview Questions

---

## Q1: "Cassandra is AP — does that mean it can never be strongly consistent?"

**What they're testing:** Understanding of tunable consistency vs CAP theorem labels.

**Strong answer:**

The "AP" label describes Cassandra's default and its design philosophy — not a hard constraint. Cassandra's consistency is **tunable per query**:

- With `QUORUM` writes + `QUORUM` reads, and Replication Factor = 3:
  - Write quorum = 2, Read quorum = 2. W + R = 4 > RF (3).
  - The read quorum is guaranteed to overlap with the write quorum → you always read the latest written value.
  - This is **strong consistency** in practice.

- With `ALL` + `ALL`, you get serializable-equivalent consistency (but any single node failure = unavailable).

**The nuance:** CAP's "consistency" means linearizability under a network partition. Cassandra with QUORUM is strongly consistent under normal operation but sacrifices availability if enough replicas are down (e.g., 2 of 3 nodes are partitioned away). With ONE, it flips: always available, but may return stale data.

**The real interview answer:** "Cassandra gives you a dial. Slide toward consistency (QUORUM, ALL) and you sacrifice availability when nodes fail. Slide toward availability (ONE, ANY) and you accept stale reads. Choose per query based on the business requirement."

---

## Q2: "How would you design a Cassandra schema for a Twitter-like activity feed?"

**What they're testing:** Access-pattern-driven schema design, anti-hotspot techniques.

**The naive wrong answer:** `user_id` as partition key, `created_at` as clustering key. Problem: a celebrity user (10M followers) has an enormous, ever-growing partition → hot partition.

**Strong answer:**

First, clarify the access pattern: "Show the 20 most recent activities for user X."

**Schema:**
```sql
CREATE TABLE activity_feed (
    user_id    UUID,
    bucket     TEXT,        -- 'user_abc#2024-01' — time-bucketed
    event_time TIMESTAMP,
    event_id   UUID,
    event_type TEXT,        -- 'tweet', 'like', 'follow'
    payload    TEXT,        -- JSON blob
    PRIMARY KEY ((user_id, bucket), event_time, event_id)
) WITH CLUSTERING ORDER BY (event_time DESC, event_id DESC)
  AND default_time_to_live = 7776000;  -- 90 days
```

**Why time-bucketed partition key?**
- A user who has tweeted for 5 years = one enormous partition in the naive design
- Monthly bucket caps partition size → safe, bounded
- Read spans 1–2 buckets for recent feed (current month + previous)

**Fan-out write strategy (the "amplification" discussion):**
```
Tweet from user A (followed by users B, C, D):
  Option 1 (fan-out on write):  Write to B, C, D's feed tables at tweet time
    → Reads are fast (just read your own feed)
    → Writes amplify by follower count — celebrity problem (10M writes per tweet)

  Option 2 (fan-out on read):  Only store tweet once; compute feed at read time
    → No write amplification
    → Read is slow (gather N followees' timelines + merge)

  Option 3 (Hybrid — Twitter's actual model):
    → Normal users: fan-out on write
    → Celebrities (>1M followers): fan-out on read
```

---

## Q3: "Your MongoDB collection has 50M documents. A query takes 30 seconds. Walk me through your debugging process."

**What they're testing:** Practical MongoDB performance triage.

**Step-by-step answer:**

```javascript
// Step 1: Run explain with executionStats
db.posts.find({ author_id: "user_123", status: "published" })
        .explain("executionStats")
```

**Read the output:**
```
{
  "executionStats": {
    "executionSuccess": true,
    "nReturned": 42,
    "executionTimeMillis": 28500,   ← 28.5 seconds
    "totalDocsExamined": 50000000,  ← examined ALL 50M docs
    "totalKeysExamined": 0,         ← no index used!
  },
  "winningPlan": {
    "stage": "COLLSCAN"             ← collection scan = full scan
  }
}
```

**Diagnosis:** COLLSCAN — no usable index on `author_id` or `status`.

**Fix:**
```javascript
// Option A: Compound index (if both fields always appear in queries together)
db.posts.createIndex({ author_id: 1, status: 1 })

// Option B: Separate indexes (if each can appear alone too)
db.posts.createIndex({ author_id: 1 })

// After index:
// totalKeysExamined: 42, totalDocsExamined: 42, executionTimeMillis: 1
```

**Follow-up diagnostics to mention:**
- `db.currentOp()` — check what's running right now, find the blocking query
- `db.setProfilingLevel(1, { slowms: 100 })` — log all queries > 100ms
- Check `db.posts.stats()` — document average size, total size, index sizes
- Check memory: if working set (hot data + indexes) doesn't fit in RAM → disk I/O dominates

**Key insight:** In MongoDB, an absent or wrong index is almost always the root cause of slow queries. COLLSCAN on 50M docs should never happen in production.

---

## Q4: "When would a NoSQL database give you WORSE performance than PostgreSQL?"

**What they're testing:** Intellectual honesty. Avoid sounding like a NoSQL fanboy.

**Strong answer:**

Several scenarios where NoSQL loses:

1. **Complex JOIN queries across multiple entity types.** A reporting query like "total revenue by customer segment, filtered by product category, joined to regional data" — SQL with proper indexes crushes any document store that requires multiple round trips or aggregation pipeline complexity.

2. **Ad-hoc analytics.** PostgreSQL with good indexes handles OLAP queries that access many columns. MongoDB COLLSCAN on a 500GB collection is catastrophic; Cassandra can't do the query at all without ALLOW FILTERING.

3. **Strong transactions across multiple entities.** MongoDB 4.0+ supports multi-document ACID, but it is slower than PostgreSQL's native transaction model, especially under contention. DynamoDB TransactWrite works for ≤ 25 items; anything more complex needs the Saga pattern.

4. **Low-cardinality data.** Redis sorted sets have overhead; if your leaderboard has 10 users, a Python dictionary is faster. PostgreSQL is simpler and faster for small data.

5. **Read patterns that don't match the Cassandra schema.** Cassandra requires knowing access patterns at schema-design time. If product requirements change, a new access pattern may require a new table and backfill migration — more painful than adding an index in PostgreSQL.

**The honest meta-answer:** "I reach for the right tool for the access pattern, not for the hype. I validate the bottleneck before switching data stores."

---

## Q5: "DynamoDB throttles your reads at peak. How do you handle it?"

**What they're testing:** Understanding of DynamoDB capacity model and caching strategy.

**Strong answer:**

**First, understand the cause:**
- DynamoDB limits are at the **partition level**, not just the table level. If all traffic hits a few partition keys (hot keys) — even on-demand mode has per-partition limits.
- Signs: `ProvisionedThroughputExceededException` or `RequestLimitExceeded`.

**Immediate mitigations:**

1. **Switch to on-demand mode** if on provisioned — auto-scales, but still per-partition limits.

2. **Add DAX (DynamoDB Accelerator):** Managed Redis-compatible cache in front of DynamoDB. Read hit from DAX (< 1ms), reducing RCU consumption by 90%+. Ideal for read-heavy, cache-friendly data.

3. **Exponential backoff + jitter on client:** AWS SDK retries with exponential backoff, but adding ±25% jitter prevents thundering herd of retries.

4. **Review access patterns for hot keys:** If one PK is being read 1000x more than others (e.g., trending product), implement application-level cache (Redis) for that specific item.

5. **Add a random suffix to hot keys (write sharding):**
```
key: "trending_product#123"  → too hot
split into:
  "trending_product#123#0"
  "trending_product#123#1"
  "trending_product#123#2"
  (write to a random one; read from all and merge)
```

**Prevention:** Set CloudWatch alarms on `SystemErrors` and `ThrottledRequests`. Run load tests before production launch.

---

## Q6: "Explain the difference between MongoDB replica set election and Cassandra leader election."

**What they're testing:** Deep understanding of how each system achieves high availability.

**MongoDB:**
- Uses a **Raft-like protocol** for leader election within a replica set (3+ nodes)
- Nodes vote for a new primary when the current primary fails
- Requires **majority consensus** (2 of 3 nodes must agree)
- Automatic failover in ~10–30 seconds
- One primary at all times — all writes go to primary, reads optionally to secondaries

**Cassandra:**
- **No leader election** — there is no single leader. Every node is equal.
- Consistent hashing ring determines which nodes are responsible for which key ranges
- Writes and reads can go to ANY node (coordinator); the coordinator routes to the correct replica
- "Election" in Cassandra context only means choosing a **coordinator for a request** — and even that is just client-side load balancing
- This is why Cassandra has no single point of failure and no election convergence time — any node can fail without triggering an election

**Interview punchline:**
"MongoDB's HA relies on replica set election with a brief unavailability window during failover. Cassandra has no such window because there's no 'master' to fail over from — any surviving replica can serve requests immediately."

---

## Q7: "Why might you choose Redis over Memcached when both could serve the same caching use case?"

**What they're testing:** Understanding of Redis's richer feature set and when it matters.

**Strong answer:**

For pure caching (GET key / SET key value), both work. But Redis wins as the default choice because it offers capabilities you'll almost certainly need as the system grows:

| Feature                          | Redis | Memcached |
|----------------------------------|-------|-----------|
| Data structures (lists, sets, sorted sets, hashes) | ✅ | ❌ Only strings |
| Persistence (RDB snapshots + AOF) | ✅ | ❌ In-memory only |
| Pub/Sub messaging                | ✅   | ❌               |
| Distributed lock (SET NX PX)    | ✅   | ❌               |
| Atomic increment (INCR)         | ✅   | ✅               |
| TTL per key                     | ✅   | ✅               |
| Cluster / HA                    | ✅ (Sentinel, Cluster) | ✅ (client-side consistent hashing) |
| Lua scripting                   | ✅   | ❌               |

**When Memcached is still a valid choice:**
- Pure caching, no need for persistence
- Extremely memory-sensitive (Memcached uses less overhead per key)
- Multi-threaded (Memcached is multi-threaded; Redis single-threaded by default — though Redis 6+ added I/O threading)

**In practice:** I almost always choose Redis. The added operational complexity of running two different cache systems (Redis for locks/leaderboards, Memcached for caching) isn't worth the marginal memory saving.
