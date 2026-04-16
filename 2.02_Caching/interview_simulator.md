# Caching — Interview Simulator

*Full mock scenarios. Set a 20-minute timer per simulation.*

---

## Simulation 1: Design a Caching Layer for an E-Commerce Product Catalog

### Scenario
> You're designing the caching strategy for Amazon-like product catalog. 50M products, 10M DAU, reads are 200x more frequent than writes. Product price/stock updates happen ~1M times/day. p99 read latency must be < 50ms.

---

### Guided Walkthrough

**Clarify first:**
- Is price accuracy critical? (shows wrong price → purchase fails at checkout)
- Stock count: approximate OK? ("3 left" is fine if off by 1)
- User reviews: separate from product data? (usually yes — different freshness)

**Estimate:**
```
200M reads/day = ~2,300 read QPS
1M writes/day = ~12 write QPS
Read:write = ~200:1 → ideal for caching

Product object size = ~2KB (name, description, price, images list)
Top 1% products (500K) = 80% of traffic → cache them
Cache size: 500K × 2KB = 1GB → fits in a single Redis node
```

**Caching strategy decision:**

```
Read path: Cache-Aside
  GET /product/:id → Redis HIT (< 1ms) ✅
                   → Redis MISS → PostgreSQL read → populate Redis (TTL: 5min)

Write path: Invalidate on write
  PATCH /product/:id/price → update DB → DELETE Redis key
  Next read gets fresh data from DB → cache warmed again
```

**Why 5-minute TTL?**
- Price changes: 1M/day across 50M products = 0.02 changes/product/day = rarely
- If a price change isn't immediately reflected: user might see stale price on listing → checkout will show correct price (reads from DB at payment step)
- Worst case: user sees old price for 5 min → acceptable business trade-off

**For stock count (more sensitive):**
- Don't cache stock count directly — use Redis `INCRBY` / `DECRBY` as the source of truth for in-memory stock reservation
- DB updated async every N seconds or on order completion
- This is write-back for stock, not cache-aside

**Multi-layer cache:**
```
Browser  → HTTP Cache-Control: public, max-age=60   (1min browser cache for product page)
CDN      → s-maxage=300 for product detail pages    (5min CDN cache)
Redis    → product:{id} TTL 300s                    (distributed cache)
DB       → PostgreSQL primary (source of truth)
```

**Cache warming on startup:**
```
Background job: SELECT id FROM products ORDER BY view_count DESC LIMIT 500000
→ pre-populate Redis before accepting traffic
```

**Failure mode:**
- Redis down → all reads fall through to PostgreSQL (~50ms, above SLA)
- Mitigation: Redis Sentinel for HA; in-process LRU cache as L1 for top-100 products as last resort

---

## Simulation 2: Design Caching for a Real-Time Leaderboard

### Scenario
> Design the caching strategy for a gaming leaderboard showing top 100 players by score. 1M players, scores update every 30 seconds per active player. Leaderboard must be eventually consistent (2-second lag acceptable).

---

### Guided Walkthrough

**Why this is different:**
- Leaderboard is write-heavy (1M updates/30s = ~33K writes/sec)
- Traditional cache-aside won't work — invalidating the leaderboard key on every score update = invalidation storm
- Need a data structure that supports efficient sorted reads

**Redis Sorted Set — the right tool:**
```
ZADD leaderboard <score> <player_id>   # O(log N) insert/update
ZREVRANGE leaderboard 0 99 WITHSCORES  # O(log N + K) top 100 read
ZINCRBY leaderboard 50 player_123      # atomic score increment
ZRANK leaderboard player_123           # player's rank, O(log N)
```

**Architecture:**
```
Game server → score update → Redis ZINCRBY (immediate, in-memory) ← source of truth for leaderboard
                           → async DB write via queue (for persistence)

Leaderboard read:
  Client → GET /leaderboard/top100
  API server → ZREVRANGE leaderboard 0 99 WITHSCORES
  → returns top 100 in O(log N + 100) ≈ < 1ms ✅
```

**Why not cache-aside here?**
- The leaderboard IS the cache (Redis sorted set is the read model)
- DB is just for durability/recovery, not the primary read source
- This is a write-back + CQRS pattern: writes go to Redis first, DB gets async updates

**Persistence strategy:**
```
Every 60 seconds:
  ZRANGE leaderboard 0 -1 WITHSCORES → batch upsert to PostgreSQL
```
Loss window: 60 seconds of scores if Redis crashes. Acceptable per requirements (2s eventual consistency is about read lag, not durability).

**Cache eviction:**
Sorted sets in Redis don't need LRU eviction — the leaderboard is a fixed set of 1M players, ~50MB. No eviction needed if memory is sized correctly.

---

## Simulation 3: Caching Strategy Self-Assessment

For any system design you're working on, walk through this checklist before finalizing:

### Caching Completeness Checklist

```
Read path:
  [ ] What cache strategy? (aside, through, read-through)
  [ ] What is the cache key? (specific, not too broad, not infinite cardinality)
  [ ] What is the TTL? (justified: how stale is acceptable?)
  [ ] What is the cache hit rate target?
  [ ] What happens on cache miss? (DB fallback, latency impact?)

Write path:
  [ ] How is the cache invalidated on data change?
  [ ] Who invalidates? (writer, event consumer, TTL expiry?)
  [ ] Race conditions? (replica lag, double-write, concurrent updates?)

Failure modes:
  [ ] What if Redis is unavailable? (graceful degradation to DB?)
  [ ] Cache stampede risk? (high TTL near expiry on hot keys?)
  [ ] Memory limit set? (maxmemory + eviction policy configured?)

Observability:
  [ ] Cache hit/miss rate monitored?
  [ ] Redis memory usage alerted?
  [ ] Slow key queries monitored?
```

**Score yourself:**
- 15/15: Rock solid caching design
- 10–14: Good, a few gaps to address
- < 10: Revisit the strategy — missing pieces will become production incidents
