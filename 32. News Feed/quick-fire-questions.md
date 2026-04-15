# News Feed — Quick-Fire Questions

**Q1: What's the fundamental challenge in building a news feed?**
Fan-out: when a user creates a post, it must appear in the feeds of all their followers. Write fan-out (push to all followers) creates write amplification. Read fan-out (pull from all followees) creates read amplification. The celebrity problem makes naive write fan-out catastrophic for accounts with millions of followers.

---

**Q2: What is "fan-out on write" vs "fan-out on read"?**
Fan-out on write: on post creation, push the post_id to every follower's feed list (Redis). Feed read = O(1). Fan-out on read: on feed load, query recent posts from each followed account, merge, sort. Post write = O(1), feed read = O(N × queries). Most production systems use a hybrid of both.

---

**Q3: How do you handle the celebrity problem?**
Users with > 100K followers don't write-fan-out (too expensive). Instead, their posts go into a separate `celebrity_posts:{user_id}` sorted set. When a user loads their feed, their pre-computed feed is merged with the last 10 posts from each celebrity they follow. Feed reads always include at most N celebrity queries (bounded read amplification).

---

**Q4: Why use a Redis Sorted Set for the pre-computed feed? Why not a List?**
Sorted Set allows: insertion by score (timestamp), range queries by score (time-range pagination), and efficient trimming (`ZREMRANGEBYRANK`). A List only supports indexed access — pagination by index not timestamp. Also, Sorted Set deduplicates automatically (no duplicate post_id entries).

---

**Q5: Why use Snowflake IDs for posts?**
Snowflake IDs are time-ordered — you can sort posts chronologically by ID without a separate timestamp lookup. The ID itself encodes creation time (first 41 bits). This means `ZADD feed:{uid} post_id post_id` works — the ID is both the member and the sort score.

---

**Q6: How do you implement pagination for a feed?**
Use Redis `ZREVRANGEBYSCORE feed:{user_id} {max_score} -inf LIMIT 0 20`. Client sends the score (timestamp or Snowflake ID) of the oldest seen post as `max_score` for the next page. This is cursor-based pagination — no offset, no skipping rows. Works correctly even as new posts are added.

---

**Q7: How do you handle a user who creates 1000 posts in 1 minute (spam/bot)?**
Rate limiting at the post creation API: max N posts per user per minute (Token Bucket). Flag accounts exceeding the limit for review. Don't add spam posts to the fan-out queue — check rate limit before triggering fan-out. Separate spam detection ML model that can retroactively remove posts and fan-out entries.

---

**Q8: What happens to the feed when a user unfollows someone?**
Don't eagerly remove their posts from the feed — too expensive. Use lazy cleanup: when the feed is rendered, filter out posts from accounts no longer in the user's following set. Posts will naturally age out of the feed as new posts come in. Alternatively, keep the feed as-is; posts from unfollowed users simply stop appearing as new ones are added.

---

**Q9: How do you count likes accurately without a DB write per like?**
Redis `INCR likes_pending:{post_id}` on each like. Periodic background worker (every 30s) runs `GETDEL likes_pending:{post_id}` and batch-updates `posts.like_count += N`. Display count = DB count + Redis pending. Dedup: `SADD liked_by:{post_id} {user_id}` (or Bloom Filter) prevents double-counting same user.

---

**Q10: How do you implement "what your friends liked" on the feed?**
When a post is liked, add it to `social_feed:{user_id}` sorted sets for the liker's followers (secondary fan-out). Score = timestamp of like event. On feed load, merge primary feed + social feed. Only do this secondary fan-out for users with < celebrity threshold followers (to avoid write amplification on viral likes). Capped at: only show posts liked by someone you follow in the last 24 hours.
