# Idempotency & Exactly-Once Semantics — Architecture & Concepts

## 1. The Core Problem

In distributed systems, **partial failures** are inevitable:

```
Client ──► Server: POST /charges   (request sent)
Server:   processes charge successfully
Server ──► Client: [network drops]  (response lost)
Client:   receives timeout → did the charge happen or not?
```

The client cannot distinguish between:
- "Server never received the request" → safe to retry
- "Server processed the request but response was lost" → retry = double charge

**Idempotency** resolves this: retrying the same operation produces the same result with no additional side effects, regardless of how many times it's executed.

---

## 2. HTTP Method Idempotency

| Method | Idempotent? | Safe (read-only)? | Notes |
|---|---|---|---|
| GET | ✓ Yes | ✓ Yes | Reading never changes state |
| HEAD | ✓ Yes | ✓ Yes | Same as GET, no body |
| PUT | ✓ Yes | No | Full replacement — same result if called N times |
| DELETE | ✓ Yes | No | Deleting already-deleted resource = 404 or 204 (same final state) |
| POST | **No** by default | No | Creates new resource each time unless idempotency key used |
| PATCH | **No** by default | No | `PATCH counter += 1` is not idempotent; `PATCH counter = 5` is |

**Key distinction:** "safe" means no state change. "idempotent" means N calls = 1 call in terms of final state (but may have side effects).

---

## 3. Idempotency Keys

The standard pattern for making non-idempotent operations idempotent:

**Client generates a unique key** (UUID) for each logical operation and sends it with every attempt:

```
POST /v1/charges
Idempotency-Key: idem-a1b2c3d4-e5f6-7890-abcd-ef1234567890
Content-Type: application/json

{"amount": 5000, "currency": "usd", "customer": "cus_123"}
```

**Server logic:**
```python
async def charge(request, idempotency_key: str):
    # 1. Check if this key was already processed
    cached = await redis.get(f"idem:{idempotency_key}")
    if cached:
        return json.loads(cached)   # Return identical response — no re-processing

    # 2. Acquire distributed lock to prevent concurrent duplicate processing
    lock = await redis.set(f"idem_lock:{idempotency_key}", "1", nx=True, ex=30)
    if not lock:
        raise ConflictError("Request with this key is already in progress")

    try:
        # 3. Process the actual operation
        result = await process_charge(request)

        # 4. Store the response (TTL: 24 hours per Stripe's policy)
        await redis.setex(f"idem:{idempotency_key}", 86400, json.dumps(result))
        return result

    except Exception:
        await redis.delete(f"idem_lock:{idempotency_key}")
        raise
```

**Properties:**
- Same key → same response, even if the server state changed after initial processing
- Key expiry (24h) balances storage cost vs retry window
- Lock prevents concurrent duplicate submissions (TOCTOU race)

---

## 4. Generating Idempotency Keys

**Client side — deterministic generation:**

```python
import uuid, hashlib

# Option 1: Random UUID (most common)
key = str(uuid.uuid4())   # Different for each user action

# Option 2: Deterministic / content-based
# Derived from the operation's logical identity: same cart = same key
key = hashlib.sha256(f"{user_id}:{cart_id}:{cart_version}".encode()).hexdigest()[:32]
```

**Deterministic keys** enable "safe retries with no button click" — the app can automatically retry the charge on reconnect without asking the user, knowing the same key produces the same result.

**Rule:** The key must change when the user intentionally changes the operation. If the user updates the cart and re-clicks "pay," generate a new key.

---

## 5. Exactly-Once Semantics

"Exactly-once" means an operation has its effect exactly once, regardless of failures and retries. It's formally harder than idempotency and depends on the layer:

### At the API Layer (HTTP)
Achieved via idempotency keys + server-side deduplication. "Exactly-once" at API level is really "at-most-once processing + at-least-once delivery = exactly-once effect."

### At the Messaging Layer (Kafka)
True exactly-once requires transactions:

```python
producer = KafkaProducer(
    bootstrap_servers=...,
    enable_idempotence=True,          # Per-partition, per-session dedup
    transactional_id="payment-svc-1"  # Cross-partition + consumer atomicity
)

producer.init_transactions()

with producer.transaction():
    producer.send("payments", key=order_id, value=payload)
    producer.send("audit_log", key=order_id, value=audit_entry)
    producer.send_offsets_to_transaction(offsets, consumer_group)
    # Commit makes both publishes + offset advance atomic
```

**Two guarantees:**
1. `enable_idempotence=True`: no duplicate messages on producer retry (sequence numbers per partition)
2. `transactional_id`: makes multi-partition publishes + consumer offset commit atomic — all-or-nothing

**Consumer must use `isolation.level=read_committed`** to only see committed (non-aborted) transaction messages.

