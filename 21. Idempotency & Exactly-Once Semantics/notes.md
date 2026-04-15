# Idempotency & Exactly-Once Semantics — Notes & Reference Sheet

## 1. Core Vocabulary

| Term | Definition |
|---|---|
| **Idempotent operation** | Calling N times produces the same result as calling once. No additional side effects on repeats. |
| **Safe operation** | Causes no state change (read-only). All safe operations are also idempotent. |
| **At-most-once** | 0 or 1 executions. May lose data. Used where retries are dangerous and data loss is acceptable. |
| **At-least-once** | 1 or more executions. May duplicate. Requires idempotent consumer to be safe. |
| **Exactly-once** | Exactly 1 execution effect regardless of failures. Requires idempotency + deduplication. |
| **Idempotency key** | Client-generated unique ID (UUID) sent with each request. Server uses it to deduplicate. |
| **Outbox pattern** | Write DB record + event in one transaction. Separate poller publishes event to broker. |

---

## 2. HTTP Method Idempotency

```
GET    — safe + idempotent    (reading never changes state)
HEAD   — safe + idempotent
PUT    — idempotent only      (full replacement: same body → same result)
DELETE — idempotent only      (deleting twice → same final state: resource gone)
POST   — neither by default   (creates new resource each time)
PATCH  — depends on content   (PATCH amount=5 → idempotent; PATCH amount+=1 → NOT)
```

**Rule of thumb:** Make POST idempotent via idempotency key. Make PATCH use absolute values, not relative increments.

---

## 3. Idempotency Key Pattern

```
Client ──► Server POST /charges
          Header: Idempotency-Key: <uuid>

Server logic:
  1. Check Redis: GET idem:<key>
     → Hit:  return cached response (no re-processing)
     → Miss: continue

  2. Try: SET idem_lock:<key> 1 NX EX 30
     → Fail: return 409 (in-flight duplicate)
     → OK:   continue

  3. Process request
  4. SETEX idem:<key> 86400 <response>
  5. Return response
```

**TTL choices:**
- Lock TTL: max expected processing time + buffer (e.g., 30s)
- Response cache TTL: 24 hours (Stripe policy); 7 days for failed charges (Stripe)
- Error responses are cached too — retry with same key returns same error

**Key generation:**
- Random UUID: new per user action (most common)
- Deterministic: `SHA256(user_id + cart_id + cart_version)[:32]` — allows safe auto-retry without user interaction

---

## 4. Stripe's Model (Canonical Reference)

| Property | Value |
|---|---|
| Header name | `Idempotency-Key` |
| TTL | 24 hours |
| Scope | `(api_key, idempotency_key)` — per-account |
| On error | Stores error response; retry with same key returns same error |
| Different body same key | Returns 422 — body mismatch detected |
| Concurrent duplicate | Returns 409 — in-flight |

---

## 5. Delivery Guarantees Comparison

| Guarantee | Description | How to achieve | Kafka config |
|---|---|---|---|
| At-most-once | 0 or 1 | Don't retry; fire-and-forget | `acks=0`, `retries=0` |
| At-least-once | 1 or more | Retry + idempotent consumer | `acks=all`, `retries=MAX_INT`, consumer reads all msgs |
| Exactly-once (producer) | No duplicate messages in broker | Idempotent producer | `enable_idempotence=True` |
| Exactly-once (end-to-end) | 1 execution effect | Kafka transactions + idempotent consumer | `transactional_id=...` + `isolation_level=read_committed` + consumer dedup |

---

## 6. Kafka Exactly-Once Quick Reference

```python
# Producer
producer = KafkaProducer(
    enable_idempotence=True,           # sequence numbers per partition
    transactional_id="svc-prod-1",    # unique per producer instance
    acks="all",
)
producer.init_transactions()

with producer.transaction():
    producer.send("topic-a", ...)
    producer.send("topic-b", ...)
    producer.send_offsets_to_transaction(offsets, consumer_group)
    # Commit = atomic: both sends + offset advance happen together

# Consumer
consumer = KafkaConsumer(
    isolation_level="read_committed",  # Skip aborted-transaction messages
    enable_auto_commit=False,          # Manual commit for control
)
```

