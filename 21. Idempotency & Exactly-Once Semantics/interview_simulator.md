# Idempotency & Exactly-Once Semantics — Interview Simulator

Three complete mock interview scenarios. Read the prompt, close the answer, design for 10–15 minutes, then compare.

---

## Scenario 1: Design a Payment Processing API (Stripe-Like)

**Interviewer prompt:**

> "Design the backend for a payment API. Merchants use POST /charges to charge a customer's card. Requirements: (1) a charge must never happen twice for the same user intent, (2) the API must be safe to retry on network failures, (3) the system must handle 10,000 charges per second. Walk me through your idempotency design."

---

### Strong Answer

**Framing the problem:**

The critical constraint is: a user's card must never be charged twice for a single purchase. The danger is the "lost response" scenario: merchant server submits a charge, our payment API processes it, response is lost in transit. Merchant retries. Without idempotency → double charge.

**Client contract:**

Every POST /charges request must include `Idempotency-Key: <uuid>`. The client generates a UUID per checkout intent and reuses it for all retries of that specific checkout. Different checkout = new UUID.

**Server idempotency design:**

```
POST /charges
Idempotency-Key: idem-uuid-here

1. Validate Idempotency-Key format (UUID)
2. Check Redis: GET idem:{key}
   → Hit:  return cached response (HTTP 200 or cached error) → done
3. Try SET idem_lock:{key} NX EX 30
   → Fail: return 409 (concurrent duplicate in-flight)
4. Charge the card via payment processor (Stripe → Adyen)
5. Write to DB: INSERT INTO charges (charge_id, order_id, idempotency_key, amount, status)
               ON CONFLICT (idempotency_key) DO NOTHING ← belt-and-suspenders
6. Cache: SETEX idem:{key} 86400 {response_json}
7. Return 200 with charge object
On failure at step 4/5: DELETE idem_lock:{key} (release so client can retry)
```

**Two deduplication layers:**

- **Redis (fast path):** TTL=24h, catches retries within the user session
- **DB UNIQUE(idempotency_key):** Permanent guard — catches retries after Redis TTL expires, data race where Redis dies between GET (miss) and SETEX

**Scale: 10,000 charges/second:**

- Redis handles GET + SETEX at >500k ops/sec easily; 10k/s is trivial
- DB: charges table is append-only; with proper indexing (`UNIQUE(idempotency_key)`, shard by `merchant_id`) PostgreSQL handles 10k writes/sec with connection pooling (PgBouncer, 100 connections per node)
- If payment processor rate limits: queue excess via a work queue (SQS FIFO or Kafka), process asynchronously, return 202 Accepted with a charge ID; client polls `GET /charges/{charge_id}` for status

**Error response caching:**

A key subtlety: even error responses are cached. If the card is declined (402), the cached `{status: "declined"}` is returned for retries with the same key. This prevents the merchant from accidentally re-pinging a declined card in a loop. The merchant must generate a new key if they want to retry with a corrected card.

**What interviewers probe:**
- "What if the payment processor times out?" → We don't know if the charge went through. Release the Redis lock, return 500 to the client. Client retries with same key. Our retry to the processor should also be idempotent — most payment processors accept a `reference_id` that deduplicates on their side.
- "What if Redis is down?" → Degrade gracefully: skip Redis check, fall through to DB `ON CONFLICT DO NOTHING`. Performance degrades (every request hits DB) but correctness is maintained by the DB UNIQUE constraint.

---

## Scenario 2: Event-Driven Order Processing (Exactly-Once Kafka)

**Interviewer prompt:**

> "You have three services: Order Service, Inventory Service, and Notification Service. A user places an order → Order Service writes to DB and publishes `order.created` to Kafka → Inventory Service deducts stock → Notification Service sends confirmation email. How do you ensure the stock is only deducted once and the email is only sent once, even under failures?"

---

### Strong Answer

**Identify the failure modes first:**

1. Order Service writes to DB but crashes before publishing → event lost (order created, no downstream processing)
2. Inventory Service processes the event but crashes before committing offset → message redelivered → stock deducted twice
3. Notification Service sends email but crashes before committing offset → email sent twice

**Solution layer by layer:**

**Order Service — Outbox Pattern:**

