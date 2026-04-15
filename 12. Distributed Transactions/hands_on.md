# Distributed Transactions — Hands-On Exercises

## Exercise 1: Simulate 2PC — Coordinator and Participants

```python
"""
Simulates 2-Phase Commit with coordinator and participant nodes.
Demonstrates: happy path, participant failure, coordinator crash after VOTE-YES.
"""
import time
import random
import threading
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional


class Vote(Enum):
    YES = "YES"
    NO  = "NO"


class ParticipantState(Enum):
    IDLE       = "IDLE"
    PREPARED   = "PREPARED"
    COMMITTED  = "COMMITTED"
    ABORTED    = "ABORTED"


class CoordinatorState(Enum):
    INIT       = "INIT"
    PREPARING  = "PREPARING"
    COMMITTING = "COMMITTING"
    ABORTING   = "ABORTING"
    DONE       = "DONE"


@dataclass
class Participant:
    name: str
    fail_on_prepare: bool = False   # Simulate participant refusing
    fail_on_commit:  bool = False   # Simulate commit-phase crash
    state: ParticipantState = ParticipantState.IDLE
    locks_held: list = field(default_factory=list)

    def prepare(self, operations: list) -> Vote:
        if self.fail_on_prepare:
            print(f"  [{self.name}] CRASH during prepare — voting NO")
            self.state = ParticipantState.ABORTED
            return Vote.NO

        # Simulate writing to WAL and acquiring locks
        self.locks_held = operations
        self.state = ParticipantState.PREPARED
        print(f"  [{self.name}] Acquired locks for {operations}, voting YES")
        return Vote.YES

    def commit(self):
        if self.fail_on_commit:
            print(f"  [{self.name}] CRASH during commit — locks still held!")
            # In reality: on restart, participant checks coordinator log and retries
            return False

        self.locks_held.clear()
        self.state = ParticipantState.COMMITTED
        print(f"  [{self.name}] COMMITTED — locks released")
        return True

    def abort(self):
        self.locks_held.clear()
        self.state = ParticipantState.ABORTED
        print(f"  [{self.name}] ABORTED — rolled back, locks released")


class TwoPhaseCoordinator:
    def __init__(self, transaction_id: str, crash_after_votes: bool = False):
        self.transaction_id = transaction_id
        self.crash_after_votes = crash_after_votes
        self.state = CoordinatorState.INIT
        self.decision_log: dict = {}   # Durable log — persists across restarts

    def run(self, participants: list[Participant], operations: dict) -> bool:
        """
        operations: {participant_name: [list of operations]}
        Returns True if transaction committed.
        """
        print(f"\n=== 2PC Transaction: {self.transaction_id} ===")

        # ── Phase 1: PREPARE ──────────────────────────────────────────────
        print(f"\n[Coordinator] Phase 1: Sending PREPARE to {len(participants)} participants")
        self.state = CoordinatorState.PREPARING

        votes = {}
        for p in participants:
            ops = operations.get(p.name, [])
            votes[p.name] = p.prepare(ops)

        all_yes = all(v == Vote.YES for v in votes.values())
        print(f"\n[Coordinator] Votes received: {votes}")

        # ── Simulate coordinator crash after votes but before sending decision ──
        if self.crash_after_votes:
            print("\n[Coordinator] *** CRASHED after collecting votes — before sending COMMIT ***")
            print("[Coordinator] Participants are now BLOCKED — holding locks, waiting for decision")
            print("[Coordinator] Recovery: restart reads WAL decision log, re-sends COMMIT")
            # In a real system: coordinator WAL would have the decision
            # On restart it replays from the log
            return False

        # ── Phase 2: COMMIT or ABORT ──────────────────────────────────────
        if all_yes:
            print(f"\n[Coordinator] All YES — writing COMMIT to durable log")
            self.decision_log[self.transaction_id] = "COMMIT"   # WAL write
            self.state = CoordinatorState.COMMITTING

            print(f"[Coordinator] Phase 2: Sending COMMIT")
            for p in participants:
                p.commit()
        else:
            print(f"\n[Coordinator] At least one NO — sending ABORT")
            self.decision_log[self.transaction_id] = "ABORT"
            self.state = CoordinatorState.ABORTING

            for p in participants:
                if p.state == ParticipantState.PREPARED:
                    p.abort()
                elif p.state == ParticipantState.ABORTED:
                    print(f"  [{p.name}] Already aborted")

        self.state = CoordinatorState.DONE
        result = all_yes
        print(f"\n[Coordinator] Transaction {self.transaction_id}: {'COMMITTED' if result else 'ABORTED'}")
        return result


def demo():
    # Scenario 1: Happy path
    print("\n" + "="*60)
    print("SCENARIO 1: Happy Path — All participants commit")
    p1 = Participant("OrderDB")
    p2 = Participant("PaymentDB")
    p3 = Participant("InventoryDB")
    coord = TwoPhaseCoordinator("txn-001")
    coord.run([p1, p2, p3], {
        "OrderDB":    ["INSERT orders(id=1, status=pending)"],
        "PaymentDB":  ["DEBIT account(user=alice, amount=99)"],
        "InventoryDB":["DECREMENT stock(product=A, qty=1)"],
    })

    # Scenario 2: Participant refuses
    print("\n" + "="*60)
    print("SCENARIO 2: PaymentDB refuses (insufficient funds)")
    q1 = Participant("OrderDB")
    q2 = Participant("PaymentDB", fail_on_prepare=True)   # Insufficient funds
    q3 = Participant("InventoryDB")
    coord2 = TwoPhaseCoordinator("txn-002")
    coord2.run([q1, q2, q3], {
        "OrderDB":    ["INSERT orders(id=2, status=pending)"],
        "PaymentDB":  ["DEBIT account(user=bob, amount=999)"],    # Insufficient
        "InventoryDB":["DECREMENT stock(product=B, qty=1)"],
    })

    # Scenario 3: Coordinator crash — blocking
    print("\n" + "="*60)
    print("SCENARIO 3: Coordinator crashes after collecting all YES votes")
    r1 = Participant("OrderDB")
    r2 = Participant("PaymentDB")
    coord3 = TwoPhaseCoordinator("txn-003", crash_after_votes=True)
    coord3.run([r1, r2], {
        "OrderDB":   ["INSERT orders(id=3)"],
        "PaymentDB": ["DEBIT account(user=carol, amount=50)"],
    })
    print(f"  Order DB state: {r1.state.value} — locks held: {r1.locks_held}")
    print(f"  Payment DB state: {r2.state.value} — locks held: {r2.locks_held}")


if __name__ == "__main__":
    demo()
```

