# API Design — Hands-On Exercises

## Exercise 1: Build a Well-Designed REST API with FastAPI

```python
"""
Production-quality REST API demonstrating:
- Correct HTTP method + status code usage
- Cursor-based pagination
- Consistent error envelope
- OpenAPI docs via FastAPI
"""
import uuid
from datetime import datetime, UTC
from typing import Optional, Annotated
from enum import Enum

from fastapi import FastAPI, HTTPException, Query, Path, Response, status
from pydantic import BaseModel, EmailStr, field_validator

app = FastAPI(title="Orders API", version="1.0.0")


# ── Models ────────────────────────────────────────────────────────────────────
class OrderStatus(str, Enum):
    PENDING   = "pending"
    PAID      = "paid"
    SHIPPED   = "shipped"
    CANCELLED = "cancelled"


class OrderCreate(BaseModel):
    user_id:    str
    product_id: str
    quantity:   int
    amount:     float

    @field_validator("quantity")
    @classmethod
    def quantity_positive(cls, v):
        if v <= 0:
            raise ValueError("quantity must be > 0")
        return v

    @field_validator("amount")
    @classmethod
    def amount_positive(cls, v):
        if v <= 0:
            raise ValueError("amount must be > 0")
        return v


class Order(BaseModel):
    id:         str
    user_id:    str
    product_id: str
    quantity:   int
    amount:     float
    status:     OrderStatus
    created_at: datetime


class PaginatedOrders(BaseModel):
    data:        list[Order]
    next_cursor: Optional[str]
    has_more:    bool


class ErrorResponse(BaseModel):
    code:       str
    message:    str
    request_id: str
    field:      Optional[str] = None


# ── In-memory store (replace with asyncpg in production) ──────────────────────
_orders: dict[str, Order] = {}


def _make_request_id() -> str:
    return f"req_{uuid.uuid4().hex[:12]}"


# ── Error helper ──────────────────────────────────────────────────────────────
def api_error(status_code: int, code: str, message: str,
              field: str = None, request_id: str = None):
    return HTTPException(
        status_code=status_code,
        detail=ErrorResponse(
            code=code,
            message=message,
            request_id=request_id or _make_request_id(),
            field=field
        ).model_dump()
    )


# ── Endpoints ─────────────────────────────────────────────────────────────────
@app.post(
    "/v1/orders",
    response_model=Order,
    status_code=status.HTTP_201_CREATED,
    responses={422: {"model": ErrorResponse}}
)
def create_order(body: OrderCreate, response: Response):
    """
    Create a new order.
    Returns 201 with Location header pointing to the new resource.
    """
    order = Order(
        id=str(uuid.uuid4()),
        **body.model_dump(),
        status=OrderStatus.PENDING,
        created_at=datetime.now(UTC)
    )
    _orders[order.id] = order
    response.headers["Location"] = f"/v1/orders/{order.id}"
    return order


@app.get("/v1/orders/{order_id}", response_model=Order)
def get_order(order_id: Annotated[str, Path(description="Order UUID")]):
    """Fetch a single order by ID."""
    order = _orders.get(order_id)
    if not order:
        raise api_error(404, "ORDER_NOT_FOUND", f"Order {order_id} does not exist")
    return order


@app.get("/v1/orders", response_model=PaginatedOrders)
def list_orders(
    status_filter: Optional[OrderStatus] = Query(None, alias="status"),
    limit: int = Query(20, ge=1, le=100),
    after: Optional[str] = Query(None, description="Cursor from previous page"),
):
    """
    List orders with cursor-based pagination.
    Use `after` cursor from previous response to get the next page.
    """
    all_orders = sorted(_orders.values(), key=lambda o: o.created_at)

    # Apply status filter
    if status_filter:
        all_orders = [o for o in all_orders if o.status == status_filter]

    # Apply cursor (cursor = last seen order_id)
    if after:
        try:
            start_idx = next(i for i, o in enumerate(all_orders) if o.id == after) + 1
        except StopIteration:
            raise api_error(400, "INVALID_CURSOR", "Cursor is invalid or expired")
        all_orders = all_orders[start_idx:]

    page = all_orders[:limit]
    has_more = len(all_orders) > limit
    next_cursor = page[-1].id if has_more else None

    return PaginatedOrders(data=page, next_cursor=next_cursor, has_more=has_more)


@app.patch("/v1/orders/{order_id}", response_model=Order)
def update_order_status(order_id: str, new_status: OrderStatus):
    """Partial update — change order status only."""
    order = _orders.get(order_id)
    if not order:
        raise api_error(404, "ORDER_NOT_FOUND", f"Order {order_id} does not exist")

    # State machine guard
    invalid_transitions = {
        OrderStatus.CANCELLED: [OrderStatus.PAID, OrderStatus.SHIPPED],
        OrderStatus.SHIPPED:   [OrderStatus.PENDING],
    }
    blocked = invalid_transitions.get(new_status, [])
    if order.status in blocked:
        raise api_error(
            409, "INVALID_STATUS_TRANSITION",
            f"Cannot transition from {order.status} to {new_status}"
        )

    _orders[order_id] = order.model_copy(update={"status": new_status})
    return _orders[order_id]


@app.post("/v1/orders/{order_id}/cancel", status_code=status.HTTP_204_NO_CONTENT)
def cancel_order(order_id: str):
    """
    Named action endpoint — cancel an order.
    Uses POST /resource/action pattern for non-CRUD state transitions.
    Returns 204 No Content on success.
    """
    order = _orders.get(order_id)
    if not order:
        raise api_error(404, "ORDER_NOT_FOUND", f"Order {order_id} does not exist")
    if order.status in (OrderStatus.SHIPPED,):
        raise api_error(409, "CANNOT_CANCEL", "Shipped orders cannot be cancelled")

    _orders[order_id] = order.model_copy(update={"status": OrderStatus.CANCELLED})


@app.delete("/v1/orders/{order_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_order(order_id: str):
    """Delete an order. Idempotent: deleting a non-existent order returns 204."""
    _orders.pop(order_id, None)   # idempotent — no error if not found
```

