# Microservices vs Monolith — Architecture & Concepts

## 1. The Architectural Spectrum

Systems don't jump from monolith to microservices — they evolve through recognisable stages:

```
Monolith ──► Modular Monolith ──► Macro-services ──► Microservices
```

**Monolith**: Single deployable unit. All code compiled and deployed together. One database schema.

**Modular Monolith**: Code organised into strict modules (bounded contexts) with explicit interfaces between them. Single deployment, but internal design discipline enforced by package/module visibility rules. *Martin Fowler calls this the underrated "sweet spot" for many teams.*

**Macro-services**: 3–10 large services decomposed by domain (auth, payments, inventory). Not fine-grained microservices. Lower operational complexity than full microservices.

**Microservices**: Many small services, each owning its data, independently deployable. Fine-grained decomposition (sometimes per business capability).

---

## 2. Monolith — Strengths and Failure Modes

### Why monoliths work well initially

- **Simple deployment**: one artifact, one server
- **Simple debugging**: single process, full stack trace
- **No network latency**: function calls, not RPC
- **Transactions**: ACID across the entire domain in one DB
- **Easy refactoring**: IDE can rename a function across the whole system

### When monoliths break down

| Symptom | Root Cause |
|---|---|
| Deployment takes 2 hours | All teams sharing one CI/CD pipeline |
| Unrelated team's bug takes down the whole site | No failure isolation |
| Scaling a single hot feature requires scaling everything | No independent scalability |
| Teams waiting on each other to merge | Tight coupling via shared code + DB |
| DB schema migration blocks all teams | Single schema ownership |
| Can't adopt new language/runtime for a specific domain | Technology lock-in |

**Key insight**: A monolith's problems are **organisational and operational** at scale, not purely technical.

---

## 3. Microservices — Decomposition Strategies

### 3.1 Decompose by Business Capability

Align services to what the business *does*, not how the code is layered.

```
E-commerce example:
  ├── Order Service         (create, track, cancel orders)
  ├── Inventory Service     (stock levels, reservations)
  ├── Payment Service       (charge, refund)
  ├── Notification Service  (email, SMS, push)
  ├── User Service          (accounts, auth)
  └── Search Service        (product catalog search)
```

**Benefit**: Matches Conway's Law — team structure mirrors service structure. Each team owns one service end-to-end.

### 3.2 Decompose by Subdomain (Domain-Driven Design)

Identify **Bounded Contexts**: the explicit boundary within which a domain model applies.

```
"Order" in checkout context ≠ "Order" in warehouse context
  - Checkout Order: items, price, discount, payment status
  - Warehouse Order: pick list, bin location, packing instructions
  → Two separate models, two separate services
```

**Strategic DDD patterns**:
- **Core Domain**: competitive differentiator → build (e.g., recommendation engine)
- **Supporting Domain**: needed but not differentiating → build lean or buy
- **Generic Domain**: commodity → use off-the-shelf (e.g., email sending → SendGrid)

### 3.3 Strangler Fig Pattern (Incremental Migration)

Don't rewrite — incrementally extract services from a running monolith:

```
1. New requests routed to new service via API Gateway (Facade)
2. New service proxies unimplemented paths back to monolith
3. Gradually migrate functionality from monolith → service
4. When fully migrated, remove the proxy path
5. Repeat for the next domain
```

```
Client ──► API Gateway ──► New Auth Service (new code)
                      └──► Monolith (legacy, slowly decomposed)
```

The monolith never gets a big-bang rewrite. It *strangles* slowly as capabilities move out.

---

## 4. Inter-Service Communication

### 4.1 Synchronous (Request-Response)

**HTTP/REST**: standard, widely understood, tooling abundant. Best for external-facing APIs and simple service-to-service calls.

**gRPC**: binary protocol (Protocol Buffers), strongly typed, ~10× faster than REST for internal communication. Best for internal high-throughput service calls, streaming.

```
gRPC advantages:
  - Strongly typed contracts (proto files = shared interface definition)
  - Code generation for multiple languages
  - Streaming: unary, server-streaming, client-streaming, bidirectional
  - HTTP/2 multiplexing
```

**When to use sync**: Client needs the result immediately. User-facing reads. Simple request-response.

**Problems with sync**:
- **Cascading failures**: if Service B is slow, callers pile up → Service A's thread pool exhausts → A is now slow → callers pile up upstream
- **Availability product**: if 3 services chained, each 99.9% → chain = 99.7%
- **Tight availability coupling**: B must be running for A to succeed

### 4.2 Asynchronous (Event-Driven)

**Message queues / event streams (Kafka, RabbitMQ, SQS)**:

```
Order Service ──publishes──► "order.created" ──► Kafka
                                               ├──► Inventory Service (reserve stock)
                                               ├──► Notification Service (email)
                                               └──► Analytics Service (log)
```

**Benefits**:
- Services are decoupled in time — downstream can be slow or down without blocking Order Service
- Easy fan-out: new subscriber without changing producer
- Natural backpressure: slow consumer doesn't block the producer
- Replay: reprocess events from the beginning of the Kafka log

**Problems with async**:
- Debugging harder (no single call stack)
- Eventual consistency: Order placed but inventory not yet reserved — what does the UI show?
- Ordering guarantees limited to a partition
- Compensation (Saga pattern needed for multi-step transactions)

