# Distributed Transactions — Interview Simulator

**Instructions:** Set a 35-minute timer per scenario. Attempt completely before reading guidance. A senior interviewer expects pattern justification, failure mode awareness, compensation design, and delivery guarantees.

---

## Scenario 1: E-Commerce Order Checkout Pipeline

**Prompt:**
> "Design the transaction model for an e-commerce checkout flow. When a user clicks 'Buy', the system must: (1) create an order, (2) charge the payment, (3) decrement inventory, (4) send a confirmation email. Each of these lives in a separate microservice with its own database. You cannot use a distributed database. Walk me through the complete design, failure handling, and the guarantees you provide to the user."

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "Payment succeeds but inventory is out of stock. The user was charged. What happens?"
2. "The orchestrator crashes after charging payment but before checking inventory. What now?"
3. "The same user double-clicks 'Buy' and you receive two identical requests within 100ms. What prevents a double charge?"
4. "You said the email can't be compensated. What's your fallback?"

---

### Guidance (Read After Attempting)

**Pattern choice: Saga (Orchestration) + Outbox**

Orchestration over choreography because:
- Complex conditional failure paths (payment fail vs inventory fail have different compensations)
- Audit trail required for financial operations
- Clear state machine needed for support team visibility

**Saga Steps + Compensations:**

```
Step 1: CreateOrder (OrderService)
  ↓ success
Step 2: DebitPayment (PaymentService)
  ↓ success
Step 3: ReserveStock (InventoryService)
  ↓ success
Step 4: SendConfirmationEmail (NotificationService)
  ↓ success
DONE

Compensation table:
  Step 3 fails → compensate Step 2 (refund), compensate Step 1 (cancel order)
  Step 2 fails → compensate Step 1 (cancel order)
  Step 4 fails → no compensation needed (email is best-effort; order is already complete)
```

**Outbox for reliable event publishing:**
- Each service step is triggered by the orchestrator sending a command via a message queue
- The orchestrator writes the next command to an outbox table atomically with updating its own state
- This ensures that if the orchestrator crashes after updating state but before sending the next command, the outbox publisher re-sends it

**Orchestrator state schema:**
```sql
CREATE TABLE saga_executions (
    saga_id         UUID PRIMARY KEY,
    current_step    TEXT NOT NULL,
    completed_steps JSONB NOT NULL DEFAULT '[]',
    status          TEXT NOT NULL,   -- running | completed | compensating | compensated | failed
    context         JSONB NOT NULL,  -- order_id, user_id, amount, product_id
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Idempotency for double-click:**
- Client generates a UUID idempotency key before sending the checkout request
- Server stores `(idempotency_key → saga_id)` on first request
- Second request with same key returns the existing saga result immediately — no second Saga starts
- Storage: `checkout_idempotency(key UUID PK, saga_id UUID, created_at TIMESTAMPTZ)`

**Email compensation:**
Cannot undo a sent email. Compensation: send a "Your order was cancelled" email. This is a business-level semantic compensation, not a data rollback. Make this explicit to the interviewer — it's a sign of strong real-world thinking.

**Payment service down during compensation:**
Orchestrator persists `COMPENSATING` state + pending compensation steps. Retries with exponential backoff. Alert fires after 3 failed retries. Human escalation path documented for ops team. Kafka DLQ captures permanently failed events.

**End-to-end guarantee to user:**
"Your order is either fully placed (all four steps completed) or fully cancelled and any charge is refunded. We cannot guarantee the confirmation email arrives, but we will always send a cancellation email if the order fails."

---

## Scenario 2: Bank Transfer — Cross-Account Money Movement

**Prompt:**
> "Design a bank-to-bank transfer: debit account A by $500 and credit account B by $500. Account A is in PostgreSQL database 1. Account B is in PostgreSQL database 2. Both databases are in the same AWS region. The business requirement: money must never be lost or created — if the debit happens, the credit must happen. What transaction model do you use and why? How do you handle failures?"

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "You chose 2PC — the coordinator crashed after both participants voted YES. Both accounts are locked. How long are users blocked?"
2. "You chose Saga — payment was debited but credit crashed. Compensation runs and refunds the debit. The customer sees '$500 missing for 3 seconds'. Is that acceptable for a bank?"
3. "Could you solve this with a single database instead?"
4. "What if the two accounts are in different countries with 200ms latency between them?"

---

### Guidance (Read After Attempting)

**This scenario has no perfect answer — that is the point.**

**Option A: 2PC (XA)**

Appropriate here because:
- Same region, same cloud provider — fast network (< 5ms latency)
- Both PostgreSQL — both support XA
- Strict atomicity is a genuine business requirement (not just a preference)
- Transaction volume is manageable (intrabank transfers are not Twitter-scale)

```python
# PostgreSQL XA
conn1.tpc_begin(xid1)
conn2.tpc_begin(xid2)
conn1.execute("UPDATE accounts SET balance = balance - 500 WHERE id = $1", account_a)
conn2.execute("UPDATE accounts SET balance = balance + 500 WHERE id = $1", account_b)
conn1.tpc_prepare()   # Phase 1
conn2.tpc_prepare()
conn1.tpc_commit()    # Phase 2
conn2.tpc_commit()
```

Blocking risk: coordinator crash during commit. Mitigation: coordinator logs decision to durable store before sending COMMIT; automated recovery restarts coordinator within 30 seconds; acceptable for a bank where a 30s lock is preferable to incorrect account balances.

**Option B: Saga with Escrow**

Use an intermediary escrow account to make the Saga safe:
1. Debit Account A → Escrow (local TX on DB1)
2. Credit Escrow → Account B (local TX on DB2)
3. Close Escrow (local TX on DB1)

If step 2 fails: escrow → Account A (compensate). Funds are never lost because they sit in escrow at all times — the system is always consistent at the escrow level. This is how SWIFT international transfers work.

**Cross-country scenario (200ms latency):**
2PC becomes dangerous — 200ms round-trip means locks are held for 200ms minimum. Under load, this creates severe contention. The right answer is escrow-based Saga with monotonic sequence numbers, accepting 200–500ms of "in-flight" transfer state — which is exactly how international SWIFT transfers work (they take hours and use escrow accounts throughout).

**Single database solution:**
If both accounts are in the same DB (or same Postgres cluster), a regular local ACID transaction solves this with zero overhead:
```sql
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = $1 AND balance >= 500;
UPDATE accounts SET balance = balance + 500 WHERE id = $2;
COMMIT;
```
This is always the right answer when possible. Distributed transactions are a solution to a problem you only have when you've already distributed your data.

---

## Scenario 3: Ride-Sharing — Driver Assignment Flow

**Prompt:**
> "You're designing the ride request flow for a ride-sharing platform. When a user requests a ride: (1) the Trip Service creates a trip record, (2) the Matching Service assigns an available driver, (3) the Notification Service pushes alerts to both rider and driver, (4) the Billing Service reserves payment authorization. Each service has its own database. Design the transaction model, failure handling, and explain how you prevent double-charging and double-assignment."

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "A driver accepts the trip but their phone dies before they receive the push notification. What do you do?"
2. "Two users simultaneously request rides, and the Matching Service assigns the same driver to both. How do you prevent this?"
3. "The Billing Service rejects the payment auth (insufficient funds). The driver was already notified. What's the UX and the compensation?"
4. "At peak (Friday evening), you process 50,000 ride requests per minute. Does your design hold?"

---

### Guidance (Read After Attempting)

**Pattern: Saga (Orchestration) + Outbox, with explicit driver reservation locking**

**Saga Steps:**
```
Step 1: CreateTrip (TripService) → trip_id, status=matching
Step 2: ReserveDriver (MatchingService) → driver_id assigned, driver locked
Step 3: AuthorizePayment (BillingService) → authorization_id
Step 4: NotifyRider (NotificationService) → push sent
Step 5: NotifyDriver (NotificationService) → push sent
→ COMPLETED