---

## Exercise 2: Saga — Choreography with Compensations

```python
"""
Order checkout Saga using choreography.
Events flow through a simple in-process event bus (substitute Kafka in production).
Demonstrates happy path and compensation rollback when inventory fails.
"""
import uuid
from dataclasses import dataclass, field
from typing import Callable


# ── Event bus (in-process; replace with Kafka/SQS in prod) ──────────────────
EventHandler = Callable[[dict], None]

class EventBus:
    def __init__(self):
        self._handlers: dict[str, list[EventHandler]] = {}

    def subscribe(self, event_type: str, handler: EventHandler):
        self._handlers.setdefault(event_type, []).append(handler)

    def publish(self, event_type: str, payload: dict):
        print(f"\n[EventBus] → {event_type}: {payload}")
        for handler in self._handlers.get(event_type, []):
            handler(payload)


bus = EventBus()


# ── Order Service ────────────────────────────────────────────────────────────
orders: dict[str, dict] = {}

def handle_checkout_requested(event: dict):
    order_id = str(uuid.uuid4())[:8]
    orders[order_id] = {"status": "pending", "user": event["user"], "amount": event["amount"]}
    print(f"  [OrderService] Created order {order_id}")
    bus.publish("order.created", {"order_id": order_id, **event})

def handle_payment_succeeded(event: dict):
    orders[event["order_id"]]["status"] = "paid"
    print(f"  [OrderService] Order {event['order_id']} marked as PAID")

def handle_inventory_failed(event: dict):
    order_id = event["order_id"]
    orders[order_id]["status"] = "cancelled"
    print(f"  [OrderService] Order {order_id} CANCELLED due to inventory failure")
    # Compensate: trigger payment refund
    bus.publish("payment.refund_requested", {"order_id": order_id, "amount": event["amount"]})

def handle_order_confirmed(event: dict):
    orders[event["order_id"]]["status"] = "confirmed"
    print(f"  [OrderService] Order {event['order_id']} fully CONFIRMED")

bus.subscribe("checkout.requested",    handle_checkout_requested)
bus.subscribe("payment.succeeded",     handle_payment_succeeded)
bus.subscribe("inventory.failed",      handle_inventory_failed)
bus.subscribe("stock.reserved",        handle_order_confirmed)


# ── Payment Service ──────────────────────────────────────────────────────────
balances = {"alice": 200, "bob": 10}

def handle_order_created(event: dict):
    user   = event["user"]
    amount = event["amount"]
    if balances.get(user, 0) >= amount:
        balances[user] -= amount
        print(f"  [PaymentService] Debited {amount} from {user} (balance: {balances[user]})")
        bus.publish("payment.succeeded", {"order_id": event["order_id"], "amount": amount, "user": user})
    else:
        print(f"  [PaymentService] Insufficient funds for {user}")
        bus.publish("payment.failed", {"order_id": event["order_id"], "reason": "insufficient_funds"})

def handle_refund_requested(event: dict):
    # Compensating transaction — credit back
    order   = orders.get(event["order_id"], {})
    user    = order.get("user")
    amount  = event["amount"]
    if user:
        balances[user] = balances.get(user, 0) + amount
        print(f"  [PaymentService] REFUNDED {amount} to {user} (balance: {balances[user]})")

bus.subscribe("order.created",          handle_order_created)
bus.subscribe("payment.refund_requested", handle_refund_requested)


# ── Inventory Service ────────────────────────────────────────────────────────
stock = {"laptop": 2, "phone": 0}

def handle_payment_succeeded_inventory(event: dict):
    order   = orders.get(event["order_id"], {})
    product = order.get("product", "laptop")   # simplified
    if stock.get(product, 0) > 0:
        stock[product] -= 1
        print(f"  [InventoryService] Reserved 1x {product} (remaining: {stock[product]})")
        bus.publish("stock.reserved", {"order_id": event["order_id"]})
    else:
        print(f"  [InventoryService] OUT OF STOCK for {product}")
        bus.publish("inventory.failed", {
            "order_id": event["order_id"],
            "amount":   event["amount"],
            "product":  product
        })

bus.subscribe("payment.succeeded", handle_payment_succeeded_inventory)


# ── Demo ─────────────────────────────────────────────────────────────────────
def run_demo():
    print("="*60)
    print("SAGA A: Happy path — alice buys a laptop ($150, stock=2)")
    orders_copy = {"product": "laptop"}
    orders.update({})   # reset handled by order creation
    bus.publish("checkout.requested", {"user": "alice", "amount": 150, "product": "laptop"})
    print(f"\nFinal state — orders: {orders}, stock: {stock}, balances: {balances}")

    print("\n" + "="*60)
    print("SAGA B: Inventory failure — alice buys a phone ($50, stock=0)")
    alice_balance_before = balances["alice"]
    bus.publish("checkout.requested", {"user": "alice", "amount": 50, "product": "phone"})
    print(f"\nFinal state — orders: {orders}, stock: {stock}, balances: {balances}")
    print(f"Alice's balance restored: {alice_balance_before} → {balances['alice']}")


if __name__ == "__main__":
    run_demo()
```

