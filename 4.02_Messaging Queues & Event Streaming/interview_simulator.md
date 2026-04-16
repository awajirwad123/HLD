# Messaging Queues & Event Streaming — Interview Simulator

**Instructions:** Set a 35-minute timer per scenario. Attempt completely before reading guidance. A senior interviewer expects: broker selection justification, delivery guarantee awareness, DLQ design, backpressure handling, and throughput estimation.

---

## Scenario 1: Design the Event Pipeline for an E-Commerce Order System

**Prompt:**
> "You're building an e-commerce platform. When a user places an order, the following must happen: (1) payment is charged, (2) inventory is decremented, (3) a confirmation email is sent, (4) the order appears in the analytics dashboard within 60 seconds. All four steps run in separate microservices. Design the messaging architecture: which broker, which guarantees, how failures are handled, and how you'd scale to 50,000 orders per minute at peak."

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "Payment service is down for 3 minutes. What happens to the messages during that window and after it recovers?"
2. "The confirmation email fails 5% of the time due to a flaky SMTP provider. How do you handle this without blocking order processing?"
3. "A bug in the inventory service causes it to throw an exception for orders containing product ID `PROMO-001`. There are 10,000 such messages in the queue. What do you do?"
4. "The analytics service only needs 60-second freshness, not real-time. Does that change your design?"

---

### Guidance (Read After Attempting)

**Broker choice: Kafka**
Why: multiple independent services need the same `order.created` event (payment, inventory, email, analytics each consume independently), replay is valuable (replay to analytics after a bug fix), and 50K orders/min = ~833/sec is well within Kafka's capacity.

**Topic Design:**
```
Topic: order.created        — 12 partitions, replication-factor=3
  Partition key: order_id (distribute evenly; no ordering requirement across orders)

Topic: payment.processed    — 6 partitions
Topic: inventory.reserved   — 6 partitions

Each service has its own consumer group:
  - payment-service-group
  - inventory-service-group
  - notification-service-group
  - analytics-service-group
```

**Delivery guarantee per service:**
- Payment: at-least-once + idempotent (dedup by order_id before charging)
- Inventory: at-least-once + idempotent (`UPDATE SET stock=stock-1 WHERE stock>0 AND NOT EXISTS (dedup)`)
- Email: at-least-once → if email sent twice = acceptable; if too noisy, idempotency key per order
- Analytics: at-least-once is fine; duplicate event = slightly overcounted → correct on aggregation

**Payment service down for 3 minutes:**
Kafka retains messages indefinitely during the outage — consumer group lag grows. When service recovers, it resumes at its last committed offset and processes the backlog. This is the key advantage of Kafka retention over SQS.

**Email DLQ:**
```
notification-service: max 3 retries with exponential backoff (1s, 2s, 4s)
After 3 failures → publish to order.created.notification.DLT
Alert fires: PagerDuty notification
Ops retries DLQ messages after SMTP provider recovers
```

**Poison pill (PROMO-001 bug):**
1. Immediately identify via DLQ lag spike + error logs
2. Pause the inventory consumer group to stop further poisoning
3. Fix the bug in inventory service
4. Deploy fix
5. Reset consumer group offset to before the first PROMO-001 message
6. Resume — all 10,000 messages re-processed with fixed code

**Analytics 60-second SLA:**
Analytics consumer can use `max_poll_records=5000` and process in bulk rather than one-by-one. At 833 events/sec, the consumer has 60 seconds of buffer — as long as processing completes within 60s, SLA is met. This also enables bulk inserts (cheaper than row-by-row in a data warehouse).

**Scale to 50K orders/min:**
- 12 partitions → 12 parallel consumers per group
- 833 events/sec → each consumer ~70 events/sec — very manageable
- If throughput doubles: increase to 24 partitions, scale consumer instances accordingly

---

## Scenario 2: Notification System for a Social Platform at Twitter Scale

**Prompt:**
> "Design the messaging architecture for a social notification system: when user A follows, likes, or comments on user B's post, user B receives a push notification. The platform has 500M users; at peak (major news event), 10M notifications per minute need to be generated and delivered. Some users have 50M followers (celebrities). Walking me through broker selection, fan-out strategy, delivery guarantees, and DLQ design."

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "A celebrity posts and 50M followers each need a notification. How do you fan this out without a single queue becoming a bottleneck?"
2. "Push notifications to iOS/Android fail 30% of the time silently. How do you handle this?"
3. "A user has 250 unread notifications. How do you prevent notification storm when they open the app?"
4. "At peak, the delivery service is processing 150K notifications/sec. How do you prevent the push delivery API (APNs/FCM) from being overwhelmed?"

---

### Guidance (Read After Attempting)

**Broker: Kafka for fan-out events, SQS for delivery tasks**

Two-stage architecture:
```
Stage 1 (Kafka): Social event fan-out
  social.events topic → fan-out worker explodes one event into N notifications

Stage 2 (SQS): Push delivery per notification
  notifications-delivery queue → push workers call APNs/FCM
```

**Celebrity fan-out problem (50M followers):**
A single `post.created` event for a celebrity cannot fan out to 50M SQS messages synchronously. Solutions:

Option A — Pull model hybrid: celebrities' followers are notified lazily. When a follower opens the app, the server queries recent posts from followed celebrities (pull) instead of pre-computed push. Only non-celebrity fan-out is push.

