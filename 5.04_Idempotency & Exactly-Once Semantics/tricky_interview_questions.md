# Idempotency & Exactly-Once Semantics — Tricky Interview Questions

Questions targeting the precise gaps that separate candidates who "know about idempotency" from those who've reasoned through the edge cases.

---

## Question 1: "Your idempotency key is cached in Redis with a 24-hour TTL. A user submits a payment, gets a timeout, retries — and the second request returns the cached charge. Three days later, the user disputes the charge and asks you to refund it. But when you look up the idempotency key in Redis, it's gone. How does your system handle this?"

**The trap:** Candidates assume idempotency is the durability mechanism. It's not.

**Strong answer:**

The idempotency cache is a short-lived deduplication layer — it protects against network-error retries within a session (minutes to 24 hours). It is explicitly not the record of truth for the charge.

The charge itself was persisted to the payments database when first processed — `INSERT INTO charges (charge_id, customer_id, amount, status)` — and that record survives the Redis TTL expiry. Refunds are processed against the `charge_id`, not the `idempotency_key`. The customer service team looks up the charge by `charge_id`, confirms it in the DB, and issues a refund regardless of whether the idempotency key still exists in Redis.

The Redis TTL is intentionally set to 24 hours because: (a) clients should not need to retry a request from 3 days ago, and (b) storing idempotency responses forever would require unbounded storage. A legitimate retry within 24 hours is deduped by Redis. A legitimate retry after 24 hours would be a programming error in the client — it should use a new key.

**The real invariant:** The original charge is durable in the payment database. Idempotency keys protect the write path. Dispute resolution uses the payment database directly.

---

## Question 2: "Exactly-once delivery in Kafka requires `enable_idempotence=True` + `transactional_id`. But you said the consumer ALSO needs to deduplicate. If Kafka guarantees exactly-once, why does the consumer need to do anything?"

**The trap:** Conflating broker-level exactly-once with application-level exactly-once.

**Strong answer:**

Kafka's exactly-once guarantees operate at the broker/storage level: no duplicate messages are committed to a topic's log. With `transactional_id`, publishes across multiple partitions are atomic — either all land or none do.

But exactly-once *processing* in the consumer is a different contract. Consider:

1. Consumer reads a message
2. Consumer calls `process_order()` → updates the database
3. Consumer commits the Kafka offset
4. **Crash between steps 2 and 3**

On restart, the consumer re-reads the message from the last committed offset (before step 2). The database was already updated in step 2. Without consumer-side deduplication, the order is processed twice.

Kafka's guarantee is: the message exists in the topic exactly once. It says nothing about *what happens in your application code when the consumer reads it*. The consumer is responsible for making its processing idempotent — `ON CONFLICT DO NOTHING` in the DB, or checking a `processed_events` table.

"Exactly-once semantics" in Kafka documentation means: no producer duplicates in the broker log + atomic batch commits. It does not mean: your consumer function runs exactly once on every message.

---

## Question 3: "You implement the Outbox pattern. The outbox poller publishes an event to Kafka, then crashes before it marks the entry as `published=true` in the DB. The poller restarts and publishes the same event again. This is a duplicate. Haven't you broken exactly-once with the Outbox pattern?"

**The trap:** Believing the Outbox pattern provides exactly-once end-to-end by itself.

**Strong answer:**

The Outbox pattern provides **at-least-once** delivery from the database to the message broker — not exactly-once. The scenario you describe is expected and correct: the poller will re-deliver the event. This is intentional.

The system achieves exactly-once processing at the consumer level:

1. Each event in the outbox has a unique `event_id` (UUID)
2. The Kafka consumer checks: `SELECT 1 FROM processed_events WHERE event_id = $1`
3. If found: skip (already processed)
4. If not found: process + `INSERT INTO processed_events` atomically in the same DB transaction
5. Commit Kafka offset

The duplicate event from the poller retry hits step 2 and is silently skipped. Net result: the business logic runs exactly once despite the at-least-once delivery.

This is the fundamental distributed systems recipe: **at-least-once delivery + idempotent consumer = exactly-once processing**. The Outbox ensures no events are *lost*. The consumer's deduplication ensures no events are *double-processed*.

---

## Question 4: "Two servers received the same `POST /charges` request at the same exact millisecond (load balancer split brain). Both servers do a Redis `GET idem:<key>` — both get a miss. Now both acquire a lock with `SET NX idem_lock:<key>`. Who wins, and what happens to the loser?"

**The trap:** Checking whether "both got a miss before the lock" breaks the system.

**Strong answer:**

`SET key value NX EX ttl` in Redis is a single atomic command — the Redis server processes it serially. Even if two requests arrive microseconds apart, only one can win the `SET NX`:

- **Server A wins:** `SET NX` returns OK → proceeds to process the charge
- **Server B loses:** `SET NX` returns nil (key already set) → returns HTTP 409 Conflict: "A request with this key is already being processed. Retry after completion."

The client SDK receives the 409 and retries after a short delay (~1–2 seconds). By then, Server A has finished, stored the response in Redis, and released or expired the lock. The retry hits the cached response and returns it without re-processing.

**Edge case — Server A crashes while processing:**
- The lock TTL (e.g., 30s) expires
- Retry arrives, acquires lock fresh, processes the request

