# System Design Framework — Interview Simulator

*Full mock interview walkthroughs. Treat each as a real 45-minute session. Read the problem, set a timer, and work through it before reading the guided answer.*

---

## How to Use This File

1. Read the **problem statement** only
2. Set a **45-minute timer**
3. Work through all 4 steps on paper/whiteboard
4. Then read the **guided walkthrough** to compare

---

## Simulation 1: Design a URL Shortener (TinyURL)

### Problem Statement
> Design a URL shortening service like TinyURL. Users paste a long URL and get a short one. Clicking the short URL redirects to the original.

---

### Guided Walkthrough

#### Step 1 — Clarify (5 min)

**Questions to ask:**
- Custom short codes? (e.g., `/my-brand`) or auto-generated only?
- Expiry on short URLs?
- Analytics (click counts, geo data)?
- Scale: how many URLs created/day? How many redirects?

**Agreed scope:**
- Auto-generated codes, no custom slugs
- No expiry (or optional TTL)
- No analytics (out of scope)
- 100M DAU, 10M writes/day, 10:1 read ratio

#### Step 2 — Estimate (5 min)

```
Write QPS = 10M / 86,400 ≈ 115 QPS
Read QPS  = 115 × 10 = 1,150 QPS (redirect lookups)

URL size = 500 bytes × 10M/day × 365 × 5yr ≈ 9 TB
Short code = 7 chars (base62) → 62^7 = 3.5 trillion URLs (sufficient)
```

#### Step 3 — HLD (15 min)

```
User
 │
 ▼
[ DNS: short.ly → CDN/LB ]
 │
 ├──► Write Service (POST /shorten)
 │      │
 │      ├──► ID Generator (Snowflake/counter)
 │      └──► PostgreSQL (code → long_url)
 │
 └──► Redirect Service (GET /:code)
        │
        ├──► Redis Cache (code → long_url, TTL 24h)
        │      │ miss
        └──────► PostgreSQL
                    │
                  HTTP 301/302 redirect
```

**Key decision:** 301 (permanent) vs 302 (temporary)
- 301: browser caches → less load on servers, but can't update/track clicks
- 302: every redirect hits your server → you can track analytics, update destination
- **Choose 302** unless analytics is off and CDN caching of redirects is needed

#### Step 4 — Deep Dive (20 min)

**Deep dive 1: ID Generation**
- Option A: UUID — globally unique but 128-bit, not human-friendly
- Option B: Auto-increment DB ID → encode in base62 → 7-char code
  - Problem: single DB is bottleneck; predictable IDs
- Option C: Snowflake ID — 64-bit, timestamp + worker ID + sequence → distributed, ordered
- **Choose:** Base62 encoding of a DB sequence is simplest; switch to Snowflake at scale

**Deep dive 2: Cache strategy**
- Cache-aside: on redirect, check Redis first, fallback to DB on miss
- Cache key = short code, value = long URL
- ~95%+ of traffic hits top 20% of URLs → cache is extremely effective
- TTL: 24h for popular URLs; LRU eviction

**Failure mode:**
- Redis down → fallback to DB (slower but correct)
- DB down → serve from cache only; writes fail gracefully with 503

---

## Simulation 2: Design a Notification System

### Problem Statement
> Design a notification system that can send push notifications, emails, and SMS to users when triggered by events (e.g., "you have a new message").

---

### Guided Walkthrough

#### Step 1 — Clarify (5 min)

**Questions:**
- Who triggers notifications? (system events, user actions, marketing campaigns?)
- Channels: push only, or also email/SMS?
- Delivery guarantee: at-least-once? Exactly-once?
- Real-time or near-real-time (< 5 seconds)?
- User preferences (opt-out per channel)?

**Agreed scope:**
- System events trigger notifications
- Push + email (no SMS)
- At-least-once delivery (duplicates OK, better than missing)
- Near-real-time < 5 seconds
- User preference stored in DB

#### Step 2 — Estimate (5 min)

```
10M DAU × 5 notifications/day = 50M notifications/day
= ~580 notifications/second peak
Push: 80% → 464/sec to APNs/FCM
Email: 20% → 116/sec to SES/SendGrid
Storage: 50M × 200 bytes metadata ≈ 10 GB/day
```

#### Step 3 — HLD (15 min)

```
Event Source (order placed, message received)
  │
  ▼
[ Notification Service ] ──► check user_preferences DB
  │
  ▼
[ Message Queue (Kafka) ]
  ├──► [ Push Worker ] ──► APNs / FCM
  ├──► [ Email Worker ] ──► SendGrid / SES
  └──► [ Notification Log DB ] (delivery status)
```

**Why Kafka?**
- Durable: messages persist even if workers are down
- Replay: can resend failed notifications
- Fan-out: same event can trigger multiple channels

#### Step 4 — Deep Dive (20 min)

**Deep dive 1: Delivery guarantees**
- At-least-once: Kafka consumer commits offset AFTER successful send
- If push fails → dead letter queue → retry with backoff → alert on 3 failures
- Deduplication: track `notification_id` in Redis; skip if already sent

**Deep dive 2: Rate limiting to 3rd party APIs**
- APNs/FCM have rate limits per app (~50K/sec burst)
- Use token bucket per channel worker; back-pressure to Kafka consumer group
- Spillover to lower-priority queue (marketing notifications have lower priority than transactional)

**Failure mode:**
- APNs down → queue backed up → resume when APNs recovers (Kafka retention = retry buffer)
- Notification service down → events queue in Kafka; process on recovery
- User has no push token → skip push, log, try email

---

## Simulation 3: Framework Self-Check (30 min)

Use this as a final self-assessment. Pick any system you haven't designed before and grade yourself:

| Checkpoint                              | Score (1–5) |
|-----------------------------------------|-------------|
| Asked 3+ clarifying questions before drawing | /5    |
| Capacity estimate completed in < 5 min  | /5          |
| HLD covers: LB, cache, DB, queue        | /5          |
| Mentioned at least one failure mode     | /5          |
| Deep-dived into the actual bottleneck   | /5          |
| Mentioned observability (logs/metrics)  | /5          |
| Stated trade-offs for each key decision | /5          |
| Finished within 45 minutes              | /5          |
| **Total**                               | /40         |

**Scoring:**
- 35–40: Ready for senior-level interviews
- 25–34: Solid; work on trade-off depth and failure modes
- < 25: Review architecture.md and practice more simulations
