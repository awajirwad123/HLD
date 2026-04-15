# Distributed Transactions — Architecture Deep-Dive

## The Core Problem

A **distributed transaction** spans multiple services or databases that each own their own data. The challenge: either all steps complete, or none do — across processes that can fail independently.

```
Order Service          Payment Service         Inventory Service
─────────────          ───────────────         ─────────────────
INSERT order           DEBIT $99              DECREMENT stock by 1
      │                       │                       │
      └─────── ALL must succeed, or ALL must roll back ───────┘

Problem: after payment succeeds, the inventory service crashes.
Customer is charged. Stock is wrong. Order is incomplete.
```

Two fundamentally different approaches:
1. **2-Phase Commit (2PC)** — distributed ACID transaction using a coordinator
2. **Saga pattern** — sequence of local transactions with compensating actions

---

## 1. Two-Phase Commit (2PC)

### Protocol

```
Phase 1 — PREPARE
────────────────
Coordinator → all Participants: "Can you commit?"
Each Participant:
  - Writes to WAL (durable log)
  - Acquires locks
  - Replies VOTE-YES or VOTE-NO

Phase 2 — COMMIT (or ABORT)
───────────────────────────
If all voted YES:
  Coordinator → all: "COMMIT"
  Participants: release locks, apply changes, ACK
Else:
  Coordinator → all: "ABORT"
  Participants: rollback, release locks, ACK
```

**Timeline diagram:**
```
Coordinator      DB-1 (Orders)       DB-2 (Payments)
    │                 │                     │
    ├──── PREPARE ───►│                     │
    ├──── PREPARE ────┼────────────────────►│
    │                 │                     │
    │◄── VOTE-YES ────│                     │
    │◄── VOTE-YES ────┼─────────────────────│
    │                 │                     │
    ├──── COMMIT ────►│                     │
    ├──── COMMIT ─────┼────────────────────►│
    │                 │                     │
    │◄──── ACK ───────│                     │
    │◄──── ACK ────────────────────────────│
    │                 │                     │
  DONE             unlocked              unlocked
```

### Failure Scenarios