Option B — Batch fan-out workers: the fan-out worker reads follower shards (follower list split into pages of 10K), creates batches of notifications, and produces them to Kafka partitioned by `user_id`. Downstream delivery workers consume per-partition.

```
post.created (celebrity_id) → fan-out-worker
  → reads follower shard 1 (users 1–10K) → produces 10K notifications to 'notifications' topic
  → reads follower shard 2 (users 10K–20K) → produces 10K notifications ...
  ... × 5000 shards = 50M notifications, spread across 1–5 minutes
```

**APNs/FCM rate limit (backpressure):**
- APNs: 1M notifications/sec total per app certificate
- Rate limit consumer using `max_poll_records` per push worker instance
- Multiple push worker instances each consume separate partitions, stay below APNs limits
- Per-user-device rate limit: max 1 push/5 seconds per device (application-level dedup)

**Push failure handling:**
APNs/FCM return error codes:
- `DeviceTokenNotForTopic` / `Unregistered`: device unregistered → delete token from DB, don't retry
- `ServiceUnavailable` / `InternalServerError`: transient → retry with backoff
- After 3 retries: move to DLQ; mark notification as "undelivered" in DB

**Notification storm prevention:**
Don't push all 250 unread notifications at once. On app open: server sends a single "You have 247 new notifications" badge update, not 247 individual pushes. Push is for real-time interruption; historical notifications are fetched via API.

**Delivery guarantee:**
At-least-once is the correct choice. Duplicate pushes are annoying but rare (retry-only). Missing pushes (at-most-once) are worse — user is never notified. With idempotent delivery keys per notification, at-least-once + idempotent push service ≈ effectively exactly-once.

---

## Scenario 3: Real-Time Fraud Detection Pipeline

**Prompt:**
> "You're designing a real-time fraud detection system for a payment processor. Every payment event must be analyzed within 100ms and flagged if fraudulent. The system processes 2 million transactions per minute. Walking me through: event ingestion, stream processing, alerting on detected fraud, and what happens when the fraud detection ML model is updated and needs to reprocess historical data."

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "The fraud detection model takes 80ms per transaction. With 2M transactions per minute (~33K/sec), how many model instances do you need?"
2. "A new fraud pattern is detected. You need to re-run the model on the past 30 days of transactions. How do you do this without affecting the live pipeline?"
3. "The fraud model returns a score > 0.95 with false positives 2% of the time. How do you avoid blocking legitimate payments?"
4. "Your Kafka cluster goes down for 2 minutes. What happens to fraud detection during that window?"

---

### Guidance (Read After Attempting)

**Architecture:**

```
Payment Gateway → Kafka topic: transactions (20 partitions, acks=all)
                                  │
                        ┌─────────┴──────────┐
                        ▓ Fraud Detection     ▓  ← consumer group: fraud-scorer
                        ▓ Workers (N instances)▓  80ms ML model per transaction
                        └─────────┬──────────┘
                                  │
               ┌──────────────────┼───────────────────┐
               ▼                  ▼                    ▼
        Kafka topic:      Redis (flagged            Payment
        fraud.alerts      transaction IDs)          Gateway
        (alert pipeline)  TTL=24h                   (block/allow)
```

**Throughput math:**
- 2M transactions/min ÷ 60 = ~33,333/sec
- Each model call = 80ms → one instance handles 12.5 transactions/sec
- Instances needed: 33,333 / 12.5 = **2,667 instances** minimum
- In practice: target 70% utilization → 3,800 instances OR batch scoring (group 10 txns per model call → 10× throughput per instance → 380 instances)
- Better: GPU-accelerated batch scoring, 1,000 txns/batch/100ms → 10,000/sec/GPU → 4 GPUs

**Historical reprocessing without affecting live pipeline:**
Create a separate consumer group for the reprocessing job:
```
consumer_group="fraud-scorer-backfill-2024-q1"
auto.offset.reset="earliest"
```
Kafka retention must cover 30 days (`retention.ms=2592000000`). The backfill consumer reads historical offsets independently; the live consumer group is unaffected (different group = different offsets).

Key: set backfill consumer `max_poll_records` lower and introduce artificial sleep (`time.sleep(0.01)`) to prevent it from consuming 100% of broker I/O, throttling the live pipeline.

**False positive handling (2% false positive rate):**
- Don't hard-block at 0.95; soft-flag instead
- Score 0.95–0.99: add step-up authentication (require 3DS / SMS OTP) — user can still complete with extra verification
- Score > 0.99: block and require manual review
- Log all blocked transactions with model version + feature vector for review board

**Kafka outage (2-minute window):**
- Payments during outage: depends on architecture
  - Option A: fail open — allow payments, no fraud check (revenue continues, fraud risk accepted)
  - Option B: fail closed — reject all payments (no fraud, but revenue loss too)
  - Option C: local cache of known fraud patterns (top 1000 stolen card hashes in Redis) — crude but maintains basic protection
- After recovery: Kafka consumer resumes at last committed offset — all transactions during outage are fraud-scored (retroactively), flagged cards reversed if fraud confirmed

**Delivery guarantee:**
`acks=all` on producer — no transaction may be lost from the fraud pipeline. At-least-once on consumer — duplicate scoring is acceptable (same transaction scored twice = same decision, idempotent). Exactly-once is not needed here.
