# Notification System — Tricky Interview Questions

## Q1: "Your APNs certificate expired at 3am. How do you detect this and recover?"

**What they're testing:** Operational awareness, error handling on external provider responses, on-call readiness.

**Trap:** Saying "we'll catch the error when we try to send" — if you're not monitoring, you may not know for hours.

**Strong answer:**

Detection:
- Monitor APNs HTTP/2 response codes. Certificate expiry causes a connection-level error (`403 Forbidden` with reason `ExpiredProviderToken` or connection refused for .p12 certs). The first send attempt after expiry will fail.
- Alert on: push delivery success rate dropping below 95% over a 5-minute window.
- Proactive: set a calendar reminder / automated alert 30 days before certificate expiration. APNs certificates are valid for 1 year.

Recovery:
- Rotate to a new .p12 certificate or regenerate a JWT signing key. Config is hot-reloadable if you store it in a secrets manager (AWS Secrets Manager, Vault).
- Workers should detect connection errors, back off briefly, reload credentials, and reconnect — not crash permanently.
- Failed messages during the outage window: they're in the retry queue with exponential backoff, so they'll be retried automatically once credentials are valid again (up to the 1hr window).

Post-incident:
- Set up automated cert rotation with 30-day pre-expiry alerts.
- Switch from .p12 certificates to JWT-based auth (Provider Token) — JWT signing keys never expire automatically.

---

## Q2: "We need to guarantee every critical notification (fraud alert, OTP) is delivered at least once. How do you guarantee that?"

**What they're testing:** Delivery guarantees, at-least-once semantics, trade-offs with exactly-once.

**Strong answer:**

At-least-once delivery requires:

1. **Durable enqueue:** Write the notification to Kafka (replicated, durable). Commit the offset only after the delivery provider accepts it.
2. **Idempotent dedup key:** Since at-least-once can cause duplicates, OTP codes include a server-generated token in the SMS body. If the same code is re-delivered before expiry, the user ignores the duplicate. For fraud alerts the body is identical — receiving it twice is fine.
3. **Retry with persistence:** Track delivery state in DB (pending/sent/failed). Workers query DB for pending notifications on startup and re-attempt them. Redis retry queue complements this.
4. **Acknowledgement before Kafka offset commit:** Worker processes → sends to Twilio/APNs → waits for 200 OK → then commits Kafka offset. If the worker crashes after sending but before committing, the message is retried — hence at-least-once, not exactly-once.
5. **SMS fallback for push:** For critical notifications: send push AND SMS simultaneously (or push first → if no delivery receipt within 30s → fallback to SMS). APNs doesn't provide delivery receipts, so SMS adds the guaranteed delivery backstop.

Exactly-once is hard because external providers (APNs, Twilio) don't support idempotent keys. At-least-once with dedup logic is the realistic target.

---

## Q3: "Explain how you would send a breaking news alert to 100M users simultaneously within 30 seconds."

**What they're testing:** Fan-out at scale, bottleneck identification, burst capacity.

**Strong answer:**

100M notifications in 30 seconds = **3.3M notifications/second**. This is the "thundering herd" for notifications.

Bottlenecks (and solutions):

1. **Fan-out throughput:** Can't generate 100M Kafka messages from a single producer instantly. Solution: Use a "broadcast" topic with a single message containing the alert content + eligible audience segment. Consumer workers each pull a partition range of users and generate their own notification records. This distributes fan-out across N workers.

2. **APNs throughput:** APNs supports HTTP/2 multiplexing; you can maintain 1,000 in-flight requests per connection. With 100 APNs connections, that's 100K concurrent requests. At 5ms per request: ~20M pushes/sec maximum. You need a pool of workers with many connections.

3. **FCM:** Use FCM Topic Messaging or FCM Multicast (up to 500 tokens per batch request). Send bulk batches rather than individual API calls.

4. **Email:** Email to 100M users in 30s is unrealistic. Email is inherently slower (30-60min delivery). Use push for immediacy, email as a follow-up.

