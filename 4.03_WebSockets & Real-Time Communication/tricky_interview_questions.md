# WebSockets & Real-Time Communication — Tricky Interview Questions

## Question 1

**"You're designing a notification system for a mobile app. A PM suggests using WebSockets. Would you agree?"**

### Why it's tricky
WebSocket sounds right for real-time, but mobile has specific constraints the PM likely didn't consider.

### Strong Answer

No — or at least, not exclusively. WebSockets are a poor fit as the primary mobile notification channel for several reasons:

1. **Battery drain**: Maintaining a persistent TCP connection + heartbeat on mobile consumes radio and CPU continuously. OS background process limits (iOS 3-minute background execution limit, Doze mode on Android) will kill long-lived connections anyway.

2. **Push notifications already exist**: APNs (iOS) and FCM (Android) are designed exactly for this. They use system-level persistent connections shared across all apps, maintained by the OS — the user's battery budget is spent once, not once per app. They also work when the app is killed.

3. **Correct approach**: APNs/FCM for background/killed app. WebSocket for in-app real-time (e.g., live chat, active game). SSE for in-app one-way updates (e.g., order tracker while app is open).

**Follow-up trap**: "But APNs/FCM can be slow." → Acceptable latency is usually 100ms–2s. WebSocket in an active app adds 10–50ms but at the cost of maintaining the connection. This trade-off is only worth it for specific interactive features.

---

## Question 2

**"How do you handle a user sending a message, closing the app immediately, then reopening it and wondering why their message shows as 'not delivered'?"**

### Why it's tricky
Tests understanding of the full delivery state machine and offline sync — most candidates only think about the happy path.

### Strong Answer

This is an end-to-end delivery guarantee problem:

**Step 1 — Optimistic local state**: The client renders the message immediately as "sending" (one grey tick) before the server ACK arrives.

**Step 2 — Server ACK**: When the server persists the message to the database, it sends an ACK with the `msg_id` → client updates to "sent" (one dark tick). If the app closes before this ACK, the message is in an **unconfirmed** state.

**Step 3 — On reopening**: The client must perform a sync on startup. Send `{"type": "sync", "unconfirmed_messages": ["msg-abc"], "last_seq": N}`. Behaviour:
- If server received and persisted "msg-abc" → return ACK → client updates to "sent"
- If server never received it (connection died too early) → client resends the message with idempotency key = `msg-abc`

**Step 4 — Idempotency key**: The `msg_id` generated client-side acts as an idempotency key. The server deduplicated on insert: `INSERT INTO messages WHERE msg_id = ? ON CONFLICT DO NOTHING`. Safe to resend.

**Result**: Message eventually reaches "delivered" state. The user experience shows a brief "sending" state, not a permanent failure.

---

## Question 3

**"You're told: 'Use WebSocket and Redis Pub/Sub for fan-out. That's all you need.' What's wrong with this for a system like Slack?"**

### Why it's tricky
It's almost right — tests whether candidates can identify the gaps (offline delivery, large channel scale, message persistence).

### Strong Answer

Redis Pub/Sub is fire-and-forget with no persistence. Three major gaps:

**1. Offline users**: If a user's WebSocket is disconnected when a message is published, they get nothing. Redis Pub/Sub doesn't queue messages for offline receivers. Fix: write messages to a database first, then publish to Pub/Sub. Offline users sync from the DB on reconnect.

**2. Fan-out at scale**: A Slack channel with 50,000 members means publishing on a Redis channel triggers 50,000 deliveries. If each member is connected to a different server instance, each server must filter + forward. At celebrity/massive channel scale, this becomes a write storm. Fix: hybrid fan-out — push only to online+active users; others pull on channel open.

**3. Message ordering under concurrent sends**: Two messages published to the same channel at the same millisecond can arrive out of order at different subscribers (CPU scheduling, network jitter). Fix: server-assigned monotonic sequence numbers. Clients detect gaps and sync.

**What you actually need**: Message DB (Cassandra or Postgres) + Redis Pub/Sub for online delivery + offline sync endpoint + sequence numbers.

---

## Question 4

**"If I have 10 replicas of my WebSocket server behind a round-robin load balancer, which server handles the reconnect?"**

### Why it's tricky
Tests whether candidates understand connection affinity vs. stateless routing.

### Strong Answer

Any server can handle the reconnect — that's the point of building the connection registry in Redis. The flow:

1. Client reconnects to the load balancer.
2. Round-robin assigns it to Server 6 (it was on Server 3 before).
3. Server 6 handles the WebSocket handshake, creates a new connection entry: `HSET ws:user:alice server_id "srv-6" conn_id "new-conn-id"` (overwrites the old one).
4. Server 6 reads the client's `last_seq` from the sync payload, fetches missed messages from the database, and delivers them.
5. If Server 3 still has a stale entry for Alice and tries to deliver a message, the registry now points to Server 6, so Server 3 publishes to Redis and Server 6 delivers it.

