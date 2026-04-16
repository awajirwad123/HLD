# Idempotency & Exactly-Once Semantics — Hands-On Exercises

## Exercise 1: Idempotency Key Middleware (FastAPI)

Build a full FastAPI idempotency middleware: store request/response pairs in Redis, prevent concurrent duplicates, return cached responses for retries.

```python
"""
Goal: Implement idempotency key handling as FastAPI middleware.
      Requirements:
      - Check Redis for existing response on idempotency key
      - Acquire a lock to prevent concurrent duplicate processing
      - Store the response after successful processing (TTL=24h)
      - Return 409 Conflict if the same key is in-flight on another request
"""

import json
import uuid
import asyncio
from typing import Optional
from fastapi import FastAPI, Request, Response, HTTPException
from pydantic import BaseModel
import redis.asyncio as aioredis

app = FastAPI()
redis_client: Optional[aioredis.Redis] = None

IDEM_TTL = 86400    # 24 hours — same as Stripe's policy
LOCK_TTL = 30       # 30 seconds — max expected request processing time


async def lifespan(app: FastAPI):
    global redis_client
    redis_client = await aioredis.from_url("redis://localhost:6379", decode_responses=True)
    yield
    await redis_client.aclose()

app = FastAPI(lifespan=lifespan)


class IdempotencyMiddlewareHelper:
    """Helper encapsulating idempotency logic."""

    def __init__(self, redis: aioredis.Redis):
        self.redis = redis

    async def get_cached_response(self, key: str) -> Optional[dict]:
        data = await self.redis.get(f"idem:{key}")
        return json.loads(data) if data else None

    async def acquire_lock(self, key: str) -> bool:
        """Returns True if lock acquired, False if already locked."""
        return await self.redis.set(
            f"idem_lock:{key}", "1", nx=True, ex=LOCK_TTL
        )

    async def release_lock(self, key: str) -> None:
        await self.redis.delete(f"idem_lock:{key}")

    async def cache_response(self, key: str, status_code: int, body: dict) -> None:
        payload = {"status_code": status_code, "body": body}
        await self.redis.setex(f"idem:{key}", IDEM_TTL, json.dumps(payload))


# ─── Dependency ───────────────────────────────────────────────────────────────

async def handle_idempotency(request: Request) -> Optional[dict]:
    """
    FastAPI dependency. Returns cached response if exists,
    None if the request should be processed fresh.
    Raises 409 if in-flight.
    Raises 400 if key format is invalid.
    """
    idem_key = request.headers.get("Idempotency-Key")
    if not idem_key:
        return None   # No idempotency key — process normally (idempotent GET/DELETE)

    # Validate key is a valid UUID to prevent injection / abuse
    try:
        uuid.UUID(idem_key)
    except ValueError:
        raise HTTPException(status_code=400, detail="Idempotency-Key must be a UUID")

    helper = IdempotencyMiddlewareHelper(redis_client)

    # 1. Check for existing cached response
    cached = await helper.get_cached_response(idem_key)
    if cached:
        request.state.idem_cached = cached
        return cached

    # 2. Try to acquire lock for in-flight protection
    acquired = await helper.acquire_lock(idem_key)
    if not acquired:
        raise HTTPException(
            status_code=409,
            detail="A request with this Idempotency-Key is already being processed. "
                   "Wait for it to complete before retrying."
        )

    # Store lock key in request state so endpoint can release it
    request.state.idem_key = idem_key
    request.state.idem_cached = None
    request.state.idem_helper = helper
    return None


# ─── Domain model ─────────────────────────────────────────────────────────────

class ChargeRequest(BaseModel):
    amount: int          # in cents
    currency: str
    customer_id: str


class ChargeResponse(BaseModel):
    charge_id: str
    amount: int
    currency: str
    status: str


async def create_charge_in_db(req: ChargeRequest) -> ChargeResponse:
    """Simulate payment processor call."""
    await asyncio.sleep(0.05)   # simulate latency
    return ChargeResponse(
        charge_id=f"ch_{uuid.uuid4().hex[:16]}",
        amount=req.amount,
        currency=req.currency,
        status="captured",
    )


# ─── Endpoint ─────────────────────────────────────────────────────────────────

@app.post("/v1/charges", response_model=ChargeResponse)
async def create_charge(
    body: ChargeRequest,
    request: Request,
    cached: Optional[dict] = None,   # from handle_idempotency dependency
):
    # If cached response exists (set by dependency), return it directly
    if not hasattr(request.state, "idem_cached"):
        await handle_idempotency(request)

    if request.state.idem_cached:
        cached_data = request.state.idem_cached
        return Response(
            content=json.dumps(cached_data["body"]),
            status_code=cached_data["status_code"],
            media_type="application/json",
        )

    idem_key = getattr(request.state, "idem_key", None)
    helper = getattr(request.state, "idem_helper", None)

    try:
        result = await create_charge_in_db(body)
        response_body = result.model_dump()

        # Cache the successful response
        if idem_key and helper:
            await helper.cache_response(idem_key, 200, response_body)

        return result

    except Exception:
        # On failure: release the lock so the client can retry
        if idem_key and helper:
            await helper.release_lock(idem_key)
        raise


# ─── Test the behavior ────────────────────────────────────────────────────────

"""
Run with: uvicorn solution:app --reload

Test with httpx:

import httpx, uuid
idem_key = str(uuid.uuid4())
headers = {"Idempotency-Key": idem_key}
body = {"amount": 5000, "currency": "usd", "customer_id": "cus_123"}

r1 = httpx.post("http://localhost:8000/v1/charges", json=body, headers=headers)
print(r1.json())   # {"charge_id": "ch_abc123...", "amount": 5000, ...}

r2 = httpx.post("http://localhost:8000/v1/charges", json=body, headers=headers)
print(r2.json())   # SAME charge_id — no new charge created

# Different key = different charge
r3 = httpx.post("http://localhost:8000/v1/charges", json=body,
                 headers={"Idempotency-Key": str(uuid.uuid4())})
print(r3.json())   # New charge_id
"""
```

