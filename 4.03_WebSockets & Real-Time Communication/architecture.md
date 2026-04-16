# WebSockets & Real-Time Communication — Architecture Deep-Dive

## The Core Problem

HTTP is a request-response protocol — the server cannot push data to a client without the client asking first. Real-time applications (chat, live sports scores, collaborative editing, stock tickers) need server-initiated updates with low latency. Three mechanisms solve this with different trade-offs.

```
HTTP request-response (not real-time):
  Client → "Give me new messages" → Server
  Client ← [messages]             ← Server
  ... wait N seconds ...
  Client → "Give me new messages" → Server   ← client must keep polling

Real-time push (server-initiated):
  Client ←─── "New message!" ─── Server
  Client ←─── "User X typing"─── Server
  Client ←─── "Score: 3-2"   ─── Server
```

---

## 1. Long Polling

The simplest approximation of real-time: client sends a request; server holds it open until data is available (or a timeout), then responds. Client immediately re-issues the request.

```
Client                       Server
  │                              │
  │── GET /messages?since=t1 ───►│  (server holds request open)
  │                              │  ... waiting for new data ...
  │◄── [new messages] ──────────│  (responds when data arrives, or timeout ~30s)
  │── GET /messages?since=t2 ───►│  (immediately re-issues)
  │                              │
```

**Characteristics:**
- Works with standard HTTP/1.1 — no protocol upgrade needed
- Each "push" requires a full HTTP handshake overhead (headers ~800 bytes)
- Server holds one connection per client — resource intensive at scale
- Latency: typically 0–500ms depending on poll timeout and processing
- Load balancer friendly — each request is independent

**Use when:** Simple admin dashboards, legacy systems that can't upgrade, clients behind aggressive HTTP proxies.

---

## 2. Server-Sent Events (SSE)

Client opens one long-lived HTTP connection; server streams events over it as plain text. Unidirectional — server to client only.

```
Client                        Server
  │                               │
  │── GET /stream ───────────────►│  (one connection, kept open)
  │                               │
  │◄── data: {"score": "2-1"} ───│  (server pushes when ready)
  │◄── data: {"score": "3-1"} ───│
  │◄── data: {"score": "3-2"} ───│
  │                               │  (connection stays open)
```

**Wire format:**
```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

data: {"type": "message", "content": "Hello"}\n\n
id: 42\n
data: {"type": "typing", "user": "alice"}\n\n
retry: 3000\n\n
```

**Characteristics:**
- Native browser support (`EventSource` API) with **automatic reconnect**
- `Last-Event-ID` header: client sends last received event ID on reconnect — server can replay missed events
- Works over HTTP/1.1 (one connection per tab) and HTTP/2 (multiplexed)
- Unidirectional: client cannot push data to server over the same stream
- Load balancer: requires sticky sessions (or stateless server + Redis pub/sub)
- Proxy/firewall friendly — standard HTTP response

**Use when:** Live dashboards, news feeds, stock prices, notifications — anything where server pushes and client reads.

---

## 3. WebSockets

Full-duplex, persistent TCP connection. After an HTTP upgrade handshake, both client and server can send frames at any time without request-response overhead.

```
Client                        Server
  │                               │
  │── HTTP GET /ws ──────────────►│  (upgrade request)
  │   Upgrade: websocket          │
  │   Sec-WebSocket-Key: ...      │
  │◄── 101 Switching Protocols ──│
  │                               │
  │◄══════ WebSocket Tunnel ═════►│  (binary/text frames, bidirectional)
  │── "hello" ──────────────────►│
  │◄── "world" ─────────────────│
  │◄── "typing: alice" ─────────│
  │── "send message: hi" ───────►│
```

**Framing:** After the handshake, data is sent as lightweight frames (2–14 bytes overhead vs ~800 bytes for HTTP headers). Frames can be text or binary; large messages can be fragmented.

**WebSocket vs SSE vs Long Polling:**

