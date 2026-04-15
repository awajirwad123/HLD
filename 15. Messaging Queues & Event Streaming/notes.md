# Messaging Queues & Event Streaming — Notes & Reference

## Queue vs Stream — One-Line Distinction

| | Queue | Stream |
|--|-------|--------|
| **Message fate** | Consumed and removed | Retained for configurable period |
| **Consumers** | Competing (one gets each msg) | Independent groups (all get all msgs) |
| **Replay** | ❌ | ✅ From any offset |
| **Mental model** | Task queue | Commit log |

---

## Delivery Guarantee Cheat Sheet

| Guarantee | Loss? | Duplicate? | How | Consumer requirement |
|-----------|:-----:|:----------:|-----|---------------------|
| **At-most-once** | Possible | Never | Commit offset before processing | None |
| **At-least-once** | Never | Possible | Commit offset after processing | Idempotent |
| **Exactly-once** | Never | Never | Kafka Transactions / SQS FIFO dedup | Idempotent (external writes) |

**Key fact:** True end-to-end exactly-once requires broker-level deduplication AND idempotent consumer side effects. Kafka's exactly-once only covers the Kafka pipeline itself — writing to a DB outside Kafka is still at-least-once unless you use the Outbox pattern or a transactional DB.

---

## Kafka Key Numbers

| Parameter | Default | What it controls |
|-----------|---------|-----------------|
| `acks` | `1` | Durability — use `all` for production |
| `min.insync.replicas` | `1` | Minimum replicas that must ack → set to `2` with `acks=all` |
| `retention.ms` | `604800000` (7 days) | How long messages are kept |
| `max.poll.records` | `500` | Backpressure — max messages pulled per poll |
| `max.poll.interval.ms` | `300000` (5 min) | Max time between polls before session expires |
| `auto.commit.interval.ms` | `5000` (5s) | Auto-commit interval — set `enable.auto.commit=false` |
| `replication.factor` | `1` (dev) | Number of replicas — use `3` in production |
| `num.partitions` | `1` | Parallelism — max consumers = num.partitions per group |

---

## Kafka Durability Config (Production)

```python
# Producer
producer = KafkaProducer(
    acks="all",                 # Wait for all in-sync replicas
    enable_idempotence=True,    # Deduplicate retries at broker
    retries=5,
    retry_backoff_ms=200,
)

# Broker topic config
# min.insync.replicas=2  (require 2 of 3 replicas to ack)
# replication.factor=3
```

---

## Partition + Consumer Group Rules

```
Rule: max useful consumers per group = number of partitions
  - 6 partitions + 3 consumers → 2 partitions per consumer
  - 6 partitions + 6 consumers → 1 partition per consumer  (max parallelism)
  - 6 partitions + 8 consumers → 2 consumers idle  (waste)

Ordering guarantee: strict within a partition, none across partitions
  → Use same partition key for messages that must be ordered (e.g., user_id)
  → All events for user_id=42 go to same partition → processed in order
```

---

## Dead Letter Queue Design Rules

1. **Always include failure metadata** — reason, retry count, original offset, timestamps
2. **Alert on DLQ depth > 0** — every DLQ message is a bug or bad data that needs investigation
3. **DLQ messages must be re-processable** — after fixing the bug, replay DLQ
4. **Never silently discard** from DLQ — set a long retention (14 days)
5. **Separate DLQ per topic** — `orders.DLT`, `payments.DLT` (not one global DLQ)

---

## Backpressure Recognition + Response

```
Symptom                          Signal
───────                          ──────
Consumer lag growing             consumer_group_lag metric rising
Consumer poll interval breached  kafka.consumer.session_timeout alert
Queue depth unbounded            queue_depth CloudWatch metric rising
Consumer OOM / GC pressure       JVM/process heap metrics

Response
────────
1. Scale out consumers (up to partition count)
2. Reduce max_poll_records (process less per batch)
3. Optimize processing function (DB query, I/O)
4. Increase partitions (allows more consumer parallelism)
5. Apply circuit breaker on downstream calls in consumer
```

---

## System Comparison Reference Card

| | Kafka | RabbitMQ | SQS (Standard) | SQS FIFO |
|--|-------|----------|----------------|----------|
| Retention | Days/forever | Until consumed | 1–14 days | 1–14 days |
| Throughput | 10M+ msg/sec | 50–100K/sec | Unlimited | 3K/sec per group |
| Ordering | Per partition | Per queue | Best-effort | Per MessageGroupId |
| Delivery | At-least-once / exactly-once (tx) | At-least-once | At-least-once | Exactly-once (dedup ID) |
| Replay | ✅ | ❌ | ❌ | ❌ |
| DLQ | Manual (DLT pattern) | `x-dead-letter-exchange` | `RedrivePolicy` | `RedrivePolicy` |
| Managed? | Self-hosted (or Confluent Cloud) | Self-hosted (or CloudAMQP) | ✅ Fully managed | ✅ Fully managed |

---

## RabbitMQ Exchange Types Summary

| Type | Routing | Use Case |
|------|---------|----------|
| **Direct** | Exact routing key match | Point-to-point (one event → one queue) |
| **Topic** | Pattern (`orders.*`, `*.failed`) | Content-based routing to multiple queues |
| **Fanout** | Broadcast to all bound queues | Pub/Sub — one event → all services |
| **Headers** | Match message headers | Rich routing without key-based logic |

---

## Producer-Consumer Pattern Selection

| Pattern | When | Implementation |
|---------|------|----------------|
| **Competing consumers** | Scale processing of independent tasks | N consumers on same queue/group |
| **Pub/Sub fan-out** | One event → multiple independent reactions | Fanout exchange (RabbitMQ) or separate consumer groups (Kafka) |
| **Priority queue** | High-priority tasks must jump the queue | RabbitMQ `x-max-priority`; or separate high/low queues |
| **Request-reply** | Async RPC via messaging | `reply-to` header + correlation ID |
| **Event sourcing** | Full history of state changes | Kafka with infinite retention + log compaction |

---

## Key Interview Phrases

- **"At-least-once + idempotent consumer = effectively exactly-once at the application level"** — this is the standard production pattern
- **"Consumer group lag is the primary health metric for a Kafka consumer"**
- **"Partition count = max degree of parallelism for a consumer group"**
- **"DLQ messages are bugs, not noise — treat them like failed jobs that must be investigated"**
- **"Kafka is a log, not a queue — messages aren't consumed, they're read at an offset"**
- **"Don't increase poll timeout to fix slow consumers — fix the processing or scale out"**
