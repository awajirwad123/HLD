# Notification System — Interview Simulator

## Scenario 1: "Design a Notification System for a Social App (100M Users)"

**Prompt:** "Design a system that sends push, email, and SMS notifications triggered by social events: likes, comments, follows, DMs. Users can configure their preferences. The system should handle 1B notifications/day."

---

**Clarifying questions to ask:**

1. What types of events trigger notifications? (Social: like, comment, follow. Transactional: password reset, OTP?)
2. Do we need read receipts or delivery confirmation?
3. What are the latency requirements? (Real-time social < 5 seconds, marketing < 5 minutes?)
4. Multi-device support (one user, 3 devices)?
5. Can we use third-party providers (APNs, FCM, SendGrid)?

---

**Model answer outline:**

**Capacity:**
- 1B notifications/day ÷ 86,400 = ~11,600/sec average; peak ~50K/sec
- 100M users, avg. 3 devices each = 300M device token records
- Storage: 100 bytes/notification × 1B/day × 30 days = ~3TB/month

**Component walkthrough:**

```
Social Event → Kafka "events" topic
     ↓
Notification Orchestrator (consumer)
  - Reads event
  - Looks up who to notify (post owner, followers in DM case)
  - Checks per-user preferences (Redis cache, DB fallback)
  - Applies dedup key
  - Applies rate limiting / aggregation buffering
  - Publishes to channel-specific Kafka topics
     ↓
Kafka topics: notif.push.ios | notif.push.android | notif.email | notif.sms
     ↓
Channel Workers:
  APNs Worker → Apple APNs (HTTP/2, batch, 100 in-flight connections)
  FCM Worker  → Google FCM (batch 500 tokens per call)
  Email Worker → SendGrid API
  SMS Worker  → Twilio API
     ↓
Delivery status written to DB + DLQ on final failure
```

**Preferences:**
- `notification_preferences` table: push_enabled, email_enabled, sms_enabled, quiet_hours, opted_out_types
- Cached in Redis for 5 minutes; actively invalidated on user change
- Checked at Orchestrator, not at fan-out

**Device tokens:**
- Stored in `device_tokens` table (user_id, token, platform, is_active)
- Invalidated on `BadDeviceToken` response from APNs/FCM
- Multi-device: fan out to all active tokens for the user

**Retry/DLQ:**
- Redis Sorted Set: score = scheduled_at timestamp
- 5 attempts with exponential backoff (0s, 30s, 2min, 10min, 1hr)
- Dead letter queue for human review after all attempts fail

**Aggregation:**
- Buffer social events per (user, type) in Redis for 5 minutes
- If > threshold, send digest: "Alice, Bob, and 3 others liked your post"
- Prevents notification spam on viral content

**Open questions to mention proactively:**
- Should we store all notifications in an inbox table visible in-app? (Yes, needed for unread count badge)
- How to handle app uninstalls? (APNs/FCM return 410, mark inactive)
- Email deliverability? (SPF, DKIM, DMARC on sending domain)

---

## Scenario 2: "Amazon Order Notification System"

**Prompt:** "Design the notification system for Amazon order events: order confirmed, shipped, out for delivery, delivered. 100M orders/day, each with up to 5 status transitions."

---

**Key design decisions:**

**Notification triggers are state transitions, not user events:**
```
order_state_machine:
  placed → confirmed → shipped → out_for_delivery → delivered
                                                   → delivery_failed
```
Each transition should fire exactly one notification.

**Exactly-once per transition:**
- Risk: order_state_change event processed twice → user gets 2 "Your order shipped!" emails.
- Solution: Idempotency key = `{order_id}:{transition}` stored in `sent_notifications` table.
- Before sending: `INSERT INTO sent_notifications (key) VALUES (...) ON CONFLICT DO NOTHING` → only proceed if INSERT succeeds.

**Channel mix for transactional notifications:**
- Email: full receipt with order details, estimated delivery date, tracking link
- Push: brief alert "Your order has shipped!" with deep link to order page
- SMS: only if user opted in or for delivery failure alerts

