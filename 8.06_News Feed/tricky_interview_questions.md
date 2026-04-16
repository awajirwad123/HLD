# News Feed — Tricky Interview Questions

## Q1: "A user with 1M followers posts. Your fan-out service sends 1M writes to Redis within 1 second. Redis throughput is 1M ops/sec. Will it fall over?"

**The problem:** Fan-out happens synchronously at post creation. 1M `ZADD` commands in < 1 second saturates a single Redis node (even pipelined, this is close to capacity).

**Solutions:**

1. **Async fan-out via Kafka.** Post creation → Kafka topic. Fan-out consumers read from Kafka and write to Redis. Multiple consumer instances parallelize the work. 1M writes at 100K/sec per consumer × 10 consumers = 10 seconds total. Independent of post creation latency.

2. **Fan-out in batches.** Process followers in batches of 1,000. Pause 100ms between batches. Spreads 1M writes over 100 seconds. Acceptable: followers see the post within 2 minutes, not instantly.

3. **Skip celebrities from write fan-out (hybrid model).** If the user has > 100K followers, don't write to individual feed lists at all. Followers fetch from `celebrity_posts:{user_id}` at read time. Zero write amplification.

4. **Redis Cluster sharding.** Feed keys are by user_id: `feed:{user_id}`. 1M different users' feed keys → distributed across 16,384 hash slots → across N Redis nodes. 1M writes are distributed across all nodes in parallel. Each node handles 1M/N writes.

**The production answer:** All three combined. Celebrity threshold avoids write fan-out for top influencers. Async Kafka fan-out handles the rest with parallelism. Redis Cluster distributes load. Feed writes to a single user's feed list from different fan-outs are naturally serialized per key.

---

## Q2: "Instagram uses an algorithmic (non-chronological) feed. How do you implement ranking without adding 500ms of latency to every feed load?"

**The problem:** A real-time ranking model computing scores for 500 post candidates takes 50–100ms per prediction. At feed load time, users expect < 200ms total. You can't score 500 posts in-request.

**Solutions:**

1. **Pre-score at fan-out time.** When writing post_id to a follower's feed list, compute an initial relevance score and use it as the Sorted Set score instead of the timestamp. Score = `f(poster_affinity, engagement_velocity, content_type_preference)`. Stale by read time but "good enough" for most users.

2. **Background re-scoring.** A background job periodically re-scores each user's top 100 feed candidates using a heavier ML model. Updates the Sorted Set scores. Next feed load uses the pre-scored ordering. Re-scoring runs every 5 minutes — acceptable staleness.

3. **Two-phase fetch.**
   - Phase 1: `ZREVRANGE feed:{user_id} 0 499` — get 500 candidate post IDs (~1ms)
   - Phase 2: Enrich 500 posts with lightweight features (like count, age) using Redis MGET (~5ms)
   - Phase 3: Apply fast scoring model (linear + a few features) — 500 items × lightweight model = ~10ms
   - Return top 20

4. **Candidate pool sizing.** Reducing from 500 to 200 candidates with a faster initial filter (age × poster_tier) before the heavy ML model reduces scoring time proportionally.

**Netflix/YouTube approach:** The expensive model runs as a background batch job. At read time, a pre-compiled ranking lookup is fetched (almost O(1)).

---

## Q3: "A user deletes a post. How do you remove it from all followers' feed lists efficiently?"

**The problem:** User A has 500K followers. Each follower's Redis feed list may contain the deleted post_id. Iterating 500K feed lists and calling `ZREM feed:{follower_id} {post_id}` on each = 500K Redis commands. Too slow for a synchronous delete.

**Options:**

1. **Tombstone pattern (recommended):** Don't remove from feed lists. Instead, mark the post as deleted in the DB and cache: `SET post_deleted:{post_id} 1 EX 86400`. At feed render time, filter out deleted post IDs. Post naturally ages out of feed lists within the retention window. Zero write amplification on delete.

2. **Async background cleanup.** Publish a "post deleted" event to Kafka. Fan-out consumers lazily scan feed lists for the post_id and remove it. This happens over minutes/hours rather than synchronously. User sees the post gone within ~5 minutes.

3. **Short feed cache TTL + DB source of truth.** Feed lists have a 30-minute TTL. On regeneration, the deleted post isn't included (DB query filters it). User sees stale feed for at most 30 minutes. Simplest operationally but slowest for the deleting user's experience.

**Production answer:** Tombstone in Redis (immediate effect on feed render) + async background cleanup of feed lists. The post_deleted flag is checked for every post ID at render time with a Redis MGET (batched, very fast).

---

## Q4: "Twitter's 'For You' feed uses ML ranking. How does it work at 350M DAU without adding seconds of latency?"

**The two-tower architecture:**

**Tower 1 — Candidate Retrieval (runs in batch, every few hours):**
- Retrieves 1,500 post candidates per user from the global post pool
- Uses approximate nearest-neighbor search (FAISS) on user interest embeddings
- Pre-computes and stores the 1,500 candidates per user in Redis

**Tower 2 — Ranking (runs at request time, < 50ms):**
- Takes the 1,500 pre-retrieved candidates
- Applies a more complex model: 100+ features per post (engagement signals, author affinity, media type, recency)
- Returns top 20 ranked posts

**Why this works at scale:**
- Heavy candidate retrieval = batch job, not on the critical path
- Light ranking model = runs in < 50ms even for 1,500 candidates
- Separation of concerns: retrieval maximizes recall; ranking maximizes precision

**Infrastructure:** Twitter uses Cortex (equivalent to TensorFlow Serving) for model inference. Models are deployed as microservices. Feed service calls ranking endpoint with candidate + user feature vectors. The ranking service runs on GPU-equipped instances with model warming.
