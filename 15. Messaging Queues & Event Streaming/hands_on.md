# Messaging Queues & Event Streaming — Hands-On Exercises

## Exercise 1: Kafka Producer + Consumer with Manual Offset Commit

```python
"""
Kafka producer and consumer demonstrating:
- Manual offset commit (at-least-once delivery)
- Idempotent consumer (deduplication via PostgreSQL processed_events table)
- Backpressure via max_poll_records

pip install kafka-python asyncpg
Requires: Kafka running on localhost:9092
"""
import json
import uuid
import time
import asyncio
import asyncpg
from kafka import KafkaProducer, KafkaConsumer, TopicPartition
from kafka.errors import KafkaError

KAFKA_BROKER = "localhost:9092"
TOPIC        = "orders"
GROUP_ID     = "order-processor"
DB_DSN       = "postgresql://user:password@localhost:5432/mydb"


# ── Producer ──────────────────────────────────────────────────────────────────
def make_producer() -> KafkaProducer:
    return KafkaProducer(
        bootstrap_servers=KAFKA_BROKER,
        value_serializer=lambda v: json.dumps(v).encode("utf-8"),
        key_serializer=lambda k: k.encode("utf-8") if k else None,
        # Durability: wait for all in-sync replicas
        acks="all",
        # Idempotent producer: deduplicates retries at the broker
        enable_idempotence=True,
        # Retry on transient failures
        retries=5,
        retry_backoff_ms=200,
    )


def produce_orders(n: int = 10):
    producer = make_producer()
    print(f"[Producer] Sending {n} orders to topic '{TOPIC}'")

    for i in range(n):
        order = {
            "order_id":   str(uuid.uuid4()),
            "user_id":    f"user_{i % 3}",
            "amount":     round(10.0 + i * 5.5, 2),
            "product_id": f"prod_{i % 5}",
        }
        # Partition key = user_id ensures same user's orders go to same partition (ordering)
        future = producer.send(TOPIC, key=order["user_id"], value=order)
        try:
            record_metadata = future.get(timeout=5)
            print(f"  Sent order {order['order_id'][:8]}... → "
                  f"partition={record_metadata.partition} offset={record_metadata.offset}")
        except KafkaError as e:
            print(f"  [ERROR] Failed to send: {e}")

    producer.flush()
    producer.close()
    print("[Producer] Done")


# ── Consumer with idempotent processing ──────────────────────────────────────
class IdempotentOrderConsumer:
    """
    Consumes orders from Kafka.
    Uses a PostgreSQL processed_events table to prevent duplicate processing.
    Commits offsets AFTER processing (at-least-once → idempotent = effectively exactly-once).
    """

    def __init__(self):
        self.consumer = KafkaConsumer(
            TOPIC,
            bootstrap_servers=KAFKA_BROKER,
            group_id=GROUP_ID,
            value_deserializer=lambda m: json.loads(m.decode("utf-8")),
            key_deserializer=lambda k: k.decode("utf-8") if k else None,
            # Manual offset commit
            enable_auto_commit=False,
            # Backpressure: pull at most 50 messages per poll
            max_poll_records=50,
            # If we don't poll within 5 min, we're kicked from the group
            max_poll_interval_ms=300_000,
            # Start from earliest if no committed offset (useful for replay)
            auto_offset_reset="earliest",
        )
        self.db: asyncpg.Connection = None

    async def connect_db(self):
        self.db = await asyncpg.connect(DB_DSN)
        await self.db.execute("""
            CREATE TABLE IF NOT EXISTS processed_events (
                event_id TEXT PRIMARY KEY,
                processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
            )
        """)

    async def process_order(self, order: dict) -> bool:
        """Returns True if processed, False if duplicate."""
        event_id = order["order_id"]

        # Idempotency check: try to insert; skip if already exists
        result = await self.db.execute(
            "INSERT INTO processed_events (event_id) VALUES ($1) ON CONFLICT DO NOTHING",
            event_id
        )

        if result == "INSERT 0 0":
            print(f"  [Consumer] DUPLICATE skipped: {event_id[:8]}")
            return False

        # Actual business logic (replace with real processing)
        print(f"  [Consumer] Processing order {event_id[:8]} "
              f"user={order['user_id']} amount=${order['amount']}")
        await asyncio.sleep(0.01)   # Simulate work
        return True

    async def run(self, max_messages: int = 20):
        await self.connect_db()
        print(f"[Consumer] Listening on topic '{TOPIC}' (group={GROUP_ID})")

        processed = 0
        try:
            while processed < max_messages:
                # Pull-based: consumer controls how much it takes
                records = self.consumer.poll(timeout_ms=2000)

                if not records:
                    print("  [Consumer] No messages — waiting...")
                    continue

                for tp, messages in records.items():
                    for msg in messages:
                        ok = await self.process_order(msg.value)
                        processed += 1 if ok else 0

                # Commit AFTER processing the entire batch
                # (crash before this = re-process batch = idempotency required)
                self.consumer.commit()
                print(f"  [Consumer] Committed offsets (processed={processed})")

        finally:
            self.consumer.close()
            await self.db.close()


async def run_consumer():
    consumer = IdempotentOrderConsumer()
    await consumer.run(max_messages=20)


if __name__ == "__main__":
    import sys
    if "produce" in sys.argv:
        produce_orders(15)
    else:
        asyncio.run(run_consumer())
```

