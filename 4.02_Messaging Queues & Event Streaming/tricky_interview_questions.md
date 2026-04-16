# Messaging Queues & Event Streaming — Tricky Interview Questions

## Q1: "We use Kafka with `enable.auto.commit=true`. Is our system at-least-once or exactly-once?"

**Why it's tricky:** Auto-commit sounds safe — the framework handles it. In reality it creates a subtle data-loss window that candidates almost always miss.

**Strong Answer:**

With `enable.auto.commit=true`, the consumer automatically commits offsets every `auto.commit.interval.ms` (default 5 seconds). This creates a **at-most-once** risk window, not at-least-once:

```
Timeline:
  t=0:   Consumer polls messages 1–50
  t=1s:  Starts processing message 1
  t=5s:  AUTO-COMMIT fires → offsets 1–50 marked as processed
  t=6s:  Consumer crashes before finishing message 10
  t=7s:  Consumer restarts — reads from offset 51
         Messages 10–50 were NEVER processed but offset was committed → LOST
```

For at-least-once delivery you need `enable.auto.commit=false` and commit offsets only after all messages in the batch are fully processed. The trade-off: crash after processing but before commit = messages re-delivered (duplicate), so consumers must be idempotent.

Auto-commit is only safe for truly fire-and-forget telemetry where at-most-once is explicitly acceptable.

---

## Q2: "We have 12 Kafka partitions and 15 consumer instances in our group. Is this fine?"

**Why it's tricky:** It sounds like more consumers = more throughput. It actually wastes 3 instances and can cause churn.

**Strong Answer:**

Not fine — 3 consumer instances will be permanently idle because each partition is assigned to at most one consumer per group. Having more consumers than partitions means some consumers have nothing to read.

Worse: adding/removing consumers triggers a **rebalance** — the partition-to-consumer assignment is recalculated for the entire group. During a rebalance, all consumption in the group pauses. Idle consumers that periodically poll can trigger unnecessary rebalances if they timeout.