---

## Exercise 3: Saga — Orchestration with State Machine

```python
"""
Order checkout Saga using an explicit orchestrator.
The orchestrator drives state transitions and tracks which
compensations to run on failure.
"""
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional, Callable
import uuid


class SagaState(Enum):
    STARTED           = "STARTED"
    ORDER_CREATED     = "ORDER_CREATED"
    PAYMENT_DEBITED   = "PAYMENT_DEBITED"
    STOCK_RESERVED    = "STOCK_RESERVED"
    COMPLETED         = "COMPLETED"
    COMPENSATING      = "COMPENSATING"
    COMPENSATED       = "COMPENSATED"
    FAILED            = "FAILED"


@dataclass
class SagaExecution:
    saga_id: str
    state: SagaState = SagaState.STARTED
    completed_steps: list[str] = field(default_factory=list)
    error: Optional[str] = None


# ── Service stubs ────────────────────────────────────────────────────────────
class OrderService:
    orders: dict = {}

    def create_order(self, saga_id: str, user: str, amount: float) -> str:
        order_id = f"ord-{saga_id[:6]}"
        self.orders[order_id] = {"status": "pending", "user": user, "amount": amount}
        print(f"    [OrderService] Created order {order_id}")
        return order_id

    def confirm_order(self, order_id: str):
        self.orders[order_id]["status"] = "confirmed"
        print(f"    [OrderService] Confirmed order {order_id}")

    def cancel_order(self, order_id: str):
        self.orders[order_id]["status"] = "cancelled"
        print(f"    [OrderService] CANCELLED order {order_id} (compensating)")


class PaymentService:
    balances: dict = {"alice": 200, "charlie": 5}

    def debit(self, user: str, amount: float) -> bool:
        if self.balances.get(user, 0) >= amount:
            self.balances[user] -= amount
            print(f"    [PaymentService] Debited ${amount} from {user} (balance: {self.balances[user]})")
            return True
        print(f"    [PaymentService] Insufficient funds for {user}")
        return False

    def refund(self, user: str, amount: float):
        self.balances[user] = self.balances.get(user, 0) + amount
        print(f"    [PaymentService] REFUNDED ${amount} to {user} (compensating) balance={self.balances[user]}")


class InventoryService:
    stock: dict = {"widget": 3, "gadget": 0}

    def reserve(self, product: str, qty: int) -> bool:
        if self.stock.get(product, 0) >= qty:
            self.stock[product] -= qty
            print(f"    [InventoryService] Reserved {qty}x {product} (remaining: {self.stock[product]})")
            return True
        print(f"    [InventoryService] OUT OF STOCK: {product}")
        return False

    def release(self, product: str, qty: int):
        self.stock[product] = self.stock.get(product, 0) + qty
        print(f"    [InventoryService] Released {qty}x {product} (compensating) stock={self.stock[product]}")


# ── Orchestrator ─────────────────────────────────────────────────────────────
class OrderSagaOrchestrator:
    def __init__(self):
        self.order_svc   = OrderService()
        self.payment_svc = PaymentService()
        self.inventory_svc = InventoryService()
        self.executions: dict[str, SagaExecution] = {}

    def execute(self, user: str, amount: float, product: str, qty: int = 1) -> SagaExecution:
        saga_id  = str(uuid.uuid4())[:8]
        saga     = SagaExecution(saga_id)
        self.executions[saga_id] = saga
        print(f"\n[Orchestrator] Starting saga {saga_id}: {user} buys {qty}x {product} for ${amount}")

        # Context carried through steps
        ctx = {}

        # ── Step 1: Create Order ───────────────────────────────────────────
        print(f"[Orchestrator][{saga_id}] Step 1 → CreateOrder")
        try:
            ctx["order_id"] = self.order_svc.create_order(saga_id, user, amount)
            saga.state = SagaState.ORDER_CREATED
            saga.completed_steps.append("create_order")
        except Exception as e:
            saga.error = str(e)
            saga.state = SagaState.FAILED
            return saga

        # ── Step 2: Debit Payment ──────────────────────────────────────────
        print(f"[Orchestrator][{saga_id}] Step 2 → DebitPayment")
        if not self.payment_svc.debit(user, amount):
            saga.error = "payment_failed"
            self._compensate(saga, ctx, user=user, amount=amount, product=product, qty=qty)
            return saga

        saga.state = SagaState.PAYMENT_DEBITED
        saga.completed_steps.append("debit_payment")

        # ── Step 3: Reserve Stock ──────────────────────────────────────────
        print(f"[Orchestrator][{saga_id}] Step 3 → ReserveStock")
        if not self.inventory_svc.reserve(product, qty):
            saga.error = "inventory_failed"
            self._compensate(saga, ctx, user=user, amount=amount, product=product, qty=qty)
            return saga

        saga.state = SagaState.STOCK_RESERVED
        saga.completed_steps.append("reserve_stock")

        # ── Step 4: Confirm Order ──────────────────────────────────────────
        print(f"[Orchestrator][{saga_id}] Step 4 → ConfirmOrder")
        self.order_svc.confirm_order(ctx["order_id"])
        saga.state = SagaState.COMPLETED
        print(f"[Orchestrator][{saga_id}] ✅ COMPLETED")
        return saga

    def _compensate(self, saga: SagaExecution, ctx: dict,
                    user: str, amount: float, product: str, qty: int):
        """Run compensations in reverse order for all completed steps."""
        print(f"[Orchestrator][{saga.saga_id}] ⚠️ Compensating — error: {saga.error}")
        saga.state = SagaState.COMPENSATING

        for step in reversed(saga.completed_steps):
            if step == "reserve_stock":
                self.inventory_svc.release(product, qty)
            elif step == "debit_payment":
                self.payment_svc.refund(user, amount)
            elif step == "create_order":
                self.order_svc.cancel_order(ctx.get("order_id", "?"))

        saga.state = SagaState.COMPENSATED
        print(f"[Orchestrator][{saga.saga_id}] 🔄 COMPENSATED — all steps rolled back")


def demo():
    orch = OrderSagaOrchestrator()

    r1 = orch.execute("alice", amount=50.0,  product="widget", qty=1)
    print(f"Result: {r1.state.value}\n")

    r2 = orch.execute("charlie", amount=50.0, product="widget", qty=1)  # Insufficient funds
    print(f"Result: {r2.state.value}\n")

    r3 = orch.execute("alice", amount=30.0, product="gadget", qty=1)  # Out of stock
    print(f"Result: {r3.state.value}")


if __name__ == "__main__":
    demo()
```