If Server A had already written to the payment database before crashing, the retry would create a *logical* duplicate. This is why the payment processing itself must also be idempotent at the DB level: `INSERT INTO charges ... ON CONFLICT (idempotency_key) DO NOTHING`. The idempotency key is stored permanently in the `charges` table — even after Redis TTL expires — as the definitive deduplication record.

---

## Question 5: "Engineer says: 'We generate a new UUID for each payment attempt in the frontend. The user panics and clicks Pay rapidly 5 times. We now have 5 different idempotency keys for what should be one charge. Isn't idempotency useless here?'"

**The trap:** Conflating idempotency (retry deduplication) with duplicate action prevention (UX/domain logic).

**Strong answer:**

Idempotency protects against *network retries of the same operation*. It was never designed to protect against a user clicking a button 5 times — that's a different problem: **duplicate submission prevention**.

They are two orthogonal layers:

**1. UI-level prevention (frontend):**
- Disable the Pay button after the first click
- Show a loading spinner
- Once the response arrives (success or error), re-enable

**2. Domain-level prevention (backend):**
- The charge table has a UNIQUE constraint on `order_id` (or `cart_id + cart_version`):
  `INSERT INTO charges (order_id, ...) ON CONFLICT (order_id) DO NOTHING`
- Even if 5 requests with 5 different idempotency keys arrive, only one charge for `order_id=X` commits

**3. Idempotency keys (retry deduplication):**
- If one of the 5 requests times out and the SDK retries it with the same UUID, the server returns the same response without creating a 6th charge

The three layers address different failure modes. Using deterministic/content-based keys (SDK generates `SHA256(user_id + order_id + amount)` rather than random UUID per click) would collapse all 5 clicks into one idempotency key — but this only works when the logical operation is unambiguously identifiable before the request is sent.

---

## Question 6: "Interviewer: 'The payment service publishes a `charge.captured` event to Kafka. The billing service consumes it, sends a receipt email, and updates the invoice. You said the consumer needs to be idempotent. But sending an email is a side effect that can't be `ON CONFLICT DO NOTHING` in a database. How do you make email sending idempotent?'"

**The trap:** Candidates who know DB deduplication but haven't thought about non-DB side effects.

**Strong answer:**

Email is the classic non-idempotent external side effect. The pattern is:

**Record before acting:**
```python
async with db.transaction():
    inserted = await db.fetchval(
        "INSERT INTO sent_emails (event_id, email_type, recipient) "
        "VALUES ($1, $2, $3) ON CONFLICT (event_id) DO NOTHING RETURNING event_id",
        event_id, "receipt", customer_email
    )
    if inserted is None:
        return  # Already sent this email — skip

    # Mark as "pending send"
    # (Don't send inside the transaction — email send can't be rolled back)

# Outside transaction: send the email
await send_email(to=customer_email, subject="Your receipt", body=receipt_html)

# Mark as delivered
await db.execute(
    "UPDATE sent_emails SET sent_at = now() WHERE event_id = $1", event_id
)
```

**Why outside the transaction:** Email sending can't be rolled back. If it's inside the transaction and the DB commit fails after the email sends, the email was sent but the DB row wasn't written — future retries would send again.

**The residual risk:** If the process crashes between sending the email and writing `sent_at`, the next retry will attempt to send again. The `ON CONFLICT DO NOTHING` on `(event_id)` prevents the insert → skips the send. So the `sent_emails` INSERT acts as the deduplication guard even though the email itself is external.

**Accept, don't fight:** For truly external services (email, SMS, webhooks), idempotency is best-effort. Some systems accept a small probability of duplicate email delivery (e.g., charge receipts) rather than building elaborate distributed locks. The goal is: minimize duplicates to "rare" (< 0.01%) and make them benign (a duplicate receipt is annoying, not a double charge).

---

## Question 7: "Stripe accepts an idempotency key, then the charge succeeds. The client code has a bug and immediately calls the same endpoint again with the SAME idempotency key but the user intended a second, separate charge. The server returns the cached first charge. How does Stripe know the user didn't intend a new charge?"

**The trap:** Argues that idempotency keys can mask genuine duplicate user intent.

**Strong answer:**

Stripe explicitly does not know — it cannot distinguish "client retry after timeout" from "client bug submitting intentional second charge with wrong key." Stripe's model is: **same key = same operation**. It's the client's responsibility to use a new key for a new logical operation.

This is by design — the guarantee is: "if you retry with the same key, you will never be double-charged." The client takes on the responsibility: "if I want a second separate charge, I must use a different key."

In practice, well-designed clients:
1. Generate the key from the logical operation's identity: `SHA256(user_id + order_id + cart_version)` — the key changes when cart_version changes (user explicitly checks out again)
2. Or generate a random UUID once per checkout session, store it locally, and reuse it only for automated retries of that specific checkout

The "bug" scenario you describe (same key, intended different charge) is a client programming error. Stripe's documentation explicitly warns: "if you want to retry, use the same key; if you want a new charge, use a new key." Stripe validates that the request body hash matches the original for the same key — which would catch "different charge amount, same key" — but not "same body, second intended charge."

The conclusion: idempotency keys are a contract between the client and server. The server enforces the mechanic; the client is responsible for the semantics.
