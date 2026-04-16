# Chat System — Notes & Reference

## Protocols Comparison

| Protocol | Connection | Latency | Server Push | Use Case |
|---|---|---|---|---|
| HTTP short poll | New per request | High | No | Simple status check |
| HTTP long poll | Held open until event or timeout | Medium | Simulated | Legacy chat |
| Server-Sent Events (SSE) | Persistent, server→client only | Low | Yes (unidirectional) | News feeds, notifications |
| WebSocket | Persistent, bidirectional | Very low | Yes (bidirectional) | Chat, real-time gaming |

**Chat systems use WebSocket** — bidirectional is essential (typing indicators, delivery receipts both go client↔server).

---

## Message Delivery States (WhatsApp Model)

```
Sent      → ✓  (server received message, stored in Cassandra)
Delivered → ✓✓ (recipient's device received, app not necessarily open)
Read      → ✓✓ (blue ticks — recipient opened the conversation)
```

Implementation:
- `✓`: Server sends ACK to sender after writing to Cassandra
- `✓✓`: Recipient's app sends a "delivered" ACK to server when message is pushed
- Blue `✓✓`: Recipient's app sends "read" event when user opens conversation

---

## Message Storage: Why Cassandra

| Requirement | Why Cassandra |
|---|---|
| High write throughput (1M msg/sec) | Append-only writes, LSM tree (no B-tree maintenance) |
| Time-range queries per conversation | Partition by conversation_id; TIMEUUID clustering key |
| Horizontal scale | Add nodes, data rebalances automatically |
| No ACID needed | Messages don't need transactions |
| TTL-based expiry | Native TTL per row |

**Partition key = conversation_id.** All messages for a chat are co-located. Range query = read one partition (fast).

**Anti-pattern:** Partitioning by `user_id` — then a conversation between A and B stores messages in two different partitions. You'd need two reads to reconstruct the conversation.

---

## Message ID Design Options

| Option | Ordering | Uniqueness | Central Coord? |
|---|---|---|---|
| Server timestamp | Approximate (clock skew) | No (collisions at high rps) | No |
| Snowflake ID | Time-ordered, sortable | Global (machine+seq) | No |
| Cassandra TIMEUUID | Time-ordered | Global (mac+clock) | No |
| DB auto-increment | Strict per-table | Global if single DB | Yes — bottleneck! |
| Lamport clock per conversation | Strict per-conversation | Per-conversation only | Redis INCR per conv |

**Best choice:** Snowflake ID or TIMEUUID — sortable, globally unique, no central coordinator.

---

## Presence System Design

```
Redis Hash: presence:{user_id} = {server_id: "s-7", status: "online"}
TTL: 90 seconds (heartbeat every 30s refreshes TTL)

Online check:  EXISTS presence:{user_id}   → O(1)
Server lookup: HGET presence:{user_id} server_id
```

**Scale:** 500M users × ~100 bytes/entry = 50GB in Redis — manageable with a Redis Cluster.

**Last seen:** Store `last_seen` timestamp in the presence hash or in a separate DB. Expose via API with privacy controls (can hide from non-contacts).

---

## Fan-out in Group Chats

| Approach | Storage | Delivery latency | Max group size |
|---|---|---|---|
| Fan-out on write (1 copy per member) | N × message size | Fast | ~100 (amplification) |
| Fan-out on read (1 copy total) | 1 × message size | Read on demand | Unlimited |
| Hybrid (same server = direct; different = Kafka) | 1 copy | Low | ~1024 (WhatsApp limit) |

---

## Push Notification Flow (Offline Users)

```
Message arrives for offline user
  ↓
Chat Server: delivery attempt fails (no WebSocket)
  ↓
Push Notification Service
  ├── iOS: APNs (Apple Push Notification Service)
  └── Android: FCM (Firebase Cloud Messaging)
        ↓
Device wakes up → app opens WebSocket → Chat Server delivers queued messages
```

Push payload is lightweight (notification only, not message content for privacy). App fetches actual messages via WebSocket after reconnection.

---

## Key Numbers

| Metric | Value |
|---|---|
| Messages/day | 100 billion |
| Messages/sec | 1.15 million |
| Active connections | 350M WebSockets |
| Chat servers needed | ~3,500 |
| Message avg size | ~200 bytes |
| Storage/year (compressed) | ~730 TB |
| Presence Redis size | ~50 GB |
| Cassandra partitions | 1 per conversation |
| Max group size (WhatsApp) | 1,024 members |

---

## Security Considerations

- **End-to-end encryption (E2EE):** Messages encrypted on sender's device, decrypted only on recipient's device. Server stores ciphertext (Signal Protocol: double ratchet algorithm)
- **Message transport:** TLS 1.3 for WebSocket connections
- **Auth:** JWT or session token sent on WebSocket upgrade request
- **Group key distribution:** When a group member is added/removed, a new group encryption key must be distributed to all current members