---

## Exercise 4: Outbox Pattern — Reliable Event Publishing

```python
"""
Outbox pattern implementation using PostgreSQL.
Demonstrates: atomic write + outbox insert, polling publisher,
and idempotent consumer.

Run with: pip install asyncpg
Requires: PostgreSQL with tables created (see schema below)
"""
import asyncio
import json
import uuid
from datetime import datetime, UTC

import asyncpg

DSN = "postgresql://user:password@localhost:5432/mydb"

# ─── Schema ───────────────────────────────────────────────────────────────────
SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS orders (
    id         UUID PRIMARY KEY,
    user_id    TEXT NOT NULL,
    amount     NUMERIC NOT NULL,
    status     TEXT NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS outbox (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    event_type   TEXT NOT NULL,
    aggregate_id TEXT NOT NULL,
    payload      JSONB NOT NULL,
    status       TEXT NOT NULL DEFAULT 'pending',
    published_at TIMESTAMPTZ,
    retry_count  INT NOT NULL DEFAULT 0
);

-- Partial index: only index rows that still need publishing
CREATE INDEX IF NOT EXISTS idx_outbox_pending
    ON outbox (created_at)
    WHERE status = 'pending';

-- Idempotency table for consumers
CREATE TABLE IF NOT EXISTS processed_events (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
"""


# ─── Application Layer ────────────────────────────────────────────────────────
async def create_order(conn: asyncpg.Connection, user_id: str, amount: float) -> str:
    """
    Atomically:
      1. Insert order row
      2. Insert outbox event in the SAME transaction

    Either both are written or neither — no dual-write problem.
    """
    order_id = str(uuid.uuid4())

    async with conn.transaction():
        await conn.execute(
            """INSERT INTO orders (id, user_id, amount, status)
               VALUES ($1, $2, $3, 'pending')""",
            order_id, user_id, amount
        )

        await conn.execute(
            """INSERT INTO outbox (event_type, aggregate_id, payload)
               VALUES ($1, $2, $3)""",
            "order.created",
            order_id,
            json.dumps({"order_id": order_id, "user_id": user_id, "amount": amount})
        )

    print(f"[App] Created order {order_id} + outbox entry (atomic)")
    return order_id


# ─── Outbox Publisher ─────────────────────────────────────────────────────────
async def simulate_kafka_publish(event_type: str, payload: dict) -> bool:
    """Simulates publishing to Kafka. Returns True on success."""
    print(f"  [Publisher] Sending to Kafka topic '{event_type}': {payload}")
    await asyncio.sleep(0.01)   # Network I/O simulation
    return True


async def outbox_publisher(conn: asyncpg.Connection, batch_size: int = 100):
    """
    Polls for pending outbox rows and publishes them.
    Uses SELECT FOR UPDATE SKIP LOCKED to allow multiple publisher instances
    without double-processing (safe for horizontal scaling).
    """
    while True:
        async with conn.transaction():
            rows = await conn.fetch(
                """SELECT id, event_type, aggregate_id, payload
                   FROM outbox
                   WHERE status = 'pending'
                   ORDER BY created_at
                   LIMIT $1
                   FOR UPDATE SKIP LOCKED""",
                batch_size
            )

            if not rows:
                pass  # nothing to publish
            else:
                for row in rows:
                    payload = dict(row["payload"])
                    success = await simulate_kafka_publish(row["event_type"], payload)

                    if success:
                        await conn.execute(
                            """UPDATE outbox
                               SET status = 'published', published_at = NOW()
                               WHERE id = $1""",
                            row["id"]
                        )
                    else:
                        await conn.execute(
                            """UPDATE outbox
                               SET retry_count = retry_count + 1
                               WHERE id = $1""",
                            row["id"]
                        )

        await asyncio.sleep(0.1)   # Poll interval; replace with CDC in production


# ─── Idempotent Consumer ──────────────────────────────────────────────────────
async def consume_order_created(conn: asyncpg.Connection, event_id: str, payload: dict):
    """
    Idempotent consumer: inserts event_id into processed_events before processing.
    If event_id already exists, skip — handles at-least-once re-delivery safely.
    """
    async with conn.transaction():
        # Try to mark this event as processed
        result = await conn.execute(
            """INSERT INTO processed_events (event_id)
               VALUES ($1)
               ON CONFLICT (event_id) DO NOTHING""",
            uuid.UUID(event_id)
        )

        if result == "INSERT 0 0":
            print(f"  [Consumer] Event {event_id} already processed — SKIPPING (idempotent)")
            return

        # Process the event (runs inside the same transaction)
        order_id = payload.get("order_id")
        print(f"  [Consumer] Processing order.created for order {order_id}")
        await conn.execute(
            "UPDATE orders SET status = 'processing' WHERE id = $1",
            order_id
        )
        print(f"  [Consumer] Order {order_id} moved to processing")


# ─── Demo orchestration ───────────────────────────────────────────────────────
async def main():
    conn = await asyncpg.connect(DSN)
    await conn.execute(SCHEMA_SQL)

    # Create two orders atomically
    oid1 = await create_order(conn, "alice", 99.99)
    oid2 = await create_order(conn, "bob",   49.00)

    # Run publisher for one cycle
    print("\n[Publisher] Running publish cycle...")
    await outbox_publisher(conn, batch_size=10)

    # Simulate idempotent consumption (same event delivered twice)
    print("\n[Consumer] First delivery:")
    await consume_order_created(conn, oid1, {"order_id": oid1})

    print("\n[Consumer] Duplicate delivery (Kafka re-delivery simulation):")
    await consume_order_created(conn, oid1, {"order_id": oid1})

    await conn.close()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Exercise 5: FastAPI — End-to-End Order Saga (Orchestration + Outbox)

```python
"""
Production-ready checkout endpoint combining:
  - Saga orchestration for multi-service coordination
  - Outbox pattern for reliable event publishing
  - Idempotency key to prevent duplicate checkouts
"""
import json
import uuid
from contextlib import asynccontextmanager
from typing import Annotated