---

## Exercise 2: Database-Level Deduplication

Implement exactly-once processing of payment events using PostgreSQL `ON CONFLICT DO NOTHING` and optimistic locking:

```python
"""
Goal: Process incoming payment events exactly once.
      Use: (1) event_id deduplication, (2) account balance update with version lock.
      Simulate concurrent duplicate events and verify no double-processing.
"""

import asyncio
import uuid
from dataclasses import dataclass
import asyncpg


@dataclass
class PaymentEvent:
    event_id: str
    account_id: str
    amount: int     # negative = debit
    description: str


# ─── Schema ───────────────────────────────────────────────────────────────────
SCHEMA = """
CREATE TABLE IF NOT EXISTS accounts (
    id      TEXT PRIMARY KEY,
    balance INT  NOT NULL DEFAULT 0,
    version INT  NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS processed_events (
    event_id     TEXT PRIMARY KEY,
    account_id   TEXT NOT NULL,
    amount       INT  NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
"""


# ─── Core processing function ─────────────────────────────────────────────────

async def process_payment_event(
    conn: asyncpg.Connection,
    event: PaymentEvent,
) -> bool:
    """
    Process a payment event exactly once.
    Returns True if processed, False if duplicate (already processed).
    """
    async with conn.transaction():
        # 1. Attempt to record this event (idempotency guard)
        # ON CONFLICT DO NOTHING means duplicates silently skip
        inserted = await conn.fetchval(
            """
            INSERT INTO processed_events (event_id, account_id, amount)
            VALUES ($1, $2, $3)
            ON CONFLICT (event_id) DO NOTHING
            RETURNING event_id
            """,
            event.event_id, event.account_id, event.amount
        )

        if inserted is None:
            # Duplicate event — already processed
            print(f"  [SKIP] {event.event_id} already processed")
            return False

        # 2. Apply the balance change with optimistic lock
        max_retries = 3
        for attempt in range(max_retries):
            row = await conn.fetchrow(
                "SELECT balance, version FROM accounts WHERE id = $1 FOR UPDATE",
                event.account_id
            )
            if not row:
                raise ValueError(f"Account {event.account_id} not found")

            new_balance = row["balance"] + event.amount
            if new_balance < 0:
                raise ValueError(f"Insufficient funds: balance={row['balance']}, debit={event.amount}")

            updated = await conn.fetchval(
                """
                UPDATE accounts
                SET balance = $1, version = version + 1
                WHERE id = $2 AND version = $3
                RETURNING version
                """,
                new_balance, event.account_id, row["version"]
            )

            if updated is not None:
                print(f"  [OK]   {event.event_id}: balance {row['balance']} → {new_balance} (v{updated})")
                return True

            # Version mismatch — concurrent update, retry within the same transaction
            if attempt == max_retries - 1:
                raise RuntimeError("Max optimistic lock retries exceeded")
            await asyncio.sleep(0.01)

    return False


# ─── Simulation ───────────────────────────────────────────────────────────────

async def main():
    conn = await asyncpg.connect("postgresql://postgres:password@localhost/test")
    await conn.execute(SCHEMA)

    # Seed an account
    await conn.execute(
        "INSERT INTO accounts (id, balance) VALUES ('acc-1', 10000) "
        "ON CONFLICT DO NOTHING"
    )

    # Create a payment event
    event = PaymentEvent(
        event_id=str(uuid.uuid4()),
        account_id="acc-1",
        amount=-2000,      # debit $20
        description="Coffee subscription"
    )

    print("=== First processing (should succeed) ===")
    await process_payment_event(conn, event)

    print("\n=== Duplicate processing (should be skipped) ===")
    await process_payment_event(conn, event)

    print("\n=== Another duplicate (retry simulation) ===")
    await process_payment_event(conn, event)

    balance = await conn.fetchval("SELECT balance FROM accounts WHERE id = 'acc-1'")
    print(f"\nFinal balance: {balance} (expected 8000 — only one $20 debit applied)")

    await conn.close()


asyncio.run(main())
```