**Timing / scheduling:**
- "Out for delivery" in the morning (7am local time) — user wants to know before leaving home
- Never send at 3am — schedule to next morning if order transitions happen overnight
- Exception: "Delivery failed" sends immediately regardless of time (user may be home)

**Template personalization:**
```
email template "order_shipped":
  Subject: "Your order is on the way, {first_name}!"
  Body: "Order #{order_id} containing {product_name} shipped via {carrier}. 
         Tracking: {tracking_url}. Estimated delivery: {delivery_date}."
```
Templates stored in a CMS, rendered with Jinja2/Handlebars at dispatch time.

**Tracking URL generation:**
- Call carrier API to get tracking number → embed in notification
- Tracking URLs should be long-lived (order exists for years)
- Don't use URL shortener for transactional links (phishing perception)

**Scale:**
- 100M orders × 5 transitions = 500M state change events/day
- ~6,000 events/sec average, 30K/sec peak (Prime Day)
- Email volume: ~150M emails/day (not all orders trigger all transitions)
- SLA: "shipped" email within 5 minutes of carrier scan

**Disaster scenario:** Carrier API outage → tracking URL unavailable. Solution: Send notification without tracking link, add "tracking will appear within 1 hour" message. Async job retries fetching tracking URL and sends a follow-up push when available.

---

## Scenario 3: "Debugging — Notifications Are Sent to Opted-Out Users"

**Prompt:** "Customers are complaining they're receiving marketing push notifications even though they disabled them in settings. How do you debug this?"

---

**Systematic debugging approach:**

**Step 1: Gather data**
- How many complaints in what time window? (Widespread vs isolated)
- Which notification type? (All marketing or specific type?)
- When did they opt out? (Before or after the notification was sent?)
- What channel? (Push only or email too?)

**Step 2: Check preference propagation**

Is the opt-out actually saved?
```sql
SELECT push_enabled, opted_out_types, updated_at
FROM notification_preferences
WHERE user_id = {complaining_user_id};
```
Is Redis cache stale?
```redis
GET prefs:{user_id}
```
If Redis shows `push_enabled = true` but DB shows `false`: cache invalidation bug. The preference write didn't call `DEL prefs:{user_id}`.

**Step 3: Check when the notification was produced**

Pull the delivery log:
```sql
SELECT created_at, status, channel, notification_id
FROM notification_deliveries
WHERE user_id = {user_id} ORDER BY created_at DESC LIMIT 10;
```
Compare `created_at` with when the user opted out (`notification_preferences.updated_at`).

If `created_at < opt_out_time`: notification was in flight before the preference change — this is expected behavior (window of in-flight events).

If `created_at > opt_out_time`: bug — preference check is not being honored.

**Step 4: Identify the code path**

Was preference checked at fan-out or at delivery?
- If checked only at fan-out time (not delivery), in-flight events in Kafka bypass the check.
- Fix: move preference check to the delivery worker (just before APNs/FCM call), not just fan-out.

**Step 5: Check for batch marketing job**

Marketing campaigns often have their own code path that bulk-exports user lists to an email/push vendor. This path may not go through the regular preference check. Verify that the marketing campaign job:
1. Joins with `notification_preferences` and filters opted-out users before export
2. Respects `opted_out_types` array, not just `push_enabled` boolean

**Root causes (in priority order):**
1. Redis cache TTL too long + no active invalidation on preference change (most common)
2. Marketing job bypasses preference check entirely
3. Preference check at fan-out time only — not re-checked at delivery
4. DB replication lag — preference write to primary not yet visible on read replica used by worker
5. Race condition: notification event produced before opt-out, but processed after — borderline acceptable but confusing to users

**Fix:**
```python
# At delivery time (not just fan-out), always re-read preferences:
prefs = await get_prefs(user_id)  # Redis first, DB fallback
if event.event_type in prefs.opted_out_types:
    log_suppressed(event)
    return  # Don't send
```
Add monitoring: track `notifications_suppressed_by_preference` counter separately from `notifications_sent`. If suppressed count is zero, something is wrong.
