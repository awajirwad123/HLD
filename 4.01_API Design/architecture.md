# API Design — Architecture Deep-Dive

## The Core Problem

An API is a contract between a producer and a consumer. A poorly designed API:
- Forces clients to make multiple round-trips for a single UX action
- Breaks existing clients on every server change
- Makes incorrect behavior easy (non-idempotent retries, silent data loss)
- Cannot be extended without a full rewrite

Good API design makes the correct usage obvious and the incorrect usage impossible or explicit.

---

## 1. REST — Principles and Best Practices

### The Six Constraints (Roy Fielding, 2000)

| Constraint | Meaning |
|-----------|---------|
| **Client-Server** | UI and data storage are decoupled — each evolves independently |
| **Stateless** | Each request contains all context needed; server stores no session state |
| **Cacheable** | Responses must declare whether they can be cached |
| **Uniform Interface** | Resource identification via URIs, manipulation via representations, self-descriptive messages, HATEOAS |
| **Layered System** | Client cannot tell whether it's talking to origin server or intermediary (proxy, CDN, LB) |
| **Code on Demand** (optional) | Server can send executable code (JavaScript) to clients |

### Resource Modelling

REST resources are **nouns**, not verbs. HTTP methods express the action.

```
❌ POST /createUser
❌ GET  /getUserById?id=42
❌ POST /deleteOrder/5

✅ POST   /users              (create)
✅ GET    /users/42           (read)
✅ PUT    /users/42           (replace)
✅ PATCH  /users/42           (partial update)
✅ DELETE /orders/5           (delete)
```

### HTTP Method Semantics

| Method | Safe | Idempotent | Body | Use |
|--------|:----:|:----------:|------|-----|
| GET | ✅ | ✅ | No | Read resource |
| HEAD | ✅ | ✅ | No | Read headers only |
| POST | ❌ | ❌ | Yes | Create / non-idempotent action |
| PUT | ❌ | ✅ | Yes | Replace entire resource |
| PATCH | ❌ | ❌* | Yes | Partial update |
| DELETE | ❌ | ✅ | Opt | Delete resource |

*PATCH is not inherently idempotent — depends on implementation (set vs increment)

### HTTP Status Codes — Correct Usage

```
2xx  Success
  200 OK           — read / update success with body
  201 Created      — POST that created a resource (include Location header)
  202 Accepted     — async operation started (return operation_id)
  204 No Content   — success with no response body (DELETE, some PATCHs)

3xx  Redirection
  301 Moved Permanently  — permanent redirect (update bookmarks)
  304 Not Modified       — cached response still valid (ETag / If-None-Match)

4xx  Client Error
  400 Bad Request        — malformed syntax, invalid parameters
  401 Unauthorized       — not authenticated (missing/invalid token)
  403 Forbidden          — authenticated but not authorized
  404 Not Found          — resource doesn't exist
  409 Conflict           — resource state conflict (duplicate key, optimistic lock)
  410 Gone               — resource permanently deleted
  422 Unprocessable Entity — syntactically valid but semantically invalid
  429 Too Many Requests  — rate limit exceeded (include Retry-After header)

5xx  Server Error
  500 Internal Server Error — unexpected server failure
  502 Bad Gateway           — upstream service returned invalid response
  503 Service Unavailable   — server overloaded or in maintenance
  504 Gateway Timeout       — upstream service timed out
```

### URL Design Rules

```
Use plural nouns for collections:
  /users          (collection)
  /users/42       (single resource)
  /users/42/orders (sub-resource)

Use kebab-case (never camelCase or snake_case in URLs):
  ✅ /order-items
  ❌ /orderItems
  ❌ /order_items

Filter/sort/paginate via query params:
  GET /orders?status=pending&sort=created_at&page=2&limit=20

Actions that don't fit CRUD — use sub-resources or verbs explicitly:
  POST /orders/42/cancel    (trigger state machine action)
  POST /payments/42/refund
  POST /auth/refresh-token
```

