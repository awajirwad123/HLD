# Microservices vs Monolith — Notes & Cheat Sheet

## 1. Architectural Spectrum One-Liners

| Style | Summary |
|---|---|
| **Monolith** | Single deployable unit, single DB, all code coupled |
| **Modular Monolith** | Enforced module boundaries in-process; one deploy |
| **Macro-services** | 3–10 large domain services; practical middle ground |
| **Microservices** | Many small services, each owns its data; independently deployable |

**Start here**: *Always monolith first. Extract when you feel the pain at scale.*

---

## 2. When Microservices Make Sense — Checklist

- [ ] Team > 50 engineers, multiple teams stepping on each other
- [ ] Specific component has drastically different scaling needs
- [ ] Regulatory isolation required (PCI-DSS for payments)
- [ ] Independent deployment velocity matters per team
- [ ] Domain boundaries are well-understood (completed DDD analysis)
- [ ] Kubernetes + observability + CI/CD are mature
- [ ] Monolith is already modular (don't skip this step)

---

## 3. Decomposition Strategies

**By Business Capability**: Align to what the business *does*, not code layers.
- Order Service, Payment Service, Inventory Service, Notification Service

**By Bounded Context (DDD)**: Split where domain language diverges.
- "Order" in checkout ≠ "Order" in warehouse → two separate services

**Strangler Fig Pattern**:
```
New requests → API Gateway Facade → new microservice  (for extracted paths)
                                  → monolith           (for unextracted paths)
```
Incrementally extract, never big-bang rewrite.

---

## 4. Inter-Service Communication Summary

### Synchronous

| Protocol | Use case | Overhead |
|---|---|---|
| HTTP/REST | External-facing, simple S2S calls | Headers + JSON |
| gRPC | Internal high-throughput, streaming | Protobuf binary (~10× faster) |

**Sync pitfalls:**
- Cascading failures: slow downstream = slow upstream
- Availability chain: 3 services at 99.9% = 99.7% combined
- Tight coupling: B must be up for A to succeed

### Asynchronous

| Pattern | Tool | "Guarantee" |
|---|---|---|
| Event-driven | Kafka/RabbitMQ | At-least-once |
| Command queue | SQS/RabbitMQ | At-least-once |
| Event sourcing | Kafka (log) | Ordered, replayable |

**Async benefits:**
- Temporal decoupling: downstream can be down
- Fan-out: new subscriber without changing producer
- Natural backpressure

**Async pitfalls:**
- Eventual consistency → UI must handle "pending" states
- Debugging: no single stack trace — need distributed tracing
- Multi-step failures need Saga pattern

---

## 5. Data Patterns

**Database-per-service**: Each service owns its schema. No cross-service DB joins.

**API Composition**: Aggregator service calls multiple services and joins in-memory.
- Drawback: N synchronous calls → O(N) latency

**CQRS** (Command Query Responsibility Segregation):
- Write model (events) → async projection → Read model (denormalised, fast)
- Use when read and write patterns differ significantly

**Outbox Pattern** (safe async publish):
```
In same DB transaction:
  1. INSERT into business table
  2. INSERT into outbox table (event to publish)
Outbox poller: reads outbox → publishes to Kafka → marks published
```
Prevents lost events when service crashes between DB write and Kafka publish.

---

## 6. Service Mesh — Key Concepts

| Concept | Description |
|---|---|
| **Sidecar proxy** | Envoy runs alongside each service pod; intercepts all traffic |
| **Data plane** | All Envoy sidecars across the cluster |
| **Control plane** | Istiod (Istio) — configures all Envoys dynamically via xDS API |
| **mTLS** | Mutual TLS between services — automatic via Citadel (certificate authority) |
| **VirtualService** | Routing rules (canary %, A/B, header-based) |
| **DestinationRule** | Load balancing policy + circuit breaker config per service |
| **AuthorizationPolicy** | RBAC: which service can call which endpoint |

**xDS API**: Envoy's dynamic configuration protocol. Control plane pushes new configs to all sidecars without restarting Envoy.

---

## 7. Resilience Patterns

### Circuit Breaker States
```
CLOSED → (N failures in window) → OPEN → (timeout elapsed) → HALF_OPEN → (M successes) → CLOSED
                                                             → (1 failure) → OPEN
```

| State | Behaviour |
|---|---|
| CLOSED | Pass through — normal operation |
| OPEN | Fast-fail immediately — don't call downstream |
| HALF_OPEN | Let one request through — probe recovery |

### Retry with Backoff
```python
sleep = min(base * (2 ** attempt) * (0.5 + random()), max_delay)
```
Always add jitter (× random 0.5–1.5) to prevent thundering herd on retry storms.

### Bulkhead
Isolate thread pools per downstream service. Service B slowness can't exhaust thread pool for Service C calls.

---

## 8. Conway's Law

> "Organisations design systems that mirror their communication structure."

**Inverse Conway Manoeuvre**: Structure your *teams* around the architecture you want.
- Team owns one service end-to-end: code, deploy, operate, on-call
- Don't split services if teams are still coupled (shared standups, shared sprints)

If your org has 3 teams with blurry boundaries → your "microservices" will look like a monolith split across processes.

---

## 9. Microservices Operational Checklist

- [ ] **CI/CD per service**: independent pipeline, independent deploy
- [ ] **Containerisation**: Docker + Kubernetes for service isolation
- [ ] **Service discovery**: Kubernetes DNS, Consul, or Istio
- [ ] **Distributed tracing**: OpenTelemetry → Jaeger (trace_id in every request)
- [ ] **Structured logging**: `{trace_id, service, level, timestamp, message}`
- [ ] **Health checks**: `/health/live` (is process alive?) + `/health/ready` (can serve traffic?)
- [ ] **Circuit breakers**: Resilience4j / Istio DestinationRule
- [ ] **Retry budgets**: don't retry unboundedly — exponential backoff + cap
- [ ] **Idempotency keys**: all mutating API calls must be idempotent for safe retry

---

## 10. Key Numbers & Rules of Thumb

| Rule | Value |
|---|---|
| Microservices justified team size | 50+ engineers / Bezos "2 pizza rule" per service |
| gRPC speedup vs REST | ~5–10× (binary + HTTP/2 vs JSON + HTTP/1.1) |
| Saga compensation time | Ideally < 1s for user-facing, async for background |
| Availability chain of 3 × 99.9% | 99.7% combined |
| Istio sidecar overhead | ~5ms p99 latency, ~0.5 vCPU per 1000 RPS |
| Typical service mesh threshold | Enable at 10+ services |

---

## 11. Interview Key Phrases

- *"I start with a modular monolith — strict module boundaries enforced via interfaces. Extract to microservices only when we hit a specific pain point: scaling, team autonomy, or regulatory isolation."*
- *"The Strangler Fig pattern lets me extract incrementally without a big-bang rewrite — the API Gateway routes new paths to the microservice and falls back to the monolith."*
- *"Async communication decouples services in time — the Order Service doesn't fail because the Payment Service is slow. The cost is eventual consistency."*
- *"A service mesh moves cross-cutting concerns — mTLS, retries, circuit breakers, tracing — out of application code into the sidecar. Teams stop reinventing this per service."*
- *"Conway's Law: if the teams aren't split, the services won't be either. Architecture is social before it's technical."*
