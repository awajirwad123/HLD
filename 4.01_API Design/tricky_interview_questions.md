# API Design — Tricky Interview Questions

## Q1: "REST says everything should be a resource. What do you do when your operation doesn't fit — like 'send password reset email' or 'bulk delete'?"

**Why it's tricky:** Strict REST purists and pragmatists will give very different answers. The interviewer is testing whether you can reason beyond the textbook.

**Strong Answer:**

Two valid approaches, and the choice depends on how much you value REST purity vs practical clarity.

**Option A — Model the action as a resource (REST-pure):**
```
POST /password-reset-requests   { email: "alice@example.com" }
→ 201 Created: { id: "prr_123", status: "pending", expires_at: "..." }

POST /bulk-deletions   { order_ids: [1, 2, 3] }
→ 202 Accepted: { job_id: "job_456", status: "processing" }
GET  /bulk-deletions/job_456     ← poll for result
```

**Option B — Named action sub-resource (pragmatic):**
```
POST /users/42/password-reset
POST /orders/bulk-delete   { ids: [1, 2, 3] }
```

In practice, Option B is more readable and immediately obvious to a developer consuming the API. Option A is more "correct" by Fielding's constraints but adds cognitive overhead for simple actions.

The key rule: whatever you choose, be **consistent across your entire API**. Mixed styles (REST resources here, verb endpoints there) are worse than either pure approach.

---

## Q2: "PATCH is idempotent in theory. Explain a case where PATCH is NOT idempotent and what you do about it."

**Why it's tricky:** Most candidates say "PATCH is idempotent" without thinking about what the patch operation actually does.

**Strong Answer:**

PATCH is idempotent only when the operation is a **set** (set field X to value Y). It is NOT idempotent when the operation is a **relative change**:

```python
# Idempotent PATCH — calling twice gives same result
PATCH /accounts/1   {"balance": 100}   # Sets balance to 100
PATCH /accounts/1   {"balance": 100}   # Sets balance to 100 again — same result

# Non-idempotent PATCH — calling twice gives different result
PATCH /accounts/1   {"balance": {"$increment": 50}}   # +50 → 150
PATCH /accounts/1   {"balance": {"$increment": 50}}   # +50 → 200 ← different!
```

The `$increment` operation is the classic case — appending to an array, incrementing a counter, etc.

**Solutions:**
1. Avoid relative operations in PATCH — express them as a new endpoint (`POST /accounts/1/deposits`) that is POST (explicitly non-idempotent and documents the intent).
2. If you allow relative operations, require an idempotency key so duplicate requests are detected and ignored.
3. Use optimistic locking with `If-Match: {etag}` — the server rejects a request if the resource has changed since the client last read it, preventing unintended double-application.

---

## Q3: "The engineering manager says 'Just use GraphQL for everything — it's more flexible than REST.' How do you respond?"

**Why it's tricky:** Candidates often capitulate or swing to the other extreme. The right answer is nuanced.

**Strong Answer:**

GraphQL is excellent for specific scenarios but brings real costs that make "use it for everything" a bad policy.

**GraphQL strengths:** Eliminates over/under-fetching, ideal for complex data graphs, excellent for a BFF (Backend for Frontend) aggregating multiple services for a mobile app.

**GraphQL weaknesses in a "use everywhere" policy:**

1. **HTTP caching is broken.** GraphQL queries are POST to a single URL (`/graphql`). CDNs and browser caches cache by URL + HTTP method — every query looks identical. REST GET endpoints cache trivially by URL. For public read-heavy APIs, this is a significant performance regression.

2. **N+1 attack surface.** Every deeply nested query is a potential DB explosion. DataLoader mitigates it but requires careful implementation at every resolver. A junior developer adding a new resolver field can silently introduce N+1 without knowing.

3. **Query complexity attacks.** Without query depth/complexity limiting, a malicious client can crash your API with a single deeply-nested query. REST endpoints have fixed cost.

4. **Operational overhead.** Schema-first, resolvers, DataLoader, complexity scoring, persisted queries — GraphQL adds significant tooling and maintenance burden that is unjustified for simple CRUD.

**My recommendation:** Use GraphQL at the BFF layer (mobile/web gateway) where multiple services need aggregation and field selection matters. Use REST for service-to-service, public APIs, and simple CRUD. Use gRPC for internal high-performance service mesh.

---

## Q4: "You have a payment API. A client sends a POST /payments request, gets a timeout, and doesn't know if the payment was processed. Walk me through the full solution."

**Why it's tricky:** Tests end-to-end understanding of idempotency, timeout handling, and client retry design all together.

**Strong Answer:**

The root problem: the server may have committed the charge or may have crashed before committing. Without idempotency, retrying creates a double charge.

**Complete solution:**

**Step 1 — Client generates idempotency key BEFORE the first request:**
```python
key = str(uuid.uuid4())    # Generated once, stored client-side
```

**Step 2 — Client sends with key, stores key + expected amount:**
```
POST /payments
Idempotency-Key: uuid-abc123
{ amount: 99.99, card_token: "tok_..." }
```

**Step 3 — Timeout occurs. Client retries with the SAME key:**
```
POST /payments
Idempotency-Key: uuid-abc123   ← same key
{ amount: 99.99, card_token: "tok_..." }
```

