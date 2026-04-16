# Idempotency & Exactly-Once Semantics — Quick-Fire Interview Q&A

**Format:** Cover the answer before reading it. Aim for 30–60 seconds per question.

---

**Q1. What is idempotency, and how does it differ from a "safe" operation?**

An idempotent operation produces the same result and final state regardless of how many times it's called. A safe operation is read-only — it causes no state change. All safe operations are idempotent (reading the same value N times gives the same result), but not all idempotent operations are safe. `PUT /users/123` with the same body is idempotent (same final state) but not safe (it writes to the database).

---

**Q2. Which HTTP methods are idempotent by specification?**

GET, HEAD, PUT, DELETE are idempotent. POST is not (each call may create a new resource). PATCH is context-dependent — `SET amount = 5` is idempotent; `INCREMENT amount BY 1` is not. A client-side idempotency key makes POST idempotent in practice.

---

**Q3. A user clicks "Pay" in a mobile app. The network drops after the server processes the charge but before the client receives the response. The app shows an error. What should happen next?**

The client should retry the request using the same idempotency key. On the server, the idempotency key check finds the cached response from the first processing and returns it — no second charge is created. The user's account is charged exactly once despite the retry.

---

**Q4. Who generates the idempotency key — client or server?**

Always the client. The server cannot generate it because only the client knows whether this is a retry of a previous request or a new operation. If the server generated the key, a network-dropped response would mean the client has no key to retry with — defeating the purpose.

---

**Q5. With Stripe's idempotency model, what happens if you retry a request with the same key but a different request body?**

Stripe returns HTTP 422 Unprocessable Entity with an error indicating the body doesn't match the original. The server detects that `(api_key, idempotency_key)` was already processed with a different payload and refuses the request. The key is tied to both the operation identity and the payload hash.

---

**Q6. What is the difference between "at-least-once" and "exactly-once" delivery in Kafka?**

At-least-once: messages are always delivered but may be duplicated (producer retries on failure). The consumer may process the same message more than once. Exactly-once: Kafka's `enable_idempotence` + `transactional_id` ensures the broker stores each message exactly once and the consumer offset advances atomically with the publish — no duplicates reach an `isolation_level=read_committed` consumer.

---

**Q7. `enable_idempotence=True` is set on a Kafka producer. What guarantees does it provide?**

Kafka assigns a Producer ID (PID) and a per-partition sequence number to each message. If the producer retries (due to a timeout/network error), the broker uses the sequence number to detect and drop the duplicate — the message appears in the topic only once. This guards against producer-level duplicates within a session. It does NOT provide cross-session or cross-partition guarantees — that requires `transactional_id`.

---

**Q8. What does setting `transactional_id` on a Kafka producer add beyond `enable_idempotence`?**

It enables Kafka transactions: a group of sends to multiple partitions (and optionally a consumer offset commit via `send_offsets_to_transaction`) becomes atomic — all land or none land. The consumer must use `isolation_level=read_committed` to filter out messages from aborted transactions. Common use: publishing an event and advancing the consumer offset in a single atomic operation — "consume-process-produce" exactly-once.

---

**Q9. If a Kafka consumer commits the offset after processing but crashes between processing and committing, what happens on restart?**

The consumer re-reads the message from the last committed offset — the message is redelivered. This is the fundamental at-least-once scenario. The consumer must be idempotent: check a `processed_events` table with the message's `event_id` before processing, and insert into it atomically with the business logic update.

---

**Q10. What is the Outbox pattern and what problem does it solve?**

It solves the dual-write problem: if you write to a database AND publish to Kafka, one can fail and the other succeed — leaving data inconsistent. In the Outbox pattern, the business logic write and the outbox entry are in the same DB transaction. A separate poller reads unpublished outbox entries and publishes them to Kafka, then marks them published. Even if the poller crashes between publishing and marking, the retry will re-publish (at-least-once) — the consumer deduplicates on event_id.

---

**Q11. Can you just write to the database and publish to Kafka in the same code block without the Outbox? Why or why not?**

No — there's no distributed transaction between a relational DB and Kafka. If the DB write succeeds and the Kafka send fails (network error, broker down), you've lost the event. If the Kafka send succeeds first and the DB write fails, you've published an event for a transaction that never happened. The Outbox pattern makes the publish-to-Kafka a recoverable, retryable operation derived from the DB write.

