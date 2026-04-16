# Microservices vs Monolith — Interview Simulator

## Scenario 1: E-Commerce Platform Migration

**Prompt:**
> "You're a senior engineer at an e-commerce company. The monolith that started as a 5-person startup now has 120 engineers and 3-hour deployment windows. Leadership wants microservices. Where do you start?"

**Time limit:** 30 minutes

---

### Clarifying Questions You Should Ask

1. What is the biggest pain point right now — deployment time, team velocity, or scaling a specific component?
2. Is the monolith already modular internally, or is it genuinely tangled?
3. Do we have CI/CD maturity, observability, and containerisation in place?
4. What is the risk tolerance — can we afford a 6-month migration with no user-facing features?
5. Are there regulatory constraints (PCI-DSS for payments, GDPR for user data) that drive a specific extraction first?

---

### Expected Architecture Walkthrough

**Phase 0 — Assess Before Cutting**

Don't start with extraction. Start with understanding:
- Draw the existing module dependency graph. Where are the circular dependencies?
- Identify bounded contexts by talking to domain experts (checkout, fulfilment, catalogue, user identity)
- Identify the "fracture planes": components with clearly different scaling needs, team ownership, or deployment cadence

**Phase 1 — Modularise the Monolith First**

Before extracting any service, enforce module boundaries within the monolith:
- Enforce package-level visibility: `checkout` module cannot import `warehouse` module's internals
- All cross-module calls go through explicit interfaces (protocols in Python, interfaces in Java/Go)
- Each module owns a clearly defined data namespace within the shared DB (schema-per-module or table-prefix-per-module)

This step costs ~2–3 months but produces a clean target for extraction. It also often reveals whether microservices are even necessary — many pain points disappear with modular discipline.

**Phase 2 — Extract High-Value Targets Using Strangler Fig**

Priority order for extraction (based on common e-commerce pain points):

| Service | Why Extract First |
|---|---|
| **Notification Service** | Stateless, no domain data, lowest risk. Good first slice. |
| **Payment Service** | PCI-DSS compliance requires isolation. Independent security patching. |
| **Search Service** | Different tech stack (Elasticsearch). Independent scaling (query spikes). |
| **User/Auth Service** | Hot path — every request hits it. Must scale independently. |
| **Order Service** | Core domain — extract last after others are stable. |

**Strangler Fig execution per service**:
```
1. Build new service with complete feature parity for target domain
2. Deploy API Gateway (or Nginx) in front of monolith
3. Route traffic for /api/notifications/** → Notification Service
4. Monitor: error rates, latency, parity in business metrics
5. Once stable: remove notification code from monolith
6. Repeat
```

**Database Extraction Strategy**:
Don't copy the DB schema. Each new service gets a fresh DB from the start. Seed initial data via a one-time migration. Going forward, the new service owns its writes. The monolith's reference to the same table is replaced with an API call.

**Phase 3 — Service Mesh at ~10 services**

Before 10 services: cross-cutting concerns (mTLS, retries, tracing) are manageable per-service.
At 10+ services: introduce Istio/Envoy to centralise:
- mTLS enforcement between all services
- Consistent retry + timeout policies
- Distributed tracing (OpenTelemetry → Jaeger)

---

### Key Decisions to Justify

**"Why not extract the Order Service first?"**
Order is the core domain — it touches payments, inventory, fulfilment, and notifications. Extracting it requires all its dependencies to have stable APIs first. Extract the leaves of the dependency graph before the roots.

**"The monolith's DB has a `users` table that 15 different parts of the codebase join against. How do you handle this?"**
1. Create the User Service with its own DB.
2. For the monolith: replace direct DB joins with a `UserServiceClient` that calls `/users/{id}` and caches results (Redis, 5min TTL).
3. For queries that need joins (e.g., "orders with user email"): build a read-side projection that subscribes to `user.updated` and `order.created` events and maintains a denormalised view.

---

### Follow-up Questions

**"You've extracted 5 services. Deployment frequency went from weekly to daily. But error rates doubled. Why?"**
Likely causes: (1) no circuit breakers — one slow service cascades; (2) inconsistent timeouts — one service defaults to 30s while callers expect 2s; (3) missing distributed tracing → bugs take hours to locate; (4) retry amplification without coordination. The fix: enforce service mesh policies for retries and timeouts; add distributed tracing before anything else.

**"How do you measure if the migration is successful?"**
- **Deployment frequency per team**: should increase (independent deploys)
- **Change failure rate**: monolith → should decrease (smaller blast radius)
- **Mean time to recover (MTTR)**: should decrease (narrower fault isolation)
- **Team lead time** (PR → production): should decrease
- These are the DORA metrics — the standard industry measure of engineering velocity

---

## Scenario 2: Greenfield SaaS — Should We Start with Microservices?

**Prompt:**
> "You're the founding engineer at a startup building a B2B SaaS analytics platform. There are 4 engineers. The CTO came from Netflix and insists on a microservices-first approach. The first user-facing feature ships in 6 weeks. What do you do?"

**Time limit:** 20 minutes

---

### Expected Answer

**Politely and clearly push back with data.**

The CTO's Netflix experience is valid *for Netflix* — 1,000+ engineers, services that independently see 10M+ requests/day, distinct scaling profiles across dozens of domains. None of that is true at a 4-person startup in week 1.

