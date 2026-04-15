# Messaging Queues & Event Streaming — Architecture Deep-Dive

## The Core Problem

Services communicating synchronously (HTTP/gRPC) are tightly coupled: if the downstream service is slow or down, the upstream caller blocks or fails. Messaging decouples them:

```
Synchronous (tight coupling):
  OrderService ──HTTP──► PaymentService   ← blocked until payment responds
                                           ← order fails if payment is down

Asynchronous (loose coupling):
  OrderService ──► [Message Broker] ──► PaymentService
                                         ← order returns immediately
                                         ← payment processes whenever it's ready
```

Two fundamentally different messaging models:
1. **Message Queue** — point-to-point delivery, message consumed and removed
2. **Event Stream** — persistent log, consumers read at their own pace, messages retained

---

## 1. Queues vs Streams

### Message Queue (RabbitMQ, SQS, ActiveMQ)

```
Producer → [Queue] → Consumer

Key properties:
- Message is CONSUMED (removed) after one consumer processes it
- Competing consumers: N consumers share a queue, each message goes to exactly ONE
- No message history — consumers can't re-read past messages
- Broker tracks delivery state (delivered / acked / unacked)
```

**Mental model:** A task queue — each job is picked up once by the next available worker.

```
                  ┌──── Worker A  ← processes msg 1, 3, 5
Producer → Queue ─┤
                  └──── Worker B  ← processes msg 2, 4, 6
```

### Event Stream (Kafka, Kinesis, Pulsar)

```
Producer → [Partitioned Log] → Consumer Group A (offset N)
                             → Consumer Group B (offset M)
                             → Consumer Group C (offset P)

Key properties:
- Message is RETAINED for a configurable time (hours, days, forever)
- Multiple independent consumer groups each maintain their own offset
- Any consumer can replay from any offset in history
- Ordered within a partition; parallel across partitions
```

**Mental model:** A commit log — any reader can replay from any point, independently.

```
                              ┌─► Analytics Service (offset 100)
Producer → Topic Partition ───┼─► Notification Service (offset 99)
                              └─► Audit Service (offset 85 — replay in progress)
```

### Queue vs Stream Decision

| Factor | Queue (RabbitMQ/SQS) | Stream (Kafka) |
|--------|---------------------|----------------|
| Message retention | Until consumed | Days / forever |
| Multiple consumers | Competing (one gets each msg) | Independent groups (each gets all msgs) |
| Replay | ❌ Not possible | ✅ Replay from any offset |
| Ordering | FIFO within queue (SQS FIFO) | Strict per partition |
| Throughput | Moderate (100K msg/sec) | Very high (10M+ msg/sec per cluster) |
| Message size | Small (SQS: 256KB) | Up to 1MB default (configurable) |
| Use case | Task queues, work distribution | Event sourcing, audit logs, stream processing |
| Operational complexity | Lower | Higher |

---

## 2. Kafka Architecture Deep-Dive

### Core Concepts

```
Cluster: N brokers (Kafka servers)

Topic: logical channel divided into Partitions
  - topic "orders" might have 12 partitions spread across 3 brokers

Partition: an ordered, append-only log
  - each message gets a monotonically increasing offset (0, 1, 2, ...)
  - a partition lives on exactly one broker (+ replicas on others)

Producer: writes to a topic
  - chooses partition via key hash or round-robin

Consumer Group: a set of consumers that jointly consume a topic
  - each partition assigned to exactly ONE consumer in the group
  - scale consumers = scale partitions

Leader: the broker that handles reads/writes for a partition
Replica: copies on other brokers for fault tolerance
```

**Partition assignment:**
```
Topic "orders" — 6 partitions, consumer group "payment-service" with 3 consumers:

Consumer 1 ← Partitions 0, 3
Consumer 2 ← Partitions 1, 4
Consumer 3 ← Partitions 2, 5

Add a 4th consumer: rebalance triggers
Consumer 1 ← Partitions 0, 4
Consumer 2 ← Partitions 1, 5
Consumer 3 ← Partitions 2
Consumer 4 ← Partition 3
```

### Producer Acknowledgement Modes (`acks`)

| `acks` | Meaning | Durability | Throughput |
|--------|---------|-----------|-----------|
| `0` | Fire and forget — no ack | Lowest (data can be lost) | Highest |
| `1` | Leader acknowledges write | Medium (lost if leader fails before replication) | Medium |
| `all` (-1) | All in-sync replicas acknowledge | Highest | Lowest |

**Production recommendation:** `acks=all` + `min.insync.replicas=2` for durability. `acks=1` for high-throughput analytics where occasional loss is acceptable.

### Consumer Offset Commit

```
Consumer reads messages → processes them → commits offset

Auto-commit: consumer commits offset every auto.commit.interval.ms
  Risk: offset committed before processing completes → crash = message lost

Manual commit (recommended for production):
  consumer.poll(timeout)
  for message in messages:
      process(message)
  consumer.commit()   ← only commit AFTER processing
  Risk: crash after processing but before commit → message re-delivered (at-least-once)
```