---

## Exercise 2: Idempotency Key Middleware

```python
"""
FastAPI middleware that enforces idempotency for POST endpoints.
Stores (Idempotency-Key → response) in Redis with a 24h TTL.
On duplicate key: returns the stored response immediately.
On in-flight duplicate: returns 409 Conflict.
"""
import json
import hashlib
from typing import Callable

import redis.asyncio as aioredis
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware

REDIS_URL = "redis://localhost:6379"
IDEMPOTENCY_TTL = 86_400   # 24 hours


class IdempotencyMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, redis_client: aioredis.Redis,
                 protected_paths: set[str] = None):
        super().__init__(app)
        self.redis = redis_client
        self.protected = protected_paths or set()

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        # Only apply to POST on protected paths
        if request.method != "POST" or request.url.path not in self.protected:
            return await call_next(request)

        idem_key = request.headers.get("Idempotency-Key")
        if not idem_key:
            return JSONResponse(
                status_code=400,
                content={"code": "MISSING_IDEMPOTENCY_KEY",
                         "message": "Idempotency-Key header is required for this endpoint"}
            )

        # Namespace key: path + provided key (prevents cross-endpoint collision)
        redis_key = f"idem:{request.url.path}:{idem_key}"
        lock_key  = f"idem_lock:{request.url.path}:{idem_key}"

        # ── Check for existing result ──────────────────────────────────────
        cached = await self.redis.get(redis_key)
        if cached:
            stored = json.loads(cached)
            return JSONResponse(
                status_code=stored["status_code"],
                content=stored["body"],
                headers={**stored.get("headers", {}), "Idempotency-Replayed": "true"}
            )

        # ── Acquire in-flight lock (NX = only if not exists) ──────────────
        locked = await self.redis.set(lock_key, "1", ex=30, nx=True)
        if not locked:
            return JSONResponse(
                status_code=409,
                content={"code": "REQUEST_IN_FLIGHT",
                         "message": "A request with this idempotency key is already being processed"}
            )

        try:
            # ── Process the request ────────────────────────────────────────
            response = await call_next(request)

            # Collect body (streaming response must be read)
            body_chunks = []
            async for chunk in response.body_iterator:
                body_chunks.append(chunk)
            body_bytes = b"".join(body_chunks)
            body_data  = json.loads(body_bytes) if body_bytes else None

            # ── Store result in Redis ──────────────────────────────────────
            stored = json.dumps({
                "status_code": response.status_code,
                "body":        body_data,
                "headers":     dict(response.headers)
            })
            await self.redis.set(redis_key, stored, ex=IDEMPOTENCY_TTL)

            return Response(
                content=body_bytes,
                status_code=response.status_code,
                headers=dict(response.headers),
                media_type=response.media_type
            )
        finally:
            await self.redis.delete(lock_key)


# ── Usage ─────────────────────────────────────────────────────────────────────
async def lifespan(app):
    app.state.redis = aioredis.from_url(REDIS_URL, decode_responses=True)
    yield
    await app.state.redis.close()

app = FastAPI(lifespan=lifespan)

# Register middleware — protect payment and order creation endpoints
@app.on_event("startup")
async def add_middleware():
    app.add_middleware(
        IdempotencyMiddleware,
        redis_client=app.state.redis,
        protected_paths={"/v1/payments", "/v1/orders"}
    )


@app.post("/v1/payments")
async def create_payment(request: Request):
    body = await request.json()
    return {"payment_id": "pay_123", "status": "charged", "amount": body.get("amount")}
```

