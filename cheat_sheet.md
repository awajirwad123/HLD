# System Design Cheat Sheet — All 36 Topics

One-page reference: concept summary, key number, and the "gotcha" you must not miss.

---

## Core Concepts (T1–T13)

| # | Topic | One-Liner | Key Number | Gotcha |
|---|---|---|---|---|
| 1 | **System Design Framework** | Requirements → Estimates → API → Data Model → Design → Deep Dive → Trade-offs | 5 phases × 5–15 min each | Never jump to design without 5 min of requirements |
| 2 | **Networking Basics** | TCP guarantees delivery + order (3-way handshake). UDP is fast but lossy. DNS maps name → IP. | TCP handshake: 1.5 RTTs | HTTP/2 multiplexes; HTTP/1.1 head-of-line blocking |
| 3 | **Client-Server Architecture** | Client requests; server responds. Stateless preferred (horizontally scalable). | Stateless server = no shared memory between requests | Sessions in sticky cookies vs server-side sessions vs JWTs |
| 4 | **Load Balancing** | Distribute traffic across servers. L4 (TCP) vs L7 (HTTP). Algorithms: round-robin, least-conn, IP-hash. | 1M+ RPS per modern L7 LB | IP-hash breaks with multiple LB nodes; use consistent hashing |
| 5 | **Caching** | Read-through, write-through, write-back, cache-aside. Redis (in-memory, single-threaded). | Redis: 100K–1M QPS | Cache stampede on expiry — use jitter or probabilistic refresh |
| 6 | **Databases — SQL** | ACID, B-tree indexes, normalisation. Vertical scaling limit. Use read replicas for reads. | Single PostgreSQL: ~5K complex QPS | Missing index on JOIN column = full table scan |
| 7 | **Databases — NoSQL** | Eventual consistency, horizontal scale. Partition key determines data locality. | Cassandra: 10K–50K writes/s per node | Wide partition problem — unbounded lists in a single partition |
| 8 | **Data Partitioning & Sharding** | Split data across nodes by range, hash, or directory. Consistent hashing minimises resharding. | Adding 1 node: consistent hashing moves ~1/N keys | Low-cardinality shard key → hotspot |
| 9 | **Replication** | Primary → replica. Synchronous (durability) vs async (performance). Semi-sync: one ack needed. | Replication lag: 1–100ms typical | Replica may return stale reads; use `read-your-writes` routing if needed |
| 10 | **CAP Theorem** | CA / CP / AP — can only guarantee 2/3 during network partition. Real choice: CP vs AP. | Partition happens ~minutes/year | "CA" doesn't exist in distributed systems — partitions are inevitable |
| 11 | **Consistency Models** | Strong → linearisable (sync). Eventual → BASE (async). Sequential, causal, read-your-writes in between. | Linearisability: 1 RTT extra cost | "Eventual consistency" means conflicts must be resolved (LWW, CRDTs, etc.) |
| 12 | **Distributed Transactions** | 2PC (blocking, coordinator SPOF). Saga (compensating transactions, async). Outbox pattern for at-least-once. | 2PC adds 2 network RTTs per transaction | Saga lacks isolation: partial failures leave intermediate visible state |
| 13 | **Consensus Algorithms** | Raft (leader election + log replication). Need quorum (n/2 + 1). etcd powers Kubernetes. | Raft election timeout: 150–300ms | Even number of nodes can deadlock — always use 3/5/7 |

---

## Architecture Patterns (T14–T26)

| # | Topic | One-Liner | Key Number | Gotcha |
|---|---|---|---|---|
| 14 | **API Design** | REST (resource-oriented), GraphQL (client-specified), gRPC (typed, binary). Versioning: URI or header. | gRPC 10× smaller payload than JSON REST | Backward compatibility: never remove fields; deprecate then delete |
| 15 | **Messaging Queues & Event Streaming** | Queue (one consumer per message). Stream (Kafka: multiple consumers, log retention). At-least-once default. | Kafka: 1M+ messages/s per broker | Exactly-once requires idempotent producers + transactional consumers |
| 16 | **WebSockets & Real-Time** | Full-duplex over TCP. Server-Sent Events (SSE) for server-to-client only. Long-polling as fallback. | WebSocket: 100K+ concurrent connections per node | Sticky sessions needed or pub/sub layer (Redis) for multi-node WS |
| 17 | **Microservices vs Monolith** | Monolith first (simpler). Split when team ownership, deploy frequency, or scale diverges. | Service call: 1–5ms overhead (vs 0.1μs in-process) | Distributed monolith = microservices without independent deployability |
| 18 | **Rate Limiting** | Token bucket (burst-friendly), sliding window (precise), fixed window (simplest). Redis for distributed. | Redis INCR + TTL: atomic window counter in < 1ms | Sliding window log uses O(requests) memory per user — cap window size |
| 19 | **Circuit Breaker & Resilience** | Closed → Open → Half-open. Bulkhead (thread pool isolation). Retry with exponential backoff + jitter. | Circuit opens after N failures in X seconds | Retry without jitter = thundering herd on recovery |
| 20 | **Consistent Hashing** | Virtual nodes on a ring. Adding a node moves only 1/N keys on average. | 150 virtual nodes per physical node (typical) | Without virtual nodes, node removal dumps all its keys onto one successor |
| 21 | **Idempotency & Exactly-Once** | Idempotency key prevents duplicate processing. Outbox pattern for reliable event emission. | Idempotency key TTL: 24h–7 days | Deduplication without distributed lock can still process twice in a race |
| 22 | **Blob / Object Storage** | Upload large files directly to S3 (presigned URL). Chunk for > 5 GB. CDN in front for reads. | S3 PUT latency: ~100–300ms | Never stream large files through your API servers — let clients go direct to S3 |
| 23 | **Search Systems** | Inverted index maps terms → document IDs. TF-IDF + BM25 for relevance. Elasticsearch shards. | Elasticsearch: 10K–50K queries/s per node | Search is eventually consistent — just-added docs may not appear immediately |
| 24 | **Time-Series Databases** | Data is append-only. Downsampling for long retention. Tags = indexed dimensions. | Prometheus TSDB: 1–3 bytes per sample (compressed) | Cardinality explosion from high-cardinality tag values crashes Prometheus |
| 25 | **Observability** | Three pillars: logs (events), metrics (numbers), traces (paths). Four Golden Signals: latency, traffic, errors, saturation. | Error budget @ 99.9% SLO = 43.8 min/month | Alert on symptoms (high P99), not causes (CPU at 80%) |
| 26 | **Service Discovery** | Client-side (client queries registry) vs server-side (LB does it). Consul/etcd for registry. K8s DNS built-in. | K8s EndpointSlice update: < 1s | DNS TTL stale records: short TTL (5–30s) in dynamic envs |

