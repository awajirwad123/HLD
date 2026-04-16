# Caching — Tricky Interview Questions

*Questions that test deep understanding — not just "add Redis."*

---

## Q1: "You cached user profile data with a 1-hour TTL. A user updates their email, but for the next hour other parts of the system still see the old email. Is this acceptable? How do you fix it?"

**Trap:** Saying "just lower the TTL" — that only reduces staleness, doesn't eliminate it.

**Strong answer — distinguish the use cases:**

First, ask: *who* reads the cached user profile, and does the old email matter to them?

- **Personalized UI display** ("Welcome, Alice") — 1 hour staleness is probably fine
- **Email outbox service** sending password resets — **not fine** — must use fresh email

**Fix options ranked by strength:**

1. **Invalidate on write** (best for most cases):
   ```python
   def update_email(user_id, new_email):
       db.update(user_id, email=new_email)
       redis.delete(f"user:{user_id}")   # immediate invalidation
   ```
   Next read gets fresh data from DB. Gap window = zero.

2. **Write-through** — update cache and DB on every write simultaneously. Cache always fresh. Slower writes.

3. **Domain partitioning** — don't cache the email field separately from the profile. For the email outbox service, bypass the cache entirely and always read from DB (strong consistency for critical fields).

**Senior signal:**
> *"Cache key design matters. If `user:{id}` caches the full profile, any field change must invalidate the entire key. Alternatively, cache subsets: `user:{id}:prefs` can have a 1-hour TTL; `user:{id}:contact` should be invalidated on write or never cached for services that need fresh contact info."*

---

## Q2: "Your cache hit rate is 45%. The interviewer says that's too low. What do you do?"

**45% = nearly every other request hits the DB. Something is wrong.**

**Diagnostic checklist:**

```
1. Are TTLs too short?
   → keys expire before they can be hit again
   → fix: increase TTL for stable data

2. Is the cache capacity too small?
   → LRU is evicting keys before they get a second hit
   → fix: increase maxmemory or add cache nodes

3. Is the key space too high-cardinality?
   → every request generates a unique key (e.g., cached sorted result with dynamic page+filters)
   → fix: simplify key, cache at coarser granularity

4. Is the workload truly non-cacheable?
   → write-heavy workload (updates > reads) — caching doesn't help
   → one-time reads (long-tail user data) — each key is only read once
   → fix: don't cache these; focus cache on hot-read paths only

5. Are cache keys being invalidated too aggressively?
   → every write flushes the cache even for unrelated data
   → fix: narrow invalidation scope (delete specific keys, not full flush)
```

**Real example:**
> "I've seen this at a marketplace: every search request included sort order and page number in the cache key → `search:shoes:price_asc:page_3`. The key space was effectively infinite so every request was a miss. The fix: cache only the first page of top results (`search:shoes:price_asc:page_1`), which accounts for 80% of traffic."

---

## Q3: "Should you cache the result of a database transaction that spans multiple tables?"

**Nuanced answer — it depends on what you're caching.**

**The problem:**
```
Order summary = JOIN orders + order_items + products + user
Cached as: order_summary:{order_id}

Now:
  - Product price changes → cache is stale
  - Order item added → cache is stale
  - User profile updates → cache is stale
```

Invalidating `order_summary:{order_id}` requires knowing which cache keys are affected by which table writes — complex cross-table dependency tracking.

**Strategies:**

1. **Cache only final, stable aggregates** — completed orders rarely change; their summary can be cached long-term. In-progress orders: don't cache or use very short TTL.

2. **Denormalize at write time** — when an order is placed, compute and store the full order summary in a separate document store (MongoDB/DynamoDB). No JOIN cache needed.

3. **Tag-based invalidation (surrogate keys)**:
   - Tag `order_summary:{id}` with `product:{pid}`, `user:{uid}`
   - When product price changes, invalidate all cache entries tagged `product:{pid}`
   - Cloudflare Cache Tags, Varnish's `xkey` header support this natively

4. **Accept staleness explicitly** — order history page can show stale product names for 5 minutes. State this in the design.

---

## Q4: "Write-back caching sounds great — why doesn't everyone use it?"

**Trap:** Candidates don't know the failure modes.

**The real risks:**

1. **Data loss on cache crash**:
   ```
   Write → Cache (100 updates buffered)
   Redis crashes before async flush
   DB never received those 100 updates
   → permanent data loss
   ```

2. **Inconsistency window**:
   - DB is stale during the buffer period
   - Another service reading directly from DB sees old data
   - Hard to reason about system state

3. **Ordering issues**:
   - Async workers may process writes out of order
   - Update then delete might arrive as delete then update

4. **Recovery complexity**:
   - After a crash, must replay cache WAL (write-ahead log) to DB
   - Added operational overhead