**Sticky sessions** are an alternative: the LB always routes Alice to Server 3. Simpler, but: fails if Server 3 dies, prevents rebalancing, and doesn't work well with auto-scaling. The Redis registry pattern is preferred for resilience.

---

## Question 5

**"Presence detection via heartbeat TTL marks a user offline 65 seconds after their last heartbeat. But a sudden network blip kills the TCP connection and the client reconnects in 2 seconds. The user was never really offline. Does this cause a bug?"**

### Why it's tricky
Exposes the impedance mismatch between TCP disconnect detection and presence state.

### Strong Answer

It can — but only if you're broadcasting presence changes too eagerly. The nuanced answer:

**What TTL is good for**: Detecting a user that is truly gone (app killed, device powered off). The 65s window absorbs brief network instability — if the client reconnects within 65s and calls `SETEX` again, the key never expires and presence was never lost.

**The actual problem**: Emitting a `user_went_offline` event to all room members the instant the WebSocket disconnects (before the TTL expires). This triggers an unnecessary flash of "Alice is offline" for 2 seconds.

**Fix**: Don't emit offline immediately on WebSocket disconnect. Instead:
1. Mark the connection as "disconnected but presence TTL still alive."
2. Set a short grace period (5–10s): if the user reconnects within the grace period, no offline event is published.
3. Only publish `user_went_offline` after the grace period without reconnect.

Twitter and Slack both use this pattern — you see "last seen" transitions only after 30s–60s, not on every micro-disconnect.

---

## Question 6

**"Interviewer: 'Great, you've designed a WebSocket chat. Now add end-to-end encryption.' What breaks in your architecture?"**

### Why it's tricky
E2E encryption fundamentally challenges server-side operations that candidates' architectures likely depend on.

### Strong Answer

Several server-side features break or require redesign:

**1. Server-side message search**: Can't search ciphertext. If you need search, you need client-side search over locally decrypted messages (Signal does this) or a server-side searchable encryption scheme (complex).

**2. Push notification content**: APNs/FCM notifications can't show message previews because the server only has ciphertext. Signal and WhatsApp send empty notification payloads ("You have a new message") and the app decrypts on receipt.

**3. Multi-device support**: Each device has its own key pair. Sender must encrypt the message separately for every recipient device. If Alice has 3 devices, the sender encrypts 3 times. Server stores N ciphertexts per message.

**4. Group message fan-out**: In a Signal-style group of 100 members with 3 devices each → sender encrypts 300 copies. At scale, this is the sender's CPU/bandwidth cost, not the server's.

**5. Message storage**: Server stores ciphertext only. Key management (Signal Protocol: Double Ratchet + X3DH) must live entirely on the client. Server has no ability to decrypt for moderation/compliance — design trade-off to call out explicitly.

---

## Question 7

**"Why might WebSocket connections start failing after you scale to 100 servers, even though each server is only at 30% load?"**

### Why it's tricky
Points to infrastructure-level issues (file descriptors, ephemeral port exhaustion) rather than application logic.

### Strong Answer

Three common culprits at scale that have nothing to do with CPU:

**1. File descriptor limits**: Each WebSocket connection is a file descriptor on Linux. Default `ulimit -n` is 1024. At 100K connections × 100 servers = 10M file descriptors needed across the fleet. Each server must have `ulimit -n` raised (e.g., to 1,000,000) and the OS `fs.file-max` tuned.

**2. Ephemeral port exhaustion on the load balancer**: The LB proxies connections using source IP + port. The port range 1024–65535 = ~64K ports per destination IP. With 100 backend servers, the LB may hit this limit if it's proxying via a single IP. Solution: multiple LB IPs, or use IP_TRANSPARENT mode.

**3. Redis Pub/Sub connection saturation**: Each server subscribes to Redis. At 100 servers, Redis must maintain 100 subscribe connections. If each server also subscribes to thousands of room channels with `SUBSCRIBE`, Redis channel memory and message dispatch CPU can become the bottleneck. Solution: use wildcard `PSUBSCRIBE room:*` (one subscription per server) and filter in-process.

**4. Heartbeat CPU at scale**: 100 servers × 100K connections = 10M heartbeats every 30s = ~333K heartbeat frames/second across the fleet. Each one needs Redis `SETEX`. This becomes a Redis write bottleneck. Solution: batch heartbeat renewals with Lua scripts or client-side TTL sliding.