Never write to DB + publish to Kafka in the same code path (they can't be made atomic).  

```python
async with db.transaction():
    order_id = await db.write_order(customer_id, items, amount)
    await db.execute(
        "INSERT INTO outbox (event_id, event_type, payload) VALUES ($1, $2, $3)",
        str(uuid.uuid4()), "order.created",
        json.dumps({"order_id": order_id, "items": items, "amount": amount})
    )
```

Outbox poller publishes unpublished entries to Kafka and marks them `published=true`. Delivery is at-least-once — the poller may send duplicates on crash-restart. Consumer deduplicates.

**Kafka Consumer Config — Isolation:**

```python
consumer = KafkaConsumer(
    group_id="inventory-service",
    enable_auto_commit=False,
    isolation_level="read_committed",   # Filter aborted Kafka transactions
)
```

**Inventory Service — DB Deduplication:**

```python
async def handle_order_created(event):
    async with db.transaction():
        inserted = await db.fetchval(
            "INSERT INTO processed_events (event_id) VALUES ($1) "
            "ON CONFLICT DO NOTHING RETURNING event_id",
            event["event_id"]
        )
        if not inserted:
            return   # Duplicate — skip

        # Deduct stock with optimistic lock
        for item in event["items"]:
            updated = await db.execute(
                "UPDATE inventory SET qty = qty - $1 "
                "WHERE product_id = $2 AND qty >= $1",
                item["qty"], item["product_id"]
            )
            if updated == 0:
                raise InsufficientStockError(item["product_id"])

    consumer.commit()  # Advance offset AFTER successful DB commit
```

The `processed_events` insert and stock deduction are in the same transaction — atomically either both succeed or neither does. No "processed but offset not committed" split.

**Notification Service — Email Deduplication:**

```python
async def handle_order_created_email(event):
    # Guard: record intent to send before sending
    inserted = await db.fetchval(
        "INSERT INTO sent_emails (event_id, email_type, recipient) "
        "VALUES ($1, 'order_confirmation', $2) ON CONFLICT DO NOTHING RETURNING event_id",
        event["event_id"], event["customer_email"]
    )
    if not inserted:
        consumer.commit()
        return   # Already sent

    # Send (outside transaction — can't roll back an email)
    await send_email(event["customer_email"], event["order_id"])
    await db.execute("UPDATE sent_emails SET sent_at = now() WHERE event_id = $1", event["event_id"])
    consumer.commit()
```

**Accept limited email duplicate risk:** If process crashes between `send_email` and `UPDATE sent_emails SET sent_at`, the next retry sees `sent_at IS NULL` (not fully committed) and may resend. One option: consider `INSERT ... RETURNING event_id` as the permanent dedup guard — if it already exists, never send regardless of `sent_at`. Accept the rare duplicate confirmation email as benign.

**Summary of exactly-once contract:**
- Order: exactly-once write (DB write is the source of truth)
- Inventory: exactly-once stock deduction (DB dedup on event_id)
- Email: at-most-once-ish (DB dedup on event_id, rare edge-case duplicate)

---

## Scenario 3: Distributed Payment in a Ride-Sharing App (Uber-Like)

**Interviewer prompt:**

> "When a ride completes, Uber charges the rider and pays the driver. This involves: (1) charge the rider's stored payment method, (2) create a 'trip completed' event, (3) calculate and schedule the driver payout. How do you make this exactly-once? What happens if the payment step partially succeeds?"

---

### Strong Answer

**State machine first:**

A trip is not "charged" or "not charged" — it has states:

```
trip_completed → payment_initiated → payment_captured → payout_scheduled → done
                                   ↘ payment_failed    → retry_or_manual_review
```

The system should be able to resume from any state on crash/retry.

**Part 1: Charging the rider**

The trip is the natural idempotency key: `trip_id` uniquely identifies one charge attempt. The DB has `UNIQUE(trip_id)` on the payments table.

```python
# Idempotent payment initiation
async def initiate_payment(trip_id: str, rider_id: str, amount: int):
    existing = await db.fetchrow(
        "SELECT charge_id, status FROM payments WHERE trip_id = $1", trip_id
    )
    if existing:
        return existing   # Already initiated — return current status

    # Call payment processor with trip_id as reference (deduplication on their side)
    charge = await payment_processor.charge(
        reference_id=trip_id,    # Processor deduplicates on this
        customer_id=rider_id,
        amount=amount,
    )
    await db.execute(
        "INSERT INTO payments (trip_id, charge_id, amount, status) "
        "VALUES ($1, $2, $3, $4) ON CONFLICT (trip_id) DO NOTHING",
        trip_id, charge.id, amount, charge.status
    )
    return charge
```

**Key:** Pass `trip_id` as `reference_id` to the payment processor. Stripe/Adyen accept a client reference and deduplicate: if they've already seen this `reference_id`, they return the existing charge — even if our server timed out and we're re-calling.

**Part 2: Exactly-once payout trigger**

Once the charge is captured, schedule the driver payout:

```python
# Outbox pattern for payout event
async def mark_payment_captured(trip_id: str, charge_id: str):
    async with db.transaction():
        await db.execute(
            "UPDATE payments SET status = 'captured' WHERE trip_id = $1", trip_id
        )
        await db.execute(
            "INSERT INTO outbox (event_id, event_type, payload) "
            "VALUES ($1, 'payment.captured', $2) ON CONFLICT (event_id) DO NOTHING",
            f"payout-trigger-{trip_id}",   # deterministic event_id
            json.dumps({"trip_id": trip_id, "charge_id": charge_id})
        )
```

Deterministic `event_id` (`"payout-trigger-{trip_id}"`) — if the outbox insert is retried (on transaction retry), it's a no-op due to `ON CONFLICT DO NOTHING`.

**Part 3: Payout scheduler consumer**

```python
async def handle_payment_captured(event):
    trip_id = event["payload"]["trip_id"]

    # Idempotency guard
    inserted = await db.fetchval(
        "INSERT INTO scheduled_payouts (trip_id) VALUES ($1) "
        "ON CONFLICT DO NOTHING RETURNING trip_id",
        trip_id
    )
    if not inserted:
        consumer.commit()
        return   # Already scheduled

    # Calculate driver earnings (fare - commission)
    earnings = await calculate_driver_earnings(trip_id)
    await payout_service.schedule(driver_id=trip.driver_id, amount=earnings)
    consumer.commit()
```

**Part 4: Handling payment processor timeout (partial success unknown)**

This is the hardest case: we called the payment processor, got a timeout, don't know if it succeeded.

1. Query the processor with `GET /charges?reference_id=trip_id` — processors support this idempotent lookup
2. If charge exists: use it, update local DB to match
3. If not: retry the charge with the same `reference_id` — processor deduplicates

This "check before retry" pattern is critical. Never assume "timeout = failure" for external payment calls.

**What interviewers look for:**
- Passing `trip_id` as `reference_id` to the payment processor (not just handling our own DB idempotency)
- State machine for trip payment — not a single operation but a multi-step recoverable flow
- Distinguishing "idempotency key TTL expired" from "permanent charge record" — the DB is truth, the Redis cache is an optimization
- Handling the timeout/unknown-state case — probing the processor before retrying
