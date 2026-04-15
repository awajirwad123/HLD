# Microservices vs Monolith — Tricky Interview Questions

## Question 1

**"Our team is building a new product. The CTO wants microservices from day 1. What do you think?"**

### Why it's tricky
The "right" answer contradicts a senior executive's stated preference. Tests whether candidates can reason confidently under social pressure.

### Strong Answer

I'd push back — respectfully but clearly.

Microservices solve a specific set of problems: **team scaling, deployment autonomy, and component-level scaling**. They introduce substantial costs: distributed tracing, service discovery, inter-service communication patterns, eventual consistency, and exponential operational complexity. On day 1, you haven't yet suffered any of those problems, and you don't yet understand your domain well enough to draw bounded context boundaries correctly.

Sam Newman (author of *Building Microservices*) and Martin Fowler both explicitly advise against starting with microservices. The right move:

1. Build a **modular monolith** — domain modules with strict interface boundaries (not spaghetti). This gives you most of the developer experience benefit (module isolation, clear ownership) without the operational overhead.
2. Use a strangler fig when a specific module demonstrates one of the valid extraction triggers: a different scaling profile (the recommendation engine needs GPUs), a different release cadence, regulatory isolation (PCI-DSS for payments), or a team that's blocked by shared ownership.

Getting the bounded contexts wrong in a monolith → a refactor. Getting them wrong in microservices → a distributed monolith that takes 2 years to untangle.

---

## Question 2

**"You've extracted a Payment Service from the monolith. When a user places an order, the Order Service calls Payment Service synchronously. Payment Service goes down. Suddenly Order Service is also down. How do you fix this?"**

### Why it's tricky
Tests cascading failure reasoning and knowledge of specific patterns. Many candidates stop at "add retries" — which makes the problem worse.

### Strong Answer

This is a cascading failure — the classic synchronous coupling trap. Adding retries here makes it worse: every Order Service caller that retries amplifies load on the already-struggling Payment Service.

**Short-term fix:** Add a circuit breaker in Order Service for calls to Payment Service. After N failures, open the circuit and fast-fail new Payment calls immediately (with an error returned to the user: "Payment temporarily unavailable") rather than letting threads pile up waiting for timeouts.

**Proper architectural fix:**
Decouple synchronously by switching the payment step to asynchronous:

1. Order Service validates the order, persists it to the DB as `status=pending_payment`, and publishes an `order.created` event to Kafka.
2. Order Service returns immediately to the user: "Your order is being processed."
3. Payment Service consumes `order.created`, attempts the charge asynchronously.
4. On success: publishes `payment.completed` → Order Service updates status to `confirmed`, sends confirmation email.
5. On failure: publishes `payment.failed` → Order Service cancels the order, notifies the user.

The user experience changes to eventual confirmation (a few seconds), but the system is now resilient. Payment Service being down for 5 minutes means orders queue up in Kafka — when it recovers, they drain. Order Service is never affected.

**Trade-off to surface**: Not all payment flows can go async. A hotel reservation paid at checkout can't show "pending" — the user needs to know now. In that case, use sync with strict timeout (3s), circuit breaker, and a fallback page — not a dangling spinner.

---

## Question 3

**"Two of your microservices share a database. A teammate says 'it's just for reading — Service B only reads from Service A's table.' Is this a problem?"**

### Why it's tricky
Sounds harmless (reads are safe, right?), but it fundamentally breaks service autonomy.

### Strong Answer

Yes, it's a serious problem — arguably worse than sharing for writes, because it's invisible until something breaks.

**What breaks immediately:**
- Service A cannot change its schema without checking if Service B's queries still work. Two teams are now coupled on every migration.
- Service A cannot switch databases (e.g., Postgres → Cassandra) without breaking Service B.
- Service A cannot partition its data, rename columns, or normalise differently without coordination.

**What breaks later:**
- As team sizes grow, Service A's team has no visibility into what Service B queries. They make a "safe" index change and Service B's query suddenly does a full table scan.
- Service A's schema version is now a public API — but it was never designed as one.

**The fix:** Service B should get the data it needs from Service A's **public API** (REST/gRPC endpoint or event stream). Service A decides what data is part of its public interface. Queries become explicit contracts instead of hidden schema dependencies.

If the read latency is a concern: Service B can maintain a local **read replica** of the data it needs by subscribing to Service A's `entity.updated` events and projecting into its own store. Public API for ownership, events for propagation.

---

## Question 4

**"You've migrated to microservices. A user reports: 'I clicked Checkout and my money was charged but I never got an order confirmation.' Describe how you debug this."**

### Why it's tricky
Distributed system debugging where a partial failure caused an inconsistent state. Tests practical knowledge of observability, not just architecture theory.

### Strong Answer

This is a classic partial failure: the payment leg succeeded, but a downstream step silently failed. The investigation path:

**Step 1 — Get the trace ID**: The user's checkout request should have been assigned a `trace_id` at the API Gateway. Every downstream service call propagates this header. Without a trace ID, I'm blind.

**Step 2 — Query distributed tracing (Jaeger/Zipkin)**: Find the trace for this user's checkout. I'll see:
- Which services were called
- Where the call chain terminated
- Which spans failed or timed out

In this case I'd likely see: `OrderService → PaymentService` (success) → `NotificationService` (failed / timed out) or → `Order DB write` (failed).

