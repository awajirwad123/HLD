# Notification System — Notes & Reference

## Notification Channel Comparison

| Channel | Provider | Latency | Cost | When to use |
|---|---|---|---|---|
| iOS Push | APNs | ~1s | Free* | Engaged mobile users |
| Android Push | FCM | ~1s | Free* | Engaged mobile users |
| Email | SendGrid / SES | 1–5 min | ~$0.0009/email | Transactional, marketing |
| SMS | Twilio / SNS | 5–30 sec | ~$0.0075/SMS | OTP, critical alerts |
| In-App | WebSocket / Poll | ~100ms | Free | Active session users |
| Web Push | FCM (VAPID) | ~1s | Free* | Desktop browser users |

*Free but requires Apple/Google developer account.

---

## APNs vs FCM Comparison

| Aspect | APNs (Apple) | FCM (Firebase/Google) |
|---|---|---|
| Protocol | HTTP/2 | HTTP v1 API |
| Auth | JWT or .p12 certificate | Service account JSON key |
| Token | Per-device token (changes on reinstall) | Per-device FCM registration token |
| Payload limit | 4KB | 4KB |
| Priority | normal (APNS-priority: 5) or high (10) | normal or high |
| Silent push | Yes (content-available: 1) | Yes (data-only message) |
| Delivery receipt | No (fire-and-forget) | Optional delivery receipt |
| APNS environment | Sandbox vs Production (separate!) | Unified |

**Common mistake:** Using sandbox token against production APNs → always `BadDeviceToken` error.

---

## Device Token Lifecycle

```
App installs → registers with APNs/FCM → receives token → sends to your server
                                                               ↓
                                               stored in device_tokens table

App updated/reinstalled → new token assigned → old token invalid
OS routine refresh → token may change → app sends updated token

APNs/FCM response "Invalid token" or 410 Gone
    → mark token as inactive in DB
    → stop sending to it
    → user must reopen app to get fresh token
```

**Token refresh code (mobile side, conceptual):**
```swift
// iOS
func application(_ application: UIApplication,
    didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    let tokenString = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
    sendTokenToServer(tokenString)  // Your POST /device-tokens endpoint
}
```

---

## DB Schema Reference

```sql
-- User preferences
CREATE TABLE notification_preferences (
    user_id             BIGINT PRIMARY KEY,
    push_enabled        BOOLEAN DEFAULT TRUE,
    email_enabled       BOOLEAN DEFAULT TRUE,
    sms_enabled         BOOLEAN DEFAULT FALSE,
    quiet_hours_start   SMALLINT,              -- UTC hour
    quiet_hours_end     SMALLINT,
    timezone            TEXT DEFAULT 'UTC',
    opted_out_types     TEXT[] DEFAULT '{}',   -- e.g. '{"post_liked","promo"}'
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Device tokens (one user can have many devices)
CREATE TABLE device_tokens (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    token       TEXT NOT NULL UNIQUE,
    platform    TEXT NOT NULL CHECK (platform IN ('ios', 'android', 'web')),
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    last_used   TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_tokens_user ON device_tokens (user_id) WHERE is_active = TRUE;

-- In-app notification inbox
CREATE TABLE notifications (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    type        TEXT NOT NULL,
    title       TEXT,
    body        TEXT NOT NULL,
    metadata    JSONB,
    is_read     BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_notifs_user_unread ON notifications (user_id, created_at DESC)
    WHERE is_read = FALSE;

-- Delivery log for auditing/debugging
CREATE TABLE notification_deliveries (
    id              BIGSERIAL PRIMARY KEY,
    notification_id BIGINT REFERENCES notifications(id),
    channel         TEXT NOT NULL,
    provider        TEXT,
    status          TEXT CHECK (status IN ('pending','sent','failed','deduped')),
    attempt         SMALLINT DEFAULT 1,
    provider_id     TEXT,       -- APNs APNS-ID, FCM message_id, SendGrid msg_id
    error_code      TEXT,
    sent_at         TIMESTAMPTZ
);
```

---

## Retry and Dead Letter Queue Pattern