### Pagination Patterns

| Type | Format | Trade-off |
|------|--------|-----------|
| **Offset/Limit** | `?page=2&limit=20` | Simple; inconsistent when rows inserted during paging |
| **Cursor** | `?after=eyJpZCI6MTAwfQ==` (opaque base64 ID) | Stable under concurrent writes; can't jump to arbitrary page |
| **Keyset** | `?after_id=100&after_created=2024-01-01` | Efficient (uses index); tied to sort order |

**Cursor pagination is preferred for production APIs** — offset-based pagination performs a full `OFFSET N` scan and is unstable under concurrent writes.

### Response Envelope vs Bare Resource

```json
// Bare resource (preferred for simple APIs)
GET /users/42
{
  "id": 42,
  "email": "alice@example.com"
}

// Envelope (useful when metadata needed — pagination, errors)
GET /users
{
  "data": [...],
  "meta": { "total": 1042, "next_cursor": "abc123" }
}

// Error shape (always consistent)
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "email must be a valid email address",
    "field": "email",
    "request_id": "req_abc123"
  }
}
```

---

## 2. GraphQL

### Core Concepts

- **Schema-first:** Everything defined in a SDL (Schema Definition Language) type system
- **Single endpoint:** `POST /graphql` — clients declare exactly what they want
- **Resolver tree:** Each field resolved independently; root resolvers → nested resolvers

```graphql
# Schema
type User {
  id:     ID!
  email:  String!
  orders: [Order!]!
}
type Order {
  id:      ID!
  total:   Float!
  status:  String!
}
type Query {
  user(id: ID!): User
}

# Client query — fetches exactly these fields
query {
  user(id: "42") {
    email
    orders {
      id
      total
    }
  }
}
```

### GraphQL Problems to Know

**N+1 Problem:**
Resolving `orders` for each user fires a separate DB query per user.
```
Query: users(limit: 10) { orders { id } }
→ 1 query: SELECT * FROM users LIMIT 10
→ 10 queries: SELECT * FROM orders WHERE user_id = ?  ← N+1
```
**Fix: DataLoader** — batches and deduplicates resolver calls within a single request tick.

```python
from strawberry.dataloader import DataLoader

async def load_orders_by_user(user_ids: list[str]) -> list[list[Order]]:
    rows = await db.fetch("SELECT * FROM orders WHERE user_id = ANY($1)", user_ids)
    by_user = {uid: [] for uid in user_ids}
    for row in rows:
        by_user[row["user_id"]].append(row)
    return [by_user[uid] for uid in user_ids]

order_loader = DataLoader(load_fn=load_orders_by_user)
```

**Over-fetching / Under-fetching:**
GraphQL solves both: clients request exactly the fields they need.

**Query complexity / depth attacks:**
A malicious client can send deeply nested queries that explode resolver execution.
```graphql
# Denial of service via deep nesting
{ user { friends { friends { friends { friends { ... } } } } } }
```
**Fix:** Query complexity scoring + depth limiting at the gateway.

---

## 3. gRPC

### Core Concepts

- **Protocol Buffer (Protobuf)** schema defines services and messages
- **HTTP/2** transport — multiplexing, binary framing, header compression
- **Code generation** — client and server stubs generated from `.proto` files
- **4 streaming modes:** Unary, Server Streaming, Client Streaming, Bidirectional Streaming

```protobuf
syntax = "proto3";

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (Order);              // Unary
  rpc ListOrders(ListOrdersRequest) returns (stream Order);         // Server stream
  rpc UploadItems(stream OrderItem) returns (Order);                // Client stream
  rpc TrackOrder(stream TrackRequest) returns (stream TrackEvent);  // Bidirectional
}

message CreateOrderRequest {
  string user_id    = 1;
  string product_id = 2;
  int32  quantity   = 3;
  double amount     = 4;
}
```

### gRPC vs REST vs GraphQL Comparison

