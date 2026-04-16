# URL Shortener — Architecture

## Problem Statement

Design a service like TinyURL that:
- Accepts a long URL and returns a short code (e.g., `https://short.ly/aB3xZ`)
- Redirects `short.ly/aB3xZ` → original URL with < 10ms P99 latency
- Handles ~100M new URLs/day, ~10B redirects/day

---

## Capacity Estimation

```
Writes:  100M URLs/day  → ~1,160 writes/sec
Reads:   10B clicks/day → ~115,000 reads/sec (read:write ≈ 100:1)

Storage per URL:
  short_code (7 chars):  7 bytes
  long_url (avg 200):  200 bytes
  created_at:            8 bytes
  user_id:               8 bytes
  ≈ 250 bytes/record

Storage for 5 years:
  100M/day × 365 × 5 × 250 bytes ≈ 45 TB
```

**Dominant concern: read throughput (115k rps on redirects).**

---

## Core Design

### Short Code Generation

The short code must be:
1. Unique — two long URLs never get the same short code
2. Short — 7 characters covers 62^7 ≈ 3.5 trillion combinations
3. Not guessable — random prevents enumeration

**Option A: Hash-based**

```
MD5(long_url) → 128-bit hash → take first 43 bits → Base62 encode → 7 chars
```

Problem: collision when two different users shorten the same URL — they get the same code but may want separate analytics. Also, pre-image guessing is possible with well-known URLs.

**Option B: Auto-increment ID + Base62 (Recommended)**

```
DB generates ID (1, 2, 3, ...) → Base62 encode → short code
```

```
Base62 alphabet: 0-9A-Za-z

ID 1         → "0000001"
ID 3,521,614,606 → "aB3xZ00"
```

Problem: sequential IDs are guessable (enumerate `a`, `b`, `c`...)

**Option C: Random token (Best for anti-enumeration)**

```python
import secrets, string
ALPHABET = string.ascii_letters + string.digits  # 62 chars
token = ''.join(secrets.choice(ALPHABET) for _ in range(7))
```

Check uniqueness against the DB. Collision probability ≈ 1 in 3.5 trillion per attempt — negligible until ~1B URLs exist (birthday paradox threshold ≈ 62^7 / 2 ≈ 1.75T).

**Option D: Snowflake ID + Base62 (Production choice)**

Distributed services each generate unique IDs via Snowflake (timestamp + machine ID + sequence). Base62-encode the 64-bit integer → 7–11 chars. No DB roundtrip for uniqueness check.

---

## High-Level Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  Clients (browser / mobile app / API)                          │
└────────────────┬───────────────────────────────────┬───────────┘
                 │ POST /shorten                      │ GET /{code}
                 ▼                                    ▼
         ┌──────────────┐                   ┌──────────────────┐
         │  Write API   │                   │    Redirect API  │
         │  (FastAPI)   │                   │    (FastAPI)     │
         └──────┬───────┘                   └────────┬─────────┘
                │ INSERT short_code                   │ GET short_code
                ▼                                     │
         ┌─────────────┐                              │ Cache MISS
         │  Primary DB │◄─────────────────────────────┤
         │  (Postgres) │                              │
         └──────┬──────┘                              │ Cache HIT
                │ async replication                   ▼
         ┌──────▼──────┐                   ┌──────────────────┐
         │  Read       │                   │  Redis Cluster   │
         │  Replica(s) │◄──────────────────│  (short→long     │
         └─────────────┘                   │   URL cache)     │
                                           └──────────────────┘
                                                      │
                                           ┌──────────▼────────┐
                                           │  CDN Edge Cache   │
                                           │  (for popular     │
                                           │   short codes)    │
                                           └───────────────────┘