**The cost at 4-person scale:**
- Each microservice needs its own CI/CD pipeline, its own Dockerfile, its own DB migration tooling, its own health checks, its own deployment manifests
- 10 microservices × 10 configs = 100 things to create, maintain, and debug
- Debugging a bug in a distributed system with 4 engineers and no established observability stack is a multi-day exercise

**The actual proposal:** Start with a well-structured modular monolith.

```
analytics-platform/
  ├── modules/
  │   ├── ingestion/     ← data ingest pipeline
  │   ├── compute/       ← aggregation engine
  │   ├── auth/          ← user/org management
  │   └── api/           ← external HTTP API
  ├── shared/
  │   └── domain/        ← value objects only (no logic)
  └── main.py            ← composition root
```

Rules:
- Modules communicate only via explicit interfaces — no cross-module direct imports
- Module owns its DB tables (schema-per-module naming convention)
- Each module has its own test suite runnable in isolation

**What this buys you:**
- Deploys in 6 weeks, not 6 months
- When you reach 20 engineers and feel the actual pain, the modules are already cleanly bounded and ready to extract
- The bounded contexts are validated by real usage, not hypothesised

**When to reconsider microservices at this company:**
- Specific component needs different scaling (GPU compute for ML models)
- Regulatory requirement (SOC 2 Type II for auth isolation)
- Second team hired that will own a specific domain independently

---

## Scenario 3: Designing the Service Boundaries

**Prompt:**
> "Design the microservice architecture for a ride-sharing platform (Uber-like) at scale. Focus on service decomposition, communication patterns, and one critical failure scenario."

**Time limit:** 35 minutes

---

### Clarifying Questions

1. Scope: just the ride booking flow, or also driver earnings, maps, pricing?
2. Scale: 10M rides/day? (Uber is ~14M — use as reference)
3. Consistency requirements: is it OK for a driver's location update to be 1–2s stale to the rider?

---

### Service Decomposition

```
Core Services:
  ├── Rider Service          — rider accounts, ride history, payment methods
  ├── Driver Service         — driver accounts, vehicle info, documents
  ├── Location Service       — real-time driver location (write-heavy, ~5 updates/sec per driver)
  ├── Matching Service       — pairs rider requests with nearby available drivers
  ├── Trip Service           — lifecycle of a trip (requested→matched→started→completed)
  ├── Pricing Service        — surge pricing, fare estimation
  ├── Payment Service        — charge rider, pay driver, split fare
  └── Notification Service   — push notifications (driver ETA, trip start, receipt)
```

**Communication patterns:**

| Flow | Pattern | Reason |
|---|---|---|
| Rider requests ride → Matching Service | Sync REST | User waits for confirmation |
| Matching Service → Driver app (notify available ride) | WebSocket / push | Low latency bidirectional |
| Driver updates location → Location Service | UDP / WebSocket | High frequency, loss-tolerant |
| Trip completed → Payment Service | Async event | Don't block trip end on payment |
| Payment completed → Notification Service | Async event | Email/SMS is non-blocking |
| Pricing surge calculation | Sync gRPC at ride request time | Need price before confirming |

**Database choices:**

| Service | DB | Reason |
|---|---|---|
| Location Service | Redis + Geo index | Sub-100ms geospatial queries; data is ephemeral |
| Trip Service | PostgreSQL | ACID for trip state machine |
| Matching Service | In-memory priority queue + Redis | Stateful, latency-sensitive |
| Rider/Driver | PostgreSQL | Relational, transactional |
| Payments | PostgreSQL + event store | Auditability, idempotency |

---

### Critical Failure Scenario: Driver Accepts Ride, Payment Service Down

**Flow at failure point:**
```
Trip completed → Trip Service publishes "trip.completed" to Kafka
→ Payment Service consumer is down (redeploying)
→ Event sits in Kafka topic (Consumer Lag = 1)
```

**Why this is fine (by design):**
- Kafka persists the `trip.completed` event with default retention (7 days)
- When Payment Service recovers (restarts, 2 minutes later), the consumer resumes at the committed offset
- It processes `trip.completed`, charges the rider, marks payment as complete
- Rider gets receipt email 2 minutes late — but is not double-charged, nothing is lost

**What would break this:**
- If the Trip Service tried to call Payment Service synchronously at trip end — Payment Service down = Trip Service can't mark the trip as complete = driver is stuck waiting
- This is exactly the argument for async for non-blocking flows

**Idempotency requirement:**
The `trip.completed` event may be delivered more than once (at-least-once Kafka). The Payment Service must be idempotent: `INSERT INTO charges (trip_id, ...) ON CONFLICT (trip_id) DO NOTHING`. Charging twice is catastrophic; the idempotency key is `trip_id`.

---

### Follow-up Questions

**"Location Service gets 50M location updates/minute. How?"**
Use UDP (not TCP) from the driver app — no connection overhead, tolerate occasional packet loss (next update arrives in 1s anyway). At the server: use an append-only write path to Redis with Geo sets. Batch writes with a short buffer (~100ms) to reduce Redis write pressure.

**"How do you handle surge pricing for a concert ending with 100K people requesting rides simultaneously?"**
Pricing Service runs as a separate compute tier reading from Location Service's driver density. It pre-computes surge multipliers per geohash cell every 30s and caches the result. At ride request time: Pricing Service reads the cached surge multiplier (fast) — not a live computation at request time. Surge calculation is decoupled from the hot request path.
