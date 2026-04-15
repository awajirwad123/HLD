# URL Shortener — Interview Simulator

## Scenario 1: "Design TinyURL"

**Interviewer:** "Design a URL shortening service like TinyURL. Walk me through your full design."

---

### Strong Answer Walkthrough

**Step 1 — Clarify requirements (2 min)**

> "Before diving in, a few quick questions: Do we need custom aliases? Click analytics? Expiration support? Is this global or single-region? What's our scale target?"

Assume answers: Yes to all features, global, 100M new URLs/day, 10B redirects/day.

**Step 2 — Capacity estimate (2 min)**

Writes: 100M / 86,400s ≈ 1,160/sec  
Reads: 10B / 86,400s ≈ 115,000/sec  
Storage: 5 years × 100M/day × 250 bytes ≈ 45 TB

Read:Write = 100:1 → heavily read-optimized design.

**Step 3 — API design**

```
POST /shorten
  Body: { "url": "https://...", "alias"?: "...", "ttl_days"?: 7 }
  Returns: { "short_url": "https://short.ly/aB3xZ" }

GET /{short_code}
  Returns: HTTP 302 Location: https://original-url.com
```

**Step 4 — Short code generation**

Recommend Snowflake ID → Base62. Explain why:
- Snowflake: globally unique, no DB roundtrip for uniqueness
- Base62: 7 chars covers 3.5 trillion URLs

For custom aliases: directly use the alias as the short_code, validated for format and uniqueness.

**Step 5 — Architecture**

```
CDN → Redirect API → Redis → DB Read Replica
Write API → DB Primary
```

Explain the write-behind click counter pattern. Explain 302 vs 301 and why 302.

**Step 6 — Scale discussion**

- CDN absorbs top 0.1% URLs = 90% traffic
- Redis handles remaining cache load
- DB read replicas for cold misses
- Kafka + ClickHouse for click analytics

**Step 7 — Data model**

Walk through the `urls` table schema.

**Step 8 — Failure scenarios**

"If Redis goes down, we fall back to DB read replicas. RTT increases from 1ms to 5ms but we stay under SLA with local in-process L1 cache in each redirect service instance."

---

## Scenario 2: "URL Shortener at WhatsApp Scale"

**Interviewer:** "WhatsApp shares ~100M links per day. Clicking a shared link in WhatsApp generates 10 redirects/second per link at peak (viral content). Design for this."

---

### Key Differences to Address

**Problem:** Power-law distribution. One link can get 1M clicks in 5 minutes when a message goes viral. Your Redis handles the key, but CDN becomes essential.

**Design additions:**

1. **Aggressive CDN caching.** Cache-Control: `max-age=86400` (1 day) for non-expiring links. This means viral links are served entirely at CDN edge at < 1ms.

2. **Preemptive warming.** When a link reaches 1,000 clicks in 60 seconds, trigger a CDN prefetch to push the cache entry to all PoPs globally. This avoids the thundering herd of a massive CDN cache miss across 20+ global PoPs simultaneously.

3. **Hot key detection.** At the Redis level, monitor per-key access rate. If any key is getting > 50,000 hits/sec from a single Redis node, replicate that key to all Redis nodes (read from any node) to spread load.

4. **Analytics offloading.** Viral links distort per-link counters. Use approximate counting (HyperLogLog for unique visitors, Redis Counter for total clicks) to avoid lock contention.

5. **Shortened URL preview.** WhatsApp renders link previews — the long URL's OpenGraph metadata must be fetched. Add a `/preview/{short_code}` endpoint that returns the OG title/image from the original URL. Cache this aggressively (TTL = 24h).

---

## Scenario 3: "Self-Hosted URL Shortener for an Enterprise"

**Interviewer:** "A bank wants to run an internal URL shortener to share documents and internal tools. They need an audit trail of every click, who clicked, from which device, and they cannot use public cloud. Design it."

---

### Key Additions for Enterprise

**Audit trail per click:**

Every click must be logged with user identity. The redirect API must be authenticated (not publicly accessible):

```
GET /{short_code}
Headers: Authorization: Bearer <JWT>
```

JWT contains `user_id`, `department`, `device_id`. On redirect:
1. Verify JWT with enterprise IdP (LDAP/AD via OAuth2/SAML)
2. Log: short_code, user_id, department, device_id, ip, timestamp to audit DB
3. Return 302 redirect

**Audit DB:** PostgreSQL (separate schema from URL store). Immutable append-only log — no DELETEs allowed (compliance). Use Row-Level Security so users can only see their own click history.

**RBAC for URL creation:**
- Only authenticated employees can create short URLs
- URLs can be tagged as "internal only" — blocked for unauthenticated access
- Admins can deactivate any URL

**On-prem architecture:**
- PostgreSQL (primary + HA replica, Patroni)
- Redis Sentinel (3 nodes) for HA
- Nginx as load balancer + reverse proxy (no CDN — internal traffic only)
- Keycloak/Auth0 on-prem for identity

**Compliance:**
- Audit logs retained 7 years (financial regulation)
- `expires_at` cannot be more than 1 year for security (links to sensitive docs must expire)
- URL scanning against internal threat intelligence before creation
