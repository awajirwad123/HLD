# Messaging Queues & Event Streaming — Quick-Fire Q&A

**Format:** Answer each question in ≤2 sentences before checking the answer.

---

**Q1. What is the fundamental difference between a message queue and an event stream?**
In a message queue, a message is consumed and removed once a consumer processes it. In an event stream (like Kafka), messages are retained in a durable log and multiple independent consumer groups can each read all messages at their own pace.

---

**Q2. What is a Kafka partition and why does it matter for scalability?**
A partition is an ordered, append-only segment of a Kafka topic that lives on a specific broker. The number of partitions determines the maximum parallelism for a consumer group — at most one consumer per group reads from each partition simultaneously.

---

**Q3. What does `acks=all` mean in a Kafka producer configuration?**
It means the producer waits for all in-sync replicas (ISRs) to acknowledge the write before considering it successful. Combined with `min.insync.replicas=2`, this provides the strongest durability guarantee: a message is not lost even if the leader broker fails immediately after the ack.

---

**Q4. What is consumer group lag and why is it important?**
Consumer group lag is the difference between the latest message offset in a partition and the consumer group's committed offset — how far behind the consumer is. Rising lag indicates the consumer cannot keep up with the producer, signalling a backpressure problem.

---

**Q5. Describe at-least-once delivery in Kafka. What must consumers do to make it safe?**
At-least-once delivery means messages are never lost but may be delivered more than once — achieved by committing offsets only after processing. Consumers must be idempotent: processing the same message twice must produce the same result as processing it once.

---

**Q6. What is at-most-once delivery and when is it acceptable?**
At-most-once means a message may be lost but is never delivered twice — achieved by committing the offset before processing. It is acceptable for high-frequency, low-importance data like metrics, telemetry, or logging where occasional data loss is tolerable but duplicate processing would be costly.

---

**Q7. What does Kafka's `enable.idempotence=true` producer setting do?**
It assigns a unique sequence number to each message per producer session. The broker deduplicates retries from the same producer (same producer ID + sequence number), preventing duplicate writes caused by producer retries after transient network failures.

---

**Q8. What is a Dead Letter Queue (DLQ)?**
A DLQ captures messages that could not be processed successfully after a configured number of retries. It prevents a "poison pill" message from blocking the queue indefinitely and provides a place for operators to inspect, fix, and replay failed messages.

---

**Q9. In RabbitMQ, what happens when `basic_nack` is called with `requeue=False`?**
The message is not re-enqueued to its original queue. Instead, if the queue has a dead-letter exchange configured (`x-dead-letter-exchange`), the message is routed there — effectively moving it to the DLQ.

---

**Q10. What is the maximum number of useful consumer instances in a Kafka consumer group for a topic with 6 partitions?**
Six. Each consumer in a group is assigned one or more partitions exclusively, and a partition can only be read by one consumer at a time. Adding a 7th consumer leaves it idle with no partition assignment.

---

**Q11. What is log compaction in Kafka and when do you use it?**
Log compaction retains only the latest message per key in a partition, discarding older values for the same key while preserving all keys. It is used for change-data-capture topics (e.g., user profiles) where you want the latest state per entity, not the full history.

---

**Q12. How does SQS FIFO achieve exactly-once delivery?**
SQS FIFO uses a `MessageDeduplicationId` — a producer-provided identifier stored for a 5-minute deduplication window. If a message with the same `MessageDeduplicationId` is sent within that window, SQS silently discards the duplicate without delivering it.

---

**Q13. What is a RabbitMQ fanout exchange and when do you use it?**
A fanout exchange broadcasts every received message to all queues bound to it, ignoring routing keys. It is used for pub/sub scenarios where a single event (e.g., `order.created`) needs to trigger multiple independent services simultaneously.

---

**Q14. What is the difference between a Kafka consumer group and a standalone consumer?**
A consumer group distributes topic partitions across its members so each message is processed by exactly one member. A standalone consumer (no group) reads all partitions independently — useful for replication, auditing, or full-log processing where every consumer needs every message.

---

**Q15. Why is `max_poll_interval_ms` critical for Kafka consumers?**
If a consumer takes longer than `max_poll_interval_ms` between calls to `poll()`, the broker considers it dead, removes it from the consumer group, and triggers a rebalance — reassigning its partitions to other consumers. Slow processing that exceeds this timeout causes repeated rebalances and double-processing.

---

**Q16. What is backpressure in the context of messaging, and how does Kafka implement it?**
Backpressure occurs when a consumer processes messages slower than the producer sends them, causing queue depth or lag to grow unboundedly. Kafka implements consumer-side backpressure via the pull model: the consumer calls `poll(max_poll_records=N)` to fetch only as many messages as it can handle per batch — it never has more pushed to it than it requests.

---

**Q17. What metadata should always be included in a DLQ message?**
Original topic, partition, and offset (for replay); failure reason and stack trace; retry count; first and last failure timestamps; and the original payload unchanged. This is the minimum needed to diagnose the failure and replay the message after a fix.

---

**Q18. How do Kafka transactions achieve exactly-once in a stream-processing pipeline?**
Kafka transactions atomically group a output topic write and an input consumer offset commit into a single transaction — either both commit or both abort. A downstream consumer using `isolation.level=read_committed` only reads messages from committed transactions, preventing processing of partial results.

---

**Q19. What SQS configuration routes messages to a DLQ after 3 failed delivery attempts?**
Set a `RedrivePolicy` on the source queue with `maxReceiveCount=3` and `deadLetterTargetArn` pointing to the DLQ's ARN. After a message is received (but not deleted) three times, SQS automatically moves it to the DLQ.

---

**Q20. When would you choose SQS over Kafka for a new system?**
SQS is preferable when: you are already on AWS and want a fully managed, zero-ops queue; your workload is simple task distribution without need for replay or multiple independent consumers; and throughput requirements are moderate. Kafka is needed when replay is required, multiple independent consumer groups must all receive every event, or throughput exceeds what a single SQS queue manages cost-effectively.
