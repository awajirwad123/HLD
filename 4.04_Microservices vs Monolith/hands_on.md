# Microservices vs Monolith — Hands-On Exercises

## Exercise 1: Modular Monolith with Enforced Boundaries

```python
"""
A modular monolith with strict module boundaries enforced via explicit interfaces.
Three domains: orders, inventory, notifications.
Modules communicate only via their public service interfaces — no cross-module imports.

This demonstrates the "modular monolith" stage before extracting microservices.
"""
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime, UTC
from enum import Enum
from typing import Protocol
import uuid


# ── Shared kernel (only value objects used across domains) ────────────────────
@dataclass(frozen=True)
class Money:
    amount: int    # Cents
    currency: str = "USD"

    def __add__(self, other: Money) -> Money:
        assert self.currency == other.currency
        return Money(self.amount + other.amount, self.currency)


# ── Inventory Module ──────────────────────────────────────────────────────────
# The ONLY way other modules interact with inventory is via InventoryPort.
class InventoryPort(Protocol):
    def reserve(self, product_id: str, quantity: int) -> bool: ...
    def release(self, product_id: str, quantity: int) -> None: ...
    def get_stock(self, product_id: str) -> int: ...


class InventoryService:
    """Concrete implementation — internal to the Inventory module."""

    def __init__(self):
        # product_id → available stock
        self._stock: dict[str, int] = {
            "SKU-001": 50,
            "SKU-002": 10,
            "SKU-003": 0,
        }
        self._reservations: dict[str, dict[str, int]] = {}   # order_id → {product: qty}

    def reserve(self, product_id: str, quantity: int, order_id: str = None) -> bool:
        available = self._stock.get(product_id, 0)
        if available < quantity:
            return False
        self._stock[product_id] -= quantity
        if order_id:
            self._reservations.setdefault(order_id, {})[product_id] = quantity
        print(f"[Inventory] Reserved {quantity}× {product_id} — remaining: {self._stock[product_id]}")
        return True

    def release(self, product_id: str, quantity: int) -> None:
        self._stock[product_id] = self._stock.get(product_id, 0) + quantity
        print(f"[Inventory] Released {quantity}× {product_id} — stock: {self._stock[product_id]}")

    def get_stock(self, product_id: str) -> int:
        return self._stock.get(product_id, 0)


# ── Notification Module ───────────────────────────────────────────────────────
class NotificationPort(Protocol):
    def send(self, user_id: str, event_type: str, data: dict) -> None: ...


class NotificationService:
    def send(self, user_id: str, event_type: str, data: dict) -> None:
        print(f"[Notification] → user:{user_id} | {event_type} | {data}")


# ── Orders Module ─────────────────────────────────────────────────────────────
class OrderStatus(Enum):
    PENDING     = "pending"
    CONFIRMED   = "confirmed"
    CANCELLED   = "cancelled"


@dataclass
class OrderItem:
    product_id: str
    quantity: int
    unit_price: Money


@dataclass
class Order:
    id: str
    user_id: str
    items: list[OrderItem]
    status: OrderStatus = OrderStatus.PENDING
    created_at: datetime = field(default_factory=lambda: datetime.now(UTC))
    total: Money = field(init=False)

    def __post_init__(self):
        self.total = sum(
            (Money(i.unit_price.amount * i.quantity) for i in self.items),
            start=Money(0)
        )


class OrderService:
    """
    The Order module interacts with Inventory and Notifications ONLY via their Ports.
    No direct imports of InventoryService or NotificationService.
    Dependency injection ensures runtime binding.
    """

    def __init__(self, inventory: InventoryPort, notifications: NotificationPort):
        self._inventory    = inventory
        self._notifications = notifications
        self._orders: dict[str, Order] = {}

    def place_order(self, user_id: str, items: list[dict]) -> Order:
        order_id = str(uuid.uuid4())[:8]
        order_items = [
            OrderItem(i["product_id"], i["quantity"], Money(i["unit_price"]))
            for i in items
        ]
        order = Order(id=order_id, user_id=user_id, items=order_items)

        # Reserve stock (via interface — doesn't know how inventory is implemented)
        for item in order.items:
            reserved = self._inventory.reserve(item.product_id, item.quantity)
            if not reserved:
                # Roll back any previous reservations in this order
                for prev in order.items:
                    if prev is item:
                        break
                    self._inventory.release(prev.product_id, prev.quantity)
                raise ValueError(f"Insufficient stock for {item.product_id}")

        order.status = OrderStatus.CONFIRMED
        self._orders[order_id] = order

        # Notify (via interface)
        self._notifications.send(user_id, "order_confirmed", {
            "order_id": order_id,
            "total": order.total.amount,
        })

        return order

    def cancel_order(self, order_id: str) -> None:
        order = self._orders.get(order_id)
        if not order or order.status != OrderStatus.CONFIRMED:
            raise ValueError("Cannot cancel order")

        for item in order.items:
            self._inventory.release(item.product_id, item.quantity)

        order.status = OrderStatus.CANCELLED
        self._notifications.send(order.user_id, "order_cancelled", {"order_id": order_id})

    def get_order(self, order_id: str) -> Order:
        return self._orders[order_id]


# ── Composition Root (wires modules together — only place with all imports) ───
def create_app():
    inventory    = InventoryService()
    notifications = NotificationService()
    orders       = OrderService(inventory, notifications)
    return orders, inventory


# ── Demo ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    orders, inventory = create_app()

    print("=== Stock before order ===")
    print(f"SKU-001: {inventory.get_stock('SKU-001')}")

    print("\n=== Place Order ===")
    order = orders.place_order("user-1", [
        {"product_id": "SKU-001", "quantity": 2, "unit_price": 2999},
    ])
    print(f"Order {order.id} status: {order.status.value}, total: ${order.total.amount/100:.2f}")

    print("\n=== Try to order out-of-stock item ===")
    try:
        orders.place_order("user-2", [{"product_id": "SKU-003", "quantity": 1, "unit_price": 999}])
    except ValueError as e:
        print(f"Expected failure: {e}")

    print("\n=== Cancel order ===")
    orders.cancel_order(order.id)
    print(f"SKU-001 stock restored: {inventory.get_stock('SKU-001')}")
```

