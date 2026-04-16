# Distributed Transactions — Quick-Fire Q&A

**Format:** Answer each question in ≤2 sentences before checking the answer.

---

**Q1. What are the two phases in 2-Phase Commit and what happens in each?**
Phase 1 (Prepare): the coordinator asks all participants if they can commit; each acquires locks, writes to WAL, and votes YES or NO. Phase 2 (Commit/Abort): if all voted YES the coordinator sends COMMIT and participants apply changes; if any voted NO it sends ABORT and all roll back.

---

**Q2. What is the blocking problem in 2PC?**
If the coordinator crashes after all participants have voted YES but before sending COMMIT, participants are stuck holding locks and cannot proceed — they don't know whether to commit or abort. This window lasts until the coordinator is recovered and replays its durable decision log.

---

**Q3. How does a coordinator recover after a crash mid-2PC?**
On restart, the coordinator reads its durable WAL log. If a COMMIT decision was written before the crash, it re-sends COMMIT to all participants. If no decision was recorded, it sends ABORT. Participants accept these replayed decisions.

---

**Q4. What is a Saga?**
A Saga is a sequence of local ACID transactions, one per service, where each step publishes an event that triggers the next step. If any step fails, previously completed steps are undone by executing compensating transactions in reverse order.

---

**Q5. What is the difference between Saga Choreography and Orchestration?**
In choreography, each service independently reacts to events from a message broker — there is no central brain. In orchestration, a dedicated Saga Orchestrator explicitly commands each service step and manages compensation, holding all state in one place.

---

**Q6. What is a compensating transaction? Give an example.**
A compensating transaction is a business operation that semantically undoes a previously committed local transaction — not a database rollback. Example: if an order was created but payment failed, the compensating transaction for order creation is cancelling that order.

---

**Q7. Can every action be compensated? What do you do when it can't?**
No — sending an email, shipping a package, or posting a notification cannot be truly undone. The best approach is to execute a best-effort compensation (e.g., send a cancellation email, initiate a return flow) and accept that Sagas provide business-level eventual correctness, not strict ACID atomicity.

---

**Q8. What is the dual-write problem?**
Writing to a database and a message broker (Kafka, SQS) as two separate operations, where either can fail independently. If the app crashes after the DB write but before the Kafka publish, the event is lost and downstream services never proceed — causing data inconsistency.

---

**Q9. How does the Outbox pattern solve the dual-write problem?**
Instead of writing to Kafka directly, the event is written to an `outbox` table in the same database transaction as the main entity. A separate outbox publisher process reads pending rows and publishes them to Kafka, guaranteeing that if the DB write commits, the event will eventually be published.

---

**Q10. What delivery guarantee does the Outbox pattern provide?**
At-least-once. If the publisher sends the event to Kafka but crashes before marking the row as `published`, it will re-send on next poll. Consumers must therefore be idempotent to handle duplicate deliveries.

---

**Q11. What is `SELECT FOR UPDATE SKIP LOCKED` and why is it used with the Outbox pattern?**
It acquires row-level locks on selected rows but skips rows already locked, allowing multiple outbox publisher instances to process different batches concurrently without picking up the same row twice. This enables horizontal scaling of the publisher.

---

**Q12. What is the difference between 2PC and Saga in terms of atomicity?**
2PC provides strict ACID atomicity — either all changes commit or none do, with no intermediate state visible. Sagas provide eventual business correctness — partial states are temporarily visible during execution, and failures result in compensations rather than true rollback.

---

**Q13. When would you choose 2PC over Sagas?**
When strict atomicity is required (e.g., financial systems where a partial commit would cause real money loss), services are in the same cluster or same cloud region, latency is acceptable, and lock contention is manageable. 2PC is appropriate for intra-DB scenarios (PostgreSQL XA, Spanner).

---

**Q14. What does an Orchestrator-based Saga need to persist?**
The orchestrator must durably persist its current state (which step is in progress, which steps have completed) so that if it crashes mid-Saga, it can resume and continue or compensate from the exact point of failure. Without durable state, a crash loses the Saga execution entirely.

---

**Q15. What is CDC and how does it improve on polling for outbox delivery?**
CDC (Change Data Capture) streams database WAL changes directly to Kafka via a connector like Debezium, eliminating the need for a polling process. It provides sub-10ms latency vs 100ms–1s polling intervals, adds zero extra DB query load, and is more reliable than maintaining a polling service.

---

**Q16. What is an idempotency key and where is it generated?**
An idempotency key is a client-generated UUID included in API requests so the server can detect and deduplicate retries. The client generates it (not the server) and the server stores it alongside the result so that retrying with the same key returns the cached result rather than re-executing the operation.

---

**Q17. In a Saga with 5 steps, step 4 fails. Which compensations run and in what order?**
Compensations run for steps 3, 2, and 1 — in reverse order. Step 4 failed before committing, so it needs no compensation. Step 5 never ran. Compensation order is reverse chronological to preserve dependency correctness (undo effects of later steps before earlier ones).

---

**Q18. What is an XA transaction?**
XA is the X/Open standard for two-phase commit across heterogeneous resource managers (databases, message brokers). It defines `xa_start`, `xa_end`, `xa_prepare`, `xa_commit`, `xa_rollback` interfaces. PostgreSQL, MySQL, and most JEE application servers support XA.

---

**Q19. Does adding the Outbox pattern mean you no longer need idempotent consumers?**
No. The Outbox publisher provides at-least-once delivery — if it crashes after publishing but before marking the row as published, the message is re-sent. Consumers must still be idempotent to handle duplicate messages correctly.

---

**Q20. What is the key operational difference between 2PC's blocking and a Saga's failure handling?**
In 2PC, a coordinator failure leaves all participants blocked holding locks — the system cannot make progress without recovery, which can take minutes. In a Saga, a step failure triggers explicit compensations that run immediately without blocking other operations; the system remains available throughout.