---

**Q12. What is optimistic locking and when do you need it?**

Optimistic locking (Compare-And-Swap) reads a row with a `version` field, performs work, then updates with `WHERE id = $1 AND version = $2`. If another process updated the row first, the version won't match → 0 rows updated → retry. It prevents lost updates without holding row locks for the entire request duration. Use it when two concurrent requests could read the same state and both try to write (e.g., two threads updating account balance).

---

**Q13. How does `INSERT ... ON CONFLICT DO NOTHING` provide idempotency?**

By making the unique key of the insert operation (e.g., `event_id`) a database UNIQUE constraint. If the same `event_id` is inserted twice, the second INSERT silently does nothing and returns no rows. The caller checks if any rows were returned — zero rows means duplicate.

---

**Q14. What is a deterministic idempotency key and when should you use one?**

A key derived from the operation's logical identity: `SHA256(user_id + cart_id + cart_version)[:32]`. Because it's deterministic, the client can regenerate it after an app crash and auto-retry without asking the user. Use when the app should transparently recover from network failures. Use a random UUID key (new per action) when each user click should always be treated as a new operation.

---

**Q15. SQS FIFO queues have a "deduplication window." What does that mean?**

If you send a message with the same `MessageDeduplicationId` within a 5-minute window, SQS discards the duplicate. After 5 minutes, the same deduplication ID is treated as a new message. The deduplication window is a sliding 5-minute period — not based on acknowledgment. Use this for consumer-side protection against producer retries within that window.

---

**Q16. What race condition can occur between checking and acquiring an idempotency lock in Redis, and how do you prevent it?**

If you do `GET idem:key` (check) then `SET idem_lock:key 1 NX` (lock) as two separate commands, another concurrent request could pass the check and acquire the lock between your two commands — both proceed to process the same request. Prevention: use `SET NX` atomically (which is a single Redis command) or use a Lua script for check-then-lock atomically. Redis single-command `SET key value NX EX ttl` is inherently atomic.

---

**Q17. A payment service crashes after writing to the database but before publishing to Kafka. With the Outbox pattern, what happens?**

On restart, the outbox poller reads all `published=false` entries (including the one for the crashed operation). It publishes them to Kafka and marks them as published. The event is eventually published — guaranteed. This is why at-least-once delivery from the outbox to Kafka is acceptable as long as the Kafka consumer is idempotent.

---

**Q18. A team argues: "We use PUT for our payment API, so it's already idempotent — no need for idempotency keys." Is this correct?**

Incorrect in practice. HTTP PUT is idempotent in *specification* — same body → same state. But payment APIs typically use POST because they create a new charge resource with a server-assigned ID. A PUT that creates a charge (`PUT /charges/client-generated-id`) can be "idempotent" if the client controls the ID — but this is non-standard and requires the client to generate globally unique, collision-resistant IDs. The industry standard is POST + `Idempotency-Key: <uuid>` header.

---

**Q19. What happens if the Redis instance storing idempotency keys goes down?**

All idempotency checks fail — either: (a) always miss (treat as fresh) → potential duplicates, or (b) always error → no requests processed. Option (a) is the safer degraded mode for a payment system: duplicate charges are better than no charges, but you'd need downstream deduplication to reconcile. In production: use a Redis Cluster with replication, or failover sentinel. Stripe uses multi-region Redis for idempotency key storage specifically because this is a critical path.

---

**Q20. How does the "consume-transform-produce" Kafka pattern work with exactly-once semantics?**

A service reads from topic A, transforms the data, and writes to topic B while advancing its offset atomically:
```python
producer.begin_transaction()
producer.send("topic-b", key, transformed_value)
producer.send_offsets_to_transaction(
    {TopicPartition("topic-a", partition): OffsetAndMetadata(offset+1)},
    consumer_group_id
)
producer.commit_transaction()
```
The commit is atomic: the message lands in topic-B AND the offset advances in topic-A's consumer group at the same moment. On crash and restart, the consumer resumes from the committed offset — no reprocessing, no duplicate in topic-B.