**Step 3 — Check the Outbox**: If the confirmation email was supposed to be sent via an async event (`order.confirmed` → Notification Service), check if the event was published to Kafka. If it wasn't (process crashed between DB write and Kafka publish), the Outbox pattern would have caught this — check the `outbox` table for unpublished rows.

**Step 4 — Check Kafka consumer lag**: Did the Notification Service fall behind? Is there a consumer lag of hours on the `order.confirmed` topic? If so, the email will arrive eventually (recoverable) vs. the event being lost (not recoverable without replay or manual trigger).

**Fix for good**: This scenario is exactly why you'd implement the Outbox Pattern for the critical `payment charged → order confirmed` step. The payment and the outbox entry write in one DB transaction. If the process dies, the poller catches the outbox entry. The event is never lost.

---

## Question 5

**"A service mesh adds ~5ms latency per hop. In a chain of 5 synchronous microservices, that's 25ms of mesh overhead alone. Was microservices the right call?"**

### Why it's tricky
Challenges the candidate to defend (or reject) an architectural choice quantitatively. Tests whether they can reason about latency budgets, not just architecture patterns.

### Strong Answer

5ms per hop × 5 hops = 25ms overhead is the worst-case framing. Let me stress-test it:

**Is 25ms significant?** For a user-facing API with a 200ms SLA: 25ms is 12.5% of the budget — non-trivial. For a background data pipeline: irrelevant.

**But is the chain of 5 synchronous services the right design?** A 5-deep synchronous chain is itself an architectural smell called a "chatty" service graph. If every user checkout requires 5 sequential hops, I'd revisit whether those 5 things need to be separate synchronous services at all. Options:

1. **Async fan-out**: Instead of A→B→C→D→E synchronously, A does its work, publishes one event, and B/C/D/E act independently in parallel → total latency is `max(B, C, D, E)` not `B+C+D+E`.
2. **API composition at the edge**: An aggregator/BFF calls B, C, D, E in parallel via `asyncio.gather` — 4 simultaneous 5ms calls, not 20ms sequential.
3. **Collapsing fine-grained services**: If B and C always need to be called together and have no independent scaling needs, they might be better as one service.

The service mesh's 5ms overhead is a red herring — the real smell is the synchronous chain. Fix the architecture; the overhead disappears.

---

## Question 6

**"How do you handle a breaking API change in a microservice that has 10 consumers?"**

### Why it's tricky
Tests practical versioning strategy. Most candidates say "version the API" without understanding the operational complexity.

### Strong Answer

**First principle**: A microservice's API is a public contract. Treat breaking changes like a Library author — deprecation window + migration period.

**Strategy:**

1. **Expand-Contract (parallel change) pattern**:
   - Phase 1 (Expand): Add the new field/endpoint alongside the old one. Both are supported. Publish deprecation notice.
   - Phase 2: Consumers migrate to the new version at their own pace (tracked via usage metrics or feature flag).
   - Phase 3 (Contract): Remove the old version once all consumers have migrated.

2. **Versioning in practice**:
   - REST: URI versioning `/v1/orders` → `/v2/orders` is visible and explicit
   - gRPC: field numbers in Protobuf are stable — you can add new fields; never remove or renumber existing ones. Use `reserved` for retired fields.

3. **Consumer-driven contract testing (Pact)**:
   - Each consumer defines what fields it actually uses ("consumer contract")
   - Provider CI runs these contracts on every PR — if a change would break any consumer, the build fails *before* deployment
   - This catches breaking changes at development time, not production

4. **Operational tracking**: Use API observability to find which consumer is still calling `/v1/orders`. Don't guess.

5. **Sunset headers** (RFC 8594): Include `Sunset: Sat, 01 Jan 2027 00:00:00 GMT` and `Deprecation: true` in responses to the old version — tooling can surface this automatically.

---

## Question 7

**"Your microservices are deployed on Kubernetes with Istio. A new engineer says 'I'll add retry logic in my service code to handle transient errors.' What would you tell them?"**

### Why it's tricky
Surfaces the "where does cross-cutting concern logic live" question. Both approaches are valid, but combining them has a dangerous interaction.

### Strong Answer

The intent is right, but doing it in application code when you already have a service mesh creates a **retry amplification problem**.

If the application code retries 3 times, and Istio's `VirtualService` is configured to retry 3 times, a single failed request triggers 3 × 3 = **9 attempts** to the downstream service. At scale, that amplification can overwhelm a recovering service.

**The right approach**: pick one layer for retries.

- **Prefer the mesh layer** for infrastructure-level retries (network timeouts, 503 Service Unavailable). Configure in Istio's `VirtualService`:
  ```yaml
  retries:
    attempts: 3
    perTryTimeout: 2s
    retryOn: "5xx,gateway-error,connect-failure"
  ```
  This is infrastructure policy controlled by the platform team, applied consistently across all services.

- **Keep application-level retry logic for business-semantic retries** — cases where the application understands the semantics (e.g., "retry with exponential backoff on optimistic lock conflict from the DB"). These aren't retried by the mesh.

- **Document and enforce via ADR**: Write an Architecture Decision Record specifying "retry logic lives in the mesh; do not add HTTP-level retry in application code." The Istio config is in the repo — it's auditable.

Also flag: retries are only safe on **idempotent operations**. Retrying a payment charge without an idempotency key can double-charge the user. Make sure the charge endpoint is idempotent before configuring retries there.
