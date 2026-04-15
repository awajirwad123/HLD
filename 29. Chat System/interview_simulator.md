# Chat System — Interview Simulator

## Scenario 1: "Design WhatsApp"

**Interviewer:** "Design a messaging system like WhatsApp. Walk me through your full design."

---

### Strong Answer Flow

**Step 1 — Clarify (2 min)**
> "A few questions: 1:1 and group chats? Read receipts and typing indicators? Message history retention? Do we need E2EE? Scale target?"

Assume: 500M DAU, 100B messages/day, 1:1 + groups up to 256 members, read receipts, 5-year history, E2EE optional (mention it).

**Step 2 — Estimate**
- 1.15M messages/sec; 350M concurrent WebSockets
- Storage: ~2TB/day compressed; ~730TB/year
- Dominant challenge: **350M persistent connections**

**Step 3 — API design**
```
WebSocket: wss://chat.example.com/ws
  connect → authenticate (JWT in header)
  
  Client → Server frames:
    { type: "message", to: "user_id", content: "...", client_msg_id: "uuid" }
    { type: "typing",  to: "user_id" }
    { type: "read",    conversation_id: "...", up_to_message_id: "..." }
  
  Server → Client frames:
    { type: "ack",     client_msg_id: "...", server_msg_id: "..." }
    { type: "message", from: "...", ... }
    { type: "typing",  from: "..." }
    { type: "receipt", message_id: "...", status: "delivered|read" }
```

**Step 4 — Architecture**
- **Connection Gateway:** L7 load balancer (sticky by user_id hash); ~3,500 Chat Server instances
- **Presence Service:** Redis Cluster; user_id → server_id mapping
- **Message Bus:** Kafka (partitioned by conversation_id); ensures ordered delivery per conversation
- **Message Store:** Cassandra (partition by conversation_id, cluster by TIMEUUID)
- **Push Notifications:** Separate service consuming Kafka for offline users; APNs + FCM

**Step 5 — Message flow**
Walk through sender→receiver happy path. Then cover: recipient offline, cross-server delivery, group fan-out.

**Step 6 — Failure handling**
"If a Chat Server crashes, its users reconnect within 30 seconds (WebSocket reconnect + exponential backoff). Undelivered Kafka messages are consumed from the committed offset by the replacement server. No messages lost."

---

## Scenario 2: "The typing indicator is expensive. Every keystroke sends a WebSocket frame. 500M users typing = billions of events/sec. How do you optimize?"

---

### Answer

**Problem:** If every keystroke triggers a typing indicator frame, 500M active users typing at 5 keystrokes/sec = 2.5B events/sec — completely impractical.

**Optimization 1: Debounce at the client.**
Client starts a 3-second timer when the user begins typing. The first keystroke sends a `typing_start` event. Subsequent keystrokes reset the timer. After 3 seconds of no keystrokes, send `typing_stop`. Instead of 5 frames/sec, we send at most: 1 `typing_start` + 1 `typing_stop` per typing session ≈ ~0.1 events/sec per user.

**Optimization 2: TTL instead of stop event.**
Server sets `typing:{conv_id}:{user_id} = 1` in Redis with TTL=5s. Client sends `typing_start` only; server handles expiry automatically. If client sends `typing_start` every 3s, Redis key stays alive. When user stops typing, key expires in 5s → recipient sees "typing..." stop naturally.

```redis
SET typing:{conv_id}:{user_id} 1 EX 5  # Refresh every 3s from client
```

Recipient's server subscribes to `keyspace:typing:{conv_id}:*` events to push typing indicators. This avoids even sending a `typing_stop` frame explicitly.

**Optimization 3: Don't persist to Cassandra.** Typing events are ephemeral. They go: `Client → Chat Server → recipient's Chat Server → recipient's WebSocket`. No Kafka, no DB. If delivery fails (recipient offline), it's dropped silently.

---

## Scenario 3: "How do you handle message synchronization across multiple devices for the same user?"

**Interviewer:** "A user has WhatsApp on both their phone and laptop. Messages sent on phone should appear on laptop. Messages sent from web should appear on phone. Design multi-device sync."

---

### Design

**Core model change:** Messages are logically addressed to a **user** (not a device). Each device subscribes independently.

**Device registration:**
```
user_devices:{user_id} → SET of {device_id, device_token, last_seen_message_id}
```

**Message delivery:**
When User A sends to User B:
- Fan out to ALL of User B's registered devices (phone + laptop)
- Each device connection gets its own WebSocket
- Each device independently ACKs delivery

**Message sync state per device:**
```
device_sync:{user_id}:{device_id} → last_received_message_id
```

When a device comes online:
1. Fetch `last_received_message_id` for this device
2. Query Cassandra: `SELECT * FROM messages WHERE conversation_id = X AND message_id > last_id LIMIT 100`
3. Receive all missed messages since last sync

**Sent messages from one device appear on other devices:**
When User A sends a message from phone:
- Server delivers to all of User B's devices ✓
- Server also delivers back to all of **User A's other devices** (laptop, etc.) — same fan-out

**Read state sync:**
When User A reads a message on phone, send `read` event to server. Server broadcasts to all of User A's other devices: `{type: "read_sync", conversation_id: X, up_to_message_id: Y}`. Laptop marks messages up to Y as read.

**E2EE complication:** With E2EE, each device has its own key pair. Sender must encrypt the message separately for each of the recipient's N devices AND each of the sender's own N devices. This is why WhatsApp Web had to keep the phone active initially — it used a relay model. Signal handles this with sealed-sender and multi-key fan-out.
