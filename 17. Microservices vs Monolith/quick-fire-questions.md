# Microservices vs Monolith — Quick-Fire Q&A

## Architecture Fundamentals

**Q1. What is Conway's Law, and why does it matter for microservices?**
"Organisations design systems that mirror their communication structure." If your teams are siloed by function (frontend, backend, DB), your architecture will reflect that — not business domains. The Inverse Conway Manoeuvre deliberately restructures teams to match your target architecture before trying to implement it.

**Q2. What is the "two-pizza rule" and how does it relate to microservices?**
Jeff Bezos's heuristic: a team should be feedable with two pizzas (~6–10 people). Each microservice should be owned end-to-end (code, deploy, operate) by one such team. If two teams jointly own a service, it's an organisational smell.

**Q3. What is a Bounded Context in DDD?**
The explicit boundary within which a domain model is internally consistent and its language is unambiguous. "Order" in the checkout context (user-facing, contains discount, payment status) differs from "Order" in the warehouse context (bin location, pick list). Each bounded context is a candidate for its own service.

**Q4. What is the Strangler Fig pattern?**
An incremental migration strategy: route new traffic to a new microservice via a facade/API Gateway, while falling through to the monolith for unextracted paths. Gradually extract functionality until the monolith is empty and can be decommissioned — no big-bang rewrite.

**Q5. What's the difference between a monolith and a distributed monolith?**
A distributed monolith has microservice-like deployment units but still has tight coupling: shared databases across services, synchronous chains of calls where Service A → B → C all fail together, or services that must be deployed in order. It has all the complexity of microservices with none of the benefits.

---

## Communication

**Q6. When should you use gRPC instead of REST for service-to-service communication?**
When you need high throughput, low latency, or streaming. gRPC uses Protocol Buffers (binary) over HTTP/2 — roughly 5–10× faster than JSON/REST. Also use it when you need strong typing: the `.proto` file is a shared, versioned contract.

**Q7. What is the availability multiplication problem with synchronous microservices chains?**
If each service has 99.9% uptime and you chain 3 services synchronously, combined availability = 99.9% × 99.9% × 99.9% ≈ 99.7%. Chain 10 services: ~99%. The more dependencies in a synchronous call chain, the lower the overall availability.

**Q8. What is the Outbox Pattern and why is it needed?**
In async event-driven systems, a service must write to its DB and publish an event atomically. If it writes to DB and crashes before publishing to Kafka, the event is lost. The Outbox Pattern: write both to the business table and an `outbox` table in the same DB transaction. A separate poller reads the outbox and publishes to Kafka, then marks rows as published. Atomicity guaranteed.

**Q9. What is CQRS?**
Command Query Responsibility Segregation: separate the write path (commands → normalised DB) from the read path (queries → denormalised read model). Events from the write side asynchronously update the read model. Useful when read and write access patterns are very different.

---

## Service Mesh & Infra

**Q10. What is a sidecar proxy in a service mesh?**
A proxy (typically Envoy) that runs in the same pod as each service, intercepting all inbound and outbound traffic. It handles mTLS, retries, circuit breaking, load balancing, and tracing injection — without any code changes to the service. The service speaks plain HTTP; the sidecar upgrades to mTLS transparently.

**Q11. What does Istio's control plane do?**
Istiod configures all Envoy sidecars dynamically via the xDS API (no restarts needed). It includes:
- **Pilot**: pushes routing rules
- **Citadel**: issues mTLS certificates (SPIFFE-compatible X.509)
- The operator declares config via `VirtualService` and `DestinationRule` Kubernetes CRDs

**Q12. What is the difference between a `VirtualService` and a `DestinationRule` in Istio?**
- `VirtualService`: *where* traffic goes — routing rules (canary, A/B, header-based routing, retries, timeouts)
- `DestinationRule`: *how* traffic behaves when it gets there — load balancing algorithm, connection pool settings, circuit breaker thresholds

---

## Resilience

**Q13. Describe the three states of a circuit breaker.**
- **CLOSED**: requests pass through normally; failures are counted
- **OPEN**: requests fast-fail immediately without calling the downstream; opens after N failures in a window
- **HALF_OPEN**: after the recovery timeout, one probe request is allowed; if it succeeds, close the circuit; if it fails, reopen

**Q14. What is a bulkhead pattern?**
Isolate thread pools (or connection pools) per downstream service. If Service B becomes slow and exhausts the thread pool allocated for B calls, it doesn't affect the thread pool for Service C calls. Named after ship bulkheads that isolate flooding to one compartment.

**Q15. Why should retries always include jitter?**
Without jitter, all clients that experienced a failure retry at the same backoff interval simultaneously — creating a "thundering herd" that re-floods the recovering service. Jitter (random ±50% of the backoff) staggers the retries across time.

---

## Tradeoffs & Pitfalls

**Q16. Name three things that are easier in a monolith than in microservices.**
1. **Debugging**: single stack trace spans the entire operation
2. **Transactions**: ACID across all domain data in one DB
3. **Refactoring**: rename a type, and the IDE updates all callers across the system automatically

**Q17. What is database-per-service and what is its main cost?**
Each microservice owns its schema — no other service queries it directly. This enables independent schema changes and polyglot persistence. The cost: cross-domain queries (e.g., "show all orders with user names") require API composition (join in application code) or maintaining a read-side projection, instead of a simple SQL JOIN.

**Q18. When does extracting a service make things *worse*?**
When the business logic spans multiple services and requires a synchronous chain (A → B → C → D) to complete one user action. Each network hop adds latency and failure surface. If a single user action requires 5 synchronous service calls, you've distributed the monolith without gaining autonomy.

**Q19. What is the "shared library" trap in microservices?**
Teams create shared libraries (e.g., `common-models`) used across services. When that library needs a schema change, every service must be updated and redeployed. The services become coupled via the library — defeating the autonomy goal. Prefer duplication of simple models over shared library coupling.

**Q20. What is a health check, and what is the difference between liveness and readiness?**
- **Liveness** (`/health/live`): Is the process alive? If this fails, Kubernetes restarts the container.
- **Readiness** (`/health/ready`): Is the service ready to serve traffic? (Has DB connection, cache warmed, etc.) If this fails, Kubernetes removes the instance from the load balancer but doesn't restart it. A service can be live but not ready (e.g., still warming up).