---

## Exercise 2: Dead Letter Queue — Retry + DLQ Pattern

```python
"""
Demonstrates retry-then-DLQ pattern.
On processing failure: retry up to N times with backoff.
After max retries: forward to DLQ topic with failure metadata.

pip install kafka-python
"""
import json
import time
import traceback
from dataclasses import dataclass, field, asdict
from datetime import datetime, UTC
from kafka import KafkaConsumer, KafkaProducer

KAFKA_BROKER  = "localhost:9092"
SOURCE_TOPIC  = "payments"
DLQ_TOPIC     = "payments.DLT"   # Dead Letter Topic convention: {topic}.DLT
GROUP_ID      = "payment-processor"
MAX_RETRIES   = 3
RETRY_BACKOFF = [1.0, 2.0, 4.0]   # Seconds: exponential backoff per attempt


@dataclass
class DLQMessage:
    original_topic:     str
    original_partition: int
    original_offset:    int
    original_key:       str
    original_payload:   dict
    failure_reason:     str
    failure_traceback:  str
    retry_count:        int
    first_failure_at:   str
    last_failure_at:    str


def process_payment(payload: dict) -> None:
    """Simulates payment processing. Raises on bad data."""
    if payload.get("amount", 0) <= 0:
        raise ValueError(f"Invalid amount: {payload.get('amount')}")
    if payload.get("card_token") == "tok_fail":
        raise RuntimeError("Card processing service unavailable")
    print(f"  [Processor] Charged ${payload['amount']} for order {payload.get('order_id', '?')[:8]}")


def send_to_dlq(producer: KafkaProducer, msg, dlq_msg: DLQMessage):
    producer.send(
        DLQ_TOPIC,
        key=msg.key,
        value=json.dumps(asdict(dlq_msg)).encode(),
        headers=[
            ("x-original-topic",     msg.topic.encode()),
            ("x-failure-reason",     dlq_msg.failure_reason.encode()),
            ("x-retry-count",        str(dlq_msg.retry_count).encode()),
        ]
    )
    producer.flush()
    print(f"  [DLQ] Forwarded failed message to {DLQ_TOPIC} "
          f"(reason: {dlq_msg.failure_reason})")


def run_consumer_with_dlq():
    consumer = KafkaConsumer(
        SOURCE_TOPIC,
        bootstrap_servers=KAFKA_BROKER,
        group_id=GROUP_ID,
        value_deserializer=lambda m: json.loads(m.decode("utf-8")),
        key_deserializer=lambda k: k.decode("utf-8") if k else None,
        enable_auto_commit=False,
        max_poll_records=10,
    )
    producer = KafkaProducer(
        bootstrap_servers=KAFKA_BROKER,
        value_serializer=lambda v: v if isinstance(v, bytes) else json.dumps(v).encode(),
        acks="all",
    )

    print(f"[Consumer] Listening on '{SOURCE_TOPIC}' — DLQ: '{DLQ_TOPIC}'")

    for msg in consumer:
        payload = msg.value
        first_failure_at = None
        last_error = None
        last_tb = ""

        for attempt in range(MAX_RETRIES + 1):
            try:
                process_payment(payload)
                # Success — commit this message's offset
                consumer.commit({
                    msg.topic + "_" + str(msg.partition):
                        {type(msg): msg.offset + 1}
                })
                break

            except Exception as e:
                last_error = str(e)
                last_tb    = traceback.format_exc()
                now        = datetime.now(UTC).isoformat()

                if first_failure_at is None:
                    first_failure_at = now

                if attempt < MAX_RETRIES:
                    backoff = RETRY_BACKOFF[attempt]
                    print(f"  [Retry {attempt+1}/{MAX_RETRIES}] "
                          f"Error: {e} — retrying in {backoff}s")
                    time.sleep(backoff)
                else:
                    # Exhausted retries → DLQ
                    dlq_msg = DLQMessage(
                        original_topic=msg.topic,
                        original_partition=msg.partition,
                        original_offset=msg.offset,
                        original_key=msg.key or "",
                        original_payload=payload,
                        failure_reason=last_error,
                        failure_traceback=last_tb,
                        retry_count=MAX_RETRIES,
                        first_failure_at=first_failure_at,
                        last_failure_at=now,
                    )
                    send_to_dlq(producer, msg, dlq_msg)
                    # Commit original offset so we don't re-process poison pill
                    consumer.commit()

    consumer.close()
    producer.close()
```