```

**Redirect flow (hot path):**
1. CDN edge node checks: does it have this code cached? → 302 redirect immediately (< 1ms)
2. If CDN miss → Redis lookup (< 1ms)
3. If Redis miss → DB read replica lookup (< 5ms) → warm Redis + CDN
4. HTTP 302 (Found) redirect to original URL

---

## Redirect: 301 vs 302

| | 301 Permanent | 302 Temporary |
|---|---|---|
| Browser behavior | Caches redirect forever | Always asks server |
| Analytics | Browser never visits short URL again after first hit | Every click hits server → accurate click counting |
| CDN behavior | CDN caches aggressively | CDN honors Cache-Control TTL |
| Use when | You want maximum redirect speed, no analytics | You want per-click analytics (TinyURL, Bitly) |

**Production choice:** 302 with `Cache-Control: max-age=3600` — browser re-checks every hour, CDN caches for 1 hour.

---

## Database Schema

```sql
CREATE TABLE urls (
    id           BIGSERIAL PRIMARY KEY,
    short_code   CHAR(7)      NOT NULL UNIQUE,
    long_url     TEXT         NOT NULL,
    user_id      BIGINT       REFERENCES users(id),
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
    expires_at   TIMESTAMPTZ,                         -- optional TTL
    click_count  BIGINT       NOT NULL DEFAULT 0,
    is_active    BOOLEAN      NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_short_code ON urls (short_code);     -- Primary lookup
CREATE INDEX idx_user_id    ON urls (user_id);        -- "My links" page
CREATE INDEX idx_expires_at ON urls (expires_at)
    WHERE expires_at IS NOT NULL;                     -- Cleanup job
```

---

## Click Analytics

Naive approach: `UPDATE urls SET click_count = click_count + 1` on every redirect.
Problem: 115,000 writes/sec on a hot row → lock contention, DB bottleneck.

**Better:** Write-behind counter with Redis:

```
On redirect hit:
  INCR redis:clicks:{short_code}

Background job (every 10s):
  GETDEL redis:clicks:{short_code} → batch UPDATE urls
```

**Even better for rich analytics:** Stream click events to Kafka → ClickHouse/BigQuery for analytics queries. Never write to the primary DB on the redirect critical path.

---

## Expiration and Cleanup

- Store `expires_at` in the URL record
- Redirect API checks: if `now() > expires_at` → 410 Gone
- Background job deletes expired rows nightly (avoid hot-path deletes)
- Soft-delete first (`is_active = false`), hard-delete later after CDN/Redis TTL expires

---

## Custom Alias Support

Users want `short.ly/my-brand`. Approach:
- Allow any alphanumeric + dash alias (1–20 chars)
- Store in same `short_code` column (no separate table needed)
- Validation: check uniqueness; reject reserved words (`api`, `admin`, `static`)

---

## Scaling Bottleneck Analysis

| Component | Bottleneck | Solution |
|---|---|---|
| Short code generation | Sequential IDs require centralized auto-increment | Snowflake IDs or token range pre-allocation per service instance |
| Read path | 115k rps on redirect lookups | Redis cluster (95%+ cache hit ratio for popular codes), CDN for top 0.1% |
| Write path | 1,160 writes/sec | Single primary DB is sufficient; add read replicas for analytics queries |
| Analytics writes | 115k click increments/sec | Redis counters + async flush; never hit primary DB |
| DB size | 45 TB in 5 years | Time-based partitioning (`PARTITION BY RANGE(created_at)`); archive old partitions to cold storage |

---

## What Can Go Wrong — Failure Modes

| Failure | Impact | Mitigation |
|---|---|---|
| Redis down | All reads hit DB directly | Redis Sentinel/Cluster for HA; DB read replicas can handle burst without Redis |
| DB primary down | Writes fail | Auto-failover to replica (Patroni/RDS Multi-AZ); write queue in Redis (persist + retry) |
| Code collision | Two URLs get same code | UNIQUE constraint in DB causes INSERT failure → retry with new code |
| URL enumeration | Attacker scans all short codes | Add CAPTCHA for automated POST; use random token (not sequential) |
| Phishing URLs | Users shortened malicious URLs | Scan long_url against Google Safe Browsing API on write; block flagged domains |
