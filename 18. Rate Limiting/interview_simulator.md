# Rate Limiting — Interview Simulator

## Scenario 1: Design a Rate Limiter Service

**Prompt:**
> "Design a distributed rate limiter that will be used as a shared service by all teams in a 500-engineer company. It must support: per-user, per-IP, and per-API-key limits; multiple algorithms; plan-based limits; and handle 500,000 requests/second globally."

**Time limit:** 35 minutes

---

### Clarifying Questions You Should Ask

1. Is this a library embedded in each service, a sidecar, or a standalone service over the network?
2. Should limits be configurable at runtime without redeployment?
3. What's the tolerance for over-allowance? (Exact vs soft)
4. Multi-region? (Global rate limit or per-region?)
5. Is this security-critical (login endpoint) or fairness-only (API metering)?

---

### Expected Architecture Walkthrough

**Deployment model decision:**

| Model | Latency | Accuracy | Ops complexity |
|---|---|---|---|
| Library in each service | 0ms overhead | Per-service only (not global) | Low |
| Sidecar (per pod) | <1ms | All traffic via sidecar | Medium |
| **Standalone service (recommended)** | 2–5ms round-trip | Global accuracy | Higher |

At 500K RPS shared across 50 services, a standalone service with a Redis cluster backend is the right call. Sidecar at this scale adds network hops without benefit; a library would only give per-instance limits.

**Component Architecture:**

```
Client Request
  ↓
Rate Limiter Service (stateless, 10 replicas)
  ├── Receives: {key, limit, window, algorithm}
  ├── Executes: Lua script on Redis Cluster
  └── Returns:  {allowed, remaining, retry_after}
  ↓
Redis Cluster (6 shards, 1 primary + 1 replica each)
  ├── Shard 1: users A–F
  ├── Shard 2: users G–M
  ...
  └── Consistent hashing of rate limit keys → shards
  ↓
Config Service (Redis + DB)
  └── Stores plan limits: {org_id → {limit, window, algorithm}}
```

**Redis at 500K RPS:**
- 500K `INCR`-equivalent operations/second
- Single Redis: ~500K ops/sec max → bottleneck
- Redis Cluster with 10 primary shards: 50K ops/shard — comfortable headroom
- Key sharding: `HASH_SLOT(rate:{user_id})` naturally distributes across shards

**Algorithm choice:**
- Default: Sliding window counter (2 counters per key, O(1) memory, ~0% error)
- Security endpoints: Sliding window log (exact)
- API metering with burst: Token bucket (via Lua script)

**Lua script per algorithm prevents race conditions. All checks are atomic.**

**Config storage:**
```python
# Redis hash per org/user with plan-based limits
HSET plan:org-123  limit 10000  window 60  algorithm sliding_counter
HSET plan:user-456 limit 1000   window 60  algorithm token_bucket
```

Config changes propagate without redeployment. Rate limiter service reads config from Redis with a 10s TTL cache in memory.

**Response headers:**
Every response from the Rate Limiter Service includes:
```
{
  "allowed": true/false,
  "remaining": 47,
  "limit": 100,
  "retry_after": 0,   # Non-zero only on rejection
  "reset_at": 1700000060
}
```

Calling service attaches `X-RateLimit-*` headers to its HTTP response to the client.

---

### Scaling Calculation

```
500K RPS global
÷ 10 Redis shards = 50K RPS per shard
× 1 Lua script per request = 50K Lua executions/sec per shard
Redis single-thread Lua: handles 200K+ Lua scripts/sec per shard ✓

Rate Limiter Service replicas: 500K RPS ÷ 50K handled per replica = 10 replicas
Each replica: 1 Redis round-trip (~0.5ms) + logic = ~1ms p50, ~5ms p99
```

---

### Follow-up Questions

**"What if one user has 10 million followers and generates a fan-out that hits the rate limiter 10M times per second?"**
This is a hotspot problem. One Redis key (`rate:{celebrity_user_id}`) gets 10M operations/second — far beyond a single shard. Solution: for known high-traffic keys, use local counters on each Rate Limiter Service instance and sync to Redis every 100ms. Accept up to 100ms × 10 instances × local_rate over-allowance. For security limits, use a dedicated shard for known hot keys.

**"Multi-region: Europe and US. Should rate limits be global or per-region?"**
If the goal is API cost control / fairness: global limit (EU + US combined). Cross-region Redis sync latency is 50–200ms — too slow for per-request enforcement. Solution: use a 70/30 regional split with periodic rebalancing via async sync. If the goal is DDoS protection: per-region limits are sufficient (an attacker in the EU can only flood EU; US is protected by its own regional limit).

---

## Scenario 2: Stripe-Like API Rate Limiter

**Prompt:**
> "Design the rate limiting system for a payment API similar to Stripe. API keys belong to merchants. Rate limits vary by plan. Write endpoints (charges, refunds) must be more tightly limited than reads. Retries with idempotency keys must not be double-counted."

**Time limit:** 25 minutes

---

### Expected Architecture Walkthrough

**Identity hierarchy:**
```
Account (merchant)
  └── API Key (test vs live; multiple keys per account)
       └── Rate limit applied at API key level
```

**Plan-based limits:**
```
Free:    10  writes/sec,  100  reads/sec
Growth:  100 writes/sec,  1000 reads/sec
Scale:   500 writes/sec,  5000 reads/sec
```