| Dimension | REST | GraphQL | gRPC |
|-----------|------|---------|------|
| **Protocol** | HTTP/1.1 or HTTP/2 | HTTP/1.1 or HTTP/2 | HTTP/2 only |
| **Payload format** | JSON (text) | JSON (text) | Protobuf (binary) |
| **Schema** | Optional (OpenAPI) | Required (SDL) | Required (.proto) |
| **Code generation** | Optional | Optional | Required |
| **Streaming** | SSE / WebSocket (bolt-on) | Subscriptions | Native (4 modes) |
| **Browser support** | ✅ Native | ✅ Native | ⚠️ Requires grpc-web proxy |
| **Payload size** | Larger (verbose JSON) | Medium (exact fields) | Smallest (binary) |
| **Latency** | Medium | Medium | Lowest |
| **Type safety** | Optional (OpenAPI codegen) | Strong (SDL) | Strongest (proto) |
| **Best for** | Public APIs, CRUD, mobile | Mobile with complex data graphs, BFF | Internal microservices, streaming, high-perf |

### When to Use Each

```
Public-facing REST API (mobile app, third-party developers):
  → REST — universal compatibility, easy to explore, OpenAPI docs

Complex data aggregation for a single frontend:
  → GraphQL — eliminates over-fetching, BFF pattern

Internal microservice-to-microservice:
  → gRPC — lowest latency, binary efficiency, strong contracts, bidirectional streaming

Real-time data push to browser:
  → REST + SSE or WebSockets (gRPC needs grpc-web adapter)
```

---

## 4. API Versioning Strategies

### Why Versioning Matters

Once an API is public, clients depend on its exact shape. Changes that break clients = **breaking changes**:
- Removing a field
- Renaming a field
- Changing a field's type
- Changing HTTP method for an endpoint
- Changing status codes

### Versioning Approaches

#### URI Path Versioning (Most Common)

```
GET /v1/users/42
GET /v2/users/42
```

**Pros:** Explicit, easy to route at CDN/LB level, obvious in logs.
**Cons:** URI is supposed to identify a resource — version pollutes resource identity. Old versions must be maintained indefinitely.

#### Header Versioning

```
GET /users/42
Accept: application/vnd.myapi.v2+json
```
or
```
X-API-Version: 2
```

**Pros:** Clean URLs, can negotiate version per client.
**Cons:** Not visible in URL (harder to debug, share links); not supported by all HTTP clients.

#### Query Parameter Versioning

```
GET /users/42?version=2
```

**Pros:** Easy to test in browser.
**Cons:** Query params should be for filtering — mixing versioning is messy. Easy to forget.

#### Content Negotiation (Most REST-pure)

```
Accept: application/vnd.company.resource-v2+json
Content-Type: application/vnd.company.resource-v2+json
```

**Pros:** True REST, version negotiated via media type.
**Cons:** Verbose, complex, rarely used in practice.

### Non-Breaking vs Breaking Changes

```
Non-breaking (additive — safe):
  ✅ Add a new optional field to response
  ✅ Add a new optional request parameter
  ✅ Add a new endpoint
  ✅ Add a new enum value (caution: clients must handle unknown values)

Breaking (requires new version):
  ❌ Remove or rename a field
  ❌ Change a field's type (string → int)
  ❌ Change HTTP method
  ❌ Change URL structure
  ❌ Remove an endpoint
  ❌ Change error codes
```

### Version Lifecycle Best Practice

1. Announce deprecation in response header: `Deprecation: true`, `Sunset: Sat, 31 Dec 2025 23:59:59 GMT`
2. Log usage of deprecated endpoints to track which clients still use them
3. Enforce sunset: return 410 Gone after sunset date
4. Minimum deprecation window: 6 months for internal, 12 months for public APIs

---

## 5. Idempotency and Retries

### Idempotency Defined

An operation is **idempotent** if applying it multiple times produces the same result as applying it once.

