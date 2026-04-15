# Common Interview Mistakes — All 36 Topics

The most frequent errors candidates make, curated from tricky questions across all topics. Use this as a final check before your interview.

---

## Category A: Skipping Requirements

**Mistake:** Jumping straight to "I'll use Kafka + Cassandra + 5 microservices" without asking a single clarifying question.

**Why it fails:** Interviewers are testing whether you can scope a problem, not just implement one. A requirement question like "is consistency important here?" changes the entire design.

**Fix:** Spend the first 5 minutes on functional and non-functional requirements. Write them on the board. Reference them when justifying choices.

---

## Category B: Using Averages Instead of Percentiles

**Mistake:** "Our average latency is 50ms so we're fine."

**Why it fails:** Average hides tail latency. If P99 is 5 seconds while P50 is 5ms, 1% of users experience 5-second page loads. That's thousands of users at scale.

**Fix:** Always state P95/P99 for latency. Design SLOs around percentiles. Alert on P99, not average.

---

## Topic-Specific Mistakes

### T2 — Networking Basics
- **Mistake:** Treating TCP and UDP as interchangeable. "I'll use UDP for video because it's faster."
- **Fix:** UDP is reliable for video where occasional packet loss is tolerable (RTP/RTCP). But real-time chat or financial data needs TCP's guaranteed delivery.

### T4 — Load Balancing
- **Mistake:** Using IP-hash load balancing with multiple LB nodes.
- **Fix:** IP-hash only guarantees stickiness when the same LB node sees the request. With multiple LBs, the same IP can hash to different servers. Use consistent hashing or session cookies for stickiness.

### T5 — Caching
- **Mistake:** Setting all cache TTLs to the same value (e.g., 1 hour).
- **Fix:** At scale, all keys expiring simultaneously causes a cache stampede: every key misses at the same time, hammering the DB. Use jittered TTL (`base_ttl + random(0, jitter)`) or probabilistic early expiration.

### T6 — SQL Databases
- **Mistake:** Adding indexes on every column "for performance."
- **Fix:** Indexes slow writes (must update index on insert/update). Only index columns you actively filter or join on. Over-indexing on write-heavy tables destroys throughput.

### T7 — NoSQL Databases
- **Mistake:** Treating Cassandra as a general-purpose DB and putting JOINs / complex queries on it.
- **Fix:** NoSQL thrives on known access patterns with a well-designed partition key. If you need complex queries, either pre-compute them or use a separate analytics store (ClickHouse, Redshift).

### T8 — Data Partitioning & Sharding
- **Mistake:** Choosing a low-cardinality shard key (e.g., `country` or `gender`).
- **Fix:** Low-cardinality → few shards → hotspots. The shard key must have high cardinality (`user_id`, `order_id`), not be monotonically increasing (sequential IDs cause all writes to the last shard), and be in the most common query filter.

### T9 — Replication
- **Mistake:** Reading immediately after a write from a replica and assuming consistency.
- **Fix:** Replicas have replication lag (1–100ms typical, higher under load). For read-your-writes consistency, either route the post-write read to the primary or use synchronous replication (with latency trade-off).

### T10 — CAP Theorem
- **Mistake:** Saying a system is "CA" (consistent and available with no partition tolerance).
- **Fix:** "CA" doesn't exist in distributed systems — network partitions are unavoidable. The real question is: during a partition, do you choose CP (stop accepting writes to preserve consistency) or AP (continue serving potentially stale data)?

### T12 — Distributed Transactions
- **Mistake:** Using 2PC in a high-throughput system because "it's safer."
- **Fix:** 2PC has two problems: (1) it's synchronous — the coordinator waits for all participants, blocking throughput; (2) if the coordinator crashes after PREPARE, participants are stuck indefinitely. Use Saga with compensating transactions for long-running distributed workflows.