---

## Exercise 3: Kafka Exactly-Once Producer + Idempotent Consumer

Produce payment events to Kafka exactly-once using transactions, then consume with deduplication:

```python
"""
Goal: Demonstrate Kafka exactly-once semantics:
  Producer: enable_idempotence=True + transactional_id
  Consumer: read_committed isolation + PostgreSQL event dedup table
"""

import asyncio
import json
import uuid
from kafka import KafkaProducer, KafkaConsumer
from kafka.errors import KafkaError


# ─── Exactly-once Producer ────────────────────────────────────────────────────

def create_idempotent_producer() -> KafkaProducer:
    """
    enable_idempotence: Kafka broker deduplicates retries within a session.
                         Uses sequence numbers per partition.
    transactional_id:   Groups a batch of sends + offset-commit into one atomic unit.
                         Unique per producer instance (restart with same ID = resume).
    acks='all':          Required by enable_idempotence — all ISR replicas must ack.
    """
    return KafkaProducer(
        bootstrap_servers=["localhost:9092"],
        enable_idempotence=True,
        transactional_id="payment-service-producer-1",
        acks="all",
        retries=5,
        value_serializer=lambda v: json.dumps(v).encode("utf-8"),
        key_serializer=lambda k: k.encode("utf-8") if k else None,
    )


def publish_payment_event(
    producer: KafkaProducer,
    order_id: str,
    amount: int,
    account_id: str,
) -> None:
    """
    Publish to 'payments' and 'audit_log' topics atomically.
    Either both messages land in Kafka, or neither do.
    """
    event_id = str(uuid.uuid4())
    payment_payload = {
        "event_id": event_id,
        "order_id": order_id,
        "account_id": account_id,
        "amount": amount,
        "event_type": "payment.captured",
    }
    audit_payload = {
        "event_id": event_id,
        "order_id": order_id,
        "action": "payment_captured",
        "amount": amount,
    }

    producer.init_transactions()

    try:
        producer.begin_transaction()

        producer.send(
            "payments",
            key=order_id,
            value=payment_payload,
        )
        producer.send(
            "audit_log",
            key=order_id,
            value=audit_payload,
        )

        producer.commit_transaction()
        print(f"[PRODUCER] Transaction committed: order={order_id}, event={event_id}")

    except KafkaError as e:
        print(f"[PRODUCER] Transaction aborted: {e}")
        producer.abort_transaction()
        raise


# ─── Idempotent Consumer ──────────────────────────────────────────────────────

# NOTE: Uses a simple in-memory set for demo. In production: PostgreSQL processed_events table.

processed_event_ids: set[str] = set()


def process_message(record) -> None:
    """
    Consumer with deduplication.
    isolation.level='read_committed' ensures we only see committed messages
    (aborted transaction messages are filtered by the broker).
    """
    payload = json.loads(record.value)
    event_id = payload.get("event_id")

    if not event_id:
        print(f"[CONSUMER] Message missing event_id — skipping invalid message")
        return

    if event_id in processed_event_ids:
        print(f"[CONSUMER] Duplicate {event_id} — skipping")
        return

    # Process the business logic
    print(f"[CONSUMER] Processing {event_id}: "
          f"order={payload['order_id']}, amount={payload['amount']}")

    # In production: write to DB and insert into processed_events
    # atomically within the same DB transaction

    # Mark as processed (in production: INSERT INTO processed_events)
    processed_event_ids.add(event_id)


def run_consumer():
    consumer = KafkaConsumer(
        "payments",
        bootstrap_servers=["localhost:9092"],
        group_id="payment-processor",
        auto_offset_reset="earliest",
        enable_auto_commit=False,        # Manual commit — control exactly when offset advances
        isolation_level="read_committed",  # Only read messages from committed transactions
        value_deserializer=lambda v: v,    # Raw bytes — decode in process_message
    )

    print("[CONSUMER] Starting...")
    try:
        for record in consumer:
            process_message(record)
            consumer.commit()   # Commit offset after successful processing
    finally:
        consumer.close()


# ─── Demo: simulate duplicate delivery ────────────────────────────────────────

def demo_deduplication():
    producer = create_idempotent_producer()

    order_id = f"order-{uuid.uuid4().hex[:8]}"
    print(f"\n=== Publishing order {order_id} ===")
    publish_payment_event(producer, order_id, 5000, "acc-1")

    print(f"\n=== Producer retry (same message — idempotent) ===")
    # simulate exactly once: even if producer dies and retries, broker deduplicates
    # using the sequence number embedded in the transactional batch

    producer.close()

    # Consumer deduplication: even if Kafka delivers the same message twice
    # (e.g., consumer crashed after processing but before commit), the
    # processed_event_ids check prevents double-processing
    print(f"\n=== Consumer processing same event twice ===")
    fake_record = type("Record", (), {
        "value": json.dumps({
            "event_id": "fixed-event-id-001",
            "order_id": order_id,
            "amount": 5000
        }).encode()
    })()

    process_message(fake_record)
    process_message(fake_record)   # duplicate — should be skipped
```