**Two layers:**
1. `enable_idempotence` — broker-level dedup of producer retries (within session)
2. `transactional_id` — multi-partition atomic commit (cross-partition + offset)

---

## 7. Database Deduplication Patterns

### INSERT ... ON CONFLICT DO NOTHING
```sql
INSERT INTO processed_events (event_id, ...)
VALUES ($1, ...)
ON CONFLICT (event_id) DO NOTHING
RETURNING event_id;
-- NULL return means duplicate — skip business logic
```

### Idempotent Upsert
```sql
INSERT INTO payments (order_id, amount, status)
VALUES ($1, $2, 'pending')
ON CONFLICT (order_id) DO UPDATE
  SET status = EXCLUDED.status;
-- Safe to call multiple times with same order_id
```

### Optimistic Lock (CAS)
```sql
UPDATE accounts
SET balance = $1, version = version + 1
WHERE id = $2 AND version = $3;
-- 0 rows updated = concurrent modification detected → retry
```

---

## 8. Outbox Pattern

```
┌───────────────────────────────────────────────────────────┐
│  Application transaction (atomic)                          │
│                                                           │
│  INSERT INTO orders (...)       ← business write         │
│  INSERT INTO outbox (event_id, payload, published=false)  │
└───────────────────────────────────────────────────────────┘
                          │
                 Commits atomically
                          │
                          ▼
         Outbox poller (separate process/thread):
         SELECT * FROM outbox WHERE published = FALSE
                          │
                          ▼
              Publish to Kafka / SQS / SNS
                          │
                          ▼
         UPDATE outbox SET published = TRUE
```

**Guarantees:** If DB transaction commits → event eventually published (at-least-once). Consumer must deduplicate.

**Polling options:**
- Direct DB polling every 1–5 seconds (simple, slight latency)
- Debezium CDC: tails PostgreSQL WAL → near-real-time, no DB polling overhead

---

## 9. Partial Failure Cheat Sheet

| Scenario | With Idempotency Key | Without |
|---|---|---|
| Request lost before server receives | Retry → fresh processing (key not in cache) | Retry → OK |
| Server processes, response lost | Retry → cached response returned, no re-charge | Retry → **duplicate** |
| Server timeout (unknown if processed) | Retry with same key → safe | Retry → **duplicate** |
| Server crashes mid-process (no DB commit) | Key never cached → retry processes fresh | Retry → OK |
| Duplicate Kafka message delivered | Consumer checks processed_events table | Consumer processes twice |

---

## 10. Key Numbers to Memorize

| Metric | Value |
|---|---|
| Stripe idempotency key TTL | 24 hours |
| SQS FIFO dedup window | 5 minutes |
| Kafka sequence number scope | Per (producer, partition) |
| Kafka transaction ID scope | Per producer instance (restart = resume) |
| Redis SET NX (lock) complexity | O(1) |
| Optimistic lock retry typical max | 3 attempts |
| UUID format | `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx` (UUID v4) |

---

## 11. Decision Guide

```
Should I use idempotency keys?
  └─ Is the operation non-idempotent by nature (POST, charge, debit)?
       ├─ YES: Add idempotency key support
       └─ NO: GET/PUT/DELETE — already safe

Where should I generate the key?
  └─ Client generates it (server cannot — it doesn't know the client's intent)

Deterministic or random key?
  ├─ Deterministic: when you want app-level auto-retry without user interaction
  │                 (same logical operation = same key = safe retries)
  └─ Random: when each button click = new operation, no auto-retry needed

At-least-once or exactly-once for Kafka?
  ├─ At-least-once + idempotent consumer = most common, simpler
  └─ Kafka transactions: only when multi-topic atomicity is required
```
