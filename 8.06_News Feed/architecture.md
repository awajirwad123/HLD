# News Feed / Timeline (Twitter / Instagram) — Architecture

## Problem Statement

Design a news feed/timeline system:
- User sees a chronologically ranked feed of posts from people they follow
- ~500M users, each following ~200 accounts on average
- 10M new posts/day created
- Feed must load in < 200ms

---

## Capacity Estimation

```
Posts created: 10M/day → ~116/sec
Feed reads: 500M × 5 feed refreshes/day → 2.5B reads/day → ~29K reads/sec
Average following: 200 accounts

Fan-out on write (naive): 116 posts/sec × 200 followers = 23,200 writes/sec to fan-out lists
Heavy user (1M followers): 1 post → 1M writes
```

---

## Core Design Challenge: Fan-Out

When User A posts, their followers should see it in their feed. There are two fundamental approaches:

### Approach A: Fan-Out on Write (Push Model)

When a post is created, immediately push it to every follower's feed list.

```
Post created by User A (has 200 followers)
      ↓
Fan-out Service:
  GET followers(A) → [user_1, user_2, ..., user_200]
  For each follower: LPUSH feed:{follower_id} {post_id}
  (200 writes)
```

**Pros:** Feed reads are instant (pre-computed), read latency is O(1)
**Cons:** Celebrity problem — Elon Musk has 50M followers → 1 tweet = 50M Redis writes = catastrophic write amplification

### Approach B: Fan-Out on Read (Pull Model)

When a user opens their feed, dynamically fetch recent posts from all accounts they follow.

```
User opens feed:
  GET following(user) → [A, B, C, ..., 200 accounts]
  For each account: GET recent_posts(account, limit=10)
  MERGE + SORT by timestamp → return top 50
```

**Pros:** No write amplification, always fresh
**Cons:** 200 queries per feed read → fan-in is expensive; amplified at 29K reads/sec = 5.8M DB queries/sec

### Approach C: Hybrid (Twitter / Instagram actual approach)

- **Normal users (<100K followers):** Fan-out on write — push post_id to followers' Redis feed lists
- **Celebrity users (>100K followers):** Fan-out on read — don't pre-push; when followers load feed, the celebrity's recent posts are fetched and merged at read time

```
User opens feed:
  1. Read pre-computed feed list from Redis (all non-celebrity posts)
  2. Fetch recent posts from each celebrity they follow (< 5 at most)
  3. Merge + sort → return top 50
```

This bounds read amplification (at most 5 celebrity fetches per load) while avoiding celebrity write amplification.

---

## Data Model

### Post Storage (Cassandra or PostgreSQL)

```sql
-- PostgreSQL for smaller scale
CREATE TABLE posts (
    id          BIGINT PRIMARY KEY,   -- Snowflake ID (time-ordered)
    user_id     BIGINT NOT NULL,
    content     TEXT,
    media_urls  TEXT[],               -- Array of S3 URLs
    like_count  BIGINT DEFAULT 0,
    comment_count BIGINT DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_posts_user_time ON posts (user_id, id DESC);
-- Enables: "last 10 posts by user X" efficiently
```

### Feed List (Redis)

```
Key: feed:{user_id}
Type: Redis Sorted Set
Score: post_created_at (Unix timestamp)
Member: post_id

ZADD feed:{user_id} {timestamp} {post_id}
ZREVRANGE feed:{user_id} 0 49              # Get 50 most recent post IDs
ZREMRANGEBYRANK feed:{user_id} 0 -1001     # Trim to last 1000 posts
```

Sorted Set allows efficient range queries by score (timestamp) and trimming old entries.

---

## Full Architecture

```
Post creation:
Client → POST /posts → Post Service → Postgres (persist) → Kafka (fan-out event)
                                                                 ↓
                                                    Fan-out Service (consumers):
                                                    - Regular user: ZADD feed:{follow_id}
                                                    - Celebrity: skip (read-time fan-out)

Feed read:
Client → GET /feed → Feed Service
  1. ZREVRANGE feed:{user_id} 0 49 → 50 post_IDs (Redis)
  2. Fetch celebrity posts (if user follows celebrities)
  3. Merge + deduplicate + sort
  4. Enrich with post data: MGET posts:{id} for each ID (Redis or DB)
  5. Return JSON
```

---

## Feed Ranking

**Chronological feed:** Sort by creation timestamp. Simple. Used by Twitter's "Latest" tab.

**Ranked/algorithmic feed (Instagram, Facebook):** Score each post candidate by:
- Recency: newer posts score higher
- Engagement: posts with high like/comment/share velocity score higher
- Author affinity: how often do you interact with this person?
- Content type preference: user watches more videos → videos score higher

This requires an **online scoring model** (fast inference at read time on the top N candidates) + a **candidate generation** step (retrieve top 500 candidates by recency, then rank the 500).

---

## Celebrity / Influencer Problem

| Scale | #Followers | Fan-out on write | Fan-out on read |
|---|---|---|---|
| Normal user | 200 | 200 writes → fast | Too few to matter |
| Mid-tier | 10K | 10K writes → OK | Expensive |
| Influencer | 1M | 1M writes → very slow | OK (1 query) |
| Celebrity | 50M | 50M writes → catastrophic | 1 query |

**Threshold (Twitter internal, estimated):** Users with > 100K followers bypass write fan-out.

---

## Handling Follows/Unfollows

**New follow:** User A follows User B.
- Immediately inject B's last 20 posts into A's feed (backfill)
- Add B to A's following set: `SADD following:{A} {B}`

**Unfollow:** User A unfollows User B.
- Remove B from A's following set
- Don't need to clean up A's feed immediately — posts from B in the feed will simply not pass the author filter on load (lazy cleanup)

---

## Key Scalability Decisions

| Decision | Rationale |
|---|---|
| Snowflake IDs for post_id | Time-ordered → can sort feed by ID without timestamp lookup |
| Redis Sorted Set for feed | O(log N) insert, O(log N + K) range query — ideal for feed |
| Hybrid fan-out | Bounded write AND read amplification |
| Pre-computed feed capped at 1000 entries | Users don't scroll infinitely; older entries are paginated from DB |
| Post metadata cache | Post data cached in Redis (content, likes, author info) for feed enrichment |