---

## Exercise 4: Stripe-Style Client SDK with Auto-Retry

Build a client library that generates stable idempotency keys and retries safely on network errors:

```python
"""
Goal: A payment SDK client that:
  1. Generates a deterministic idempotency key from the operation's logical identity
  2. Retries automatically on network/timeout errors using the SAME key
  3. Does NOT retry on 4xx errors (user errors) or on success
"""

import asyncio
import hashlib
import json
import uuid
from enum import Enum
import httpx


class RetryableError(Exception):
    pass

class NonRetryableError(Exception):
    pass


def make_idempotency_key(user_id: str, cart_id: str, cart_version: int) -> str:
    """
    Deterministic key: same cart state → same key → safe to retry without user action.
    If user changes cart (cart_version increments), a new key is generated.
    This is the "intentional change = new attempt" rule.
    """
    raw = f"{user_id}:{cart_id}:{cart_version}"
    return hashlib.sha256(raw.encode()).hexdigest()[:32]


async def charge_with_retry(
    client: httpx.AsyncClient,
    user_id: str,
    cart_id: str,
    cart_version: int,
    amount: int,
    currency: str = "usd",
    max_retries: int = 3,
) -> dict:
    """
    Charge a customer, retrying transparently on network errors.
    The idempotency key ensures retries are safe.
    """
    idem_key = make_idempotency_key(user_id, cart_id, cart_version)
    base_delay = 0.5   # seconds

    for attempt in range(max_retries + 1):
        try:
            response = await client.post(
                "/v1/charges",
                json={"amount": amount, "currency": currency, "customer_id": user_id},
                headers={"Idempotency-Key": idem_key},
                timeout=10.0,
            )

            if response.status_code == 200:
                return response.json()

            if response.status_code == 409:
                # In-flight duplicate — wait and retry (the first request is still processing)
                delay = base_delay * (2 ** attempt)
                await asyncio.sleep(delay)
                continue

            if 400 <= response.status_code < 500 and response.status_code != 409:
                # e.g., 400 Bad Request, 422 Unprocessable Entity
                # Don't retry — user error, fix the request
                raise NonRetryableError(
                    f"HTTP {response.status_code}: {response.json()}"
                )

            if response.status_code >= 500:
                if attempt < max_retries:
                    delay = base_delay * (2 ** attempt)
                    print(f"  Attempt {attempt+1} failed (HTTP {response.status_code}), "
                          f"retrying in {delay:.1f}s with same key {idem_key[:8]}...")
                    await asyncio.sleep(delay)
                    continue
                raise RetryableError(f"Server error after {max_retries} retries")

        except httpx.TimeoutException:
            if attempt < max_retries:
                delay = base_delay * (2 ** attempt)
                print(f"  Attempt {attempt+1} timed out, retrying in {delay:.1f}s "
                      f"with same key {idem_key[:8]}...")
                await asyncio.sleep(delay)
                continue
            raise RetryableError("Timed out after all retries")

        except httpx.NetworkError:
            if attempt < max_retries:
                delay = base_delay * (2 ** attempt)
                await asyncio.sleep(delay)
                continue
            raise RetryableError("Network error after all retries")

    raise RetryableError("Max retries exceeded")


# ─── Demo ─────────────────────────────────────────────────────────────────────

async def demo():
    # This key is deterministic: same user + same cart state = same key
    key_v1 = make_idempotency_key("user-123", "cart-456", cart_version=1)
    key_v2 = make_idempotency_key("user-123", "cart-456", cart_version=2)

    print(f"Cart v1 key: {key_v1}")
    print(f"Cart v2 key: {key_v2}")
    print(f"Keys are different (cart changed): {key_v1 != key_v2}")
    # True — user updated cart, new logical operation, new key required

    # Same cart = same key (safe retries on timeout):
    key_retry = make_idempotency_key("user-123", "cart-456", cart_version=1)
    print(f"Retry key matches v1: {key_retry == key_v1}")
    # True — retry with same key → server deduplicates → exactly one charge

asyncio.run(demo())
```