---

## Exercise 2: Sync vs Async Service Communication

```python
"""
Demonstrates sync (HTTP-like / direct call) vs async (event-driven) 
inter-service communication.

Sync: OrderService calls PaymentService directly — tight coupling.
Async: OrderService publishes an event — PaymentProcessor subscribes.

Uses asyncio.Queue as an in-process event bus (swap for Kafka in production).
"""
import asyncio
import uuid
import time
import random
from dataclasses import dataclass
from typing import Callable


# ── ① SYNCHRONOUS COMMUNICATION — Cascading failure demo ─────────────────────

class PaymentServiceSync:
    async def charge(self, user_id: str, amount: int) -> dict:
        # Simulate variable latency (sometimes slow)
        delay = random.uniform(0.01, 0.5)
        await asyncio.sleep(delay)
        if random.random() < 0.1:  # 10% failure rate
            raise RuntimeError("Payment gateway timeout")
        return {"status": "charged", "tx_id": str(uuid.uuid4())[:8], "latency_ms": int(delay * 1000)}


class OrderServiceSync:
    def __init__(self, payment: PaymentServiceSync):
        self._payment = payment
        self._orders  = {}

    async def place_order(self, user_id: str, amount: int) -> dict:
        order_id = str(uuid.uuid4())[:8]
        try:
            # SYNC: order service WAITS for payment service
            # If payment service is slow/down, this entire call is slow/fails
            result = await self._payment.charge(user_id, amount)
            self._orders[order_id] = {"status": "confirmed", "payment": result}
            return {"order_id": order_id, "status": "confirmed", "latency_ms": result["latency_ms"]}
        except RuntimeError as e:
            return {"order_id": order_id, "status": "failed", "reason": str(e)}


# ── ② ASYNCHRONOUS COMMUNICATION — Decoupled via event bus ───────────────────

@dataclass
class Event:
    type: str
    data: dict
    event_id: str = None

    def __post_init__(self):
        self.event_id = self.event_id or str(uuid.uuid4())[:8]


class EventBus:
    """In-process async event bus. Replace with Kafka/RabbitMQ in production."""

    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = {}
        self._queue = asyncio.Queue()

    def subscribe(self, event_type: str, handler: Callable):
        self._subscribers.setdefault(event_type, []).append(handler)

    async def publish(self, event: Event):
        await self._queue.put(event)
        print(f"[EventBus] Published: {event.type} id={event.event_id}")

    async def process(self):
        """Event loop — runs in background."""
        while True:
            event = await self._queue.get()
            handlers = self._subscribers.get(event.type, [])
            for handler in handlers:
                try:
                    await handler(event)
                except Exception as e:
                    print(f"[EventBus] Handler error for {event.type}: {e}")
            self._queue.task_done()


class OrderServiceAsync:
    def __init__(self, bus: EventBus):
        self._bus    = bus
        self._orders = {}

    async def place_order(self, user_id: str, amount: int) -> dict:
        order_id = str(uuid.uuid4())[:8]
        # Record order as PENDING
        self._orders[order_id] = {"status": "pending", "user_id": user_id, "amount": amount}

        # Fire event and return immediately — not waiting for payment
        await self._bus.publish(Event(
            type="order.created",
            data={"order_id": order_id, "user_id": user_id, "amount": amount}
        ))

        # Return immediately with "pending" — no dependency on payment service latency
        return {"order_id": order_id, "status": "pending"}

    async def on_payment_completed(self, event: Event):
        order_id = event.data["order_id"]
        if order_id in self._orders:
            self._orders[order_id]["status"] = "confirmed"
            print(f"[OrderService] Order {order_id} confirmed after payment")

    async def on_payment_failed(self, event: Event):
        order_id = event.data["order_id"]
        if order_id in self._orders:
            self._orders[order_id]["status"] = "cancelled"
            print(f"[OrderService] Order {order_id} cancelled — payment failed")


class PaymentServiceAsync:
    def __init__(self, bus: EventBus):
        self._bus = bus

    async def on_order_created(self, event: Event):
        """Subscribes to order.created — processes payment independently."""
        order_id = event.data["order_id"]
        user_id  = event.data["user_id"]
        amount   = event.data["amount"]

        await asyncio.sleep(random.uniform(0.1, 0.5))  # Simulate payment processing

        if random.random() < 0.1:  # 10% failure
            await self._bus.publish(Event(
                type="payment.failed",
                data={"order_id": order_id, "reason": "gateway_timeout"}
            ))
        else:
            tx_id = str(uuid.uuid4())[:8]
            await self._bus.publish(Event(
                type="payment.completed",
                data={"order_id": order_id, "tx_id": tx_id}
            ))
            print(f"[PaymentService] Charged ${amount/100:.2f} for order {order_id} → tx={tx_id}")


# ── Demo ──────────────────────────────────────────────────────────────────────
async def main():
    # ── Sync demo ──
    print("=" * 50)
    print("SYNC: OrderService waits on PaymentService")
    print("=" * 50)
    ps = PaymentServiceSync()
    os_sync = OrderServiceSync(ps)
    for i in range(3):
        result = await os_sync.place_order("user-1", 1999)
        print(f"  Result: {result}")

    # ── Async demo ──
    print("\n" + "=" * 50)
    print("ASYNC: OrderService returns immediately, payment processed separately")
    print("=" * 50)

    bus      = EventBus()
    os_async = OrderServiceAsync(bus)
    ps_async = PaymentServiceAsync(bus)

    # Wire up subscriptions
    bus.subscribe("order.created",      ps_async.on_order_created)
    bus.subscribe("payment.completed",  os_async.on_payment_completed)
    bus.subscribe("payment.failed",     os_async.on_payment_failed)

    # Start event processor in background
    processor = asyncio.create_task(bus.process())

    # Place 3 orders — all return immediately
    orders = await asyncio.gather(
        os_async.place_order("user-1", 1999),
        os_async.place_order("user-2", 4999),
        os_async.place_order("user-3", 999),
    )
    for o in orders:
        print(f"  Immediate result: {o}")

    # Wait for async payment processing to complete
    await asyncio.sleep(2.0)
    processor.cancel()

    print("\nFinal order states:")
    for order_id, state in os_async._orders.items():
        print(f"  {order_id}: {state['status']}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Exercise 3: Simple API Gateway / Facade for Strangler Fig

```python
"""
API Gateway that routes requests to either:
  - The new microservice (implemented)
  - The legacy monolith (proxied)

This is the Strangler Fig pattern in code.
Progressively, monolith routes become 0 as microservices cover each path.

pip install fastapi httpx uvicorn
"""
import httpx
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse

app = FastAPI(title="API Gateway")

# Route table: path_prefix → microservice URL or "monolith"
ROUTES: dict[str, str] = {
    "/api/users":    "http://user-service:8001",        # Extracted to microservice
    "/api/auth":     "http://auth-service:8002",        # Extracted to microservice
    "/api/orders":   "monolith",                         # Not yet extracted
    "/api/products": "monolith",                         # Not yet extracted
}

MONOLITH_URL = "http://legacy-monolith:8080"


async def proxy_to(request: Request, target_url: str) -> Response:
    """Forward request to target URL, preserving headers and body."""
    async with httpx.AsyncClient(timeout=30.0) as client:
        url = target_url + str(request.url.path)
        if request.url.query:
            url += f"?{request.url.query}"

        # Forward only safe headers (no hop-by-hop headers)
        forward_headers = {
            k: v for k, v in request.headers.items()
            if k.lower() not in {"host", "connection", "transfer-encoding"}
        }

        resp = await client.request(
            method=request.method,
            url=url,
            headers=forward_headers,
            content=await request.body(),
        )

        return Response(
            content=resp.content,
            status_code=resp.status_code,
            headers=dict(resp.headers),
            media_type=resp.headers.get("content-type"),
        )


@app.api_route("/{path:path}", methods=["GET", "POST", "PUT", "DELETE", "PATCH"])
async def gateway(request: Request, path: str):
    """
    Match the incoming path prefix against the route table.
    Route to microservice or fall through to monolith.
    """
    full_path = "/" + path

    # Find matching route (longest prefix match)
    target = None
    matched_prefix_len = 0
    for prefix, destination in ROUTES.items():
        if full_path.startswith(prefix) and len(prefix) > matched_prefix_len:
            target = destination
            matched_prefix_len = len(prefix)

    if target is None or target == "monolith":
        # No specific microservice — forward to monolith
        print(f"[Gateway] → MONOLITH: {request.method} {full_path}")
        return await proxy_to(request, MONOLITH_URL)

    print(f"[Gateway] → MICROSERVICE ({target}): {request.method} {full_path}")
    try:
        return await proxy_to(request, target)
    except httpx.ConnectError:
        # Microservice down — optionally fall back to monolith
        print(f"[Gateway] Microservice {target} unreachable, falling back to monolith")
        return await proxy_to(request, MONOLITH_URL)