Store in DB (PostgreSQL): `api_key_limits(api_key_id, write_rps, read_rps, algorithm)`.
Cache in Redis hash: `HGETALL limits:{api_key}` with 60s TTL.

**Endpoint classification:**
```python
WRITE_METHODS = {"POST", "PUT", "PATCH", "DELETE"}
WRITE_PATHS   = {"/v1/charges", "/v1/refunds", "/v1/payment_intents"}

def is_write(method, path):
    return method in WRITE_METHODS or any(path.startswith(p) for p in WRITE_PATHS)
```

**Idempotency key deduplication:**
```
Request arrives with header: Idempotency-Key: idem-abc-123

Step 1: Check idempotency cache
  GET idempotency:{api_key}:idem-abc-123
  Hit → return cached response, DO NOT increment rate limit counter
  Miss → continue to rate limit check

Step 2 (on miss): Rate limit check (Lua script)
  → allowed → process request → store result in idempotency cache (24h TTL)
  → rejected → return 429 (also cache the rejection to avoid re-checking)
```

**Token bucket per API key:**
```python
# Capacity = 5× write_rps (allows short burst), refill_rate = write_rps
HSET tb:{api_key}:write  tokens 500  last_ts {now}
```

**Response on rejection:**
```json
HTTP 429
{
  "error": {
    "type":    "rate_limit_error",
    "code":    "rate_limit_exceeded",
    "message": "Too many requests. Retry after 1 second.",
    "doc_url": "https://stripe.com/docs/rate-limits"
  }
}
Retry-After: 1
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
```

---

### Key Design Decisions to Justify

**"Why token bucket over sliding window for Stripe?"**
Stripe's clients are developers building integrations. Burst is legitimate — a webhook handler might fan out 50 charge requests at once when a batch job runs. Token bucket absorbs this burst (up to 5× the steady rate), then enforces the average. A strict sliding window would reject these legitimate bursts and generate confusing errors.

**"Why limit writes more tightly than reads?"**
Write operations charge cards, trigger bank calls, create financial records — each has meaningful cost and risk. Read operations (list charges, retrieve metadata) are cheap. Tighter write limits protect against runaway code (a bug that infinitely creates charges) while allowing diagnostic reading.

---

## Scenario 3: DDoS Protection at the Edge

**Prompt:**
> "A competitor is running a DDoS attack against your API: 500,000 fake requests/second from 100,000 different IPs. Your origin servers can handle 10,000 RPS. How do you design rate limiting to protect against this?"**

**Time limit:** 20 minutes

---

### Expected Architecture Walkthrough

**Key insight**: DDoS protection must happen **before** the traffic reaches your origin. Per-IP limiting at the origin is too late — parsing the request, routing it, and hitting Redis all consume origin resources.

**Layered defence:**

**Layer 1 — CDN / Edge (Cloudflare / Fastly / Akamai)**
- Rate limiting at network edge, before packets enter your data centre
- Per-IP: 100 requests/minute (aggressive for unauthenticated)
- Managed rules: block known attacker ASNs, require CAPTCHA for suspicious IPs
- Cost: CDN absorbs the 500K RPS; your origin never sees it
- Cloudflare's rate limiting uses sliding window counters per PoP

**Layer 2 — API Gateway (after CDN)**
- Only verified (CDN-cleaned) traffic arrives here
- Per-IP: 500 req/min (softer, since CDN already filtered bot traffic)
- Per-API-key: plan-based limits
- Challenge mode: for IPs with >300 req/min, serve JS challenge before allowing through

**Layer 3 — Application (origin)**
- Per-user business limits (not DDoS protection)
- Circuit breaker on Redis if attack bypasses CDN

**IP reputation + blocklist:**
- Maintain a Redis SET of known bad IPs: `SET blocked_ips:{ip} 1 EX 3600`
- Check this before the full rate limit Lua script (cheaper: just `EXISTS`)
- Feed from: fail2ban data, third-party threat intelligence, your own historical 429 patterns

**Fingerprinting beyond IP:**
- 100,000 IPs from 1 ASN → block the ASN
- All requests with `User-Agent: python-requests/2.x` in a burst → temporary UA block
- TLS fingerprint (JA3 hash) — bots often share distinctive TLS fingerprints

**Capacity math:**
```
Attack: 500K RPS from 100K IPs
CDN absorbs: 499,900 RPS (all bot traffic served 429 at edge)
Reaches origin: ~100 RPS (legitimate users, not affected by IP rate limit)
Origin capacity: 10,000 RPS — well within budget
```

---

### Follow-up Questions

**"What if the attacker uses 10 million IPs (botnet)?"**
At 10M IPs, per-IP limits are irrelevant — each IP sends one request and moves on. Defences shift to:
1. **CAPTCHA / proof-of-work challenge** for unauthenticated endpoints
2. **Authentication requirement**: force login before any meaningful API call
3. **Request shape analysis**: legitimate browsers have consistent TLS fingerprints, headers, timing patterns; bots don't
4. **Anycast routing + scrubbing centres**: CDN absorbs volumetric attacks at the network layer before HTTP parsing

**"The CDN vendor has an outage. Your origin is exposed. What now?"**
Pre-provision: have a secondary CDN (multi-vendor). Keep your own edge rate limiter (HAProxy / nginx `limit_req`) as a fallback with aggressive IP limits (`5 req/sec` burst of 50). The fallback is less sophisticated but functional. Circuit-break downstream services at origin to shed load gracefully (503 with `Retry-After: 120`).