---

## Exercise 5: Outbox Pattern — Guaranteed-Once Event Publishing

Prevent the split-brain between DB write and Kafka publish:

```python
"""
Goal: Use the Transactional Outbox pattern to guarantee that:
 - If the DB write succeeds, the event WILL eventually be published to Kafka
 - If the DB write fails, NO event is published
 - Events are never published twice (publisher marks them published)

Components:
 1. Order service: writes order + outbox entry in one transaction
 2. Outbox poller: reads unpublished outbox entries, publishes to Kafka, marks published
"""

import asyncio
import json
import uuid
from datetime import datetime, timezone
import asyncpg
from kafka import KafkaProducer


# ─── Schema ───────────────────────────────────────────────────────────────────
SCHEMA = """
CREATE TABLE IF NOT EXISTS orders (
    id          TEXT PRIMARY KEY,
    customer_id TEXT NOT NULL,
    amount      INT  NOT NULL,
    status      TEXT NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS outbox (
    id          BIGSERIAL PRIMARY KEY,
    event_id    TEXT        NOT NULL UNIQUE,
    event_type  TEXT        NOT NULL,
    payload     JSONB       NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    published   BOOLEAN     NOT NULL DEFAULT FALSE
);

CREATE INDEX IF NOT EXISTS outbox_unpublished
    ON outbox (published, id)
    WHERE published = FALSE;
"""


# ─── Order service ────────────────────────────────────────────────────────────

async def create_order(
    conn: asyncpg.Connection,
    customer_id: str,
    amount: int,
) -> str:
    """
    Write order + outbox entry atomically.
    If this transaction commits, the event WILL be published.
    If it rolls back (e.g., constraint violation), no event is published.
    """
    order_id = str(uuid.uuid4())
    event_id = str(uuid.uuid4())

    async with conn.transaction():
        await conn.execute(
            "INSERT INTO orders (id, customer_id, amount, status) "
            "VALUES ($1, $2, $3, 'pending')",
            order_id, customer_id, amount
        )
        await conn.execute(
            "INSERT INTO outbox (event_id, event_type, payload) "
            "VALUES ($1, $2, $3)",
            event_id,
            "order.created",
            json.dumps({"order_id": order_id, "customer_id": customer_id, "amount": amount})
        )

    print(f"[ORDER] Created {order_id} with outbox event {event_id}")
    return order_id


# ─── Outbox poller ────────────────────────────────────────────────────────────

async def poll_and_publish(
    conn: asyncpg.Connection,
    producer: KafkaProducer,
    batch_size: int = 100,
) -> int:
    """
    Read unpublished outbox entries, publish to Kafka, mark as published.
    Returns number of events published.
    """
    rows = await conn.fetch(
        """
        SELECT id, event_id, event_type, payload
        FROM outbox
        WHERE published = FALSE
        ORDER BY id
        LIMIT $1
        FOR UPDATE SKIP LOCKED
        """,
        batch_size
    )

    if not rows:
        return 0

    published_count = 0
    for row in rows:
        try:
            # Publish to Kafka
            future = producer.send(
                row["event_type"].replace(".", "-"),   # topic name
                key=row["event_id"].encode(),
                value=json.dumps({
                    "event_id": row["event_id"],
                    "event_type": row["event_type"],
                    "payload": json.loads(row["payload"]),
                }).encode()
            )
            future.get(timeout=5)   # wait for broker ack

            # Mark as published
            await conn.execute(
                "UPDATE outbox SET published = TRUE WHERE id = $1",
                row["id"]
            )
            published_count += 1
            print(f"[POLLER] Published {row['event_id']}")

        except Exception as e:
            print(f"[POLLER] Failed to publish {row['event_id']}: {e}")
            # Leave as unpublished — will be retried on next poll cycle
            # Kafka send is idempotent (enable_idempotence=True on producer)
            # so retrying the same event_id is safe

    return published_count


async def run_poller(conn: asyncpg.Connection, producer: KafkaProducer):
    """Continuous polling loop — runs as a background task."""
    print("[POLLER] Starting outbox poller...")
    while True:
        try:
            count = await poll_and_publish(conn, producer)
            if count == 0:
                await asyncio.sleep(1)   # nothing to publish, back off
        except Exception as e:
            print(f"[POLLER] Error: {e}")
            await asyncio.sleep(5)


# ─── Key insight ──────────────────────────────────────────────────────────────
"""
Why this is "guaranteed once":

1. If the app crashes after writing order+outbox but before the poller runs:
   → Poller picks up the unpublished entry on next startup → event published ✓

2. If the poller publishes to Kafka but crashes before marking published:
   → Poller retries on restart → Kafka receives duplicate message
   → Consumer must deduplicate on event_id (see Exercise 3)
   → Net result: exactly-once at the consumer level

3. If the order transaction rolls back:
   → No outbox entry → no event published ✓

This is the "at-least-once delivery + idempotent consumer = exactly-once processing" pattern.
"""
```
