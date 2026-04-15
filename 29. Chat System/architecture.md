# Chat System (WhatsApp) — Architecture

## Problem Statement

Design a chat system supporting:
- 1:1 and group messaging
- ~500M DAU, ~100B messages/day
- Message delivery guarantees: at-least-once, deduplicated at receiver
- Online/offline status, read receipts, typing indicators
- Message history (persistent)
- Push notifications for offline users

---

## Capacity Estimation

```
Messages/day: 100B → ~1.15M messages/sec
Message size: ~200 bytes avg (text) + metadata
Storage/day: 100B × 200B = 20TB/day → 7.3PB/year (uncompressed)
Compressed: ~2TB/day

Active connections (WebSocket): 500M × 70% online = 350M connections
```

**Dominant challenge: 350M persistent WebSocket connections.**

---

## Core Architectural Components

### 1. Connection Layer (Chat Servers)

Each user maintains one **persistent WebSocket connection** to a Chat Server:

```
Client ──WebSocket──▶ Chat Server A (manages ~100K connections each)
```

At 350M connections ÷ 100K per server = **3,500 Chat Servers**.

Chat servers are stateful (they hold open sockets). They don't do message persistence — they're pure message routers.

### 2. Message Routing

When User A (connected to Server 1) sends a message to User B (connected to Server 7):

```
User A
  │ WebSocket frame
  ▼
Chat Server 1
  │ Looks up: "which server is User B connected to?"
  │ reads from: Presence Service (Redis: user_id → server_id)
  ▼
Message Queue (Kafka topic: messages)
  ▼
Chat Server 7 (User B's server)
  │ Pushes to User B's WebSocket
  ▼
User B
```

**Why Kafka between servers?** Chat servers crash, restart, scale out. A durable message queue handles the case where Server 7 crashes before delivering to User B — the message stays in Kafka until Server 7 (or its replacement) consumes it.

### 3. Presence Service

Tracks which chat server each user is connected to:

```
Redis Hash: user_presence:{user_id} → {server_id, last_heartbeat}
```

- On WebSocket connect: `HSET user_presence:{uid} server_id "server-7" last_heartbeat {ts}`
- Heartbeat every 30s: update timestamp
- On disconnect / TTL expiry: user is "offline"
- Fan-out to user's contacts on status change: publish to `presence:{user_id}` channel

---

## Message Storage

### Database Choice: Cassandra (or ScyllaDB)

Why Cassandra for messages?
- Append-only writes (chat messages never update)
- Time-range queries: "give me the last 50 messages in conversation X"
- Very high write throughput
- Partition by conversation → all messages for a chat on the same node

**Schema:**
```sql
CREATE TABLE messages (
    conversation_id  UUID,
    message_id       TIMEUUID,     -- Time-ordered UUID (UUID v1) — acts as sort key
    sender_id        BIGINT,
    content          TEXT,
    message_type     TINYINT,      -- 0=text, 1=image, 2=video, 3=file
    client_msg_id    UUID,         -- Client-generated dedup ID
    created_at       TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC)
  AND default_time_to_live = 7776000;  -- 90 days TTL
```

**Partition key = conversation_id:** All messages in a conversation are co-located on the same Cassandra node → efficient range scans.

**message_id = TIMEUUID (UUID v1):** Contains timestamp + MAC address → time-sortable + globally unique. Solves the ordering problem without a central sequence generator.

---

## Message ID and Ordering

**Problem:** User A and User B send messages simultaneously. How do we determine the canonical order?

**Option 1: Server timestamp.** Assign timestamp at server receipt. Risk: clock skew between servers causes out-of-order messages.

**Option 2: Snowflake ID.** Each Chat Server generates Snowflake IDs. Time-ordered, unique, no coordination needed. **Best choice.**

**Option 3: Lamport clock per conversation.** `seq = INCR conv:{conv_id}:seq`. Monotonically increasing per conversation. Problem: hot Redis key if every message in a high-traffic group chat hits this counter.

---

## Message Delivery Guarantees

WhatsApp's delivery model:
```
Sender client → Chat Server → ACK to sender (message persisted)
                            ↓
               Kafka (durable)
                            ↓
               Recipient's Chat Server → push to recipient's WebSocket
                            ↓
               Recipient client ACK → server marks "delivered" ✓✓
               Recipient opens chat → server marks "read"       ✓✓ (blue)
```

**At-least-once delivery:** The Chat Server retries delivery until it receives the recipient's ACK. The client deduplicates using `client_msg_id`.

**Deduplication:** Client generates `UUID4` for each message before sending. Server stores `client_msg_id` in Cassandra with `IF NOT EXISTS`. If network retry delivers the same message twice, the second insert is a no-op (Cassandra lightweight transaction).

---

## Group Messaging

### Fan-out Approaches

**Fan-out on write (push model):**
When User A sends to a group of N members, the server writes N copies of the message (one to each member's message queue). Recipients' servers deliver from their own queue.

- Pros: delivery is fast
- Cons: storage amplification (N copies per message); impractical for groups with 10,000 members

**Fan-out on read (pull model):**
Store one copy of the message in the group conversation. When a member comes online, they fetch new messages from the conversation.

- Pros: single copy stored
- Cons: all members query the same conversation partition on login → hot partition

**WhatsApp's approach:**
Groups are capped at 1024 members (practical fan-out). Single message stored in conversation. Chat server pushes to each member's connection server simultaneously. For large groups: fan-out is async via a dedicated Fan-out service consuming from Kafka.

---

## Offline Message Delivery

If User B is offline when a message arrives:
1. Kafka message is not consumed immediately
2. Chat Server tries WebSocket push → fails → no ACK
3. After retry timeout → trigger **Push Notification Service**

**Push path:** APNs (Apple) / FCM (Google) delivers a push notification to wake the app. When user opens app → WebSocket reconnects → Chat Server delivers pending messages from Cassandra.

---

## Architecture Diagram

```
┌──────────┐    WebSocket    ┌─────────────────┐
│ Client A │◄───────────────►│  Chat Server A  │
└──────────┘                 └────────┬────────┘
                                      │
                          ┌───────────▼──────────┐
                          │   Kafka (messages)    │
                          └───────────┬────────── ┘
                                      │
┌──────────┐    WebSocket    ┌────────▼────────┐
│ Client B │◄───────────────►│  Chat Server B  │
└──────────┘                 └────────┬────────┘
                                      │
                    ┌─────────────────┼──────────────────┐
                    ▼                 ▼                   ▼
             ┌──────────┐    ┌──────────────┐    ┌────────────────┐
             │Cassandra │    │Redis Presence│    │Notification    │
             │(messages)│    │(online state)│    │Service(FCM/APNs│
             └──────────┘    └──────────────┘    └────────────────┘
```

---

## Key Scale Numbers

| Component | Scale |
|---|---|
| Chat Servers | ~3,500 (100K connections each) |
| Messages/sec | 1.15M |
| Storage/year (compressed) | ~730 TB |
| Kafka throughput needed | ~230 MB/s |
| Redis Presence entries | 500M keys (~50GB) |
| Cassandra nodes (10TB each) | ~75 nodes |