### Log Compaction

Kafka can compact topics: keep only the latest value per key.

```
Normal retention: delete all messages older than 7 days
Log compaction: keep latest message per key, discard older values for same key

Use case: user profile topic — compact to keep only latest profile per user_id
After compaction: topic is a snapshot of current state, not full history
```

---

## 3. RabbitMQ Architecture

### Core Concepts

```
Exchange: receives messages from producers, routes to queues
  - Direct: route by exact routing key
  - Topic: route by routing key pattern (orders.* matches orders.created, orders.cancelled)
  - Fanout: broadcast to all bound queues (no routing key)
  - Headers: route by message headers

Queue: holds messages until consumed
Binding: link between exchange and queue with routing key

Producer → Exchange → [binding] → Queue A → Consumer A
                    → [binding] → Queue B → Consumer B
```

### Exchange Types

```
Direct Exchange (exact match):
  orders.created → Queue "payment-processor"
  orders.cancelled → Queue "refund-processor"

Topic Exchange (wildcard):
  orders.# → Queue "order-logger"     (any orders.*)
  orders.created → Queue "inventory"  (only created)
  *.failed → Queue "alerts"           (any *.failed)

Fanout Exchange (broadcast):
  order.created → ALL bound queues simultaneously
  Use case: send the same event to email, SMS, push notification queues
```

### Message Acknowledgement in RabbitMQ

```
consumer = channel.basic_consume(queue, on_message_callback)

def on_message_callback(ch, method, properties, body):
    try:
        process(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)    # Success
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag,   # Failure
                      requeue=True)                        # Re-enqueue
```

**`basic_nack` with `requeue=False`:** sends message to DLQ (Dead Letter Queue).

---

## 4. Delivery Guarantees

### At-Most-Once

Message may be lost but never delivered twice.

```
Producer: fire-and-forget (acks=0)
Consumer: commit offset BEFORE processing

Timeline:
  Consumer reads msg, commits offset, → crash before processing
  → Message is LOST (offset advanced past it)

Use case: metrics, telemetry, high-frequency logs where occasional loss is acceptable
```

### At-Least-Once

Message is never lost but may be delivered multiple times.

```
Producer: acks=all, retries enabled
Consumer: commit offset AFTER processing

Timeline:
  Consumer reads msg, processes, → crash before committing offset
  → Consumer restarts, re-reads same message → DUPLICATE
  → Consumer must be idempotent

Use case: order processing, payment events — losing a message is unacceptable
Requirement: EVERY consumer must handle duplicates (idempotent processing)
```

### Exactly-Once

Message delivered and processed exactly once. Hard to achieve end-to-end.

#### Kafka Exactly-Once (within Kafka pipeline)

```python
# Producer side: idempotent producer (deduplicates retries at broker)
producer = KafkaProducer(
    enable_idempotence=True,    # Enable sequence numbers per producer session
    acks="all",
    retries=10
)

# Transactions: atomic write across multiple topics
producer.init_transactions()
producer.begin_transaction()
producer.send("output-topic", key, processed_value)
consumer.commit({TopicPartition("input-topic", 0): OffsetAndMetadata(new_offset)})
producer.commit_transaction()  # Atomic: output written AND offset committed
```

**Kafka exactly-once is only end-to-end exactly-once if** the consumer's processing is itself idempotent (writing to an external DB is not covered by Kafka transactions).

#### SQS Exactly-Once

AWS SQS FIFO queues provide exactly-once delivery using a `MessageDeduplicationId`:

```python
sqs.send_message(
    QueueUrl="...",
    MessageBody=json.dumps(payload),
    MessageGroupId="order-group",
    MessageDeduplicationId=f"order-{order_id}"  # SQS deduplicates within 5-minute window
)
```

### Delivery Guarantee Comparison

| Guarantee | Message Loss? | Duplicates? | Consumer Requirement |
|-----------|:------------:|:-----------:|---------------------|
| At-most-once | Possible | Never | None |
| At-least-once | Never | Possible | Idempotent processing |
| Exactly-once | Never | Never | Kafka transactions OR SQS FIFO deduplication |

**Interview key point:** True end-to-end exactly-once requires both the message broker AND the consumer's external side effect (DB write) to be atomic. This is rarely achievable without either Kafka Streams (all within Kafka) or the Outbox pattern + idempotent consumer at the DB level.

---

## 5. Dead Letter Queues (DLQ)

A DLQ (or Dead Letter Topic in Kafka) captures messages that cannot be processed successfully after N retries.

```
Normal flow:
  Message → Consumer → success → offset committed

Failure flow:
  Message → Consumer → exception → nack/retry
  ...after max_retries...
  Message → DLQ (Dead Letter Queue)
                ↓
        Ops team inspects
        Fix consumer bug
        Replay from DLQ
        (Or: alert fires, human investigation)
```

### DLQ Metadata