### 4.3 Communication Pattern Decision Matrix

| Scenario | Pattern |
|---|---|
| User queries their profile | Synchronous REST/gRPC |
| Payment confirmation needed for order to proceed | Synchronous (with timeout + Saga fallback) |
| Notify warehouse after order placed | Async event |
| Fan-out an event to 5 downstream services | Async event |
| Real-time data pipeline | Kafka streaming |
| Simple task queue (resize image) | RabbitMQ/SQS |

---

## 5. Data Management in Microservices

### Database-per-Service Pattern

Each service owns its database. No shared schema across services.

```
Order Service     ──► order_db   (PostgreSQL)
Inventory Service ──► inventory_db (MySQL)
User Service      ──► user_db    (PostgreSQL)
Search Service    ──► elasticsearch
```

**Why**:
- Schema changes in one service don't break others
- Services can choose the right database type per use case
- Independent scaling

**The trade-off**: No cross-service ACID transactions. Must use distributed patterns (Saga, Outbox).

### CQRS (Command Query Responsibility Segregation)

Separate the write model from the read model:

```
Command (write) ── Order Service DB (normalised, Postgres)
                ──becomes──► Event ──► Order Read Model (denormalised, Redis/Elastic)
Query (read)  ── Order Read Model (optimised for query patterns)
```

Useful when read and write patterns are very different (e.g., complex joins for read, simple inserts for write).

---

## 6. Service Mesh Basics

At scale (50+ microservices), cross-cutting concerns become unmanageable if implemented per-service:

- mTLS between services (authentication + encryption)
- Retry logic with backoff
- Circuit breakers
- Load balancing between service instances
- Distributed tracing (inject trace IDs into requests)
- Rate limiting per route
- Traffic splitting (canary deployments: send 5% to v2)

**Service mesh solution**: Move all of this out of application code into a **sidecar proxy** running alongside each service.

```
[Service A pod]                    [Service B pod]
  App process ──► Envoy sidecar ──►  Envoy sidecar ──► App process
                  (all outbound)      (all inbound)
                  mTLS, retry,        auth, rate limit,
                  circuit breaker     tracing inject
```

### Istio Architecture

```
Data Plane (per pod):
  Envoy proxy (sidecar) — intercepts all inbound/outbound traffic

Control Plane:
  Istiod
    ├── Pilot    — pushes routing rules to Envoy proxies
    ├── Citadel  — issues mTLS certificates (SPIFFE/X.509)
    └── Galley   — validates and distributes configuration
```

**Key Istio features**:

| Feature | Description |
|---|---|
| `VirtualService` | Routing rules (canary, A/B, header-based routing) |
| `DestinationRule` | Load balancing policy, circuit breaker config per service |
| `PeerAuthentication` | Enforce mTLS between all services in namespace |
| `AuthorizationPolicy` | RBAC: Service A is allowed to call Service B's `/api/*` |

### Envoy Proxy

Envoy (written in C++) is the de facto sidecar in modern service meshes:

- **xDS API**: Envoy's configuration is dynamically pushed by the control plane (not static files). Istio's Pilot implements the xDS API to configure all Envoy instances.
- **Listener → Filter Chain → Cluster**: Envoy matches incoming traffic against listeners, applies filter chains (auth, tracing, retries), then routes to a cluster (upstream service).
- Used standalone (as edge proxy / API gateway) or as a sidecar.

---

## 7. Microservices — Operational Costs

Things that become harder:

| Concern | Monolith | Microservices |
|---|---|---|
| Deployment | One artifact | N services, N pipelines |
| Debugging | Single stack trace | Distributed tracing required |
| Transactions | ACID | Saga + eventual consistency |
| Testing | Integration tests in-process | Contract testing (Pact), mocking |
| Schema migration | One DB, one migration | N databases to migrate independently |
| Service discovery | In-process calls | DNS / service registry |
| Data consistency | Joint query | API composition or CQRS |
| Latency | 0 (function call) | Network hop per service boundary |

**"Don't start with microservices"** — Martin Fowler, Sam Newman: monolith first, extract when specific pain is identified and team/org can support the operational overhead.

---

## 8. Conway's Law and Inverse Conway Manoeuvre

**Conway's Law**: "Organisations design systems that mirror their communication structure."

A company with 3 separate teams → 3 separate subsystems with the coupling patterns of how those teams communicate.

**Inverse Conway Manoeuvre**: Deliberately design your team structure to match the architecture you want. To build microservices, first split into teams that can each own a service end-to-end (code, deploy, operate, on-call). Don't split services without first splitting teams.

---

## 9. When to Use Each

**Start with a monolith when:**
- Early stage / startup / MVP
- Team < 10 engineers
- Domain is not yet well understood
- Rapid iteration more important than scale

**Introduce service boundaries when:**
- Specific scaling need emerges (e.g., only the recommendation engine needs GPU)
- Specific deployment isolation needed (regulatory: payment must meet PCI-DSS)
- Independent team velocity blocked by shared repo/deploy
- Technology mismatch: one capability benefits from a different stack

**Microservices make sense when:**
- Organisation has 50+ engineers with independent teams per domain
- Mature CI/CD, Kubernetes, observability tooling in place
- Domain boundaries are well-understood (DDD analysis done)
- You've already felt the pain of the monolith at scale