```
GET  /users/42      → Idempotent (reads don't change state)
PUT  /users/42      → Idempotent (setting the same resource twice = same state)
DELETE /orders/5    → Idempotent (deleting an already-deleted resource = same end state)
POST /orders        → NOT idempotent (creates a new order each time)
POST /payments      → NOT idempotent (charges the card each time) ← CRITICAL
```

### Idempotency Keys

For non-idempotent operations (POST payments, POST orders), clients provide an **idempotency key** — a unique UUID generated once per logical operation. The server stores `(key → result)` and returns the cached result on duplicate requests.

```
Client                         Server
  │                               │
  │  POST /payments               │
  │  Idempotency-Key: uuid-abc    │
  │ ────────────────────────────► │  First request: process + store result
  │ ◄──── 201 Created ─────────── │  {payment_id: "pay123"}
  │                               │
  │  (network timeout — retry)    │
  │                               │
  │  POST /payments               │
  │  Idempotency-Key: uuid-abc    │
  │ ────────────────────────────► │  Duplicate: return stored result
  │ ◄──── 201 Created ─────────── │  {payment_id: "pay123"}  ← same response
  │                               │
  │  (entirely new payment)       │
  │                               │
  │  POST /payments               │
  │  Idempotency-Key: uuid-xyz    │  New key → new operation
  │ ────────────────────────────► │
  │ ◄──── 201 Created ─────────── │  {payment_id: "pay456"}
```

**Storage:** Redis with TTL (24h–7 days) for idempotency key storage. Key: the idempotency key UUID. Value: `{status_code, response_body}`.

**Conflict handling:** If a request with the same key arrives while a prior request with that key is still in-flight, return `409 Conflict` rather than processing twice.

### Retry Strategies

| Strategy | Formula | Use Case |
|----------|---------|----------|
| **Fixed** | `wait = N ms` | Simple, predictable |
| **Exponential backoff** | `wait = base * 2^attempt` | Standard — prevents retry storms |
| **Exponential + jitter** | `wait = random(0, base * 2^attempt)` | Best — desynchronizes concurrent retries |

```python
import random, time

def retry_with_jitter(fn, max_attempts=5, base_ms=100):
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError as e:
            if attempt == max_attempts - 1:
                raise
            wait = random.uniform(0, base_ms * (2 ** attempt)) / 1000
            time.sleep(wait)
```

### What to Retry

```
✅ Retry:
  - 429 Too Many Requests (after Retry-After header delay)
  - 503 Service Unavailable
  - 504 Gateway Timeout
  - Network timeouts / connection refused

❌ Never retry:
  - 400 Bad Request (client bug — retrying won't fix it)
  - 401 Unauthorized (need new token first)
  - 403 Forbidden (permission won't change between retries)
  - 404 Not Found (resource won't appear between retries)
  - 422 Unprocessable Entity

⚠️ Retry only with idempotency key:
  - 500 Internal Server Error (server may have partially processed)
  - POST requests in general
```

### The Retry-After Header

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1712345678
```

Clients must respect `Retry-After` — retrying immediately on 429 is itself a denial-of-service.

---

## API Design Standards Summary

| Concern | Best Practice |
|---------|--------------|
| Resource naming | Plural nouns, kebab-case, hierarchical |
| HTTP methods | Use semantically correct methods (GET idempotent, POST creates) |
| Status codes | Use correct 4xx/5xx; include `request_id` in all error responses |
| Pagination | Cursor-based for production; offset-based acceptable for admin tools |
| Versioning | URI path versioning (`/v1/`) with Sunset header for deprecation |
| Idempotency | Idempotency-Key header for all POST operations that mutate state |
| Retries | Exponential backoff with jitter on 429/503/504 only |
| Authentication | OAuth2 + JWT Bearer tokens; API keys for M2M; never in URL |
| Rate limiting | 429 with Retry-After header; per-user and per-IP limits |
| Documentation | OpenAPI/Swagger for REST; SDL for GraphQL; `.proto` for gRPC |