### At the Database Layer
```sql
-- Idempotent upsert — safe to run multiple times
INSERT INTO payments (payment_id, amount, status)
VALUES ($1, $2, 'captured')
ON CONFLICT (payment_id) DO NOTHING;

-- Or: idempotent update with version guard
UPDATE accounts
SET balance = balance - $amount
WHERE account_id = $id
  AND version    = $expected_version;   -- Optimistic lock
```

---

## 6. Deduplication Strategies

### 6.1 Client-Side Idempotency Key (Primary Strategy)

Client generates UUID per operation; server deduplicates on it. Covered in §3.

### 6.2 Natural Key Deduplication

Some operations have a natural unique constraint:

```sql
-- One payment per order: order_id is the natural key
INSERT INTO payments (order_id, amount, status)
VALUES ($1, $2, 'pending')
ON CONFLICT (order_id) DO UPDATE SET status = EXCLUDED.status;
```

### 6.3 Message Deduplication in Queues

**SQS FIFO:** Built-in deduplication within a 5-minute window using `MessageDeduplicationId`:

```python
sqs.send_message(
    QueueUrl="...",
    MessageBody=json.dumps(payload),
    MessageDeduplicationId=str(uuid.uuid4()),  # Same ID = duplicate dropped
    MessageGroupId="user-group-1",
)
```

**Kafka:** No built-in consumer-side deduplication — must implement in the consumer:

```python
# Consumer: check processed_events table before processing
async def process(record):
    exists = await db.fetchval(
        "SELECT 1 FROM processed_events WHERE event_id = $1", record.key
    )
    if exists:
        return   # Already processed — skip

    async with db.transaction():
        await handle_business_logic(record)
        await db.execute(
            "INSERT INTO processed_events (event_id, processed_at) VALUES ($1, now())",
            record.key
        )
```

### 6.4 Outbox Pattern (Guaranteed-Once Publish)

Prevent the race between "write to DB" and "publish to Kafka":

```python
async with db.transaction():
    await db.execute("INSERT INTO orders ...")
    await db.execute(
        "INSERT INTO outbox (event_id, event_type, payload) VALUES ($1, $2, $3)",
        event_id, "order.created", json.dumps(payload)
    )
# Outbox poller publishes to Kafka and marks as published
# Even if crash between DB commit and Kafka publish, poller recovers
```

---

## 7. Stripe's Idempotency Model (Reference Implementation)

Stripe is the canonical real-world example:

1. Client generates `Idempotency-Key: <uuid>` header on every write request
2. Stripe stores `(api_key, idempotency_key) → response` for 24 hours
3. If the same key arrives again: return the stored response without re-processing
4. Stripe's retry library (in their SDK) automatically retries with the same key on network errors
5. Key is tied to the API key — different accounts cannot share/collide keys

**Edge case — what if the operation failed?**
- Operation threw an error → Stripe stores the error response too
- Retry with same key returns the same error (not a new attempt)
- User must use a new key to retry intentionally after fixing the cause

---

## 8. Optimistic Locking (Version-Based Idempotency)

Prevents double-execution when multiple services or threads race:

```python
# Read with version
row = await db.fetchrow("SELECT balance, version FROM accounts WHERE id = $1", account_id)

# Apply with version check (optimistic lock)
updated = await db.execute("""
    UPDATE accounts
    SET balance  = $1,
        version  = version + 1
    WHERE id      = $2
      AND version = $3
""", new_balance, account_id, row["version"])

if updated == 0:
    raise ConcurrentModificationError("Account was modified concurrently — retry")
```

If two concurrent requests both read version=5 and try to update: only one succeeds (CAS — Compare And Swap). The loser retries by re-reading — ensuring exactly one update per logical operation.

---

## 9. Partial Failure Scenarios and How Idempotency Handles Them

| Scenario | Without Idempotency | With Idempotency Key |
|---|---|---|
| Request sent, network drops before server receives | Client retries → OK (never processed) | Same — idempotency key not yet in cache |
| Server processes, response lost in transit | Client retries → **double charge** | Client retries with same key → server returns cached response, no re-charge |
| Server processes, client times out waiting | Client retries → **double charge** | Client retries with same key → cached response returned |
| Server crashes mid-process, DB write never committed | Client retries → OK (idempotency key not stored because transaction rolled back) | Same — key never entered cache |

---

## 10. Idempotency vs. Exactly-Once vs. At-Least-Once

| Guarantee | Meaning | How achieved |
|---|---|---|
| **At-most-once** | 0 or 1 executions — may lose messages | Don't retry; Kafka `acks=0` |
| **At-least-once** | 1 or more executions — may duplicate | Retry with idempotent consumer |
| **Exactly-once** | Exactly 1 execution | Idempotency key on API; Kafka transactions + idempotent consumer |

"Exactly-once" at the message/event layer still requires an idempotent consumer. Even Kafka's `transactional` producer guarantees no duplicate *messages*; if the consumer throws an exception after processing but before committing the offset, the message is redelivered — the consumer must still deduplicate.
