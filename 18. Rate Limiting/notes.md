# Rate Limiting — Notes & Cheat Sheet

## 1. Algorithm One-Liners

| Algorithm | Core idea | Burst | Memory | Boundary burst |
|---|---|---|---|---|
| **Fixed Window** | Counter per time bucket | Yes | O(1) | **Yes — double burst** |
| **Sliding Window Log** | Timestamp log per client | No | O(n requests) | No |
| **Sliding Window Counter** | Weighted prev + curr windows | ~Yes | O(1) | Approx. no |
| **Token Bucket** | Tokens fill at refill rate; spend per request | **Yes (up to capacity)** | O(1) | No |
| **Leaky Bucket** | Queue drains at constant rate | No | O(queue) | No |

**Production choice**: Sliding Window Counter (Cloudflare, NGINX) or Token Bucket (Stripe, AWS).

---

## 2. Fixed Window — Boundary Burst

```
Limit: 100/min

Window 1 [00:00–01:00]: 100 requests at 00:59 → all allowed ✓
Window 2 [01:00–02:00]: 100 requests at 01:01 → all allowed ✓
                ↑ 200 requests in 2 seconds — the "double burst" vulnerability
```

---

## 3. Sliding Window Counter Formula

$$\text{estimated} = \text{prev\_count} \times \frac{\text{window} - \text{elapsed}}{\text{window}} + \text{curr\_count}$$

- `elapsed` = seconds into current window
- Error rate: ≈ 0.003% (Cloudflare validated)
- Only stores 2 integers per key — O(1) memory

---

## 4. Token Bucket vs Leaky Bucket

| Aspect | Token Bucket | Leaky Bucket |
|---|---|---|
| Burst output | **Bursty** (up to capacity) | **Smooth** (always drain_rate) |
| Input handling | Drop if no tokens | Queue up to capacity, drop when full |
| Use case | API rate limits (burst OK) | Protecting downstream (smooth needed) |
| Key property | Average rate ≤ refill_rate | Output rate = drain_rate |

---

## 5. Redis Commands Reference

```redis
# Fixed window (atomic increment + expire)
INCR   rl:{user}:{window_start}
EXPIRE rl:{user}:{window_start}  {window_seconds}

# Sliding window log (sorted set by timestamp)
ZREMRANGEBYSCORE key 0 {now - window_ms}
ZADD             key {now_ms} {request_uuid}
ZCARD            key
EXPIRE           key {window_seconds}

# Atomic token bucket (Lua script — prevents race condition)
-- See hands_on.md Exercise 3

# Redis Cell module (GCRA / leaky bucket)
CL.THROTTLE {key}  {max_burst}  {count}  {period}  {quantity}
-- Returns: [allowed, limit, remaining, retry_after_ms, reset_after_ms]
-- Example: CL.THROTTLE user:123  99  1  1  1
--   → 100 token capacity, 1 token per second
```

---

## 6. HTTP Rate Limit Headers

```
# Standard headers (RFC 6585 + draft-ietf-httpapi-ratelimit-headers)
X-RateLimit-Limit:     100          ← max allowed in window
X-RateLimit-Remaining: 47           ← requests left in current window
X-RateLimit-Reset:     1700000060   ← Unix timestamp when limit resets
Retry-After:           30           ← seconds before retry (on 429 response)

# Response code
HTTP 429 Too Many Requests
```

Always return `Retry-After` on 429 — clients that respect it prevent thundering herd re-attacks.

---

## 7. Distributed Rate Limiting Patterns

| Pattern | Accuracy | Redis write pressure | Best for |
|---|---|---|---|
| **Centralised Redis counter** | Exact | O(1) per request | Most use cases |
| **Local + periodic sync** | Soft (sync delay × N servers) | Low | Soft limits, analytics |
| **Redis Lua script (atomic)** | Exact | O(1) per request | Token bucket |
| **redis-cell (CL.THROTTLE)** | Exact (GCRA) | O(1) | Production simplicity |

---

## 8. Where Rate Limiting Lives — Defence in Depth

```
Client
  ↓
CDN / Edge (Cloudflare)      — IP-based, global DDoS protection
  ↓
API Gateway (Kong / AWS)     — API key, tenant, plan-based limits
  ↓
Application service          — Endpoint-specific business limits
  ↓
Data store / third-party     — Protect DB connection pool, external API quotas
```

Each layer enforces different limits for different purposes. Never rely on one layer alone.

---

## 9. Rate Limiting Dimensions

```python
# Identify the key to rate limit on:
f"ip:{ip}"                        # IP-based (unauthenticated)
f"user:{user_id}"                 # Per authenticated user
f"api_key:{api_key}"              # Per API key (Stripe-style)
f"org:{org_id}"                   # Per tenant/organisation
f"user:{user_id}:{endpoint}"      # Per user per endpoint
f"global:{endpoint}"              # Hard cap regardless of user
```

---

## 10. Real-World Reference Points

| Company | Algorithm | Limits |
|---|---|---|
| **Stripe** | Token bucket | 100 req/sec per API key; lower for write endpoints |
| **GitHub** | Fixed window | 5,000 req/hour authenticated; 60 unauthenticated |
| **Cloudflare** | Sliding window counter | Per PoP, per IP, configurable |
| **Twitter API v2** | Fixed + rolling window | 300 tweet reads / 15 min |
| **OpenAI** | Token bucket + quota | RPM (requests/min) + TPM (tokens/min) |
| **AWS API Gateway** | Token bucket | 10,000 RPS burst + steady rate configurable |

---

## 11. Client Retry Strategy (Respecting Rate Limits)

```python
# Correct retry behaviour on 429
retry_after = response.headers.get("Retry-After", 5)
await asyncio.sleep(float(retry_after))

# If no Retry-After: exponential backoff with jitter
sleep = min(base * (2 ** attempt) * (0.5 + random.random()), 60)
```

**Never** retry immediately on 429 — guaranteed thundering herd.

---

## 12. Soft Limits vs Hard Limits vs Quotas

| Type | Behaviour | Example |
|---|---|---|
| **Hard limit** | Immediately reject at threshold | 429 on login attempt #6 |
| **Soft limit** | Allow burst, then throttle | Token bucket burst to capacity |
| **Quota** | Daily/monthly cap (not per-second) | 10,000 API calls/month on Free plan |
| **Throttle** | Slow down (queue) rather than reject | Leaky bucket — process at defined rate |

---

## 13. Key Numbers to Memorise

| Metric | Value |
|---|---|
| Redis `INCR` throughput | ~100K–500K ops/sec on single instance |
| Sliding window counter error | ≈ 0.003% (Cloudflare measured) |
| Lua script atomicity | Executes as single Redis command — no race condition |
| Typical API burst allowance | 2–10× steady rate |
| `Retry-After` standard | RFC 7231 §7.1.3 — seconds integer or HTTP-date |

---

## 14. Interview Key Phrases

- *"Fixed window has a boundary burst problem — a client can make 2× the limit in a 2-second window at a window boundary."*
- *"Sliding window counter approximates sliding window log with just two counters — O(1) memory with <0.1% error."*
- *"Token bucket allows bursts up to capacity then enforces average rate. Leaky bucket enforces constant output rate regardless."*
- *"Distributed rate limiting using Redis Lua scripts is atomic — prevents the check-then-increment race condition."*
- *"Always include `Retry-After` on 429 responses — clients that respect it prevent self-inflicted thundering herd."*
- *"Rate limit on multiple dimensions simultaneously: per-IP for unauthenticated protection, per-user for fairness, per-endpoint for cost control."*