Compensations:
  Step 5 fails → retry (push is best-effort); trip proceeds
  Step 4 fails → retry; trip proceeds
  Step 3 fails (payment auth rejected):
    → Release driver reservation (Step 2 compensation)
    → Cancel trip (Step 1 compensation)
    → Notify rider: "Payment method declined"
  Step 2 fails (no drivers available):
    → Cancel trip (Step 1 compensation)
    → Notify rider: "No drivers available"
```

**Preventing double-assignment (race condition):**
The Matching Service must use an atomic "reserve" operation:
```sql
UPDATE drivers
SET status = 'assigned', trip_id = $1
WHERE id = $2 AND status = 'available'   -- Optimistic lock
RETURNING id;
-- If 0 rows returned: driver was already taken → pick a different driver
```
This ensures a driver can only be assigned to one trip at a time, even under concurrent requests.

**Driver notification failure (phone dies):**
Notifications are idempotent retries at the application level. The Notification Service retries push delivery for up to 60 seconds. Meanwhile, the trip is in `status=notifying_driver`. If the driver's app comes back online within the acceptance window (30–60s), it fetches the pending trip assignment via polling.

**Payment auth failure after driver notified:**
This is where Saga shines: run compensations.
1. Mark trip `status=payment_failed`
2. Release driver back to `available` status (compensation for Step 2)
3. Send apology push to driver: "Ride request cancelled — rider's payment failed"
4. Show rider: "Payment method declined. Please update your card."

The driver loses a few seconds of availability — this is acceptable. The key is the compensation runs immediately and both parties are notified synchronously.

**Throughput at 50K requests/minute:**
- 50K/min ≈ 833/sec. Each Saga is async and independent.
- Orchestrator is stateless during execution (state in DB) → horizontally scalable
- Matching Service is the bottleneck: driver pool contention under load
  - Partition the driver pool by geo-tile (Uber's H3 hexagon grid)
  - Each matching request only contends with others in the same tile
  - 833 global requests/sec distributes to < 10 per tile at normal density
- Outbox publisher: use CDC (Debezium) for sub-10ms latency at this throughput; polling at 833 events/sec would require very short poll intervals and high DB load