---

## Exercise 3: RabbitMQ — Fanout Exchange with Per-Service Queues

```python
"""
Fanout exchange: one "order.created" event delivered to three independent queues.
Each service binds its own queue to the fanout exchange.

pip install pika
Requires: RabbitMQ running on localhost:5672
"""
import json
import pika
import threading
import time

RABBITMQ_URL = "amqp://guest:guest@localhost:5672/"
EXCHANGE     = "order_events"   # Fanout exchange
ROUTING_KEY  = ""                # Fanout ignores routing keys


def get_connection():
    return pika.BlockingConnection(pika.URLParameters(RABBITMQ_URL))


# ── Setup: declare exchange and per-service queues ────────────────────────────
def setup_topology():
    conn    = get_connection()
    channel = conn.channel()

    # Declare fanout exchange (durable = survives broker restart)
    channel.exchange_declare(
        exchange=EXCHANGE,
        exchange_type="fanout",
        durable=True
    )

    # Each service declares its own queue and binds to the exchange
    services = ["payment-service", "inventory-service", "notification-service"]
    for service in services:
        channel.queue_declare(
            queue=service,
            durable=True,              # Queue survives restart
            arguments={
                "x-message-ttl":        86_400_000,   # 24h TTL
                "x-dead-letter-exchange": "dlx",      # DLQ exchange
            }
        )
        channel.queue_bind(
            exchange=EXCHANGE,
            queue=service,
            routing_key=""   # Fanout ignores this
        )
        print(f"[Setup] Queue '{service}' bound to exchange '{EXCHANGE}'")

    conn.close()


# ── Producer ──────────────────────────────────────────────────────────────────
def publish_order_created(order: dict):
    conn    = get_connection()
    channel = conn.channel()

    channel.basic_publish(
        exchange=EXCHANGE,
        routing_key=ROUTING_KEY,
        body=json.dumps(order).encode(),
        properties=pika.BasicProperties(
            delivery_mode=2,   # Persistent message — survives broker restart
            content_type="application/json",
            message_id=order["order_id"],  # For deduplication
        )
    )
    print(f"[Producer] Published order {order['order_id'][:8]} to exchange '{EXCHANGE}'")
    conn.close()


# ── Consumer factory ──────────────────────────────────────────────────────────
def make_service_consumer(service_name: str, process_fn):
    def run():
        conn    = get_connection()
        channel = conn.channel()
        channel.basic_qos(prefetch_count=10)   # Backpressure: pull 10 at a time

        def callback(ch, method, properties, body):
            try:
                payload = json.loads(body)
                process_fn(service_name, payload)
                ch.basic_ack(delivery_tag=method.delivery_tag)
            except Exception as e:
                print(f"  [{service_name}] FAILED: {e} — sending to DLQ")
                ch.basic_nack(
                    delivery_tag=method.delivery_tag,
                    requeue=False   # Don't re-queue — send to DLQ
                )

        channel.basic_consume(queue=service_name, on_message_callback=callback)
        print(f"[{service_name}] Waiting for messages...")
        channel.start_consuming()

    return run


# ── Service handlers ──────────────────────────────────────────────────────────
def handle_payment(service, payload):
    print(f"  [{service}] Processing payment for order {payload['order_id'][:8]} "
          f"amount=${payload['amount']}")

def handle_inventory(service, payload):
    print(f"  [{service}] Reserving stock for product {payload['product_id']}")

def handle_notification(service, payload):
    print(f"  [{service}] Queuing notification email to user {payload['user_id']}")


# ── Demo ──────────────────────────────────────────────────────────────────────
def demo():
    setup_topology()

    # Start consumers in background threads (each owns its queue)
    consumers = [
        ("payment-service",       handle_payment),
        ("inventory-service",     handle_inventory),
        ("notification-service",  handle_notification),
    ]
    threads = []
    for name, handler in consumers:
        t = threading.Thread(
            target=make_service_consumer(name, handler),
            daemon=True
        )
        t.start()
        threads.append(t)

    time.sleep(0.5)

    # Publish a few orders
    import uuid
    for i in range(3):
        publish_order_created({
            "order_id":   str(uuid.uuid4()),
            "user_id":    f"user_{i}",
            "product_id": f"prod_{i}",
            "amount":     round(50.0 + i * 25, 2),
        })
        time.sleep(0.3)

    time.sleep(2)   # Let consumers process
    print("Demo complete")


if __name__ == "__main__":
    demo()
```