**Step 4 — Server behavior on duplicate key:**
- Found stored result → return `{payment_id: "pay_123", status: "succeeded"}` with `Idempotency-Replayed: true` header
- Key in flight (being processed) → return 409 (retry after a moment)
- Not found → process normally, store result

**Step 5 — Client validates returned payment_id:**
The client should verify the returned `amount` matches the expected amount — different amounts for the same key = request mismatch → return 422.

**Step 6 — Idempotency key storage:**
Redis with 24h TTL. Key: `idem:{endpoint}:{uuid}`. Value: serialized response.

**Edge case: request body mismatch on retry:**
If idempotency key matches but body is different (different amount), return 422 — the key is already "claimed" for a different operation.

---

## Q5: "What are the trade-offs between using URI versioning (/v1/) vs. not versioning at all and just making non-breaking changes?"

**Why it's tricky:** Many teams successfully avoid versioning for years, so dismissing "no versioning" is incorrect. The answer requires real nuance.

**Strong Answer:**

"No versioning, only non-breaking changes" works well under two conditions: (1) you own all clients and can deploy them simultaneously with the server, and (2) you have excellent test coverage that catches any change that breaks clients. Netflix's internal service mesh and many monorepos work exactly this way.

**When it breaks down:**

1. **External clients you don't control** (mobile apps, third-party integrations) cannot be force-upgraded. A mobile app user might not update for 18 months — you must support old response shapes indefinitely.

2. **"Non-breaking" is often misjudged.** Adding a new required field to a request is breaking. Changing an enum to include new values breaks clients that switch on all known values and have no default case. Changing the meaning of a field (not its name) is breaking without any schema change.

3. **Removing capabilities.** You will eventually need to remove something — a field, an endpoint, a supported action. Without versioning, you have no clear mechanism for this.

**URI versioning trade-offs:**
- Pros: Explicit, cacheable, logged, allows entirely different implementations per version
- Cons: Must maintain deprecated versions, clients must explicitly adopt new version, "version" in a URL means the same resource at two different addresses

**Practical recommendation:** Start with no versioning + strict non-breaking change discipline for internal services. Add URI versioning from day one for any API consumed by parties you don't control. Document a sunset policy before launching v1.

---

## Q6: "A client is retrying a request that returned 500. Is that always safe?"

**Why it's tricky:** Many engineers assume 500 is always safe to retry. It is not.

**Strong Answer:**

No. A 500 means the server encountered an unexpected error — but that error may have occurred _after_ the operation partially committed.

Example: `POST /orders` triggers:
1. INSERT into orders table ✅ committed
2. Call payment service → payment charged ✅ committed
3. Call inventory service → crashes → 500 returned to client

The client received 500 and retries. Now the order is created and the card is charged twice.

**When 500 is safe to retry:**
- GET requests (no side effects) — always safe
- POST with a server-side idempotency key — server detects duplicate
- The API is documented as idempotent for that operation

**When 500 is NOT safe to retry without an idempotency key:**
- Any POST that creates resources or charges money
- Any operation that triggers downstream side effects (emails, payments, webhooks)

**The rule:** Retry on 500 only if the operation is inherently idempotent (GET, PUT, DELETE, PATCH with set semantics) or you have an idempotency key. For anything else, surface the ambiguity to the user ("Your order may or may not have been placed — check your order history") rather than silently retrying.

---

## Q7: "The API returns data used by both a mobile app and a web dashboard. The mobile app needs 3 fields; the dashboard needs 30. How do you design the API?"

**Why it's tricky:** Tests whether candidates know the Backend-for-Frontend pattern and when GraphQL is the right tool vs overkill.

**Strong Answer:**

Three options, with different trade-offs:

**Option A: REST with field selection query param**

```
GET /users/42?fields=id,email,avatar    ← mobile gets 3 fields
GET /users/42                           ← dashboard gets all 30 fields
```
Simple to implement, no schema overhead. Works for moderate field count differences.

**Option B: GraphQL BFF**

A GraphQL gateway aggregates underlying REST/gRPC services and lets each client declare exactly what it needs:
```graphql
# Mobile query
{ user(id: "42") { id email avatar } }

# Dashboard query
{ user(id: "42") { id email avatar billing { ... } orders { ... } activity { ... } } }
```
Best when: many different clients with significantly different data needs, or when the underlying data comes from multiple services and aggregation is complex.

**Option C: Multiple specialized endpoints (BFF pattern without GraphQL)**

```
GET /mobile/v1/users/42     → 3 fields, mobile-optimized payload
GET /dashboard/v1/users/42  → 30 fields, full payload
```
Each client surface area has its own backend endpoint (or service) optimized for it. Avoids GraphQL complexity at the cost of more endpoints to maintain.

**My recommendation:** For the specific scenario described (one mobile app, one dashboard), REST with a `?fields=` parameter covers 80% of cases cheaply. If the two clients have deeply divergent data graphs or if more clients are coming, introduce a GraphQL BFF layer at the gateway. The BFF itself talks to downstream services via gRPC or REST — GraphQL is the query language, not the transport between services.