| Factor | Long Polling | SSE | WebSocket |
|--------|-------------|-----|-----------|
| Direction | Server→client (pull-based) | Server→client | Bidirectional |
| Protocol | HTTP | HTTP | WS (after upgrade) |
| Connection overhead | High (per poll) | Low (one connection) | Low (one connection) |
| Latency | 0–500ms | < 100ms | < 50ms |
| Browser support | Universal | Universal (except old IE) | Universal |
| Proxy/firewall | ✅ Works everywhere | ✅ Works everywhere | ⚠️ Some proxies block WS |
| Auto-reconnect | Client must implement | ✅ Built-in (EventSource) | Client must implement |
| Missed events on reconnect | Client must track | ✅ `Last-Event-ID` | Client must track |
| HTTP/2 multiplexing | N/A | ✅ | ❌ (WS is not HTTP/2) |
| Server resource per connection | Low (short-lived) | Medium (persistent) | Medium (persistent) |
| Client→server data | New HTTP request | New HTTP request | Over same connection |

**Choose WebSocket when:** Bidirectional real-time (chat, multiplayer game, collaborative editing, live auction bidding).
**Choose SSE when:** Unidirectional server push (live dashboard, feed, notifications — client doesn't send data over same stream).
**Choose Long Polling when:** Compatibility with legacy infrastructure, or push volume is very low (< 1 event/min).

---

## 4. WebSocket at Scale — The Stateful Connection Problem

WebSocket connections are **stateful** — each client is bound to a specific server process. This breaks horizontal scaling.

```
Problem:
  Alice (connected to Server A) sends message to Bob (connected to Server B)
  Server A doesn't know Bob is on Server B → can't deliver

  Server A: {alice_ws, carol_ws, ...}
  Server B: {bob_ws, dave_ws, ...}
```

### Solution: Pub/Sub Broker (Redis Pub/Sub or Kafka)

```
Alice → Server A → Redis PUBLISH "room:123" {msg}
                           ↓
              Redis Pub/Sub broadcasts
              ↓                    ↓
         Server A             Server B
       (delivers to         (delivers to
        Alice if in         Bob via his
        room:123)           WebSocket)
```

**Implementation pattern:**
1. Client connects via WebSocket to any server (load-balanced)
2. Server maps `connection_id → user_id` in local memory
3. Server subscribes to Redis channel for each room the user is in
4. When Message for room X arrives via Redis, server finds local connections in room X and forwards frames

**Redis Pub/Sub properties:**
- At-most-once delivery — if server is down when message published, it's lost
- No persistence — for durability, use Redis Streams or Kafka

### Connection Registry

```
Redis hash: ws:user:{user_id} → {server_id, connection_id, connected_at, last_seen}

When client connects:   HSET ws:user:alice {server_id: "srv-1", conn_id: "abc"}
When message to alice:  HGET ws:user:alice → srv-1 → publish to srv-1's channel
When client disconnects: HDEL ws:user:alice
```

---

## 5. Presence Detection

"Is user X currently online?" — Appears simple, requires careful design at scale.

### Heartbeat-based presence

```
Client sends heartbeat every 30 seconds:
  WS frame: {"type": "heartbeat"}

Server on heartbeat:
  SETEX presence:user:{user_id} 60 "online"   (TTL = 2× heartbeat interval)

Server checks presence:
  EXISTS presence:user:{user_id}  → 1 (online) or 0 (offline/expired)
```

**Why TTL matters:** If client disconnects without a clean close (mobile network drop, browser tab killed), the server never receives the disconnect event. The TTL ensures the presence key expires and user appears offline within `TTL` seconds.

### Presence at Scale (WhatsApp approach)

At 500M+ active users, presence is expensive:
- Storing every user's online status: 500M Redis keys — manageable
- Broadcasting every status change to all followers: O(followers) × events = massive fan-out

**Solutions:**
1. **Lazy presence:** Only fetch presence when user opens a chat, not proactively
2. **Group presence:** Publish to a channel per conversation ("Who in this room is online?"), not per user
3. **Rate limit presence updates:** User going online/offline rapidly shouldn't flood the system — debounce 5 seconds

---

## 6. Fan-out Patterns

Fan-out = delivering one event to many recipients. Two architectures:

### Push Fan-out (Fan-out on Write)

When a message is written, immediately push to all recipients' connections.

```
Alice sends message to room with 1000 members:
  Server receives message
  → Queries: who is in this room? → 1000 user IDs
  → For each user: find their WebSocket server → publish
  → 1000 Redis PUBLISHes in parallel
```

**Pros:** Low read latency — recipients see the message immediately
**Cons:** Write amplification — a viral post with 1M followers = 1M push operations per write

### Pull Fan-out (Fan-out on Read)

Message stored once; recipients fetch when they open the view.

```
Alice sends message to room:
  Server saves message to DB once
  Server publishes "room:123:new_message" event to Redis
  → Subscribers poll/receive notification
  → Each client fetches message from DB on demand
```

**Pros:** No write amplification — one write regardless of recipient count
**Cons:** Read amplification — many clients all read the same message simultaneously

### Hybrid (WhatsApp/Slack approach)

- **Small rooms / DMs (< 1,000 members):** Push fan-out — low recipient count, fast delivery
- **Large channels / public rooms:** Pull fan-out — store once, members fetch on open
- **Celebrity/viral content:** Fan-out to active users only; inactive users use pull on next login

---

## 7. Real-World System: WhatsApp Architecture

```
Client ← WebSocket → Edge Server (Erlang/Mnesia)
                             │
                    Message Router
                           ↓
                    Message Store (Cassandra)
                           ↓
                    Recipient's Edge Server
                           ↓
                    ACK chain back to sender

Key design decisions:
- Erlang's lightweight processes: 1 process per connection → handles 2M connections per server
- Message stored in Cassandra keyed by (conversation_id, timestamp)
- End-to-end encryption: server stores ciphertext, cannot read message content
- Delivery receipts: 1 tick (sent to server), 2 ticks (delivered to device), 2 blue ticks (read)
- Offline messages: stored in Cassandra, delivered on reconnect
```

---

## 8. Real-World System: Slack Architecture

```
Client ← WebSocket → Gateway (stateful)
                          │ (Redis Pub/Sub)
                   Channel Presence
                          │
              ┌──────────┴──────────┐
         Message Store          Search Index
         (Slack DB)             (Elasticsearch)

Fan-out model:
- Workspace with 10,000 members, message in #general:
  → 10,000 WebSocket pushes needed
  → Slack uses "channels" as Pub/Sub topics
  → Servers subscribed to channel receive push → forward to connected members
  
Presence detection:
- Per-user presence stored in Redis with 30s heartbeat TTL
- Workspace-level presence subscription: "show me online status for these 50 users"
  → Only subscribe to presence for users visible in current view (lazy)
```

---

## 9. Message Ordering and Delivery Guarantees

### Message Ordering

WebSockets don't guarantee delivery or persistence. For chat:
1. Assign each message a monotonically increasing **sequence number** per conversation
2. Client detects gaps: if it receives seq=5 but last seen was seq=3, requests seq=4 explicitly
3. Server assigns seq numbers atomically (Redis INCR, PostgreSQL SERIAL, or Snowflake IDs)

### Offline Message Delivery

```
User goes offline:
  Server detects disconnect (TCP FIN or heartbeat timeout)
  Marks user as offline in presence registry

Messages arrive while offline:
  Stored in DB (Cassandra: conversation_id → [messages])

User comes back online:
  WebSocket reconnects
  Client sends: {"type": "sync", "last_seq": 142}
  Server sends: all messages with seq > 142 in bulk
  Client replays in order
```