**When write-back IS the right call:**
- View counters, like counts — loss of 1% of increments is acceptable
- Analytics event aggregation — batch flushing to ClickHouse/InfluxDB is standard practice
- Rate limiter counters in Redis — small inaccuracy tolerable; losing counters on crash is acceptable

**Senior answer:**
> *"Write-back is a deliberate trade of durability for performance. I'd only use it for data where approximate correctness is explicitly acceptable — counters, analytics, leaderboard scores. For anything financial or identity-related, I'd never use write-back."*

---

## Q5: "You're designing a social feed. Each user's feed is personalized. How do you cache it?"

**This is one of the hardest caching problems — high fan-out invalidation.**

**The challenge:**
```
User A posts a tweet.
User A has 1M followers.
Each follower has a personalized feed that includes A's tweet.
Invalidating 1M cache keys on every tweet = expensive.
```

**Option 1: Don't cache the raw feed — cache components**
```
Cache individual tweet objects: tweet:{id} (long TTL, rarely changes after post)
Cache follower lists: followers:{user_id} (medium TTL)
Assemble feed at read time from cached components (fast if components are cached)
```
Cache hit on each component, but assembly cost is O(followed_users).

**Option 2: Fan-out on write (pre-computed timelines)**
```
User A posts → async worker pushes tweet_id into each follower's timeline in Redis
Redis sorted set: timeline:{follower_id} → [(tweet_id, timestamp), ...]
Read feed: ZREVRANGE timeline:{user_id} 0 19  → instant, no assembly
```
Write is expensive (1M followers = 1M Redis writes), but reads are O(1).
Twitter uses this approach for users with < 10K followers.

**Option 3: Hybrid (Twitter's actual solution)**
- < 10K followers: fan-out on write (pre-computed timeline in Redis)
- > 10K followers (celebrities): fan-out on read (merge at read time)
- Timeline read = pre-computed base + merge in celebrity tweets at request time

**Cache invalidation for this:**
- Deletion: user deletes tweet → remove tweet_id from all follower timelines (expensive, done async)
- Or: mark tweet as deleted in tweet store; feed assembly skips deleted tweets (lazy invalidation)

---

## Q6: "Your Redis cluster is at 95% memory usage. What do you do right now, and what do you do long-term?"

**Immediate (triage):**

1. **Check `maxmemory-policy`** — if `noeviction`, Redis is about to start rejecting writes. Change to `allkeys-lru` immediately.
2. **Identify largest keys**: `redis-cli --bigkeys` or `SCAN` with `OBJECT ENCODING` + `MEMORY USAGE`
3. **Find hottest keys**: `redis-cli --hotkeys` (requires LFU policy or sampling)
4. **Reduce TTLs on non-critical keys** temporarily to let LRU evict them faster
5. **Emergency: flush non-critical namespaces**: `redis-cli UNLINK "session:*"` (async, non-blocking)

**Long-term fixes:**

1. **Compress values** — JSON → MessagePack (~30% smaller); large strings → gzip before storing
2. **Eliminate over-caching** — use `MEMORY USAGE` to find large keys that aren't hot enough to justify their cost
3. **Increase shard count** in Redis Cluster — redistribute data across more nodes
4. **Add nodes to cluster** — `redis-cli --cluster add-node` reshards automatically
5. **Use Redis as pure cache** — don't store primary data that can't be evicted; keep Redis for cache + ephemeral data only

---

## Q7: "Two services both implement cache-aside for the same database table. Service A updates the DB and deletes the cache. Service B reads 5ms later. What could go wrong?"

**This is the double-delete / race condition problem.**

```
Service A:  write DB → delete cache key
Service B:  cache miss → read from DB (reads the NEW value ✅) → populate cache

But what if replication lag?
Service A:  write to DB primary
Service B:  cache miss → reads from DB REPLICA (which hasn't replicated yet → reads OLD value ❌)
            → populates cache WITH STALE DATA
            → cache now has stale data for the entire TTL duration
```

**This is a real production bug pattern.**

**Fixes:**

1. **Read from primary after cache miss** — for consistency-sensitive paths, bypass replicas on cache miss
2. **Replication-fenced invalidation** — delete cache key AFTER replica confirms the write (use `WAIT` command in PostgreSQL)
3. **Short TTL on the affected key** — worst case staleness is bounded by TTL (not permanent)
4. **Version + conditional set** — only populate cache if the version is newer than what's already there:
   ```python
   # Only cache if version > current cached version
   redis.set(key, value, xx=False)  # or use a Lua script for atomic check-and-set
   ```

**Senior signal:** Raise this race condition proactively in interviews. It shows you've thought about caching beyond the happy path.
