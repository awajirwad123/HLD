# News Feed — Interview Simulator

## Scenario 1: "Design Twitter's News Feed"

**Interviewer:** "Design the Twitter / X home timeline feed. Users see posts from accounts they follow, in reverse chronological order (latest-first tab)."

---

### Strong Answer

**Step 1 — Clarify**
> "500M users, 10M posts/day, average 200 follows, celebrity accounts with up to 50M followers. Chronological feed first, with algorithmic ranking as a follow-up?"

**Step 2 — Estimate**
- Posts/sec: 116
- Fan-out writes/sec: 116 × 200 = 23K writes/sec (average case)
- Feed reads/sec: ~29K
- Celebrity fan-out: 1 celebrity tweet → 50M writes (avoid with hybrid model)

**Step 3 — Post creation flow**
```
POST /tweets { content: "Hello world!" }
  → Post Service: assign Snowflake post_id
  → Write to PostgreSQL (primary persisted store)
  → Publish to Kafka topic "new_posts"

Fan-out consumer (Kafka):
  → Fetch follower_count for author
  → if follower_count <= 100K:
      Fetch followers in batches of 500
      ZADD feed:{follower_id} post_id post_id   (Snowflake = score)
      ZREMRANGEBYRANK feed:{follower_id} 0 -1001 (cap at 1000)
  → if follower_count > 100K:
      ZADD celebrity_posts:{author_id} post_id post_id (cap at 500)
```

**Step 4 — Feed read flow**
```
GET /timeline
  1. ZREVRANGE feed:{user_id} 0 49 → 50 regular post IDs
  2. SMEMBERS celebrity_follows:{user_id} → [celeb_ids]
  3. For each celeb_id (max 5): ZREVRANGE celebrity_posts:{celeb_id} 0 9
  4. Merge + deduplicate all post IDs
  5. Sort by Snowflake ID descending (= chronological)
  6. MGET post:{id} for all IDs → enrich with content, author, likes
  7. Return JSON
```

**Step 5 — Scale**
- Fan-out service: multiple Kafka consumer instances (parallelism)
- Redis Cluster: shard feed keys by user_id
- Post store: PostgreSQL with Citus (horizontal sharding) for posts table
- Feed TTL: 30 days for active users; cold keys expire naturally

---

## Scenario 2: "Instagram's algorithmic feed — design the ranking component"

**Interviewer:** "Instagram shows ranked posts (not chronological). How do you rank a user's feed in < 200ms?"

---

### Design

**Two-phase approach:**

**Phase 1 — Candidate generation (< 5ms):**
```
ZREVRANGE feed:{user_id} 0 499 → 500 candidate post IDs
MGET celebrity_posts for each celebrity followed → up to 50 more
Merge → 500–550 candidates
```

**Phase 2 — Ranking (< 30ms):**
Fetch features for each candidate from Redis MGET:
- `post_age_hours`
- `like_count`, `comment_count`, `share_count` (engagement)
- `author_follower_count`
- `user_has_liked_author_recently` (affinity)
- `media_type` (video/image/text)

Apply fast scoring formula:
```python
score = (
    recency_score(age_hours) * 0.3 +
    engagement_score(likes, comments, shares) * 0.4 +
    affinity_score(user_id, author_id) * 0.2 +
    content_type_preference_score(user_id, media_type) * 0.1
)
```

Sort by score, return top 20.

**Total latency:** < 50ms in practice (Redis MGET for 550 keys ≈ 2ms, scoring = CPU-bound < 10ms).

---

## Scenario 3: "A user with 200 follows opens the app after 2 weeks offline. What does their feed look like and how do you handle it?"

**Challenge:** Their Redis feed list has only the last 1000 posts (capped). They missed 10,000+ posts. Pre-computed feed is stale; celebrity posts have rotated out.

**Design:**
1. Detect "very stale" user: if last_seen > 7 days ago, skip pre-computed feed.
2. Regenerate feed from DB: `SELECT p.id FROM posts p JOIN follows f ON p.user_id = f.followed_id WHERE f.follower_id = X AND p.created_at > X.last_seen ORDER BY p.id DESC LIMIT 100`
3. Cache this freshly generated feed in Redis for the next 24 hours.
4. Return the fresh timeline with a "showing posts from last 2 weeks" banner.
5. Load more: subsequent scroll requests use the pre-computed cached feed.

**The insight:** Don't try to reconstruct 2 weeks of missed posts. Show a manageable "recent catch-up" slice. Twitter does this — returning to the app after days shows a curated recent subset, not the full backlog.