---

## Exercise 4: Backpressure — Consumer Lag Monitor

```python
"""
Monitors Kafka consumer group lag per partition.
Emits an alert when lag exceeds a threshold (consumer can't keep up with producer).
This is how you detect backpressure issues in production.

pip install kafka-python
"""
import time
from kafka import KafkaAdminClient, KafkaConsumer, TopicPartition
from kafka.admin import NewTopic

KAFKA_BROKER     = "localhost:9092"
TOPIC            = "orders"
GROUP_ID         = "order-processor"
LAG_ALERT_THRESH = 1000   # Alert if any partition is > 1000 messages behind


def get_consumer_lag(topic: str, group_id: str) -> dict[int, int]:
    """
    Returns {partition_id: lag} for the specified consumer group.
    Lag = end_offset - committed_offset
    """
    admin   = KafkaAdminClient(bootstrap_servers=KAFKA_BROKER)

    # Get current end offsets (latest message position per partition)
    consumer = KafkaConsumer(bootstrap_servers=KAFKA_BROKER)
    partitions = consumer.partitions_for_topic(topic)
    if not partitions:
        consumer.close()
        return {}

    tps = [TopicPartition(topic, p) for p in partitions]

    # End offsets: where the log currently ends
    end_offsets = consumer.end_offsets(tps)

    # Committed offsets: where the consumer group last processed
    # (Use kafka-python's list_consumer_group_offsets via admin)
    try:
        committed = admin.list_consumer_group_offsets(group_id)
    except Exception:
        committed = {}

    consumer.close()
    admin.close()

    lag = {}
    for tp in tps:
        end    = end_offsets.get(tp, 0)
        offset_meta = committed.get(tp)
        committed_offset = offset_meta.offset if offset_meta else 0
        lag[tp.partition] = max(0, end - committed_offset)

    return lag


def monitor_lag(topic: str, group_id: str, interval_sec: int = 10):
    """Continuously monitors consumer lag and alerts when threshold exceeded."""
    print(f"[LagMonitor] Monitoring {topic}/{group_id} — alert at lag > {LAG_ALERT_THRESH}")

    while True:
        lag = get_consumer_lag(topic, group_id)

        if not lag:
            print("  [LagMonitor] No partitions found (topic may not exist yet)")
        else:
            total_lag = sum(lag.values())
            max_lag   = max(lag.values()) if lag else 0

            status = []
            for partition, l in sorted(lag.items()):
                alert = " ⚠️ ALERT" if l > LAG_ALERT_THRESH else ""
                status.append(f"p{partition}:{l}{alert}")

            print(f"  [LagMonitor] {', '.join(status)} | total={total_lag} max={max_lag}")

            if max_lag > LAG_ALERT_THRESH:
                print(f"  🚨 BACKPRESSURE DETECTED: partition lag={max_lag} exceeds threshold.")
                print(f"      Remediation: scale up consumer instances or reduce processing time")

        time.sleep(interval_sec)


if __name__ == "__main__":
    monitor_lag(TOPIC, GROUP_ID, interval_sec=5)
```

