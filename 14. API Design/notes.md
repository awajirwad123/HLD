# API Design — Notes & Reference

## REST Cheat Sheet

### HTTP Method One-Liners

| Method | Safe | Idempotent | Typical Status | Use |
|--------|:----:|:----------:|----------------|-----|
| GET    | ✅   | ✅         | 200            | Read resource |
| POST   | ❌   | ❌         | 201 + Location | Create / named action |
| PUT    | ❌   | ✅         | 200 or 204     | Replace entire resource |
| PATCH  | ❌   | ❌*        | 200            | Partial update |
| DELETE | ❌   | ✅         | 204            | Delete (idempotent: 204 even if already gone) |

*PATCH idempotency depends on operation: `SET field=X` is idempotent; `INCREMENT field` is not.

### Status Code Decision Tree

```
Request succeeded?
├── YES
│   ├── Response has body?        → 200 OK
│   ├── New resource created?     → 201 Created  (+ Location header)
│   ├── Async job started?        → 202 Accepted (+ job_id in body)
│   └── No body needed?           → 204 No Content
└── NO
    ├── Client mistake?
    │   ├── Bad syntax/params     → 400 Bad Request
    │   ├── Not authenticated     → 401 Unauthorized
    │   ├── Authenticated, no permission → 403 Forbidden
    │   ├── Resource missing      → 404 Not Found
    │   ├── Resource permanently gone → 410 Gone
    │   ├── State conflict        → 409 Conflict
    │   ├── Semantic error        → 422 Unprocessable Entity
    │   └── Rate limited          → 429 Too Many Requests
    └── Server mistake?
        ├── Unexpected error      → 500 Internal Server Error
        ├── Upstream bad response → 502 Bad Gateway
        ├── Server overloaded     → 503 Service Unavailable
        └── Upstream timed out   → 504 Gateway Timeout
```

### URL Design Rules

```
✅  /users                     Collection
✅  /users/42                  Single resource
✅  /users/42/orders           Sub-resource (orders belonging to user)
✅  /orders?status=pending     Filter via query param
✅  /orders?sort=created_at&order=desc  Sort
✅  /orders?limit=20&after=cursor       Cursor pagination
✅  /orders/42/cancel          Named action (state machine trigger) — POST

❌  /getUsers                  Verb in path
❌  /user                      Singular (use plural for collections)
❌  /orders/42/getStatus        CRUD verb in action path
❌  /orders?page=2              Offset pagination (use cursor)
```

---

## REST vs GraphQL vs gRPC — Decision Card

| Situation | Choose |
|-----------|--------|
| Public API consumed by unknown third parties | **REST** (universal compatibility) |
| Mobile app fetching data from multiple services | **GraphQL** (exact field selection, single request) |
| Internal service-to-service in same cluster | **gRPC** (binary, low latency, strong types) |
| Audio/video streaming, real-time bidirectional | **gRPC** (native streaming) |
| Simple CRUD with browser-based frontend | **REST** |
| API for data-heavy dashboard with many optional fields | **GraphQL** |
| Strict schema enforcement, code generation required | **gRPC** |
| Needs to be callable from curl / browser devtools | **REST** or **GraphQL** |

---

## Versioning Strategy Comparison

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URI path | `/v1/users` | Visible, easy to route, CDN-friendly | Version in URL "pollutes" resource identity |
| Header | `X-API-Version: 2` | Clean URLs | Not visible in browser/curl; easy to miss |
| Accept header | `Accept: application/vnd.myapi.v2+json` | REST-pure | Verbose, complex, niche tooling |
| Query param | `?version=2` | Easy to test | Mixing filter and version semantics |

**Production recommendation:** URI path versioning (`/v1/`, `/v2/`) — most commonly understood, easiest to route in nginx/ALB, appears in logs unambiguously.

### Deprecation Response Headers (RFC 8594)

```
Deprecation: true
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Link: </v2/users>; rel="successor-version"
```

---

## Idempotency Quick Reference

### Which Operations Need Idempotency Keys?

```
POST /payments         ← YES — charging twice = disaster
POST /orders           ← YES — creating twice = duplicate order
POST /subscriptions    ← YES — subscribing twice = billing error
POST /notifications    ← Optional — sending twice is annoying but not critical
GET  /anything         ← No — inherently idempotent
PUT  /anything         ← No — inherently idempotent (replacing with same data = same result)
DELETE /anything       ← No — inherently idempotent
```

### Idempotency Key Storage

```
Redis key:   idem:{endpoint_path}:{client_provided_uuid}
Redis value: {status_code: 201, body: {...}}
TTL:         24 hours (Stripe uses 24h; adjust per use case)

Race condition (same key arrives twice simultaneously):
  → Use SET NX (only if not exists) lock key with 30s TTL
  → First request gets lock, processes, stores result, releases lock
  → Second request: no lock → 409 Conflict
  → After first finishes: second retries → finds stored result → returns it
```

---

## Retry Strategy Cheat Sheet

### When to Retry

| Status Code | Retry? | Notes |
|-------------|--------|-------|
| 400 | ❌ Never | Client bug — fix the request |
| 401 | ❌ Never | Get a new token first |
| 403 | ❌ Never | Permission won't change |
| 404 | ❌ Never | Resource won't appear |
| 422 | ❌ Never | Fix the semantic error |
| 429 | ✅ After Retry-After | Respect the header delay |
| 500 | ✅ Only with idempotency key | Server may have partially processed |
| 502 | ✅ Yes | Upstream transient error |
| 503 | ✅ Yes | Server temporarily overloaded |
| 504 | ✅ Yes | Upstream timeout |
| Network error | ✅ Yes | GET freely; POST only with idempotency key |

### Jitter Formula

```python
# Full jitter (recommended — best desynchronization)
delay = random.uniform(0, min(base * 2**attempt, max_delay))

# Equal jitter (keeps some minimum spacing)
half = min(base * 2**attempt, max_delay) / 2
delay = half + random.uniform(0, half)
```

---

## API Security Checklist

| Concern | Best Practice |
|---------|--------------|
| Authentication | OAuth2 / OIDC with JWT Bearer tokens |
| Machine-to-machine | API keys — rotate on schedule, scope to minimum permissions |
| Never | API keys in URLs (`?api_key=...` — logged by proxies, shared in browser history) |
| Authorization | Check permissions per-resource, not just per-endpoint |
| Input validation | Validate all inputs at API boundary (Pydantic, Joi, FastAPI) |
| Rate limiting | Per-user + per-IP; return 429 with Retry-After |
| HTTPS | TLS everywhere; HSTS header; no HTTP-only endpoints |
| CORS | Allowlist known origins; never `*` in production |
| Request size | Set max body size limits to prevent upload-based DoS |

---

## Key Numbers to Know

| Metric | Typical Value |
|--------|--------------|
| REST API P99 latency (well-optimized) | < 100ms |
| gRPC vs REST payload size (same data) | gRPC ~30–60% smaller (Protobuf vs JSON) |
| GraphQL DataLoader batch time | < 5ms (single event-loop tick) |
| Idempotency key TTL (Stripe) | 24 hours |
| API version deprecation window (internal) | 6 months minimum |
| API version deprecation window (public) | 12 months minimum |
| Max recommended cursor page size | 100–500 rows |
