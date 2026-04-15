# News Feed — Hands-On Exercises

## Exercise 1: Fan-Out Service Implementation

```python
"""
Implement fan-out on write with celebrity optimization.
"""

import asyncio
import json
import time
from dataclasses import dataclass

import redis.asyncio as redis
import asyncpg

REDIS_URL = "redis://localhost:6379"
DB_DSN    = "postgresql://user:pass@localhost/feed"

CELEBRITY_THRESHOLD = 100_000   # Followers > this → skip write fan-out
FEED_MAX_SIZE       = 1_000     # Max posts per user feed
FANOUT_BATCH_SIZE   = 500       # Process followers in batches


@dataclass
class Post:
    post_id: int
    user_id: int
    content: str
    created_at: float   # Unix timestamp


async def create_post(pool: asyncpg.Pool, rdb: redis.Redis, post: Post):
    """Persist post and trigger fan-out."""
    # 1. Persist post to DB
    await pool.execute(
        "INSERT INTO posts (id, user_id, content, created_at) VALUES ($1, $2, $3, to_timestamp($4))",
        post.post_id, post.user_id, post.content, post.created_at,
    )

    # 2. Cache post metadata in Redis for fast enrichment
    await rdb.setex(
        f"post:{post.post_id}",
        86400,   # TTL = 1 day
        json.dumps({"post_id": post.post_id, "user_id": post.user_id,
                    "content": post.content, "created_at": post.created_at}),
    )

    # 3. Fan-out to followers
    follower_count = await pool.fetchval(
        "SELECT follower_count FROM user_stats WHERE user_id = $1", post.user_id
    )

    if follower_count and follower_count > CELEBRITY_THRESHOLD:
        # Celebrity: skip write fan-out, mark as celebrity post
        await rdb.zadd(
            f"celebrity_posts:{post.user_id}",
            {str(post.post_id): post.created_at},
        )
        await rdb.zremrangebyrank(f"celebrity_posts:{post.user_id}", 0, -201)  # Keep last 200
        print(f"[Celebrity] Post {post.post_id} by user {post.user_id} — skipping fan-out")
    else:
        # Regular user: fan-out to all followers
        await fanout_to_followers(pool, rdb, post)


async def fanout_to_followers(pool: asyncpg.Pool, rdb: redis.Redis, post: Post):
    """Batch fan-out to all followers."""
    offset = 0
    total_written = 0

    while True:
        followers = await pool.fetch(
            "SELECT follower_id FROM follows WHERE followed_id = $1 LIMIT $2 OFFSET $3",
            post.user_id, FANOUT_BATCH_SIZE, offset,
        )
        if not followers:
            break

        # Batch write to Redis pipeline
        pipe = rdb.pipeline()
        for row in followers:
            follower_id = row["follower_id"]
            pipe.zadd(f"feed:{follower_id}", {str(post.post_id): post.created_at})
            # Trim feed to max size
            pipe.zremrangebyrank(f"feed:{follower_id}", 0, -(FEED_MAX_SIZE + 1))

        await pipe.execute()
        total_written += len(followers)
        offset += FANOUT_BATCH_SIZE

    print(f"[Fan-out] Post {post.post_id} pushed to {total_written} followers")


async def get_feed(pool: asyncpg.Pool, rdb: redis.Redis, user_id: int, page: int = 0) -> list[dict]:
    """
    Read user's feed:
    1. Get pre-computed feed from Redis
    2. Merge celebrity posts
    3. Enrich with post metadata
    """
    PAGE_SIZE = 20
    start = page * PAGE_SIZE
    stop  = start + PAGE_SIZE - 1

    # ── Get pre-computed feed ──────────────────────────────────────────────
    feed_post_ids = await rdb.zrevrange(f"feed:{user_id}", start, stop)

    # ── Get celebrities this user follows ─────────────────────────────────
    celebrity_ids = await rdb.smembers(f"celebrity_follows:{user_id}")

    celebrity_post_ids = []
    if celebrity_ids:
        pipe = rdb.pipeline()
        for celeb_id in list(celebrity_ids)[:5]:   # Max 5 celebrity queries
            pipe.zrevrange(f"celebrity_posts:{celeb_id}", 0, 9)
        celeb_results = await pipe.execute()
        for results in celeb_results:
            celebrity_post_ids.extend(results)

    # ── Merge and deduplicate ──────────────────────────────────────────────
    all_post_ids = list(set(feed_post_ids + celebrity_post_ids))

    # ── Enrich with post data ──────────────────────────────────────────────
    pipe = rdb.pipeline()
    for post_id in all_post_ids:
        pipe.get(f"post:{post_id}")
    cached = await pipe.execute()

    # For cache misses, fall back to DB
    posts = []
    miss_ids = []
    for post_id, cached_data in zip(all_post_ids, cached):
        if cached_data:
            posts.append(json.loads(cached_data))
        else:
            miss_ids.append(int(post_id))

    if miss_ids:
        rows = await pool.fetch(
            "SELECT id, user_id, content, created_at FROM posts WHERE id = ANY($1)",
            miss_ids,
        )
        for row in rows:
            posts.append(dict(row))

    # Sort by created_at descending, return page
    posts.sort(key=lambda p: p.get("created_at", 0), reverse=True)
    return posts[:PAGE_SIZE]
```

---

## Exercise 2: Like Counter with Write-Behind

```python
"""
Implement like counting:
  - Increment in Redis atomically
  - Periodic bulk flush to DB
  - Return accurate count including unflushed increments
"""

import asyncio
import asyncpg
import redis.asyncio as redis
import time


async def like_post(rdb: redis.Redis, post_id: int, user_id: int) -> bool:
    """
    Like a post. Returns True if liked, False if already liked.
    Deduplication: one like per (user_id, post_id).
    """
    # Check if already liked
    already_liked = await rdb.sadd(f"liked_by:{post_id}", str(user_id))
    if not already_liked:
        return False   # Already liked

    # Atomic increment of like counter
    await rdb.incr(f"likes_pending:{post_id}")
    return True


async def unlike_post(rdb: redis.Redis, post_id: int, user_id: int) -> bool:
    removed = await rdb.srem(f"liked_by:{post_id}", str(user_id))
    if not removed:
        return False
    await rdb.decr(f"likes_pending:{post_id}")
    return True


async def get_like_count(pool: asyncpg.Pool, rdb: redis.Redis, post_id: int) -> int:
    """Returns DB count + unflushed Redis increment."""
    db_count = await pool.fetchval("SELECT like_count FROM posts WHERE id = $1", post_id) or 0
    pending  = int(await rdb.get(f"likes_pending:{post_id}") or 0)
    return db_count + pending


async def flush_likes_worker(pool: asyncpg.Pool, rdb: redis.Redis):
    """Flush Redis like increments to DB every 30 seconds."""
    while True:
        await asyncio.sleep(30)
        keys = [k async for k in rdb.scan_iter("likes_pending:*")]
        if not keys:
            continue

        pipe = rdb.pipeline()
        for key in keys:
            pipe.getdel(key)
        counts = await pipe.execute()

        updates = [
            (int(count), int(key.split(":")[1]))
            for key, count in zip(keys, counts)
            if count and int(count) != 0
        ]
        if updates:
            async with pool.acquire() as conn:
                await conn.executemany(
                    "UPDATE posts SET like_count = like_count + $1 WHERE id = $2",
                    updates,
                )
            print(f"Flushed like updates to {len(updates)} posts")
```