5. **Rate limits:** APNs enforces per-app limits. For true breaking news scale (BBC, Twitter), negotiate dedicated APNs throughput with Apple. In practice, systems like Twitter use APNs "cloud push" with 100+ delivery shards.

6. **Precompute tokens:** Don't query device_tokens table at fan-out time. Maintain a pre-sharded snapshot (e.g., S3 files partitioned by user_id range). Workers stream tokens from S3 → batch APNs/FCM requests.

**Architecture:**
```
Admin triggers alert → Kafka "broadcast" topic (1 message)
→ 500 fan-out workers, each responsible for user_id range shard
→ Each worker: reads device tokens from S3 shard → batches 500 FCM tokens per request
→ Parallel APNs connections per worker (100 in-flight)
→ Total: 500 workers × 200K pushes/sec each = 100M in ~1 second
```

---

## Q4: "A user updates their push preferences to opt out of marketing notifications. How long until they stop receiving them, and how do you ensure they're not sent during the propagation window?"

**What they're testing:** Cache invalidation, eventual consistency, user trust.

**Strong answer:**

Write path:
- User updates preference → write to `notification_preferences` DB → also invalidate/update Redis cache key `prefs:{user_id}` with TTL.

Propagation delay sources:
- Redis cache TTL: if you cache preferences for 5 minutes, user may still receive notifications for up to 5 minutes. Solution: on explicit preference update, actively delete the cache key (not just let it expire): `DEL prefs:{user_id}`.
- In-flight notifications already in Kafka: messages already produced before the preference update will be processed. This window is typically seconds (Kafka lag), and there's little you can do about it.

Real-world target: user should stop receiving marketing notifications within < 30 seconds of toggling the setting.

Implementation:
1. Preference write → DB transaction + `DEL prefs:{user_id}` synchronously.
2. Workers always read preferences at dispatch time (not at fan-out time). This means fan-out workers produce to Kafka (where we don't check prefs), but delivery workers check prefs when they dequeue. Late opt-out is caught at delivery.
3. For near-zero false positives: add preference check at both fan-out AND delivery stages.

User trust angle: Show "Changes take effect immediately" in the UI. Accept that a tiny window (< 30s) of in-flight messages is technically breaking that promise, but practically acceptable. Log preference changes with timestamps for auditability.

---

## Q5: "Describe the notification ordering problem and how you'd address it."

**What they're testing:** Distributed ordering, Kafka partition semantics, user-visible artifacts.

**Strong answer:**

**Problem:** Two events happen in quick succession:
- t=0ms: "Alice liked your post" → worker A processes it
- t=50ms: "Alice and Bob liked your post" (aggregated after Bob likes) → worker B processes it

If worker B finishes first (network jitter), the user sees "Alice and Bob liked your post" first, then "Alice liked your post" — confusing ordering.

**Why it happens:**
- Kafka partitions can distribute events to different workers
- Workers have different processing times
- Network latency to APNs is non-deterministic

**Solutions:**

1. **Partition by user_id:** Ensure all notifications for the same user go to the same Kafka partition (use user_id as the partition key). Same partition → same consumer → sequential processing per user. Cost: one slow notification for user X blocks all of user X's notifications.

2. **Client-side ordering by timestamp:** Include a server-generated timestamp in the notification payload. Mobile app UI sorts notifications by this timestamp, not arrival order. Rendering is delayed 1s to allow out-of-order messages to arrive.

3. **Aggregation window collapses the problem:** If you aggregate "likes" over a 5-minute window, only one notification is ever sent for that batch. The ordering problem mostly disappears for social notifications.

4. **Sequence number:** Include a monotonically increasing notification sequence number per user. Client discards notifications with lower sequence number than already-displayed ones.

For most production systems, the aggregation approach (solution 3) eliminates the ordering problem naturally. Partition-by-user-id (solution 1) is the right call when precise ordering is required (e.g., chat message notifications must be ordered).