| Failure Point | Impact | Recovery |
|--------------|--------|----------|
| Participant crashes before PREPARE reply | Coordinator times out → ABORT | Safe |
| Coordinator crashes after all VOTE-YES but before COMMIT | **Blocking** — participants hold locks indefinitely | Coordinator restart reads WAL, re-sends COMMIT |
| Participant crashes after VOTE-YES but before COMMIT | Must block until coordinator recovers | Coordinator timeouts, retries; participant checks coordinator log on restart |
| Network partition after PREPARE | Participants hold locks, cannot respond | Timeout → ABORT (if coordinator hasn't committed) |

### The Blocking Problem

2PC is a **blocking protocol**: if the coordinator fails after collecting VOTE-YES, all participants are stuck with locks held — they cannot commit or rollback without the coordinator's decision. This window = coordinator failure duration.

**Mitigations:**
- Coordinator writes decision to durable log before sending COMMIT (single point of truth)
- Use 3-Phase Commit (3PC) — adds a pre-commit phase to break blocking, but adds a round-trip
- Use Paxos/Raft to replicate coordinator decision → surviving nodes can recover

### When to Use 2PC

| Context | Fit |
|---------|-----|
| Same cloud provider, same region | ✅ Good — fast network, coordinator co-located |
| Same database cluster (XA transactions) | ✅ Good — built-in coordinator |
| Cross-datacenter, cross-cloud | ❌ Bad — latency amplifies blocking window |
| High-throughput transactional system | ❌ Bad — lock contention kills throughput |
| Low-latency requirement | ❌ Bad — 2 round-trips minimum |

**Real implementations:** PostgreSQL XA (`pg_prepared_xacts`), MySQL XA, Google Spanner (uses Paxos internally, externally behaves like 2PC), JTA/Atomikos in Java EE.

---

## 2. Saga Pattern

A **Saga** is a sequence of local transactions where each step publishes an event or message that triggers the next step. If a step fails, the Saga executes **compensating transactions** to undo prior steps.

### Core Principle

```
Local TX 1 → Local TX 2 → Local TX 3
   │                            │
   │       (if TX 3 fails)      │
   └── compensate TX 1 ◄── compensate TX 2 ◄─┘
```

No distributed lock. No coordinator holding state across services. Each service owns its own ACID transaction locally.

### Choreography vs Orchestration

#### Choreography — Event-Driven, Decentralized

Each service listens for events and decides what to do next. No central brain.

```
Order Service          Kafka                Payment Service      Inventory Service
─────────────          ─────                ───────────────      ─────────────────
Create Order    ──► [order.created]
                                    ──────► Debit Payment   ──► [payment.succeeded]
                                                                            ──────► Reduce Stock
                                                             [stock.reserved] ────► Update Order Status

Failure path (payment failed):
                              ◄── [payment.failed] ◄── Debit fails
Cancel Order ◄── [order.cancelled] ◄── Emit cancel event
```

**Pros:** Loose coupling, services don't know about each other, easy to add new steps.
**Cons:** Hard to track overall state, "where is my transaction right now?" requires distributed tracing, cyclic dependencies possible.

#### Orchestration — Centralized Saga Orchestrator

A single **Saga Orchestrator** (a separate service or state machine) tells each participant what to do and handles failures explicitly.

```
                      Saga Orchestrator
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   Order Service    Payment Service   Inventory Service
```

```
Orchestrator state machine:

START
  │
  ▼
STEP 1: Command "create_order" → Order Service
  │   SUCCESS → advance
  │   FAILURE → ABORT
  ▼
STEP 2: Command "debit_payment" → Payment Service
  │   SUCCESS → advance
  │   FAILURE → compensate STEP 1, ABORT
  ▼
STEP 3: Command "reserve_stock" → Inventory Service
  │   SUCCESS → DONE
  │   FAILURE → compensate STEP 2, compensate STEP 1, ABORT
  ▼
DONE
```

**Pros:** Explicit state, easy to debug, centralized failure handling, easy to add new steps in one place.
**Cons:** Orchestrator is a single point of failure (must be made durable), introduces coupling to orchestrator service.

### Choreography vs Orchestration Comparison

| Factor | Choreography | Orchestration |
|--------|--------------|---------------|
| Coupling | Low — services react to events | Moderate — services know the orchestrator |
| Visibility | Low — state distributed | High — state in one place |
| Debugging | Hard — requires distributed tracing | Easier — orchestrator has full history |
| Single point of failure | No | Yes (orchestrator must be durable) |
| Adding new steps | Easy — new service subscribes | Easy — change orchestrator logic |
| Best for | Simple, well-defined event flows | Complex multi-step transactions requiring full control |

### Compensating Transactions

A compensating transaction **semantically undoes** a committed local transaction. It is a business operation — not a database rollback.

```
Original Action             Compensating Action
──────────────              ───────────────────
Reserve seat on flight  →   Release reservation
Debit account $99       →   Credit account $99
Send notification email →   Send cancellation email (can't unsend!)
Ship package            →   Initiate return (not always possible)
```

⚠️ **Compensations are not always possible or perfect.** Notification emails cannot be "unsent". Shipped packages require a return flow. This is acceptable — Sagas are designed for business correctness, not ACID rollback semantics.

---

## 3. Outbox Pattern

### The Dual-Write Problem

The most common Saga/event system bug: writing to the database AND publishing to a message queue in two separate operations. Either can fail independently.

```
Bad pattern:
  BEGIN
  INSERT INTO orders (status='created') WHERE id=1
  COMMIT
  ← crash here →
  kafka.send("order.created")   ← THIS NEVER EXECUTES
```

Result: order exists in the DB, but `order.created` event is never published. Downstream services never execute their steps.

### The Outbox Solution

**Write the event to an `outbox` table in the SAME database transaction as the main entity.** A separate process (the outbox publisher) reads unprocessed outbox rows and publishes them to Kafka/SQS.

```
Application Transaction (atomic):
┌─────────────────────────────────────────────────────┐
│  INSERT INTO orders (id, status) VALUES (1, 'created')  │
│  INSERT INTO outbox (event_type, payload, status)        │
│         VALUES ('order.created', '{"order_id":1}', 'pending') │
└─────────────────────────────────────────────────────┘
          (single DB transaction — either both or neither)

Outbox Publisher (separate process):
  POLL: SELECT * FROM outbox WHERE status = 'pending' LIMIT 100
  FOR EACH row:
    kafka.send(row.event_type, row.payload)
    UPDATE outbox SET status='published' WHERE id = row.id
```

**Guarantees:** At-least-once delivery. If the publisher crashes after Kafka send but before marking the row as published, the row will be re-sent. Consumers must be idempotent.

### Transactional Outbox with CDC (Change Data Capture)

Instead of polling, use CDC (Debezium + Kafka Connect) to stream database WAL changes directly to Kafka. No polling process needed.

```
PostgreSQL WAL → Debezium (CDC connector) → Kafka topic
      │                                          │
Table change captured at DB level        Event published without polling
```

**Advantages over polling:**
- Lower latency (milliseconds vs polling interval)
- No extra DB load from periodic SELECT queries
- Atomic at WAL level — no separate marking process

### Outbox Table Schema

```sql
CREATE TABLE outbox (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    event_type  TEXT NOT NULL,               -- e.g. 'order.created'
    aggregate_id TEXT NOT NULL,              -- e.g. order_id
    payload     JSONB NOT NULL,
    status      TEXT NOT NULL DEFAULT 'pending',  -- pending | published | failed
    published_at TIMESTAMPTZ,
    retry_count  INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_outbox_pending ON outbox (status, created_at)
    WHERE status = 'pending';
```

---

## Pattern Comparison: 2PC vs Saga vs Outbox

| Concern | 2PC | Saga (Choreography) | Saga (Orchestration) | Outbox |
|---------|-----|---------------------|----------------------|--------|
| Atomicity guarantee | Strict ACID | Business-level eventual | Business-level eventual | Exactly-once write + at-least-once event |
| Failure handling | Coordinator protocol | Compensating events | Orchestrator state machine | At-least-once retry |
| Latency | High (2 round-trips + locks) | Low (async) | Low (async) | Low (async) |
| Lock contention | Yes — across all participants | No | No | No |
| Cross-service coupling | Low (protocol-level) | Low (events) | Moderate (orchestrator) | Low |
| Debugging difficulty | Low (coordinator log) | High | Medium | Low |
| Throughput | Low | High | High | High |

---

## Decision Guide: Which Pattern?

```
Do you need strict atomicity (all-or-nothing, same result as a single ACID transaction)?
├── YES + same DB or low-latency intra-cluster
│   └── USE: 2PC (XA or Spanner)
│
└── NO (business eventual correctness is acceptable)
    ├── Is the flow simple with clear event boundaries?
    │   └── USE: Saga — Choreography
    ├── Is the flow complex, multi-step, requires audit trail?
    │   └── USE: Saga — Orchestration
    └── Do you need to publish events reliably after a DB write?
        └── USE: Outbox Pattern (combine with either Saga type)
```

---

## Real-World Usage

| Company / System | Pattern | Why |
|-----------------|---------|-----|
| Uber Eats order flow | Saga (Orchestration) | Complex multi-service: order, payment, restaurant, driver |
| Amazon order pipeline | Saga (Choreography) | Highly decoupled microservices |
| Stripe payment events | Outbox + CDC | Guaranteed event delivery after charge creation |
| Google Spanner writes | Distributed 2PC via Paxos | Single-region strong consistency across shards |
| Netflix publishing | Outbox + Kafka | DVD shipment and streaming notifications |