---

## Exercise 5: Exactly-Once — Kafka Transactions

```python
"""
Kafka transactions: atomically consume from input topic,
process, and produce to output topic + commit offset.
This is the read-process-write pattern for exactly-once semantics within Kafka.

pip install confluent-kafka
"""
from confluent_kafka import Producer, Consumer, KafkaError, TopicPartition
import json
import uuid

KAFKA_BROKER    = "localhost:9092"
INPUT_TOPIC     = "raw-orders"
OUTPUT_TOPIC    = "enriched-orders"
GROUP_ID        = "order-enricher"
TRANSACTIONAL_ID = "enricher-instance-1"   # Unique per producer instance


def enrich_order(raw: dict) -> dict:
    """Add computed fields to the order."""
    return {
        **raw,
        "enriched": True,
        "tax":      round(raw.get("amount", 0) * 0.08, 2),
        "total":    round(raw.get("amount", 0) * 1.08, 2),
    }


def run_transactional_processor():
    # ── Transactional producer ─────────────────────────────────────────────────
    producer = Producer({
        "bootstrap.servers":  KAFKA_BROKER,
        "transactional.id":   TRANSACTIONAL_ID,   # Enables transactions
        "acks":               "all",
        "enable.idempotence": True,
    })
    producer.init_transactions()

    # ── Consumer (reads input) ─────────────────────────────────────────────────
    consumer = Consumer({
        "bootstrap.servers":        KAFKA_BROKER,
        "group.id":                 GROUP_ID,
        "auto.offset.reset":        "earliest",
        "enable.auto.commit":       False,         # Manual commit inside transaction
        "isolation.level":          "read_committed",  # Only read committed (transactional) msgs
    })
    consumer.subscribe([INPUT_TOPIC])

    print(f"[TxProcessor] Processing {INPUT_TOPIC} → {OUTPUT_TOPIC} (exactly-once)")

    try:
        while True:
            msg = consumer.poll(timeout=1.0)
            if msg is None:
                continue
            if msg.error():
                if msg.error().code() == KafkaError._PARTITION_EOF:
                    continue
                raise Exception(f"Consumer error: {msg.error()}")

            raw = json.loads(msg.value())

            # ── Begin transaction ──────────────────────────────────────────────
            producer.begin_transaction()
            try:
                enriched = enrich_order(raw)

                # Write to output topic (inside transaction)
                producer.produce(
                    OUTPUT_TOPIC,
                    key=msg.key(),
                    value=json.dumps(enriched).encode()
                )

                # Commit input consumer offset AS PART OF THE TRANSACTION
                # This ties the output write + offset commit atomically
                producer.send_offsets_to_transaction(
                    {TopicPartition(msg.topic(), msg.partition()): msg.offset() + 1},
                    consumer.consumer_group_metadata()
                )

                # Commit: output written AND offset advanced — atomically
                producer.commit_transaction()
                print(f"  [TxProcessor] ✅ Enriched order {raw.get('order_id','?')[:8]} "
                      f"total=${enriched['total']}")

            except Exception as e:
                producer.abort_transaction()
                print(f"  [TxProcessor] ❌ Aborted transaction: {e}")

    except KeyboardInterrupt:
        pass
    finally:
        consumer.close()


if __name__ == "__main__":
    run_transactional_processor()
```