Always include in DLQ messages:
```json
{
  "original_payload": {...},
  "original_topic": "orders",
  "original_partition": 3,
  "original_offset": 10042,
  "failure_reason": "NullPointerException at line 42",
  "failure_count": 5,
  "first_failure_at": "2024-04-15T10:30:00Z",
  "last_failure_at": "2024-04-15T11:00:00Z"
}
```

### DLQ in Different Systems

| System | DLQ Implementation |
|--------|-------------------|
| RabbitMQ | `x-dead-letter-exchange` argument on queue; failed messages routed to bound exchange |
| SQS | Set `RedrivePolicy` with `deadLetterTargetArn` and `maxReceiveCount` on source queue |
| Kafka | Manual: consumer catches exception, publishes to `{topic}.DLT` with error headers |
| Spring Kafka | `@RetryableTopic` annotation with `dltTopicSuffix` — handles retries + DLT automatically |

---

## 6. Backpressure

Backpressure is the mechanism by which a slow consumer signals to producers to slow down, preventing queue overflow and OOM.

```
Without backpressure:
  Producer: 100K msg/sec
  Consumer: 10K msg/sec
  Queue depth grows: 90K msg/sec × time → unbounded growth → OOM / disk full

With backpressure:
  Queue reaches max_length → producer blocks or receives flow control signal
  → producer slows to consumer's processing rate
```

### Backpressure Strategies

| Mechanism | How | When |
|-----------|-----|------|
| **Queue max length** | Broker rejects new messages when queue full | RabbitMQ; oldest messages dropped or producer blocked |
| **Consumer pull model** | Consumer requests N messages at a time (`max.poll.records`) | Kafka; consumer controls ingestion rate |
| **Rate limiting at producer** | Producer throttles send rate based on queue depth | Application-level |
| **Circuit breaker** | Producer stops sending when downstream latency spikes | Resilience4J, etc. |
| **Flow control (Reactive Streams)** | `request(N)` signals upstream how many events consumer can handle | RxJava, Project Reactor |

### Kafka Consumer Backpressure

```python
consumer = KafkaConsumer(
    "orders",
    max_poll_records=100,            # Pull at most 100 messages per poll
    max_poll_interval_ms=300_000,    # Must process within 5 min or session dies
)

while True:
    records = consumer.poll(timeout_ms=1000)  # Pull-based — consumer controls rate
    for record in records.values():
        slow_process(record)          # If slow, next poll waits → natural backpressure
    consumer.commit()
```

If `slow_process` takes too long and exceeds `max_poll_interval_ms`, the consumer is kicked out of the consumer group (session timeout) — this is a common production bug.

---

## 7. System Comparison

| | **Kafka** | **RabbitMQ** | **AWS SQS** |
|--|-----------|-------------|------------|
| **Type** | Distributed log / event stream | Message broker (AMQP) | Managed queue |
| **Message retention** | Days / forever | Until consumed | 1–14 days |
| **Replay** | ✅ Any offset | ❌ | ❌ (FIFO: limited) |
| **Throughput** | 10M+ msg/sec (cluster) | 50K–100K msg/sec | Unlimited (managed) |
| **Ordering** | Strict per partition | FIFO per queue | FIFO per MessageGroupId (FIFO) |
| **Consumer model** | Consumer group + offset | Competing consumers | Competing consumers |
| **Routing** | By partition key | Exchanges + bindings | None (or FIFO groups) |
| **Exactly-once** | ✅ Kafka Transactions | ❌ | ✅ FIFO + dedup ID |
| **DLQ** | Manual / DLT | Built-in (`x-dead-letter-exchange`) | Built-in (`RedrivePolicy`) |
| **Operational complexity** | High (ZooKeeper/KRaft, brokers) | Medium | Low (fully managed) |
| **Best for** | Event streaming, audit log, stream processing | Complex routing, task queues | Simple managed queues, AWS-native |

---

## Producer-Consumer Patterns

### Competing Consumers (Work Queue)

```
Multiple consumer instances share a queue — each message processed by exactly one.
Enables horizontal scaling of processing.

Orders Queue ← 3 OrderProcessor instances
  Instance 1 processes msg 1, 4, 7 ...
  Instance 2 processes msg 2, 5, 8 ...
  Instance 3 processes msg 3, 6, 9 ...
```

### Pub/Sub (Fan-out)

```
Publisher sends one event; multiple subscribers each receive a copy.

"order.created" event → PaymentService receives it
                      → InventoryService receives it
                      → NotificationService receives it
                      (each independently, in their own consumer group)
```

### Priority Queue

```
High-priority messages processed before low-priority.
RabbitMQ: x-max-priority on queue.
SQS: separate queues per priority; consumers drain high before low.
```

### Request-Reply (via Messaging)

```
Requester → [Request Queue] → Replier
            ← [Reply Queue] ←

Requester sets reply-to queue + correlation-id.
Replier sends response to reply-to queue with matching correlation-id.
Use case: async RPC pattern.
```