# Health check endpoint (doesn't proxy)
@app.get("/health")
async def health():
    return {"status": "ok", "routes": ROUTES}


# ── To test locally (mock targets) ───────────────────────────────────────────
# Add these mocks before starting the server for local testing:
#
# @app.get("/api/users/me")
# async def mock_users():
#     return {"source": "user-microservice", "user_id": "u-123"}
```

---

## Exercise 4: Circuit Breaker Pattern (Manual Implementation)

```python
"""
Simple circuit breaker to prevent cascading failures in microservice calls.
Three states: CLOSED (normal) → OPEN (failing) → HALF_OPEN (testing recovery).

pip install asyncio
"""
import asyncio
import time
from enum import Enum
from typing import Callable, Any


class CircuitState(Enum):
    CLOSED    = "closed"     # Normal operation — calls pass through
    OPEN      = "open"       # Failures exceeded threshold — calls rejected fast
    HALF_OPEN  = "half_open"  # Testing if service recovered


class CircuitBreaker:
    def __init__(
        self,
        name: str,
        failure_threshold: int  = 5,    # Open after N failures in the window
        recovery_timeout: float = 30.0, # Seconds before trying HALF_OPEN
        success_threshold: int  = 2,    # Successes needed in HALF_OPEN to close
    ):
        self.name              = name
        self.failure_threshold = failure_threshold
        self.recovery_timeout  = recovery_timeout
        self.success_threshold = success_threshold

        self._state             = CircuitState.CLOSED
        self._failure_count     = 0
        self._success_count     = 0
        self._last_failure_time: float = 0

    @property
    def state(self) -> CircuitState:
        if self._state == CircuitState.OPEN:
            # Check if recovery timeout has elapsed → transition to HALF_OPEN
            if time.monotonic() - self._last_failure_time >= self.recovery_timeout:
                self._state         = CircuitState.HALF_OPEN
                self._success_count = 0
                print(f"[CB:{self.name}] OPEN → HALF_OPEN (testing recovery)")
        return self._state

    async def call(self, fn: Callable, *args, **kwargs) -> Any:
        current_state = self.state

        if current_state == CircuitState.OPEN:
            raise RuntimeError(f"Circuit {self.name} is OPEN — fast-failing")

        try:
            result = await fn(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        if self._state == CircuitState.HALF_OPEN:
            self._success_count += 1
            print(f"[CB:{self.name}] HALF_OPEN success {self._success_count}/{self.success_threshold}")
            if self._success_count >= self.success_threshold:
                self._state         = CircuitState.CLOSED
                self._failure_count = 0
                print(f"[CB:{self.name}] HALF_OPEN → CLOSED (service recovered)")
        elif self._state == CircuitState.CLOSED:
            self._failure_count = 0  # Reset on success

    def _on_failure(self):
        self._failure_count    += 1
        self._last_failure_time = time.monotonic()
        print(f"[CB:{self.name}] Failure {self._failure_count}/{self.failure_threshold}")

        if self._state == CircuitState.HALF_OPEN:
            self._state = CircuitState.OPEN
            print(f"[CB:{self.name}] HALF_OPEN → OPEN (recovery attempt failed)")

        elif self._state == CircuitState.CLOSED and self._failure_count >= self.failure_threshold:
            self._state = CircuitState.OPEN
            print(f"[CB:{self.name}] CLOSED → OPEN (threshold reached)")


# ── Demo ──────────────────────────────────────────────────────────────────────
call_count = 0

async def flaky_payment_service(amount: int) -> dict:
    global call_count
    call_count += 1
    # Fail calls 3–8, then recover
    if 3 <= call_count <= 8:
        raise ConnectionError(f"Payment service timeout (call #{call_count})")
    return {"tx_id": f"tx-{call_count}", "amount": amount}


async def demo():
    cb = CircuitBreaker("payment", failure_threshold=3, recovery_timeout=2.0)

    for i in range(15):
        await asyncio.sleep(0.1)
        # Simulate recovery timeout passing after call 9
        if i == 9:
            print("\n[Demo] Simulating 2-second recovery window elapsed...")
            await asyncio.sleep(2.1)

        try:
            result = await cb.call(flaky_payment_service, amount=1000 * i)
            print(f"  Call {i+1}: OK → {result['tx_id']} | CB state: {cb._state.value}")
        except RuntimeError as e:
            print(f"  Call {i+1}: FAST-FAIL — {e}")
        except ConnectionError as e:
            print(f"  Call {i+1}: SERVICE ERROR — {e} | CB state: {cb._state.value}")


if __name__ == "__main__":
    asyncio.run(demo())
```

---

## Exercise 5: Service Health Check & Discovery

```python
"""
Minimal service registry with health checks.
Services register themselves; stale registrations are removed by TTL.
Client-side load balancing picks a healthy instance.

In production: replace with Consul, etcd, or Kubernetes DNS.
pip install redis asyncio fastapi
"""
import asyncio
import random
import time
import uuid
import redis.asyncio as aioredis
from dataclasses import dataclass

REDIS_URL      = "redis://localhost:6379"
REGISTRY_TTL   = 30   # Seconds — service must re-register before this expires
HEARTBEAT_INTERVAL = 10


@dataclass
class ServiceInstance:
    service_name: str
    host: str
    port: int
    instance_id: str = None

    def __post_init__(self):
        self.instance_id = self.instance_id or str(uuid.uuid4())[:8]

    @property
    def address(self) -> str:
        return f"{self.host}:{self.port}"


class ServiceRegistry:
    def __init__(self, r: aioredis.Redis):
        self._r = r

    async def register(self, instance: ServiceInstance):
        """Register a service instance with TTL. Must be renewed periodically."""
        key = f"registry:{instance.service_name}:{instance.instance_id}"
        await self._r.setex(key, REGISTRY_TTL, f"{instance.host}:{instance.port}")
        print(f"[Registry] Registered {instance.service_name} → {instance.address} (id={instance.instance_id})")

    async def deregister(self, instance: ServiceInstance):
        key = f"registry:{instance.service_name}:{instance.instance_id}"
        await self._r.delete(key)
        print(f"[Registry] Deregistered {instance.service_name}:{instance.instance_id}")

    async def discover(self, service_name: str) -> list[str]:
        """Return list of healthy addresses for a service (TTL-based health)."""
        pattern = f"registry:{service_name}:*"
        addresses = []
        async for key in self._r.scan_iter(pattern):
            addr = await self._r.get(key)
            if addr:
                addresses.append(addr)
        return addresses

    async def pick_instance(self, service_name: str) -> str | None:
        """Client-side random load balancing across discovered instances."""
        instances = await self.discover(service_name)
        if not instances:
            return None
        return random.choice(instances)


async def service_heartbeat(registry: ServiceRegistry, instance: ServiceInstance):
    """Re-register periodically to renew TTL — signals liveness."""
    while True:
        await asyncio.sleep(HEARTBEAT_INTERVAL)
        await registry.register(instance)


# ── Demo ──────────────────────────────────────────────────────────────────────
async def demo():
    r = aioredis.from_url(REDIS_URL, decode_responses=True)
    registry = ServiceRegistry(r)

    # Register 3 payment service instances (e.g., on 3 different servers)
    instances = [
        ServiceInstance("payment-service", "10.0.0.1", 8001),
        ServiceInstance("payment-service", "10.0.0.2", 8001),
        ServiceInstance("payment-service", "10.0.0.3", 8001),
    ]

    for inst in instances:
        await registry.register(inst)

    # Start heartbeat tasks
    tasks = [asyncio.create_task(service_heartbeat(registry, i)) for i in instances]

    print("\n[Client] Discovering payment-service instances:")
    print(await registry.discover("payment-service"))

    print("\n[Client] Random picks (client-side LB):")
    for _ in range(6):
        picked = await registry.pick_instance("payment-service")
        print(f"  Routed to: {picked}")

    # Simulate instance 0 going down (no more heartbeats)
    print(f"\n[Demo] Instance {instances[0].address} goes down — deregistering")
    await registry.deregister(instances[0])

    print("\n[Client] Available instances after deregistration:")
    print(await registry.discover("payment-service"))

    for t in tasks:
        t.cancel()
    await r.close()


if __name__ == "__main__":
    asyncio.run(demo())
```
