# Rate Limiting — Quick-Fire Q&A

## Algorithms

**Q1. What is the boundary burst problem with fixed window counters?**
A client can make the full limit at the end of one window (e.g., 100 at 00:59) and the full limit again at the start of the next (100 at 01:01) — 200 requests in 2 seconds despite a "100/minute" limit. The window boundary resets the counter without any continuity.

**Q2. How does the sliding window log algorithm avoid the boundary burst?**
It stores a timestamp for every request in a sorted set. On each new request it removes timestamps older than the window and counts the remainder. Since the window slides with real time, there's no discrete reset boundary. Accurate to the millisecond — but memory-intensive at scale.

**Q3. What is the sliding window counter's core formula?**
$$\text{estimated} = \text{prev\_count} \times \frac{\text{window} - \text{elapsed}}{\text{window}} + \text{curr\_count}$$
It weights the previous window's count by how much of it overlaps with the current rolling window, then adds the current window's count. Only two integers per key — O(1) memory.

**Q4. Explain token bucket in one sentence.**
A bucket fills with tokens at a constant refill rate up to a maximum capacity; each request consumes one token and is rejected if the bucket is empty — allowing bursts up to the bucket capacity while enforcing an average rate equal to the refill rate.

**Q5. How does a leaky bucket differ from a token bucket?**
Leaky bucket queues incoming requests and drains them at a constant rate, producing smooth output regardless of input burstiness. Token bucket allows bursty output (up to capacity). Use leaky bucket to protect a downstream that can only handle a fixed throughput.

**Q6. Why must a token bucket implementation use a Lua script in Redis?**
Without atomicity, two concurrent requests can both read the same token count, both see "tokens >= 1", both decrement, resulting in more tokens consumed than available — a race condition. A Lua script executes atomically as a single Redis command, making the read-then-write indivisible.

---

## Distributed Systems

**Q7. If you have 10 server instances each with a local rate limiter, what's the problem?**
Each server only sees its share of requests. A client sending 1,000 req/min with "100/min" limit will be allowed by every server (each sees ~100 req/min). Total = 1,000 req/min — effectively no rate limiting across the cluster. Fix: centralise the counter in Redis.

**Q8. What is the trade-off of using a centralised Redis counter for rate limiting?**
Accuracy is perfect, but Redis is now in the hot path of every request. At 500K RPS, you have 500K Redis writes/second plus a Redis round-trip (~0.5–1ms) added to every request. For high-throughput systems, use Lua scripts to batch the check-and-increment into a single round-trip, or use the local+sync pattern for soft limits.

**Q9. Describe the local counter + periodic sync pattern.**
Each server maintains a local count and periodically (e.g., every 5s) sends a delta (`INCRBY`) to Redis. Other servers read the aggregate. The trade-off: in the sync window, the global count is stale — a client could over-send by `sync_period × server_count` requests before being throttled. Suitable for soft limits where exact enforcement isn't critical.

---

## HTTP & Headers

**Q10. What HTTP status code signals a rate limit exceeded?**
429 Too Many Requests (RFC 6585).

**Q11. Name the four standard rate limit response headers.**
- `X-RateLimit-Limit`: the maximum allowed in the current window
- `X-RateLimit-Remaining`: requests left in the current window
- `X-RateLimit-Reset`: Unix timestamp when the limit resets
- `Retry-After`: seconds to wait before retrying (on a 429 response)

**Q12. Why is `Retry-After` critical on a 429 response?**
Without it, well-meaning clients retry immediately — flooding the server with the exact burst that triggered the 429. With `Retry-After`, clients back off for the right duration, preventing a thundering herd retry storm.

---

## Real-World

**Q13. What rate limiting strategy does Stripe use?**
Token bucket per API key. 100 requests/second default, with lower limits for write endpoints. Their client library respects `Retry-After` and retries with exponential backoff + jitter automatically.

**Q14. Why does GitHub have different limits for authenticated vs unauthenticated requests?**
Unauthenticated requests (identified only by IP) are capped at 60/hour to limit scraping and abuse without a way to attribute usage. Authenticated users get 5,000/hour because they're accountable (the account can be banned), and legitimate applications need higher throughput.

**Q15. What is `redis-cell` and what algorithm does it implement?**
`redis-cell` is a Redis module providing a single `CL.THROTTLE` command that implements GCRA (Generic Cell Rate Algorithm) — a leaky bucket variant. It returns `[allowed, limit, remaining, retry_after_ms, reset_after_ms]` atomically in one round-trip.

---

## Design

**Q16. What dimensions can you rate limit on simultaneously?**
Per-IP, per-user, per-API-key, per-endpoint, per-tenant/org, and globally per-endpoint. A real system applies multiple dimensions: per-IP for unauthenticated DDoS protection, per-user for fairness, per-endpoint to allow more reads than writes.

**Q17. At what layer(s) should rate limiting be applied in a production architecture?**
Defence in depth: CDN/edge (IP-level, DDoS), API Gateway (API key / tenant), application service (endpoint-specific business limits), and optionally at the data layer to protect DB connection pools or third-party API quotas.

**Q18. What is the difference between a hard limit, a soft limit, and a quota?**
Hard limit: request is immediately rejected (429) at the threshold — no exceptions. Soft limit: allows bursting past the limit temporarily, then throttles back to the target rate. Quota: a total cap over a longer period (day/month), not a per-second rate — used for billing/plan enforcement.

**Q19. Why should rate limits apply to idempotent retries carefully?**
If a request succeeds but the response is lost, the client retries with the same idempotency key. If the rate limiter increments the counter on both the original and the retry, the retry may be rejected even though no extra work is done. Ideal: deduplicate on idempotency key and don't count a seen key against the limit.

**Q20. You're designing a rate limiter for an LLM API. Why is "requests per minute" insufficient?**
LLM inference cost is proportional to tokens, not requests. One request could cost 10 tokens, another 100,000 tokens. A "requests/minute" limit could be trivially bypassed by batching. Correct metrics: **TPM (tokens per minute)** for output rate control, plus RPM as a secondary signal. OpenAI enforces both simultaneously.
