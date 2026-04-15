# Rate Limiter Service — Quick-Fire Questions

**Q1: What are the five main rate limiting algorithms?**

Fixed window counter (simple, boundary issue), sliding window log (exact, memory-heavy), sliding window counter (approximate, efficient), token bucket (allows bursts, controlled refill), leaky bucket (smooth constant output). Most production APIs use sliding window counter or token bucket.

---

**Q2: What is the "boundary burst" problem in fixed window rate limiting?**

If limit = 100 req/min, a user can send 100 requests at 0:59 and 100 more at 1:01 — 200 requests in 2 seconds — because each burst falls in a different window. Fixed windows reset completely at each boundary. Sliding window counter eliminates this by weighting the previous window based on how much of it is still within the lookback period.

---

**Q3: How does the sliding window counter algorithm work?**

It maintains two counters: current window and previous window. At any point, the effective count is `current_count + previous_count × (overlap_fraction)`, where overlap_fraction is how much of the previous window is still within the rolling lookback. This approximates a true sliding window with only 2 stored values instead of thousands of timestamps.

---

**Q4: What is the difference between token bucket and leaky bucket?**

Token bucket allows bursts: tokens accumulate when idle (up to bucket capacity), so a burst request can consume many tokens at once. Leaky bucket outputs requests at a constant rate regardless of burst shape — requests queue up and "leak" out steadily. Token bucket is better for typical APIs where some burstiness is acceptable; leaky bucket is used in network QoS for smoothing.

---

**Q5: Why use a Lua script in Redis for rate limiting instead of plain GET + INCR?**

`GET` + `INCR` are two separate commands — a race condition exists between them. Two concurrent requests could both read count=4 (limit=5), both decide "allowed," and both increment to 5, allowing 2 requests when only 1 should pass. Lua scripts execute atomically on the Redis server: no interleaving possible.

---

**Q6: What HTTP status code do you return when a request is rate-limited?**

`429 Too Many Requests`. Include headers: `Retry-After: {seconds}`, `X-RateLimit-Limit: {limit}`, `X-RateLimit-Remaining: 0`, `X-RateLimit-Reset: {unix_timestamp}`. The `Retry-After` header tells the client exactly how long to wait before retrying.

---

**Q7: What should a rate limiter do if Redis is unavailable?**

Two strategies: **fail-open** (allow all requests when Redis is down) — preserves API availability at the cost of no rate limiting during the outage; or **fail-closed** (reject all requests) — safer for authentication endpoints. Most general-purpose APIs fail-open. Security-critical endpoints (login, payment) should fail-closed. A third option: fall back to a local in-memory limiter (may allow up to N×limit across N nodes).

---

**Q8: How do you implement multi-tier rate limiting (IP, API key, user)?**

Check each tier sequentially. On first violation, return 429 with the relevant tier's headers. On all tiers passing, allow through. Order matters: IP check first (cheapest DDoS protection), then API key (per-customer quota), then user (per-account fairness), then endpoint-specific limits.

---

**Q9: What is a rate limit "key" and how do you choose it?**

The key determines who or what is being rate-limited. Options: IP address (anonymous users, DDoS protection), API key (per-customer quotas), user ID (per-account fairness), user+endpoint (protect expensive operations). Use a composite key like `ratelimit:{user_id}:{endpoint_slug}` for endpoint-specific limits. Hash the key before storing in Redis to avoid key length issues.

---

**Q10: How does token bucket handle an idle user who then sends a burst?**

Tokens accumulate while idle. If a user is idle for 30 seconds with a refill rate of 10 tokens/sec and capacity of 100, they accumulate `min(100, 30 × 10) = 100` tokens. Then they can send a burst of 100 requests at once. After the burst, they're back to refilling at 10/sec. This is the intended behavior — short idle periods should enable short bursts.

---

**Q11: How do you rate limit a streaming API (e.g., a WebSocket connection or file download)?**

For streaming: rate limit on data volume (bytes/sec) rather than request count, using a token bucket where each token represents some bytes. E.g., 1MB/s limit with capacity=2MB: user can burst 2MB then is throttled to 1MB/s. For WebSocket: count messages, not connections. One long-lived WebSocket shouldn't bypass per-message rate limits.

---

**Q12: Where in the stack should rate limiting be enforced?**

At the API gateway / load balancer (before traffic reaches backend services): one enforcement point, no duplication. Coarse IP-level protection can be at the CDN edge. In a microservices architecture, each service can also self-limit inbound requests for defense-in-depth, but this adds per-service Redis calls. The gateway is the primary enforcement layer.

---

**Q13: How do you handle clients that "jitter" their requests to stay just under the limit?**

This is normal and expected client behavior (no burst, steady trickle). Rate limiters allow it — if a client sends exactly 99 of 100 allowed requests per minute, that's legitimate use. If you want to prevent sustained high throughput, reduce the limit. For truly adversarial circumvention (many IPs, rotating API keys), you need application-layer detection (anomaly scoring, behavioral analysis) beyond a simple rate limiter.
