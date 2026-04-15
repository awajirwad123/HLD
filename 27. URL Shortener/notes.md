# URL Shortener — Notes & Reference

## Core Requirements Summary

**Functional:**
- POST long URL → receive 7-char short code
- GET short code → redirect to original URL
- Optional: custom aliases, expiration, click analytics

**Non-functional:**
- Read heavy: 100:1 read/write ratio
- Redirect latency < 10ms P99
- 99.99% availability (users click links in emails — long TTL)
- Globally distributed (CDN edges)

---

## Short Code Options — Comparison

| Method | Uniqueness | Predictability | DB Roundtrip | Notes |
|---|---|---|---|---|
| MD5 → Base62 | Same URL → same code (collision possible) | Low | Check required | Deduplication built-in; multipart collision risk |
| Auto-increment + Base62 | Globally unique | High (sequential) | DB-generated | Simple; enumerable → security risk |
| Random token (secrets) | Probabilistic unique | None | Collision check | Best for anti-enumeration |
| Snowflake ID + Base62 | Globally unique, time-ordered | Time-sorted | None | Best performance at scale |

**Production:** Snowflake for performance; random token for small systems or user-facing codes.

---

## Base62 Reference

```
Alphabet: 0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
                                                                     (62 chars)

7 characters → 62^7 = 3,521,614,606,208 ≈ 3.5 trillion combinations
6 characters → 62^6 = 56,800,235,584   ≈ 56 billion
```

---

## HTTP Redirect Codes

| Code | Name | Browser caches? | Use |
|---|---|---|---|
| 301 | Moved Permanently | Yes (forever) | Static redirects, no analytics |
| 302 | Found (Temporary) | No (checks each time) | Short URLs with analytics |
| 307 | Temporary Redirect | No | Preserve POST method on redirect |
| 308 | Permanent Redirect | Yes | Preserve POST method permanently |

**TinyURL / Bitly use 302** because every click must hit their servers for analytics.

---

## Caching Layers (Read Path)

```
User click → CDN edge (Cache-Control: max-age=3600) → Redis (TTL 1h) → DB read replica
                ↑ ~99.9% hit for popular                ↑ ~99% hit for all active
```

- CDN: covers the top 0.1% URLs that generate 90% of traffic (power law distribution)
- Redis: covers remaining 99.9% of active URLs
- DB: only hit for cold codes or first-time cold-start

**Cache key:** `url:{short_code}` → hash with fields `{long_url, expires_at}`

---

## Click Analytics Design Options

| Approach | Writes/sec | Accuracy | Complexity |
|---|---|---|---|
| UPDATE click_count on each redirect | 115,000 | Exact | High DB load |
| Redis INCR + periodic flush | ~0 DB writes | Slightly delayed | Simple |
| Kafka + ClickHouse | 0 DB writes | Real-time analytics possible | Complex but scalable |

**Small scale:** Redis INCR + 10s flush batch
**Large scale:** Stream to Kafka → consumer writes to ClickHouse for per-click analytics (user agent, country, referer)

---

## Key Numbers

| Metric | Value |
|---|---|
| Writes per second | ~1,160 |
| Reads per second | ~115,000 |
| Short code length | 7 characters |
| Base62 7-char capacity | 3.5 trillion URLs |
| Storage per URL | ~250 bytes |
| 5-year total storage | ~45 TB |
| Redis memory per URL | ~100 bytes (hash entry) |
| 1B cached URLs in Redis | ~100 GB |

---

## DB Schema Summary

```sql
CREATE TABLE urls (
    id          BIGSERIAL PRIMARY KEY,
    short_code  CHAR(7)     UNIQUE NOT NULL,   -- indexed
    long_url    TEXT        NOT NULL,
    user_id     BIGINT,
    click_count BIGINT      DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT now(),
    expires_at  TIMESTAMPTZ,
    is_active   BOOLEAN     DEFAULT TRUE
);
```

Indexes: `short_code` (lookup), `user_id` (list my URLs), `expires_at` partial (cleanup job).

---

## Common Interview Pitfalls

1. **Choosing 301** — no analytics possible after browsers cache
2. **Synchronous click counter UPDATE** — will kill DB at 100k rps
3. **Sequential IDs without obfuscation** — enumerable, security issue
4. **No expiration support** — interviewers always ask "what if a user wants a temporary link?"
5. **No collision handling** — even random tokens need a retry path
6. **No reserved alias list** — `/api`, `/admin` must be blocked
7. **Not separating read and write path** — read API should never touch primary DB
