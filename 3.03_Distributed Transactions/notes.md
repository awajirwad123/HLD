# Distributed Transactions — Notes & Reference

## The Core Trade-Off

```
2PC          → Strict atomicity, high latency, lock contention, blocking on coordinator failure
Saga         → Eventual business correctness, low latency, no cross-service locks, requires compensation
Outbox       → Solves dual-write; orthogonal to 2PC/Saga — use it with either
```

---

## 2-Phase Commit Cheat Sheet

### Protocol Steps

```
Phase 1 (PREPARE):  Coordinator → all nodes: "Can you commit?"
                    Each node: write WAL, acquire locks, reply YES/NO

Phase 2 (COMMIT):   If all YES → Coordinator writes COMMIT to durable log → sends COMMIT → nodes apply + release locks
        (ABORT):    If any NO  → Coordinator sends ABORT → nodes rollback + release locks
```

### Failure Modes

| When | Who Fails | Impact |
|------|-----------|--------|
| Before PREPARE reply | Any participant | Coordinator times out → ABORT — safe |
| After YES, before COMMIT | Coordinator | **Blocking** — participants hold locks forever |
| After YES, before COMMIT | Participant | Coordinator retries on restart |
| Network partition | — | Participants can't respond → timeout → ABORT |

### Key Numbers

- Minimum round trips: **2** (prepare + commit)
- Lock hold duration: from PREPARE until COMMIT ACK received
- Blocking window: coordinator failure duration (seconds to minutes)

### When to Use 2PC

✅ Same database cluster (XA transactions)
✅ Intra-cluster, low-latency, same cloud region
✅ Existing JTA/XA frameworks (Java EE, Spring + Atomikos)
❌ Cross-datacenter, globally distributed
❌ High-throughput paths (lock contention)
❌ Microservices with independently deployable databases

---

## Saga Cheat Sheet

### Choreography vs Orchestration at a Glance

| | Choreography | Orchestration |
|--|--------------|---------------|
| Who drives the flow? | Each service reacts to events | Central orchestrator commands each step |
| Coupling | Services coupled to event schema | Services coupled to orchestrator |
| State visibility | Distributed — need tracing | Centralized in orchestrator |
| Failure handling | Each service emits compensating events | Orchestrator runs compensation in reverse |
| Debugging | Hard without distributed tracing | Easy — orchestrator has full log |
| Adding steps | Add a new subscriber | Change orchestrator logic in one place |
| SPOF | No | Orchestrator (must persist state durably) |

### Compensation Rules

1. **Run compensations in reverse order** — undo each step last-to-first
2. **Compensations must be idempotent** — publisher may retry
3. **Not all actions are compensatable** — emails sent, packages shipped → use best-effort (send cancellation email, initiate return)
4. **Compensation ≠ rollback** — it's a business operation that semantically undoes, not a DB rollback

### Saga Failure Response Matrix

| Which step fails | Compensation needed |
|-----------------|-------------------|
| Step 1 fails | Nothing (no committed steps) |
| Step 2 fails | Compensate Step 1 |
| Step N fails | Compensate Step N-1, N-2, … Step 1 (reverse) |

---

## Outbox Pattern Cheat Sheet

### The Dual-Write Problem

```python
# WRONG — two operations, either can fail independently
conn.execute("INSERT INTO orders ...")  # Success
kafka.send("order.created", ...)        # Crash here → order exists, event never sent
```

### The Fix

```python
# CORRECT — single atomic transaction
async with conn.transaction():
    conn.execute("INSERT INTO orders ...")
    conn.execute("INSERT INTO outbox ...")  # Same transaction — both or neither
# Outbox publisher reads outbox table and publishes to Kafka separately
```

### Delivery Guarantee

| Step | Guarantee |
|------|-----------|
| Write to DB + outbox | Atomic (exactly-once via DB transaction) |
| Outbox → Kafka (polling publisher) | **At-least-once** (publisher may crash after send, before marking published) |
| Kafka → Consumer | At-least-once (Kafka redelivers unack'd messages) |
| Consumer processing | Must be **idempotent** (dedup via `processed_events` table or consumer group offset management) |

### Polling vs CDC

| Method | Latency | DB Load | Complexity |
|--------|---------|---------|------------|
| Polling (`SELECT WHERE status=pending`) | 100ms–1s (poll interval) | Extra SELECT queries | Low |
| CDC (Debezium + Kafka Connect) | < 10ms | Reads WAL (no extra queries) | Higher (Debezium ops) |

**Choose polling** when: event volume is low, sub-second latency not required, no Kafka Connect infra.
**Choose CDC** when: high volume, sub-100ms latency required, Debezium already in stack.

### Critical Outbox Index

```sql
-- Partial index: only indexes rows needing publishing (small, fast)
CREATE INDEX idx_outbox_pending ON outbox (created_at)
WHERE status = 'pending';
```

### Scaling the Publisher

- Use `SELECT FOR UPDATE SKIP LOCKED` — multiple publisher instances can poll concurrently without double-publishing
- Partition by `aggregate_id % N` for ordered per-aggregate delivery

---

## Pattern Selection Decision Flow

```
Need all-or-nothing across multiple databases/services?
│
├── SAME database cluster / XA supported and same region?
│   └── 2PC (XA transactions) — strict ACID
│
└── DIFFERENT services with own databases (microservices)?
    └── SAGA
        ├── Simple flow, few steps, clear event boundaries?
        │   └── Choreography (events)
        └── Complex flow, audit required, failure handling complex?
            └── Orchestration (state machine)

Do any of your writes need to reliably trigger an event/message?
  └── YES → Add Outbox Pattern to whichever above you chose
```

---

## Idempotency Reminder

Sagas and outbox both produce at-least-once delivery. Every consumer MUST be idempotent:

```sql
-- Deduplication table pattern
INSERT INTO processed_events (event_id)
VALUES ($1)
ON CONFLICT (event_id) DO NOTHING;
-- If INSERT 0 0 → already processed, skip
```

Or use natural business idempotency: `UPDATE orders SET status='paid' WHERE id=$1 AND status='pending'` — applying the update again when already paid does nothing.

---

## Key Vocabulary

| Term | Definition |
|------|-----------|
| **2PC** | Two-Phase Commit — distributed ACID protocol using a coordinator and two phases (prepare + commit) |
| **XA Transaction** | Standard API (X/Open XA) for 2PC over heterogeneous databases |
| **Saga** | Long-running transaction broken into local steps with compensating actions |
| **Compensating Transaction** | Business operation that semantically undoes a committed local transaction |
| **Orchestrator** | Central service that commands Saga steps and manages compensation |
| **Choreography** | Event-driven Saga where services react to events autonomously |
| **Outbox Table** | Database table storing pending events, written atomically with the main entity |
| **CDC** | Change Data Capture — streams database WAL changes to a message broker |
| **Dual-Write** | Anti-pattern: writing to DB and message broker separately (either can fail) |
| **Idempotency Key** | Client-provided token ensuring the same request is processed exactly once |

---

## Real-World Patterns by Company

| Company | Pattern | Detail |
|---------|---------|--------|
| Uber Eats | Saga (Orchestration) | Order → payment → restaurant → driver assignment |
| Amazon | Saga (Choreography) | Each service reacts to events; massive decoupling |
| Stripe | Outbox + CDC | Charge created → event reliably published via WAL |
| Netflix | Outbox + Kafka | DVD/streaming event notifications |
| Google Spanner | Distributed 2PC via Paxos | Single-region strong consistency, multi-shard |