---

## Exercise 6: SQS Dead Letter Queue with Boto3

```python
"""
AWS SQS: main queue + DLQ setup, send messages, consume with DLQ on failure.
Demonstrates maxReceiveCount-based automatic DLQ routing.

pip install boto3
"""
import json
import uuid
import boto3
from botocore.exceptions import ClientError

# LocalStack for local testing: endpoint_url="http://localhost:4566"
sqs = boto3.client("sqs", region_name="us-east-1")


def setup_sqs_with_dlq(queue_name: str, max_receive_count: int = 3):
    """
    Creates a main SQS queue with a DLQ.
    After maxReceiveCount failed receives, message is automatically moved to DLQ.
    """
    # Create DLQ first
    dlq_name = f"{queue_name}-dlq"
    dlq_result = sqs.create_queue(
        QueueName=dlq_name,
        Attributes={
            "MessageRetentionPeriod": str(14 * 86400),   # 14 days
        }
    )
    dlq_url = dlq_result["QueueUrl"]
    dlq_attrs = sqs.get_queue_attributes(
        QueueUrl=dlq_url, AttributeNames=["QueueArn"]
    )
    dlq_arn = dlq_attrs["Attributes"]["QueueArn"]

    # Create main queue with RedrivePolicy pointing to DLQ
    main_result = sqs.create_queue(
        QueueName=queue_name,
        Attributes={
            "VisibilityTimeout":       "30",
            "MessageRetentionPeriod":  str(4 * 86400),   # 4 days
            "RedrivePolicy":           json.dumps({
                "deadLetterTargetArn": dlq_arn,
                "maxReceiveCount":     str(max_receive_count),
            }),
        }
    )
    main_url = main_result["QueueUrl"]
    print(f"[SQS] Created '{queue_name}' → DLQ: '{dlq_name}' (maxReceiveCount={max_receive_count})")
    return main_url, dlq_url


def send_order(queue_url: str, order: dict):
    sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps(order),
        # FIFO queue would add: MessageGroupId + MessageDeduplicationId
        MessageAttributes={
            "order_id": {
                "StringValue": order["order_id"],
                "DataType":    "String"
            }
        }
    )
    print(f"[SQS] Sent order {order['order_id'][:8]}")


def process_message(body: dict) -> bool:
    """Returns True on success, False on failure (will go to DLQ after maxReceiveCount)."""
    if body.get("amount", 0) <= 0:
        print(f"  [Processor] FAILED: invalid amount — will retry, then DLQ")
        return False
    print(f"  [Processor] Processed order {body['order_id'][:8]} ${body['amount']}")
    return True


def consume_queue(queue_url: str, max_messages: int = 5):
    print(f"[Consumer] Polling {queue_url}")
    received = 0

    while received < max_messages:
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=2,        # Long polling — wait up to 2s for messages
            VisibilityTimeout=30,     # Hide message for 30s during processing
        )

        messages = response.get("Messages", [])
        if not messages:
            print("  [Consumer] Queue empty")
            break

        for msg in messages:
            body    = json.loads(msg["Body"])
            receipt = msg["ReceiptHandle"]

            success = process_message(body)
            if success:
                # Delete only on success — failure leaves message visible after VisibilityTimeout
                sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=receipt)
                received += 1
            else:
                # Don't delete — SQS will re-deliver after VisibilityTimeout
                # After maxReceiveCount re-deliveries, SQS moves to DLQ automatically
                pass


def demo():
    main_url, dlq_url = setup_sqs_with_dlq("orders", max_receive_count=3)

    valid_order = {"order_id": str(uuid.uuid4()), "amount": 99.0}
    bad_order   = {"order_id": str(uuid.uuid4()), "amount": -1}

    send_order(main_url, valid_order)
    send_order(main_url, bad_order)

    consume_queue(main_url, max_messages=5)

    # After 3 failed receives, bad_order will be in DLQ
    print("\n[DLQ] Checking dead letter queue:")
    consume_queue(dlq_url, max_messages=5)


if __name__ == "__main__":
    demo()
```
