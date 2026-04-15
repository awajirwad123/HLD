# Distributed Cache — Tricky Interview Questions

## Q1: "You're using cache-aside. A user updates their profile and the cache key is deleted. Before the next read repopulates the cache, another write comes in. What happens?"

**The scenario:**

```
Thread A: PUT /users/1  → write DB → DELETE cache:user:1
Thread B: GET /users/1  → cache miss → reads DB (gets A's value) → SET cache:user:1
Thread C: PUT /users/1  → write DB → DELETE cache:user:1     ← happens between B's read and B's write
Thread B: SET cache:user:1  with C's value? No — B read before C's write. B caches STALE data.
```

**The race condition:**
1. B reads DB → gets version 1
2. C writes DB → version 2; DELETEs cache
3. B writes cache → caches version 1 (stale)
4. Cache now has version 1, DB has version 2, until B's TTL expires

**Solution options:**

1. **Add TTL as the backstop.** Accept that cache may be briefly stale (up to TTL). For most applications (profile reads), 5-minute stale is acceptable.

2. **Versioned cache keys.** Include version in the key: `user:1:v{db_version}`. Writer increments a version counter in DB. Readers always look up by current version. Old keys simply expire. Write: `INCR user:1:version → v=5 → SET user:1:v5 → DELETE user:1:v4`.

3. **Read-your-writes via sticky sessions.** Route user 1's requests to the same cache shard that just received the DELETE. Subsequent reads hit the same shard, which has no stale data — it just does a DB read.

4. **Event-driven invalidation (CDC).** DB writes emit change events (Debezium → Kafka). A dedicated cache invalidation consumer processes events in order, guaranteeing the invalidation happens after the DB write reaches the replica. No race window.

**The insight:** Pure cache-aside always has this race. Accept it with TTL, or use CDC for strict consistency.

---

## Q2: "Your Redis memory is full and evicting keys. Some of the evicted keys are NOT cached DB data — they're session tokens. How do you prevent this?"

**The problem:** You're using one Redis instance for both the session store and the DB cache. `allkeys-lru` evicts session tokens (which may be infrequently accessed) to make room for high-traffic cache data.

**Wrong answer:** Disable eviction → the cache fills up → new cache entries fail → 100% DB load.

**Correct approach: Separate Redis instances (or separate logical databases with different eviction)**

Option A: Two Redis instances:
- `redis-cache:6379` — eviction: `allkeys-lru`, no persistence
- `redis-sessions:6380` — eviction: `noeviction`, AOF persistence, `maxmemory-policy noeviction`

Session data must never be evicted because it cannot be regenerated from the DB. Cache data can always be regenerated (it IS the DB).

Option B: Single Redis with `volatile-lru`. Mark cache keys with TTL; sessions have no TTL. `volatile-lru` only evicts keys WITH TTL → sessions survive.

**The insight:** Mixing cache (ephemeral, evictable) and session store (durable, never evict) on the same Redis instance with `allkeys-lru` is a data loss waiting to happen.

---

## Q3: "We want to cache 500GB of data but our Redis nodes only have 64GB RAM each. How do you design this?"

**Option 1: Redis Cluster sharding.**
8+ nodes × 64GB = 512GB. Data distributed via CRC16 hash slots. Client routes each key to the correct node. Zero application change needed (redis-py `redis.cluster.RedisCluster` handles routing). Works transparently.

**Option 2: Tiered caching.**
Not all 500GB is equally hot. Power-law: top 10% of keys (50GB) serve 90% of reads.
- L1: Redis Cluster with 64GB total — holds top 10% (fits easily)
- L2: Memcached or Redis with larger nodes for warm tier
- L3: DB for cold data

Use a frequency-aware routing: if a key has been accessed < 10 times in 24h, don't even cache it in L1.

**Option 3: Compression.**
`OBJECT ENCODING key` — check if your cached values are raw JSON strings. Compress values before storing: `zlib.compress(json.dumps(obj)) → store bytes`. Typical JSON compresses 3–10×. 500GB uncompressed → 100GB compressed → fits on 2 large nodes.

**Option 4: Cache only metadata, not full objects.**
Rethink what goes in cache. Store only the most-queried fields rather than full documents. A user object might be 10KB; if reads only need `{name, avatar_url}`, cache just those 100 bytes for 100× density.

---

## Q4: "Redis is reporting CPU at 90% despite being single-threaded. What's causing it and how do you fix it?"

**Common causes of high Redis CPU (all unexpected for O(1) ops):**

1. **Slow commands in the key space:** `SMEMBERS`, `LRANGE 0 -1`, `SORT`, `KEYS *`, `SINTERSTORE` on large collections. These are O(N) and block the event loop. Use `SLOWLOG GET 10` to find offending commands.
   - Fix: replace `SMEMBERS` with `SSCAN`; never use unbounded `LRANGE`.

2. **AOF rewrite (BGREWRITEAOF):** CPU spike when Redis forks to rewrite AOF. Schedule rewrites during off-peak hours.

3. **Very high connection count driving I/O thread load:** Since Redis 6.0, I/O threads consume CPU. If you have 10,000 connections sending small commands, I/O overhead dominates.
   - Fix: connection pooling (fewer connections); pipelining (batch commands per connection).

4. **Lua script (EVAL) abuse:** Long Lua scripts run synchronously on the event loop.

5. **Cluster gossip overhead:** In a large cluster (>100 nodes), cluster gossip protocol consumes significant CPU. Reduce cluster size or use cluster-proxy.

**The insight:** Redis single-threaded command execution rarely saturates CPU. High CPU usually means O(N) commands or architectural issues (too many connections, large Lua scripts).

---

## Q5: "A caching system with TTL = 5 minutes for user data. A user gets banned. The ban must take effect within 30 seconds app-wide. How do you design this?"

**The problem:** With a 5-minute TTL, a banned user still appears active in cache for up to 5 minutes.

**Options:**

1. **Active invalidation on ban event.** When a ban is issued: `DEL user:{id}` on all Redis shards. Next request hits DB, gets the banned status, and doesn't re-cache (or caches `{banned: true}` with a 30-second TTL).

2. **Separate ban store with extremely short TTL.** Store a `banned:{user_id} = 1` key with TTL=30s. Auth middleware checks this key FIRST (before the full user cache). If present → reject. Ban service sets this key; it expires naturally after 30s.

3. **Event-driven invalidation via Pub/Sub.** Each application instance subscribes to `user:banned` Redis channel. When a ban fires: `PUBLISH user:banned user_id`. All subscribers immediately invalidate their local L1 cache for that user_id. Remote-origin ban propagates in < 10ms.

4. **Reduce TTL to 30s.** Simplest fix if ban latency is the SLA. Cost: 2× more DB reads. Evaluate if the DB can handle the increased load.

**Best design:** Option 2 (separate ban key) + Option 1 (active invalidation). The ban key is a cheap boolean lookup (< 0.1ms); the full user object is cached normally. Ban takes effect within 30s even without active invalidation, and immediately with it.