---

## Exercise 3: Retry Client with Exponential Backoff + Jitter

```python
"""
HTTP client with retry logic for POST and GET requests.
Demonstrates: which status codes to retry, jitter, idempotency key propagation.
"""
import asyncio
import random
import uuid
from dataclasses import dataclass
from typing import Any, Optional

import httpx


RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504}
NON_RETRYABLE_STATUS_CODES = {400, 401, 403, 404, 409, 410, 422}


@dataclass
class RetryConfig:
    max_attempts:    int   = 5
    base_delay_ms:   float = 100.0
    max_delay_ms:    float = 10_000.0
    jitter:          bool  = True


class ResilientHTTPClient:
    def __init__(self, base_url: str, config: RetryConfig = None):
        self.base_url = base_url
        self.config   = config or RetryConfig()
        self._client  = httpx.AsyncClient(base_url=base_url, timeout=30.0)

    def _delay_ms(self, attempt: int) -> float:
        delay = self.config.base_delay_ms * (2 ** attempt)
        delay = min(delay, self.config.max_delay_ms)
        if self.config.jitter:
            delay = random.uniform(0, delay)
        return delay

    async def get(self, path: str, **kwargs) -> httpx.Response:
        """GET is idempotent — retry freely on transient errors."""
        return await self._request("GET", path, **kwargs)

    async def post(self, path: str, json: Any = None,
                   idempotency_key: Optional[str] = None, **kwargs) -> httpx.Response:
        """
        POST is non-idempotent by default.
        An idempotency_key MUST be provided to safely retry.
        Without it, retries are not performed on 5xx.
        """
        headers = kwargs.pop("headers", {})
        if idempotency_key:
            headers["Idempotency-Key"] = idempotency_key
        return await self._request("POST", path, json=json,
                                   headers=headers,
                                   allow_retry_on_5xx=bool(idempotency_key),
                                   **kwargs)

    async def _request(self, method: str, path: str,
                       allow_retry_on_5xx: bool = True, **kwargs) -> httpx.Response:
        last_exc = None

        for attempt in range(self.config.max_attempts):
            try:
                response = await self._client.request(method, path, **kwargs)

                # Never retry client errors
                if response.status_code in NON_RETRYABLE_STATUS_CODES:
                    return response

                # Successful response
                if response.status_code < 400:
                    return response

                # Handle rate limit — respect Retry-After header
                if response.status_code == 429:
                    retry_after = float(response.headers.get("Retry-After", 1))
                    print(f"  [RetryClient] 429 rate limited — waiting {retry_after}s")
                    await asyncio.sleep(retry_after)
                    continue

                # Retryable server errors
                if response.status_code in RETRYABLE_STATUS_CODES:
                    if not allow_retry_on_5xx and response.status_code == 500:
                        return response  # POST without idempotency key — don't retry 500

                    if attempt < self.config.max_attempts - 1:
                        delay = self._delay_ms(attempt) / 1000
                        print(f"  [RetryClient] {response.status_code} on attempt {attempt+1} "
                              f"— retrying in {delay:.2f}s")
                        await asyncio.sleep(delay)
                        continue

                return response

            except (httpx.ConnectError, httpx.TimeoutException) as e:
                last_exc = e
                if attempt < self.config.max_attempts - 1:
                    delay = self._delay_ms(attempt) / 1000
                    print(f"  [RetryClient] Network error on attempt {attempt+1}: {e} "
                          f"— retrying in {delay:.2f}s")
                    await asyncio.sleep(delay)
                else:
                    raise

        if last_exc:
            raise last_exc

    async def aclose(self):
        await self._client.aclose()


async def demo():
    client = ResilientHTTPClient("https://httpbin.org")

    # GET with automatic retry
    print("=== GET with retry ===")
    resp = await client.get("/get")
    print(f"  Status: {resp.status_code}")

    # POST with idempotency key (safe to retry)
    print("\n=== POST with idempotency key (safe retry) ===")
    key = str(uuid.uuid4())
    resp = await client.post(
        "/post",
        json={"amount": 99.99},
        idempotency_key=key
    )
    print(f"  Status: {resp.status_code}, Key: {key}")

    # Same key again — server deduplicates
    resp2 = await client.post(
        "/post",
        json={"amount": 99.99},
        idempotency_key=key
    )
    print(f"  Retry status: {resp2.status_code}, Replayed: {resp2.headers.get('Idempotency-Replayed')}")

    await client.aclose()


if __name__ == "__main__":
    asyncio.run(demo())
```

