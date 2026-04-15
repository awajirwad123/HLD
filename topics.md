Read [](file:///c%3A/Users/ajwaji/Music/HLD/master.md)

Read [](file:///c%3A/Users/ajwaji/Music/HLD/template.md)

---

## HLD Roadmap — Backend Engineer Interview Curriculum

---

### Phase 1: Foundations

**1. System Design Framework**
- How to approach a system design interview
- Clarifying requirements (functional vs non-functional)
- Capacity estimation basics (QPS, storage, bandwidth)
- Back-of-envelope calculations

**2. Networking Basics for System Design**
- HTTP vs HTTPS, HTTP/1.1 vs HTTP/2 vs HTTP/3
- TCP vs UDP
- DNS resolution flow
- CDN fundamentals

**3. Client-Server Architecture**
- Request-response model
- Stateless vs stateful servers
- Horizontal vs vertical scaling

---

### Phase 2: Core Building Blocks

**4. Load Balancing**
- Layer 4 vs Layer 7 load balancing
- Algorithms: Round Robin, Consistent Hashing, Least Connections
- Health checks and failover
- Real-world: Nginx, AWS ALB

**5. Caching**
- Cache types: in-process, distributed (Redis, Memcached)
- Cache strategies: Cache-aside, Write-through, Write-back
- Cache invalidation and eviction (LRU, LFU, TTL)
- Cache stampede / thundering herd

**6. Databases — SQL**
- RDBMS internals (B-tree indexes, ACID)
- Query optimization, connection pooling
- Vertical vs horizontal scaling
- Master-replica, sharding

**7. Databases — NoSQL**
- Key-value, document, column-family, graph stores
- When to choose NoSQL vs SQL
- MongoDB, Cassandra, DynamoDB use cases

**8. Data Partitioning & Sharding**
- Horizontal vs vertical partitioning
- Range-based, hash-based, directory-based sharding
- Hotspot problem and consistent hashing

**9. Replication**
- Synchronous vs asynchronous replication
- Leader-follower vs leaderless
- Read replicas and write scaling

---

### Phase 3: Distributed Systems Concepts

**10. CAP Theorem**
- Consistency, Availability, Partition Tolerance
- CP vs AP systems
- Real-world classification (HBase, Cassandra, DynamoDB)

**11. Consistency Models**
- Strong, eventual, causal, read-your-writes consistency
- Trade-offs in distributed systems

**12. Distributed Transactions**
- 2-Phase Commit (2PC)
- Saga pattern (choreography vs orchestration)
- Outbox pattern

**13. Consensus Algorithms (conceptual)**
- Paxos / Raft basics
- Leader election
- Where it appears: etcd, ZooKeeper

---

### Phase 4: Communication & Integration

**14. API Design**
- REST principles and best practices
- GraphQL vs REST vs gRPC
- Versioning strategies
- Idempotency and retries

**15. Messaging Queues & Event Streaming**
- Queues vs streams (Kafka vs RabbitMQ vs SQS)
- Producer-consumer patterns
- At-least-once / at-most-once / exactly-once delivery
- Dead letter queues, backpressure

**16. WebSockets & Real-Time Communication**
- WebSockets vs SSE vs Long polling
- Presence detection, fan-out
- Real-world: WhatsApp, Slack

**17. Microservices vs Monolith**
- Monolith → modular → microservices evolution
- Service decomposition strategies
- Inter-service communication (sync vs async)
- Service mesh basics (Istio/Envoy)

---

### Phase 5: Reliability & Scalability Patterns

**18. Rate Limiting**
- Token bucket, leaky bucket, fixed window, sliding window
- Distributed rate limiting
- Real-world: API gateways, Stripe

**19. Circuit Breaker & Resilience Patterns**
- Circuit breaker, retry, fallback, bulkhead
- Timeout strategies
- Tools: Hystrix, Resilience4j, Polly

**20. Consistent Hashing**
- Virtual nodes, ring topology
- Use in load balancers, CDNs, distributed caches

**21. Idempotency & Exactly-Once Semantics**
- Idempotent APIs and operations
- Deduplication strategies
- Real-world: payment processing

---

### Phase 6: Storage at Scale

**22. Blob / Object Storage**
- Use cases for S3-style storage
- Chunking, multipart upload
- Content addressing

**23. Search Systems**
- Inverted index, TF-IDF, BM25
- Elasticsearch architecture
- Full-text search design

**24. Time-Series Databases**
- InfluxDB, Prometheus
- Downsampling, retention policies

---

### Phase 7: Observability & Operations

**25. Observability (Logging, Metrics, Tracing)**
- Structured logging
- Metrics collection (Prometheus, Grafana)
- Distributed tracing (Jaeger, OpenTelemetry)
- Alerting strategies

**26. Service Discovery & Configuration**
- Client-side vs server-side discovery
- Tools: Consul, etcd, Kubernetes DNS

---

### Phase 8: Classic System Design Case Studies

**27. URL Shortener** (TinyURL)
**28. Distributed Cache** (Redis Clone)
**29. Chat System** (WhatsApp)
**30. Video Streaming** (Netflix / YouTube)
**31. Ride-Sharing** (Uber)
**32. News Feed / Timeline** (Twitter / Instagram)
**33. Notification System**
**34. Distributed Job Scheduler**
**35. Rate Limiter Service**
**36. File Storage System** (Dropbox / Google Drive)

---

**Pick a topic number or name to begin.** I'll generate the full folder structure (`architecture.md`, `hands_on.md`, `notes.md`, `quick-fire-questions.md`, `tricky_interview_questions.md`) for it.