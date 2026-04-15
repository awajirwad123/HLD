# Notification System — Architecture

## Problem Statement

Design a notification service that:
- Sends notifications via multiple channels: push (iOS/Android), email, SMS, in-app
- ~1B notifications/day across all channels
- Handles retries, deduplication, user preferences, and quiet hours
- Fan-out: one system event might trigger notifications to millions of users

---

## Notification Types and Delivery Channels

| Type | Channel | Latency requirement | Example |
|---|---|---|---|
| Transactional | Email + SMS | Seconds | OTP, order confirmation |
| Engagement | Push + In-app | Minutes OK | "X liked your post" |
| Marketing | Email + Push | Hours OK | "New sale event" |
| Alert / Critical | SMS + Push | < 10 seconds | Fraud alert, payment failed |

---

## High-Level Architecture

```
Event Producer          Notification Service              Delivery Providers
─────────────          ─────────────────────             ──────────────────
Order Placed   ──────→  Notification Router               APNs (Apple iOS)
Like Event              ┌──────────────────┐  ─────────→  FCM (Android/Web Push)
Fraud Alert             │ Preference Check  │  ─────────→  SendGrid / SES (Email)
Marketing CMS           │ Template Render   │  ─────────→  Twilio (SMS)
                        │ Dedup Check       │  ─────────→  WebSocket (In-app)
                        │ Rate Limiting     │
                        └──────────────────┘
                               ↓ Kafka
               ┌───────────────┼──────────────────┐
               ▼               ▼                  ▼
        Push Worker      Email Worker       SMS Worker
        (APNs/FCM)       (SendGrid)         (Twilio)
               ↓               ↓                  ↓
         Delivery DB      Delivery DB        Delivery DB
         (status, retry)  (status, retry)    (status, retry)
```

---

## Notification Pipeline (Step by Step)

### Step 1: Event Ingestion

Events arrive from many upstream services via Kafka:
```json
{
  "event_type": "post_liked",
  "actor_user_id": 456,
  "recipient_user_id": 123,
  "metadata": { "post_id": 789 },
  "priority": "normal"
}
```

### Step 2: Preference Check

Before sending, check user's notification preferences:
```
Does user 123 have push notifications enabled?
Is it user 123's quiet hours? (e.g., 11pm–7am local time)
Has user 123 opted out of "like" notifications?
```

```sql
SELECT channel_prefs, quiet_hours_start, quiet_hours_end, timezone
FROM notification_preferences
WHERE user_id = 123;
```

### Step 3: Template Rendering

Turn the raw event into human-readable text:
```
"like" template:  "{actor_name} liked your post"
"comment" template: "{actor_name} commented: '{comment_preview}'"
```

### Step 4: Deduplication

Same event shouldn't send two notifications (idempotency). Dedup key:
```
dedup_key = SHA256(event_type + recipient_id + actor_id + entity_id)
Redis: SET dedup:{key} 1 NX EX 3600    # 1-hour dedup window
```

### Step 5: Rate Limiting

Prevent notification spam per recipient:
- Max 10 "like" notifications per user per 5 minutes (batch/aggregate)
- Max 100 total notifications per user per day

### Step 6: Batching / Aggregation

Instead of sending "A liked your post", "B liked your post", "C liked your post" individually, aggregate:
- Wait 5 minutes after first like event
- If > 3 likes: "A, B, and 3 others liked your post"
- Single notification for the aggregated batch

---

## Push Notification Deep Dive

### iOS: Apple Push Notification Service (APNs)

```
Your Server → APNs (Apple) → iOS Device
```

APNs requires:
- **Device Token:** Unique identifier for a specific app installation on a device. App registers with APNs → receives device token → sends to your server → stored in DB.
- **Authentication:** JWT (Provider Token) or Certificate (.p12)
- **Payload limit:** 4KB for data, 5KB for alerts

```json
{
  "aps": {
    "alert": {
      "title": "New Like",
      "body": "Alice liked your post"
    },
    "badge": 3,
    "sound": "default"
  }
}
```

### Android: Firebase Cloud Messaging (FCM)

Similar to APNs but via Google's infrastructure. FCM token = equivalent to APNs device token. Web Push also supported via FCM.

### Device Token Lifecycle

```
Token created → stored in DB when user installs app
Token refreshed → app sends new token (OS-level refresh every few months)
Token invalidated → APNs/FCM returns error "InvalidToken" → mark as inactive in DB; stop sending
```

**Common bug:** Not handling `InvalidToken` responses causes you to keep pushing to stale tokens — wastes APNs/FCM quota and masks real delivery failures.

---

## In-App Notifications

Real-time: user is active → push via WebSocket connection.
Offline: user opens app → fetch unread notifications from DB.

```sql
CREATE TABLE notifications (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    type            TEXT NOT NULL,
    title           TEXT,
    body            TEXT,
    metadata        JSONB,
    is_read         BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_notifs_user_unread ON notifications (user_id, is_read, created_at DESC)
    WHERE is_read = FALSE;   -- Partial index: only unread rows
```

Unread badge count: `SELECT count(*) FROM notifications WHERE user_id=$1 AND is_read=false` — cached in Redis.

---

## Retry Logic

Delivery failures (APNs timeout, email bounce) must be retried:

```
Attempt 1: immediately
Attempt 2: +30 seconds
Attempt 3: +2 minutes
Attempt 4: +10 minutes
Attempt 5: +1 hour
Give up → log as FAILED
```

Exponential backoff with jitter. Workers consume from Kafka with manual offset commit — don't commit offset until delivery is confirmed. On failure: publish to dead-letter queue for later retry or human review.

---

## Aggregation (Digesting)

Marketing digest: instead of 50 emails/day for each event, send a daily digest email with all activity. Implementation:
- Buffer notification events per user in Redis List
- Scheduled job (cron) processes digests at 8am local time per user
- Render "Today's activity" email from buffered events
- Clear buffer after sending

---

## Key Numbers

| Metric | Value |
|---|---|
| Notifications/day | 1 billion |
| Push notifications (APNs/FCM) | ~60% of volume |
| Email | ~30% of volume |
| SMS | ~10% of volume |
| APNs throughput per connection | ~10K push/sec |
| FCM throughput | Practically unlimited (HTTP/2) |
| SendGrid daily plan | up to 100M emails/day |
| Notification DB storage | ~100 bytes/notification |
| Retry window | Up to 1 hour |