### T13 — Consensus Algorithms
- **Mistake:** Deploying Raft/Consul/etcd with an even number of nodes.
- **Fix:** Even numbers can deadlock (2 nodes can't form a quorum if they disagree — 1/1 tie). Always use 3, 5, or 7 nodes. Quorum = ⌊N/2⌋ + 1; with 4 nodes quorum is 3 — you get no redundancy over 3 nodes for the same fault tolerance.

### T14 — API Design
- **Mistake:** Returning a breaking change in a versioned API (removing a field) without a deprecation period.
- **Fix:** Never remove fields from an API response without a deprecation cycle. Clients may depend on the field — removing it silently breaks them. Add to API, mark deprecated, give 6+ months, then remove.

### T15 — Messaging Queues
- **Mistake:** Assuming your Kafka consumer handles messages exactly-once by default.
- **Fix:** Kafka guarantees at-least-once delivery by default. For exactly-once, you need both idempotent producers (`enable.idempotence=true`) and transactional consumer processing with `isolation.level=read_committed`. This halves throughput — only use when truly needed.

### T16 — WebSockets
- **Mistake:** Routing WebSocket connections without sticky sessions to a multi-node server.
- **Fix:** WebSocket state is held in memory on the specific server node. Without sticky sessions or a shared pub/sub layer (Redis pub/sub), a message sent on node A won't reach a client connected to node B. Use a message broker to fan out across nodes.

### T18 — Rate Limiting
- **Mistake:** Implementing rate limiting entirely in the application layer without coordination.
- **Fix:** Per-instance rate limiting means a user hitting 10 servers can make 10× the allowed requests. Centralised counters (Redis atomic INCR) or token-passing are needed for distributed enforcement.

### T19 — Circuit Breaker
- **Mistake:** Retrying immediately and rapidly on failure (no backoff, no jitter).
- **Fix:** Rapid retry under failure creates a thundering herd — if a service is struggling, 1000 clients retrying every 100ms makes it worse. Use exponential backoff (`delay = base × 2^n`) plus jitter (`delay + random(0, delay)`).

### T20 — Consistent Hashing
- **Mistake:** Using basic consistent hashing (one point per node on the ring).
- **Fix:** With few nodes and one point each, key distribution is highly uneven (some nodes get far more keys). Virtual nodes (150+ per physical node) smooth the distribution dramatically.

### T21 — Idempotency
- **Mistake:** Using `idempotency_key` as just a unique ID but not storing the response and returning it on duplicate.
- **Fix:** Idempotency means same key → same response. If you receive a duplicate key, you must return the original response, not process again. Store key → response in Redis with appropriate TTL.

### T22 — Blob / Object Storage
- **Mistake:** Proxying large file uploads through your API servers.
- **Fix:** API server becomes a bottleneck and bandwidth cost multiplier. Use presigned S3 URLs — the client uploads directly to S3, your server only generates the signed URL. You never touch the file bytes.

### T23 — Search Systems
- **Mistake:** Using Elasticsearch for primary data storage (not just search).
- **Fix:** Elasticsearch is eventually consistent, has no ACID transactions, handles schema changes poorly, and is not optimised for low-latency point lookups. Use a primary DB for source of truth; sync to Elasticsearch for search.

### T24 — Time-Series Databases
- **Mistake:** Using high-cardinality labels in Prometheus (e.g., user_id, request_id).
- **Fix:** Each unique label combination = one time series. user_id with 10M users = 10M time series per metric. Prometheus crashes. Never use unbounded values as labels; use logs or traces for per-entity analysis.

### T25 — Observability
- **Mistake:** Alerting on causes (CPU > 80%) instead of symptoms (P99 latency > 1s, error rate > 1%).
- **Fix:** Cause-based alerts fire constantly for non-impacting events. Users care about symptoms. Alert on what users experience; investigate causes during the incident. Exception: predictive capacity alerts (disk filling in 4 hours) are fine.

### T26 — Service Discovery
- **Mistake:** Using a long DNS TTL (300s+) in dynamic container environments.
- **Fix:** Containers restart frequently with new IPs. A 300s TTL means clients route to dead IPs for up to 5 minutes after a pod restart. Use 5–30s TTL for service discovery DNS. Also add client-side retry with DNS re-lookup on connection failure.

### T27 — URL Shortener
- **Mistake:** Using HTTP 301 when analytics tracking is required.
- **Fix:** 301 (Permanent Redirect) is cached by browsers — subsequent clicks never hit your server, so you can't count them. Use 302 (Temporary Redirect) for click tracking, 301 only when you want the cache performance and don't need analytics.

### T28 — Distributed Cache
- **Mistake:** Not accounting for the hot key problem in Redis.
- **Fix:** A single Redis key at 1M+ QPS saturates the node handling that hash slot (in Redis Cluster). Solutions: shard the hot key by appending a suffix (`key:0..key:N`), use local in-process cache with short TTL to buffer Redis, or use read replicas for hot keys specifically.

### T29 — Chat System
- **Mistake:** Using a relational DB (PostgreSQL) for the message store without time-based partitioning.
- **Fix:** Chat tables grow unboundedly. A single `messages` table in PostgreSQL will slow dramatically past 100M rows on range scans. Use Cassandra with composite pk  `(channel_id, message_id)` (time-ordered via ULID/TSID), or partition PostgreSQL by month.

### T30 — Video Streaming
- **Mistake:** Transcoding video synchronously in the upload API handler.
- **Fix:** Transcoding a 10-minute video takes 5–30 minutes of CPU. Doing it synchronously times out the client connection and blocks the upload worker. Use a job queue (SQS/Kafka): upload stores raw file, queue triggers transcoding workers asynchronously.

### T31 — Ride-Sharing
- **Mistake:** Storing driver location updates in PostgreSQL with a GIS extension.
- **Fix:** Driver location updates arrive at 1M+ writes/second for a large fleet. PostgreSQL with GIS can handle ~10K writes/second. Use a write-optimised store (Cassandra, Redis GEOADD) for live locations; use PostgreSQL only for history/analytics.

### T32 — News Feed
- **Mistake:** Using pure fan-out-on-write for all users including celebrities.
- **Fix:** A celebrity with 50M followers posting once triggers 50M fan-out writes in seconds. That's 50M DB writes in a burst — catastrophic. Use fan-out-on-write for normal users, fan-in-on-read (pull + merge) for celebrity accounts. Hybrid threshold = ~10K followers.

### T33 — Notification System
- **Mistake:** Not handling APNs/FCM token expiry.
- **Fix:** Device tokens expire or become invalid when users reinstall the app or change devices. If you keep sending to invalid tokens, push providers may rate-limit or ban your sender ID. Process `invalid token` errors from provider responses and delete the token from your records.

### T34 — Distributed Job Scheduler
- **Mistake:** Not making scheduled jobs idempotent.
- **Fix:** With at-least-once execution semantics (leader failover mid-job, network retry), the same job can run twice. If your job sends an email or charges a user, duplicate execution is catastrophic. Design jobs to be idempotent: track a `job_execution_id` and deduplicate at the action layer.

### T35 — Rate Limiter Service
- **Mistake:** Not handling the boundary case in fixed window rate limiting.
- **Fix:** With a 100 req/min fixed window, a user can make 100 requests at 23:59:59 and 100 more at 00:00:01 — effectively 200 requests in 2 seconds. Use sliding window counter (Redis) to prevent boundary bursting.

### T36 — File Storage System
- **Mistake:** Uploading the entire file on any small change (no delta sync).
- **Fix:** A 1 GB file with a 1-byte change should only transfer the changed chunks (4–16 MB blocks). Delta sync uses content-addressable block hashing — only blocks whose hash changed are transferred. Dropbox and Google Drive both do this; forgetting it wastes bandwidth and storage.

---

## Top 5 Universal Mistakes (Any Topic)

1. **Designing without quoting numbers** — Always give estimates. "I'll use a cache" is weaker than "I'll add a Redis cache to absorb the 50K read QPS we estimated."
2. **Single point of failure** — Any component without redundancy is a SPOF. Always ask: "What happens if this server dies?"
3. **Ignoring the write path** — Most candidates design the read path well. The write path (how data gets in, what happens on write failure, idempotency) is where real complexity hides.
4. **Not stating trade-offs** — "I chose Cassandra" is incomplete. "I chose Cassandra because we need 50K writes/s and our access pattern is always by user_id — the partition key fits perfectly. The trade-off is no JOINS and eventual consistency on reads" is what interviewers want.
5. **Overcomplicating the design upfront** — Starting with Kafka, Spark, and 10 microservices for a URL shortener signals poor engineering judgment. Start simple. Add complexity only when a specific scale requirement justifies it.
