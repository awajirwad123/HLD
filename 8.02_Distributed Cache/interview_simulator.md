# Distributed Cache — Interview Simulator

## Scenario 1: "Design the caching layer for a social network profile service"

**Interviewer:** "We have 500M users. A user's profile page is viewed ~10 times per day on average. Profile updates are infrequent (maybe once a week). Design the caching strategy."

---

### Strong Answer

**Scale:** 500M users × 10 views/day ÷ 86,400s ≈ 57,870 reads/sec. Profile updates: 500M / 7 days / 86,400s ≈ 826/sec.

Read:Write ≈ 70:1 → cache-heavy design.

**What to cache:**
Don't cache the entire user object (~10KB with profile media URLs). Cache the read model for the profile page: `{name, username, bio, avatar_url, follower_count, following_count, post_count, is_verified}` ≈ 300 bytes.

**Cache key:** `profile:{user_id}` → JSON string, TTL = 1 hour.

**Eviction:** `allkeys-lfu` — popular users (celebrities) accessed far more often than average users. LFU keeps celebrities in cache permanently; LRU would evict them if they weren't accessed in the last N minutes.

**Invalidation on update:** User updates bio → write to DB → `DEL profile:{user_id}`. Next read repopulates. Accept up to ~1 second stale read during the invalidation window (tiny).

**Memory sizing:** 57,870 rps with 95% cache hit rate → 2,900 DB reads/sec. Assume 300 bytes/entry × 500M users = 150GB if you cache all users. But power-law: top 10M users generate 90% of traffic. Cache top 10M = 3GB — tiny.

**Cluster design:**
- 3-node Redis Cluster
- 64GB RAM per node → 192GB total → room for growth
- One replica per primary → 6 nodes total
- `allkeys-lfu` eviction

**Read path:** GET → Redis hit (95%) → return. Redis miss → PostgreSQL read replica → SET cache → return.
**Write path:** PUT profile → Postgres primary → DEL cache → return.

**Follow-up: What if we need real-time follower counts?**
Follower counts change on every follow/unfollow. Storing `follower_count` in the profile cache means it goes stale on every follow. Solution: store follower count in a separate Redis key as a counter (`INCR followers:{user_id}`), not in the profile cache. Profile cache stores a snapshot; the live counter is fetched separately and merged at read time.

---

## Scenario 2: "Your Redis cache is causing more problems than it solves. Cache hit rate is only 30%. Debug and fix."

**Interviewer:** "Our Redis cache has a 30% hit rate. DB is under heavy load. What's wrong?"

---

### Investigation and Fix

A 30% hit rate means 70% of reads miss the cache and go to the DB. Root causes to investigate:

**Cause 1: TTL too short.**
`TTL(cache_keys)` — what's the average remaining TTL? If most keys expire in < 60 seconds, they're evicted before being reused. Fix: increase TTL to match read frequency.

**Cause 2: Cache key not matching query patterns.**
Example: caching `user:{id}` but the app queries `user:{id}?fields=name,email` — generating a different URL-based cache key each time even for the same user. Fix: normalize cache keys to the entity ID, strip query parameters.

**Cause 3: Low cache capacity → high eviction rate.**
`INFO stats | grep evicted_keys` — are keys being evicted? If eviction rate is high, maxmemory is too small for working set. Fix: increase Redis memory or add nodes.

**Cause 4: Cache population not happening on writes.**
Application updates user in DB but forgets to SET the new value in cache (not just DELETE). Next read correctly goes to DB, but if the same request also writes a stale value, the cache key keeps getting written with old data. Review write paths.

**Cause 5: Key explosion from bad cache key design.**
Cache key includes request timestamp or nonce → every request generates a unique key → 0% reuse. Fix: cache key = entity identity only.

**Cause 6: Cold start after deployment.**
If the cache was flushed during deployment, warm-up takes time. During warm-up: hit rate starts at 0% and climbs. Check if this is a transient issue.

**Metrics to add:** Track cache hit/miss separately per entity type. Profile, post, comment — which has the worst hit rate? Focus optimization there.

---

## Scenario 3: "Design Redis as a leaderboard for a real-time gaming platform"

**Interviewer:** "1M concurrent players. Score updates every few seconds. We need: real-time rank of any player, top-100 leaderboard, player's rank + score in < 5ms."

---

### Design

**Data structure: Redis Sorted Set (ZSET)**

```redis
ZADD leaderboard {score} {player_id}
ZRANK leaderboard {player_id}           # Rank (0-indexed, low=worst)
ZREVRANK leaderboard {player_id}        # Rank (0-indexed, high=best)
ZRANGEBYSCORE leaderboard 0 +inf        # All players by score
ZREVRANGE leaderboard 0 99 WITHSCORES  # Top 100
ZSCORE leaderboard {player_id}          # Player's score
```

**Score update on kill/achievement:**
```redis
ZINCRBY leaderboard 100 player_456      # Add 100 points, O(log N)
```

**Complexity:**
- ZADD / ZINCRBY: O(log N) — N = number of players
- ZREVRANGE for top 100: O(log N + 100) ≈ O(log N)
- ZREVRANK for a player: O(log N)

At 1M players: log(1M) ≈ 20 operations. Redis handles this at ~1M ops/sec → no problem.

**Memory:** Each Sorted Set entry ≈ 50 bytes. 1M players × 50 bytes = 50MB. Trivial.

**Sharding:** 1M players fit in one Sorted Set on one node. No sharding needed.

**Separate leaderboards:** Daily, weekly, all-time:
```redis
leaderboard:alltime
leaderboard:2026-04-15   # TTL = 24 hours after day ends
leaderboard:2026-W16     # TTL = 7 days after week ends
```

Player's rank and score across all three fetched in one pipeline:
```redis
PIPELINE:
  ZREVRANK leaderboard:alltime player_456
  ZSCORE   leaderboard:alltime player_456
  ZREVRANK leaderboard:2026-04-15 player_456
  ZSCORE   leaderboard:2026-04-15 player_456
```

**Persistence:** Use AOF (`everysec`) — leaderboard data must survive Redis restart. Alternatively: DB is the source of truth; Redis is rebuilt from DB on startup (acceptable if DB query is fast).