```
Event arrives → attempt #0
    ↓ fails?
    Wait 30s → attempt #1
    ↓ fails?
    Wait 2min → attempt #2
    ↓ fails?
    Wait 10min → attempt #3
    ↓ fails?
    Wait 1hr → attempt #4
    ↓ fails?
    → DLQ (manual inspection / alerting)
```

**Implementation:** Redis Sorted Set where score = scheduled delivery timestamp.
```redis
ZADD notif:retry_queue {unix_timestamp} {notification_json}
ZRANGEBYSCORE notif:retry_queue -inf {now} LIMIT 0 100
```

---

## Rate Limiting Strategy

| Window | Limit | Purpose |
|---|---|---|
| Per user per 5 min (social) | Max 3 raw → aggregate into 1 | Prevent like-spam |
| Per user per day (total) | Max 100 notifications | Prevent overwhelming |
| Per user per day (marketing) | Max 1 promo email | Marketing cap |
| Per user per 10 min (OTP) | Max 3 OTP SMS | Anti-spam for SMS cost |

Implementation: Redis sliding window counter.
```
INCR ratelimit:{user_id}:{type}:{window}
EXPIRE ratelimit:{user_id}:{type}:{window} {window_seconds}
```

---

## Quiet Hours Implementation

1. Store quiet hours in user_preferences as UTC hours + user timezone.
2. At dispatch time: convert current UTC to user's local time, check if it's quiet.
3. For delayed delivery: schedule notification to wake up at quiet_hours_end.
4. Always bypass for CRITICAL priority (fraud, security, OTP).

```python
from zoneinfo import ZoneInfo
from datetime import datetime

def is_quiet(prefs) -> bool:
    tz = ZoneInfo(prefs.timezone)
    local_hour = datetime.now(tz).hour
    qs, qe = prefs.quiet_hours_start, prefs.quiet_hours_end
    if qs < qe:
        return qs <= local_hour < qe
    else:                              # crosses midnight
        return local_hour >= qs or local_hour < qe
```

---

## Aggregation Window Pattern

**Problem:** User gets 1,000 "X liked your post" notifications in an hour.
**Solution:** Buffer events per (user, type) for a window period, then send one digest.

```
Event arrives: "Alice liked your post"
→ Add "Alice" to Redis list: RPUSH notif:buffer:{user}:post_liked Alice
→ Set or refresh TTL: EXPIRE notif:buffer:{user}:post_liked 300

After 5 minutes (triggered by TTL expiry or scheduled flush):
→ LRANGE notif:buffer:{user}:post_liked 0 -1
→ Build "Alice, Bob, and 3 others liked your post"
→ Send single notification
→ DELETE buffer key
```

---

## Email Deliverability Checklist

| Concern | Solution |
|---|---|
| Spam filters | SPF record, DKIM signing, DMARC policy |
| Bounce handling | Suppress bounced addresses immediately |
| Unsubscribe | Include List-Unsubscribe header + one-click unsubscribe |
| IP reputation | Use dedicated sending IP; warm up gradually |
| send rate | Ramp up slowly for new sending domains/IPs |
| Content | Avoid spam trigger words, include plain-text part |

**SPF example:**
```
TXT record: v=spf1 include:sendgrid.net ~all
```

---

## Kafka Topic Layout

```
kafka topics:
  notif.push.ios        ← APNs worker consumes
  notif.push.android    ← FCM worker consumes
  notif.email           ← Email worker consumes
  notif.sms             ← SMS worker consumes
  notif.inapp           ← WebSocket server consumes
  notif.dlq             ← Dead letter queue (all channels)
```

Partition by `user_id` to preserve per-user ordering and avoid duplicate fan-out.

---

## Key Numbers

| Metric | Value |
|---|---|
| 1B notifications/day | ~11,600/sec peak |
| APNs connections needed | ~10K push/sec per connection → ~2 connections for 20K/sec |
| SendGrid throughput | ~10M emails/hour on Pro plan |
| Twilio SMS throughput | ~1 msg/sec per long-code; use short-codes for 100+/sec |
| Dedup Redis key TTL | 3600s (1hr) |
| Retry window | Up to 1hr (5 attempts) |
| In-app notification inbox | Keep 90 days; archive or delete older |
| Device token table | One row per install; expect ~3× users for multi-device |
| Preference read latency | < 5ms (Redis cache, updated lazily) |
