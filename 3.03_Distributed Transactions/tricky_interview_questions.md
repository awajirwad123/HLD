# Distributed Transactions — Tricky Interview Questions

## Q1: "2PC guarantees atomicity across services — why don't all microservices just use it?"

**Why it's tricky:** Candidates often accept "2PC is the correct solution" without understanding why it's nearly unusable in modern microservice architectures.

**Strong Answer:**

2PC works well within a single database cluster or bounded intra-cluster context, but breaks down across independently-deployed microservices for three reasons:

1. **Blocking on coordinator failure.** After all participants vote YES, if the coordinator crashes, every participant holds locks indefinitely. In a microservice system, a single coordinator outage freezes resources across multiple independent services — a cascading availability disaster.

2. **Lock contention kills throughput.** Locks acquired in Phase 1 are held across a network round-trip. Under high concurrency, this serializes requests that would otherwise be independent.

3. **You don't own the other service's database.** 2PC requires every participant to implement the XA protocol inside their own data store. If Payment Service runs MongoDB and Order Service runs PostgreSQL, you cannot run an XA transaction across them. Microservices by definition own their own storage.

Sagas sidestep all three: no distributed locks, no coordinator state machine, each service independently commits to its own database.

---

## Q2: "Sagas can leave the system in a partially applied state during execution. Isn't that a bug?"

**Why it's tricky:** This test whether candidates understand the difference between ACID atomicity and business-level eventual correctness.

**Strong Answer:**

It is a deliberate design trade-off, not a bug. Sagas replace strict atomicity with **business-level eventual correctness**: during execution, intermediate states (order created but payment not yet confirmed) are visible. The system is designed to handle them.

The key insight is that ACID's "intermediate states never visible" guarantee is expensive — it requires locking across all participating resources. Sagas accept temporary inconsistency in exchange for availability and throughput, with the commitment that either the Saga completes or all prior steps are compensated.

In practice, the UI handles this gracefully: "Order placed — processing payment" is shown until the Saga completes. If compensation runs, "Order cancelled — refund in 3–5 days" is shown. Users of every major e-commerce platform experience this every day.

The cases where partial visibility is truly unacceptable — e.g., bank-to-bank wire transfer with strict double-entry — require either 2PC within a shared database or careful use of escrow accounts and reservations that simulate atomicity at the business logic level.

---

## Q3: "Can the Outbox pattern guarantee exactly-once delivery to Kafka?"

**Why it's tricky:** Candidates conflate "exactly-once write to the DB" with "exactly-once delivery to Kafka." These are different problems.

**Strong Answer:**

No — the Outbox pattern provides **exactly-once write** (the event is atomically written to the DB alongside the entity, so it always exists if the entity was created) but only **at-least-once delivery** to Kafka.

Here's why: the outbox publisher sends to Kafka and then marks the row as `published`. If the publisher crashes between the Kafka send and the `UPDATE outbox SET status='published'`, the row remains `pending` and will be re-sent. Kafka will receive the message twice.

To achieve end-to-end exactly-once, you need:
1. Kafka's **idempotent producer** (deduplicates on the broker side using sequence numbers)
2. **Idempotent consumers** (dedup via a `processed_events` table or natural business idempotency)

Kafka Transactions (introduced in Kafka 0.11) can achieve exactly-once if the producer and consumer are both within the Kafka ecosystem. But for database-to-Kafka flows via Outbox, at-least-once with idempotent consumers is the standard and correct approach.

---

## Q4: "In a Saga, the payment was charged but the order creation failed. The compensation tries to refund but the payment service is down. What happens?"

**Why it's tricky:** Tests understanding of Saga failure durability and the need for persistent saga state.

**Strong Answer:**

This is the "stranded compensation" problem — and it is well-known. The solution is **durable saga state with retry**.

In an orchestrated Saga, the orchestrator persists the current state (which steps completed, which compensations are pending) to a durable store (database). When the orchestrator detects that the payment service is down:

1. It marks the compensation as `pending` and persists this state.
2. It retries with exponential backoff until the payment service recovers.
3. On recovery, it replays the pending compensation.

The key requirements this imposes:
- **Compensations must be idempotent** — the orchestrator may retry the same compensation multiple times if it crashes after sending but before confirming receipt.
- **The orchestrator's state must survive restarts** — it cannot be in-memory only.
- **Dead letter handling** — if compensation fails after N retries, an alert fires and a human reviews the stranded saga (manual remediation or scheduled batch job).

This is why no distributed system achieves the simplicity of a local ACID transaction — you are trading the database's built-in failure handling for your own application-level durability.

---

## Q5: "Why not just wrap every microservice call in a try/except and roll back on error? Isn't that the same as a Saga?"

**Why it's tricky:** This is a common junior developer instinct. Tests understanding of what "rollback" means across distributed state.

**Strong Answer:**

A `try/except` rollback works within a single ACID database transaction because the database engine tracks uncommitted state and discards it on rollback. Across distributed services, there is no such mechanism — each service has already committed its local transaction.

When Payment Service charges the card and commits, that `COMMIT` is final within Payment Service's database. If Order Service subsequently fails, there is no "rollback" you can send to Payment Service that undoes a committed charge. You cannot reach into another service's database and undo a committed row.

This is exactly what Saga compensations handle: they are explicit business operations (issue a refund) that semantically undo committed state, not database rollbacks. A `try/except` in the calling service that "catches the error and returns" leaves the charge sitting in Payment Service's database — the definition of data inconsistency.

The Saga pattern forces you to design your system with the understanding that distributed state cannot be rolled back atomically, and to write explicit compensation logic for every step that might need to be undone.

---

## Q6: "When would you choose choreography over orchestration for a Saga? Give a concrete scenario."

**Why it's tricky:** Most candidates can recite the definition difference but struggle to justify the choice with a real example.

**Strong Answer:**

Choreography is the better choice when the flow is a **linear, well-understood event chain** with a small number of loosely-coupled participants that are unlikely to change frequently.

**Concrete example:** User sign-up pipeline.
1. `user.registered` → Email Service sends welcome email
2. `user.registered` → Analytics Service records signup event
3. `user.registered` → Notification Service subscribes user to default alerts

Each service independently consumes the `user.registered` event. No service needs to know about the others. Adding a new service (e.g., a CRM) is trivial — it just subscribes to `user.registered`. No orchestrator changes needed.

**When choreography becomes problematic:** if the flow has conditional branches (if payment fails, follow path A; if inventory fails, follow path B), central audit needs, or complex compensation chains, choreography makes it difficult to answer "where is this transaction right now?" — debugging requires correlating events across multiple services.

Orchestration is better for payment-and-fulfillment flows (Uber Eats, Amazon orders) where the failure modes are complex, compensation must run in strict order, and business stakeholders need a clear audit log of every step.

---

## Q7: "The interviewer says: 'We write to the DB and then publish to Kafka — we've done this for two years without issues. Why change it?'"

**Why it's tricky:** Tests confidence in identifying a latent reliability bug that only manifests under failure, not in happy-path operation.

**Strong Answer:**

You've been lucky — or the failures have been invisible. The dual-write pattern has a specific failure mode that is silent and rare: the application crashes after the DB commit but before the Kafka publish. In two years of operation, this may have happened but gone unnoticed because:
- The order existed in the DB but the downstream service never ran
- The customer may have complained, been manually retried, or the issue attributed to something else
- Monitoring doesn't catch it unless you actively track publish success rate vs DB write rate

The risk profile is asymmetric: the failure is rare (only on crashes at exactly that point), but the consequence can be severe (order exists, payment never processed, inventory never decremented, customer never notified).

The Outbox pattern is a small, localized change — one extra table, one publisher process — that eliminates this failure mode entirely. The cost is an additional poll/WAL read operation. For any system where message delivery is correctness-critical (payments, order processing, reservations), this trade-off is justified regardless of historical reliability. Reliability without the outbox is probabilistic; with it, it's guaranteed.