**Correct approach:**
- Set consumer count ≤ partition count for full utilization
- If you need more throughput, increase partitions first (partitions can be increased; can't be decreased)
- Note: increasing partitions for a topic that uses key-based routing breaks the key→partition mapping for existing keys — re-partition carefully

**Exception:** Intentionally having fewer consumers than partitions is valid for fault-tolerance head-room — if a consumer fails, its partitions are reassigned to remaining healthy consumers without dropping to zero readers.

---

## Q3: "Kafka guarantees exactly-once delivery. Can I stop making my consumers idempotent?"

**Why it's tricky:** Kafka's exactly-once is prominently marketed but has a precise and limited scope that most candidates don't know.

**Strong Answer:**

No. Kafka's exactly-once guarantee applies only **within the Kafka pipeline** — and only when using Kafka Transactions (`KafkaProducer.init_transactions()`) with a consumer that sets `isolation.level=read_committed`.

It guarantees:
- A message is produced to an output topic exactly once
- An input consumer offset is committed atomically with the output write

What it does NOT cover:
- Writing to an external database (PostgreSQL, MongoDB, Redis)
- Calling an external API
- Sending an email or push notification

```
Kafka pipeline (exactly-once):
Input topic → Kafka Streams processor → Output topic  ✅ covered

External write (NOT covered):
Input topic → Consumer → UPDATE database  ❌ still at-least-once
```

For external side effects, you still need idempotent consumers (dedup table, `INSERT ... ON CONFLICT DO NOTHING`, natural business idempotency, or the Outbox pattern). Kafka exactly-once is most useful for pure stream processing pipelines where the entire computation stays within Kafka.

---

## Q4: "A single message in our payment queue is stuck. Every consumer tries to process it, fails, and retries — blocking the entire queue. How do you fix this?"

**Why it's tricky:** This is the "poison pill" problem. The fix requires a DLQ configuration, but many candidates just say "DLQ" without explaining the mechanics or the immediate mitigation.

**Strong Answer:**

This is a classic **poison pill** scenario. The fix has two parts:

**Immediate mitigation (for the blocked queue):**
1. Identify the stuck message by inspecting the queue/consumer logs
2. In RabbitMQ: use the management UI to move/delete the specific message
3. In Kafka: skip by manually advancing the consumer offset past the bad message: `kafka-consumer-groups.sh --reset-offsets --to-offset {n+1}`
4. In SQS: delete the message directly via the console or `delete_message` API

**Structural fix (prevent recurrence):**
Configure a Dead Letter Queue with a `maxReceiveCount` (SQS) or retry limit (RabbitMQ `x-dead-letter-exchange`):

```python
# SQS: automatically route after 3 failed receives
sqs.create_queue(
    QueueName="payments",
    Attributes={
        "RedrivePolicy": json.dumps({
            "deadLetterTargetArn": dlq_arn,
            "maxReceiveCount": "3"
        })
    }
)
```

After `maxReceiveCount` failures, the broker moves the message to the DLQ automatically — unblocking the main queue. Add an alert on DLQ depth > 0 so ops is notified immediately.

**Root cause investigation:** The DLQ message includes the failure reason. Common causes: schema change broke deserialization, upstream data corruption, consumer bug with specific data shape. Fix the root cause, then replay from DLQ.

---

## Q5: "We need strict ordering of all events for a given user. How do you achieve this in Kafka at scale?"

**Why it's tricky:** Kafka only guarantees ordering within a single partition. Getting global ordering while maintaining throughput is a nuanced trade-off.

**Strong Answer:**

Kafka guarantees strict ordering within a partition. Global ordering across all partitions is not possible without sacrificing throughput (using a single partition).

**The correct approach: partition by entity key**

Route all events for a given `user_id` to the same partition using the producer key:

```python
producer.send(
    topic="user-events",
    key=user_id.encode(),       # Same key → same partition via hash(key) % num_partitions
    value=event_payload.encode()
)
```

All events for `user_id=42` always land in the same partition and are consumed in order by the same consumer instance.

**What this gives you:**
- Strict ordering per user ✅
- Parallelism across users (different users on different partitions) ✅
- No cross-user ordering guarantees (user A and user B events may interleave) — usually acceptable

**Hotspot concern:** If one user generates 90% of events, their partition becomes a hotspot. Mitigations:
- Composite key: `{user_id}_{event_type}` — spread high-volume users across more partitions
- Dedicated high-volume topic with more partitions for power users

**Don't do this:** Use a single partition for global ordering → entire topic is single-threaded → throughput = one consumer, no parallelism.

---

## Q6: "What exactly is backpressure and how do you detect it before things go wrong?"

**Why it's tricky:** Candidates often describe backpressure vaguely as "the queue getting full." The strong answer quantifies it and describes the specific signals to watch.

**Strong Answer:**

Backpressure is the condition where a consumer processes messages slower than the producer generates them — causing the lag (unconsumed message backlog) to grow unboundedly. Left unchecked it results in hours of delayed processing, OOM crashes (if messages are buffered in-memory), and eventually data loss when retention expires.

**Detection — three signals to monitor:**

1. **Consumer group lag per partition** (Kafka):
   ```
   lag = end_offset - committed_offset
   ```
   Alert when lag > threshold (e.g., 10,000 for SLA-sensitive topics). Growing lag means consumer is falling behind. This is the primary signal.

2. **Queue depth** (RabbitMQ/SQS):
   - `rabbitmq_queue_messages_ready` metric rising
   - SQS `ApproximateNumberOfMessagesVisible` CloudWatch metric rising

3. **Consumer poll interval breaching** (Kafka):
   - `max.poll.interval.ms` exceeded → Kafka logs "Member ... leaving group due to poll timeout"
   - This triggers a rebalance and temporary processing halt

**Response tiers:**
- Lag growing slowly: scale consumer instances (up to partition count)
- Lag growing rapidly: reduce `max_poll_records`, optimise processing, add circuit breaker on downstream calls
- Lag not recovering: increase partition count, split topic, horizontally scale the downstream dependency

---

## Q7: "Should you use Kafka or a traditional message queue for a simple job queue that distributes email-sending tasks across workers?"

**Why it's tricky:** Kafka is fashionable. Defaulting to it regardless of the use case is a common mistake.

**Strong Answer:**

For a simple work distribution queue (email sending), a traditional queue — SQS, RabbitMQ, or even Redis with a queue library — is the better choice. Here's why:

**Kafka is over-engineered for this scenario:**
- Email jobs are consumed once, not replayed — you don't need log retention
- Workers compete for jobs — you want competing consumer semantics, not independent consumer groups all receiving every message
- Kafka's minimum operational footprint (broker cluster, ZooKeeper/KRaft, monitoring) is significant overhead for what is essentially "pick up a task, do work, mark done"
- Kafka's smallest meaningful deployment is 3 brokers — SQS is zero-ops

**Where Kafka wins and makes sense:**
- You need replay: re-process historical email jobs to fix a rendering bug
- Multiple independent services all need the same event (notification service, analytics, audit), not just email workers
- Very high throughput that SQS can't handle at reasonable cost
- The email task is part of a larger event pipeline

**What to actually use:**
- Simple job queue: SQS (AWS) or RabbitMQ
- If email is one of many consumers of an `order.created` event: Kafka consumer group per service
- If the decision is unclear: SQS now, migrate to Kafka if replay or fan-out requirements emerge

The principle: **match tool complexity to problem complexity**. Kafka is powerful but carries real operational cost and cognitive overhead that is unwarranted for a basic work queue.