---

## Exercise 4: GraphQL with DataLoader (N+1 Prevention)

```python
"""
GraphQL API using Strawberry with DataLoader to prevent N+1 queries.
Models: User → Orders relationship.
"""
import strawberry
from strawberry.fastapi import GraphQLRouter
from strawberry.dataloader import DataLoader
from fastapi import FastAPI
from typing import Optional
import asyncio

# ── Fake DB ───────────────────────────────────────────────────────────────────
USERS = {
    "1": {"id": "1", "email": "alice@example.com"},
    "2": {"id": "2", "email": "bob@example.com"},
    "3": {"id": "3", "email": "carol@example.com"},
}
ORDERS = [
    {"id": "o1", "user_id": "1", "amount": 99.0,  "status": "paid"},
    {"id": "o2", "user_id": "1", "amount": 45.0,  "status": "pending"},
    {"id": "o3", "user_id": "2", "amount": 120.0, "status": "shipped"},
]
query_log: list[str] = []   # Track DB queries for demo


# ── DataLoader batch function ─────────────────────────────────────────────────
async def load_orders_for_users(user_ids: list[str]) -> list[list[dict]]:
    """
    Called ONCE per request with ALL user_ids collected.
    Replaces N individual SELECT queries with a single IN query.
    """
    query_log.append(f"SELECT * FROM orders WHERE user_id IN {tuple(user_ids)}")
    print(f"  [DB] Batch query for users: {user_ids}  ← only ONE query!")
    result = {}
    for uid in user_ids:
        result[uid] = [o for o in ORDERS if o["user_id"] == uid]
    return [result.get(uid, []) for uid in user_ids]


# ── Strawberry types ──────────────────────────────────────────────────────────
@strawberry.type
class Order:
    id:     str
    amount: float
    status: str


@strawberry.type
class User:
    id:    str
    email: str

    @strawberry.field
    async def orders(self, info: strawberry.types.Info) -> list[Order]:
        # DataLoader batches this across ALL users resolved in the same request
        raw = await info.context["order_loader"].load(self.id)
        return [Order(**o) for o in raw]


@strawberry.type
class Query:
    @strawberry.field
    async def users(self) -> list[User]:
        query_log.append("SELECT * FROM users")
        print("  [DB] SELECT * FROM users")
        return [User(**u) for u in USERS.values()]

    @strawberry.field
    async def user(self, id: str) -> Optional[User]:
        u = USERS.get(id)
        return User(**u) if u else None


# ── Schema + App ──────────────────────────────────────────────────────────────
schema = strawberry.Schema(query=Query)

async def get_context():
    return {
        "order_loader": DataLoader(load_fn=load_orders_for_users)
    }

graphql_router = GraphQLRouter(schema, context_getter=get_context)

app = FastAPI()
app.include_router(graphql_router, prefix="/graphql")

# Test query to run:
# query {
#   users {
#     email
#     orders { id amount status }
#   }
# }
# Without DataLoader: 1 + 3 = 4 queries
# With DataLoader:    1 + 1 = 2 queries
```

---

## Exercise 5: gRPC Service — Order Management