import asyncpg
from fastapi import FastAPI, HTTPException, Header, Depends
from pydantic import BaseModel

DB_DSN = "postgresql://user:password@localhost:5432/mydb"
_pool: asyncpg.Pool | None = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    global _pool
    _pool = await asyncpg.create_pool(DB_DSN, min_size=5, max_size=20)
    yield
    await _pool.close()


app = FastAPI(lifespan=lifespan)


async def get_conn():
    async with _pool.acquire() as conn:
        yield conn


class CheckoutRequest(BaseModel):
    user_id:    str
    product_id: str
    quantity:   int
    amount:     float


@app.post("/checkout")
async def checkout(
    req: CheckoutRequest,
    idempotency_key: Annotated[str, Header()],  # Client-provided UUID
    conn: asyncpg.Connection = Depends(get_conn),
):
    """
    Idempotent checkout using Saga orchestration + Outbox.
    If the same idempotency_key is sent twice, returns the original result.
    """
    # ── Idempotency check ──────────────────────────────────────────────────
    existing = await conn.fetchrow(
        "SELECT order_id, status FROM checkout_requests WHERE idempotency_key = $1",
        idempotency_key
    )
    if existing:
        return {"order_id": existing["order_id"], "status": existing["status"], "idempotent": True}

    order_id = str(uuid.uuid4())

    # ── Saga Step 1: Create order + outbox (atomic) ────────────────────────
    async with conn.transaction():
        # Record idempotency key immediately
        await conn.execute(
            """INSERT INTO checkout_requests (idempotency_key, order_id, status)
               VALUES ($1, $2, 'processing')""",
            idempotency_key, order_id
        )

        # Create the order
        await conn.execute(
            """INSERT INTO orders (id, user_id, product_id, quantity, amount, status)
               VALUES ($1, $2, $3, $4, $5, 'pending')""",
            order_id, req.user_id, req.product_id, req.quantity, req.amount
        )

        # Outbox entry for payment service to pick up
        await conn.execute(
            """INSERT INTO outbox (event_type, aggregate_id, payload)
               VALUES ('checkout.initiated', $1, $2)""",
            order_id,
            json.dumps({
                "order_id":   order_id,
                "user_id":    req.user_id,
                "product_id": req.product_id,
                "quantity":   req.quantity,
                "amount":     req.amount
            })
        )

    # ── Return immediately — downstream Saga runs async via outbox ─────────
    return {
        "order_id": order_id,
        "status":   "processing",
        "message":  "Order accepted. Payment and inventory processing asynchronously."
    }
```
