# Notification System — Quick-Fire Questions

**Q1: What are the four main notification delivery channels, and when do you pick each one?**

Push (APNs/FCM) for mobile users who have the app installed; Email for transactional receipts and marketing; SMS for critical/OTP where guaranteed delivery is needed; In-App (WebSocket) for active sessions where real-time display matters most. Often you layer them: push first, fall back to email if push token invalid.

---

**Q2: What is a device token and what happens when it expires?**

A device token is an opaque identifier assigned by APNs/FCM to a specific app installation on a specific device. It makes no sense across devices or reinstalls. When APNs/FCM returns `InvalidToken` (or HTTP 410), you must mark it inactive and stop sending to it. The user gets a new token the next time they open the app, which re-registers with your server.

---

**Q3: How do you prevent the same notification from being sent twice?**

Compute a deduplication key from `SHA256(event_type + recipient_id + actor_id + entity_id + channel)` and try `SET dedup:{key} 1 NX EX 3600` in Redis. If the SET fails (key exists), the notification was already sent within the dedup window — skip it. This handles retries, duplicate Kafka messages, and double fan-out bugs.

---

**Q4: Explain the retry strategy for failed notification deliveries.**

Use exponential backoff: attempt immediately, then after 30s, 2min, 10min, 1hr. After 5 attempts, send to a Dead Letter Queue for human review or alerting. Each attempt state is tracked so workers can resume without reprocessing successful deliveries. Add random jitter to prevent retry storms when many notifications fail simultaneously.

---

**Q5: What is a Dead Letter Queue and when does a notification end up there?**

A DLQ is a storage location (Redis List, Kafka topic, or DB table) where messages land after all retry attempts are exhausted. A notification ends up in the DLQ when it has failed delivery on all retry attempts. Engineers review DLQ items to diagnose systemic failures (e.g., "APNs certificate expired" or "Twilio account suspended").

---

**Q6: How do you respect a user's quiet hours at scale?**

Store quiet hours start/end + timezone in the notification_preferences table. Before dispatching, convert current UTC to the user's local time and check whether it falls in the quiet window. For non-critical notifications in quiet hours: either discard, or schedule delivery for when quiet hours end (put back into the retry queue with `next_attempt_at = quiet_end_time`). Always bypass quiet hours for priority=CRITICAL.

---

**Q7: Why would you separate APNs and FCM into different Kafka topics?**

Each provider has different authentication, protocols, rate limits, and failure modes. Separating topics means you can tune consumer count, retry policy, and monitoring independently. FCM tokens and APNs tokens have no overlap. A SendGrid outage shouldn't block your push worker queue from draining.

---

**Q8: What's the difference between a visible push notification and a silent push?**

A visible push shows a banner/alert to the user (APNs `alert` payload, FCM `notification` field). A silent push is delivered in the background without any UI; the app receives it, processes it, and can fetch fresh data or update a badge. Silent pushes use `content-available: 1` on APNs. They're useful for refreshing in-app content or triggering sync without disturbing the user.

---

**Q9: How do you build an aggregated "X, Y, and 3 others liked your post" notification?**

Buffer events in a Redis List keyed by `notif:buffer:{user_id}:{event_type}` with a TTL (e.g., 5 minutes). On each new "like" event, RPUSH the actor name and EXPIRE to reset the TTL. When the buffer is flushed (TTL expiry or scheduled job), LRANGE the buffer, build a digest string, send one notification, and DELETE the key.

---

**Q10: A user gets 200 "likes" in 1 minute due to a viral post. How do you prevent notification spam?**

Two mechanisms: (1) Rate limiting — after sending the first notification for `post_liked`, suppress further ones for a window (e.g., 5 min). (2) Aggregation — buffer all actors in that window and send one digest at the end. You can also combine: send immediately for the first like, then aggregate subsequent ones into a digest.

---

**Q11: How is an in-app notification inbox implemented?**

A `notifications` table in PostgreSQL with `user_id`, `type`, `body`, `is_read`, `created_at`. A partial index on `(user_id, created_at DESC) WHERE is_read = FALSE` makes unread count queries fast. The unread count is cached in Redis (`SET notif:unread:{user_id} {count}`) and incremented on each new notification to avoid a `COUNT(*)` query on every page load.

---

**Q12: What SPF, DKIM, and DMARC are and why they matter for email notifications?**

SPF (Sender Policy Framework): DNS TXT record listing which IPs are authorized to send email for your domain. Prevents spoofing. DKIM (DomainKeys Identified Mail): Cryptographic signature on outbound emails proving they came from your domain. DMARC: Policy declaring what to do when SPF/DKIM fail (none/quarantine/reject) and where to send failure reports. Without these, your notification emails land in spam.

---

**Q13: How do you handle bulk marketing notifications vs real-time transactional notifications in the same system?**

Separate Kafka topics with different consumer groups and SLA targets. Transactional (OTP, order confirmation): high-priority Kafka partition, dedicated workers, monitored for latency < 30s. Marketing (promo emails): low-priority topic, processed in batches during off-peak hours, rate-limited per user per day. They share the same delivery workers but consume from separate queues with separate rate limits.

---

**Q14: A celebrity gains 5M new followers in an hour. Each sends them a "welcome" push. What breaks?**

The notification workers get flooded with 5M push deliveries to one APNs token (or a small set). APNs has per-endpoint rate limits and may start throttling. Memory in the job queue fills up. Solution: deduplicate "you got a new follower" events with a short window (send one notification per N new followers or per minute), and rate-limit outbound APNs connections. This is similar to the celebrity fan-out problem in social feeds.

---

**Q15: What metrics would you monitor for a notification system?**

Delivery success rate per channel, P99 delivery latency (time from event to provider accept), retry queue depth, DLQ size, APNs/FCM invalid token rate (signals app churn), email bounce rate, unsubscribe rate, opt-out rate per notification type, Kafka consumer lag per worker group.