```python
# order_service.proto
"""
syntax = "proto3";
package orders;

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (OrderResponse);
  rpc GetOrder(GetOrderRequest) returns (OrderResponse);
  rpc ListOrders(ListOrdersRequest) returns (stream OrderResponse);  // Server streaming
}

message CreateOrderRequest {
  string user_id    = 1;
  string product_id = 2;
  int32  quantity   = 3;
  double amount     = 4;
}
message GetOrderRequest {
  string order_id = 1;
}
message ListOrdersRequest {
  string user_id = 1;
  int32 limit    = 2;
}
message OrderResponse {
  string order_id   = 1;
  string user_id    = 2;
  string product_id = 3;
  int32  quantity   = 4;
  double amount     = 5;
  string status     = 6;
}
"""

# ─── Server implementation (grpcio + grpcio-tools) ────────────────────────────
# pip install grpcio grpcio-tools
# python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. order_service.proto

import grpc
import uuid
import asyncio
from concurrent import futures

# Assuming generated stubs exist:
# import order_service_pb2 as pb2
# import order_service_pb2_grpc as pb2_grpc

# ── Fake implementation (shows structure without generated code dependency) ────
class OrderServiceImpl:
    """gRPC service implementation."""
    _orders: dict = {}

    def CreateOrder(self, request, context):
        order_id = str(uuid.uuid4())
        self._orders[order_id] = {
            "order_id":   order_id,
            "user_id":    request.user_id,
            "product_id": request.product_id,
            "quantity":   request.quantity,
            "amount":     request.amount,
            "status":     "pending"
        }
        print(f"[gRPC Server] Created order {order_id}")
        # Return pb2.OrderResponse(**self._orders[order_id])
        return self._orders[order_id]

    def GetOrder(self, request, context):
        order = self._orders.get(request.order_id)
        if not order:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details(f"Order {request.order_id} not found")
            return {}
        return order

    def ListOrders(self, request, context):
        """Server-streaming: yields orders one by one."""
        user_orders = [o for o in self._orders.values()
                       if o["user_id"] == request.user_id]
        for order in user_orders[:request.limit or 20]:
            yield order   # Streamed to client


def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    # pb2_grpc.add_OrderServiceServicer_to_server(OrderServiceImpl(), server)
    server.add_insecure_port("[::]:50051")
    server.start()
    print("[gRPC Server] Listening on :50051")
    server.wait_for_termination()


# ── Client ────────────────────────────────────────────────────────────────────
def grpc_client_demo():
    channel = grpc.insecure_channel("localhost:50051")
    # stub = pb2_grpc.OrderServiceStub(channel)

    # Unary call
    # response = stub.CreateOrder(pb2.CreateOrderRequest(
    #     user_id="alice", product_id="laptop", quantity=1, amount=999.0
    # ))

    # Server streaming call
    # for order in stub.ListOrders(pb2.ListOrdersRequest(user_id="alice", limit=10)):
    #     print(f"  Streamed order: {order.order_id}")

    channel.close()
```

---

## Exercise 6: API Versioning — URI Path + Sunset Headers

```python
"""
Demonstrates URI path versioning with:
- /v1/ endpoint (deprecated)
- /v2/ endpoint (current)
- Sunset header on deprecated responses
- Version routing in FastAPI
"""
from datetime import datetime, UTC
from fastapi import FastAPI, Response, APIRouter

app  = FastAPI()
v1   = APIRouter(prefix="/v1")
v2   = APIRouter(prefix="/v2")

SUNSET_DATE = "Sat, 31 Dec 2026 23:59:59 GMT"


def add_deprecation_headers(response: Response):
    """Add RFC 8594 deprecation headers to indicate endpoint age."""
    response.headers["Deprecation"] = "true"
    response.headers["Sunset"]      = SUNSET_DATE
    response.headers["Link"]        = '</v2/users>; rel="successor-version"'


# ── v1 — Old shape (deprecated) ───────────────────────────────────────────────
@v1.get("/users/{user_id}")
def get_user_v1(user_id: str, response: Response):
    add_deprecation_headers(response)
    # v1 returns a flat structure with underscore-named fields
    return {
        "user_id":    user_id,
        "user_email": "alice@example.com",   # breaking change in v2: renamed to 'email'
        "full_name":  "Alice Smith",         # split into first_name/last_name in v2
    }


# ── v2 — New shape (current) ──────────────────────────────────────────────────
@v2.get("/users/{user_id}")
def get_user_v2(user_id: str):
    # v2 uses cleaner field names and richer structure
    return {
        "id":         user_id,
        "email":      "alice@example.com",   # renamed from user_email
        "first_name": "Alice",
        "last_name":  "Smith",
        "created_at": "2024-01-15T10:30:00Z",
    }


app.include_router(v1)
app.include_router(v2)


# ── Version detection middleware (header-based alternative) ───────────────────
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request as StarletteRequest

class VersionNegotiationMiddleware(BaseHTTPMiddleware):
    """
    Allow clients to specify version via Accept header as an alternative to URL.
    Accept: application/vnd.myapi.v2+json → routes to v2 handler
    """
    async def dispatch(self, request: StarletteRequest, call_next):
        accept = request.headers.get("Accept", "")
        if "vnd.myapi.v1" in accept and not request.url.path.startswith("/v"):
            # Rewrite path to v1
            request.scope["path"] = "/v1" + request.url.path
        elif "vnd.myapi.v2" in accept and not request.url.path.startswith("/v"):
            request.scope["path"] = "/v2" + request.url.path
        return await call_next(request)
```