---

## System Design Practicals (T27–T36)

| # | Topic | One-Liner | Key Number | Gotcha |
|---|---|---|---|---|
| 27 | **URL Shortener** | Hash long URL → 7-char ID. 301 for SEO (cached), 302 for analytics (never cached). Store in KV store. | 7 chars base62 = 62^7 = 3.5 trillion unique URLs | Hash collision: check DB before returning alias |
| 28 | **Distributed Cache** | Redis cluster with hash slots (16384). Read replica for read scaling. TTL + LRU for eviction. | Redis Cluster: 16384 hash slots across N nodes | Hot key problem — single key at 100K+ RPS overloads one node; use local cache + replication |
| 29 | **Chat System** | WebSocket for real-time delivery. Kafka for fan-out to offline users. Message store in Cassandra (time-ordered). | Message delivery P99 < 100ms (1:1). Group chat: fan-out to all members. | Message ordering in distributed chat: logical clock or server-assigned sequence ID |
| 30 | **Video Streaming** | Upload: chunked multipart to S3. Transcoding: FFmpeg workers from queue. Delivery: CDN + adaptive bitrate (HLS/DASH). | CDN edge: 100K+ concurrent streams per POP | Transcoding is CPU-intensive — queue + worker pool, not synchronous |
| 31 | **Ride-Sharing** | Geospatial indexing (quadtree or geohash) for driver matching. Two-mode: supply (drivers push location) + demand (users request). | Uber: 1M+ concurrent drivers updating location every 4s | Location update writes >> reads; use write-optimised store (Cassandra) not PostgreSQL |
| 32 | **News Feed** | Fan-out on write (push to followers' feeds) for normal users. Fan-in on read (pull + merge) for celebrities. Hybrid for both. | Facebook/Instagram: 1 post = up to 10M fan-out writes | Pure fan-out for celebrity accounts (50M followers) causes write avalanche |
| 33 | **Notification System** | Template service → routing service → per-channel workers (push/email/SMS) → external provider (APNs, Twilio). | APNs/FCM delivery: < 1s typical, retry up to 28 days | FCM/APNs tokens expire — must handle invalid token errors and clean up |
| 34 | **Distributed Job Scheduler** | Jobs stored in DB with `next_run_at`. Workers poll or use lease-based locking. Cron triggers via scheduler service. | At-least-once execution: job must be idempotent | Two workers acquiring the same job simultaneously: use row-level locking or distributed lock |
| 35 | **Rate Limiter Service** | Token bucket (burst), sliding window (precise), fixed window (simple). Redis atomic ops for distributed counting. | Redis INCR + EXPIREAT: 1 ms per check | Sliding window log at high scale uses too much memory — use sliding window counter instead |
| 36 | **File Storage System** | Chunked uploads (4–16 MB blocks). Deduplication via content hash. Block storage (S3) + metadata DB. | Dropbox chunk size: 4 MB. Block dedup ratio: 30–60% common | Delta sync: only transfer changed blocks on update, not the full file |

---

## Key Numbers to Memorise

### Latency
| Operation | Approximate Time |
|---|---|
| L1 cache reference | 1 ns |
| L2 cache reference | 4 ns |
| RAM access | 100 ns |
| SSD read (random) | 100 μs |
| Network round-trip (same DC) | 500 μs |
| HDD read (random) | 5 ms |
| Network round-trip (cross-continent) | 100 ms |
| SSD sequential read (1 MB) | 1 ms |

### Throughput
| System | Read QPS | Write QPS |
|---|---|---|
| PostgreSQL (optimised) | 5K–50K | 1K–10K |
| Cassandra per node | 30K–50K | 10K–50K |
| Redis | 100K–1M | 100K–1M |
| Kafka per broker | — | 500K–1M msg/s |
| Elasticsearch | 10K–50K | 2K–10K |

### Storage
| Data type | Size |
|---|---|
| UUID | 16 bytes |
| Int64 timestamp | 8 bytes |
| Typical DB row | 100 bytes–1 KB |
| Image (web) | 100 KB–500 KB |
| Short video (1 min, 720p) | ~50 MB |
| 1 hour video (1080p) | ~4 GB |

---

## The Three Most Important Trade-offs

1. **Consistency vs Availability** — CP vs AP (CAP). Default to eventual consistency for reads; use strong consistency only where correctness demands it (payment deduplication, inventory).
2. **Latency vs Durability** — Async writes (cache + queue) are fast but can lose data. Sync writes are safe but slow. Use async + idempotent replay as the middle ground.
3. **Simplicity vs Scale** — Don't pre-optimise. Start with 1 DB, 1 cache, 1 server. Add Kafka, sharding, and microservices when the specific bottleneck is proven.
