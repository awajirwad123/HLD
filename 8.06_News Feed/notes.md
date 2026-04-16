# News Feed — Notes & Reference

## Fan-Out Comparison

| | Fan-out on Write (Push) | Fan-out on Read (Pull) | Hybrid |
|---|---|---|---|
| Write load | High (N writes per post, N = followers) | Low | Medium (skip celebrities) |
| Read load | Very low (pre-built feed) | Very high (N queries per read) | Low (merge at read time) |
| Feed freshness | Near-instant | At read time | Slightly delayed for regular |
| Celebrity problem | Catastrophic | Fine | Solved |
| Used by | Early Twitter | Legacy systems | Twitter, Instagram, Facebook |

---

## Feed Storage in Redis

```
Key:    feed:{user_id}
Type:   Sorted Set
Score:  post_created_at (Unix timestamp, OR Snowflake ID — both monotone)
Member: post_id

Operations:
  ZADD feed:alice 1713175200 post:789      # Add post to alice's feed
  ZREVRANGE feed:alice 0 49 WITHSCORES     # Get 50 most recent
  ZREVRANGEBYSCORE feed:alice +inf {since}  # Get posts since timestamp (pagination)
  ZREMRANGEBYRANK feed:alice 0 -1001        # Trim to last 1000 posts
  ZCARD feed:alice                          # Feed size
```

---

## Celebrity Threshold Strategy

```
if follower_count > CELEBRITY_THRESHOLD (e.g., 100K):
  Store post_id in celebrity_posts:{user_id} sorted set (capped at 200 posts)
  
On feed read for User X:
  1. Regular pre-computed feed: ZREVRANGE feed:{X} 0 49
  2. Celebrity follows for X: SMEMBERS celebrity_follows:{X}  → [elonmusk, taylorswift]
  3. For each celebrity (max 5):
     ZREVRANGE celebrity_posts:{celeb_id} 0 9 (last 10 posts)
  4. Merge, sort, deduplicate, return
```

---

## Post ID: Why Snowflake

- Time-ordered: `post_id` naturally sorts chronologically
- `ZADD feed:{uid} post_id post_id` — post_id IS the score (no timestamp duplication)
- Globally unique: no two posts get the same ID
- No central coordinator: each server generates its own Snowflake IDs

---

## Like / Comment Count Design

| Approach | Write throughput | Accuracy |
|---|---|---|
| `UPDATE posts SET likes+1` per like | High DB write load on popular posts | Exact |
| Redis `INCR likes:{post_id}` + flush | Near-zero DB writes | ≤30s delay |
| Separate `likes` table + COUNT(*) | Accurate, audit trail | Expensive COUNT query |

**Deduplication:** Store `SADD liked_by:{post_id} {user_id}` in Redis. `SISMEMBER` before incrementing. For scale: use Redis Bloom Filter instead of SET (probabilistic, much smaller memory).

---

## Feed Ranking (Algorithmic)

```
Score(post) = f(recency, engagement, affinity, content_type)

Recency:    exp(-λ × age_hours)         λ = 0.1
Engagement: log(1 + likes + 2×comments + 3×shares) × velocity_multiplier
Affinity:   P(interact | author_follows) — learned from user behavior
Type:       user's historical preference for video vs image vs text

Final: weighted_sum of above factors (weights tuned by ML)
```

Candidate generation: get top 500 by recency from pre-computed feed.
Ranking: score each with the formula above. Return top 20.

---

## Follow/Unfollow Operations

```
Follow (A follows B):
  1. INSERT INTO follows (follower_id=A, followed_id=B)
  2. SADD following:{A} B
  3. INCR follower_count:{B}
  4. Backfill: inject B's last 20 posts into A's feed
     ZADD feed:{A} {post_id_score}... for B's recent posts

Unfollow (A unfollows B):
  1. DELETE FROM follows WHERE follower_id=A AND followed_id=B
  2. SREM following:{A} B
  3. DECR follower_count:{B}
  4. Do NOT clean feed:{A} immediately (lazy cleanup)
     Feed loader filters by current following set at render time
```

---

## Key Numbers

| Metric | Value |
|---|---|
| Posts/day | 10M |
| Posts/sec | ~116 |
| Feed reads/sec | ~29K |
| Fan-out writes/sec (average) | ~23K (116/sec × 200 avg followers) |
| Celebrity threshold | 100K followers |
| Feed Redis key TTL | 30 days inactivity expiry |
| Max feed size in Redis | 1,000 posts per user |
| Post metadata cache TTL | 24 hours |
