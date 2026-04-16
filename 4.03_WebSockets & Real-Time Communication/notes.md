# WebSockets & Real-Time Communication — Notes & Cheat Sheet

## 1. Transport Comparison

| Dimension | Long Polling | SSE | WebSocket |
|---|---|---|---|
| Direction | Half-duplex (client pulls) | Server → Client only | Full-duplex |
| Protocol | HTTP/1.1 | HTTP/1.1 | WS (upgraded from HTTP) |
| Connection | New conn per poll | Persistent | Persistent |
| Frame overhead | ~800 bytes (HTTP headers) | ~100 bytes | **2–14 bytes** |
| Latency | 0–500ms | ~0ms | ~0ms |
| Auto-reconnect | Client must re-implement | **Yes (built-in)** | Must implement |
| `Last-Event-ID` replay | No | **Yes (built-in)** | No |
| Proxy-friendly | Yes | Yes | Sometimes (CONNECT tunneling) |
| HTTP/2 multiplexing | Yes | **Yes** | No (separate TCP) |
| Use cases | Legacy browsers, fallback | Dashboards, feeds, notifications | Chat, gaming, collaboration |

**Decision rule:**
- Server pushes only + need replay → **SSE**
- Bidirectional low-latency → **WebSocket**
- Legacy fallback / simple → **Long Polling**

---

## 2. WebSocket Handshake

```
Client → Server:
  GET /chat HTTP/1.1
  Host: example.com
  Connection: Upgrade
  Upgrade: websocket
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13

Server → Client:
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- `Sec-WebSocket-Accept` = Base64(SHA-1(`Sec-WebSocket-Key` + magic GUID))
- After 101, the TCP connection is "hijacked" — no more HTTP

**Frame format (minimum 2 bytes):**
```
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+-+-+-+-+-------+-+-------------+
|F|R|R|R| opcode|M| Payload len |  + optional extended length + masking key
|I|S|S|S|  (4)  |A|    (7)      |
|N|V|V|V|       |S|             |
```

---

## 3. SSE Wire Format

```
id: 42\n
event: metrics\n
data: {"cpu": 45.2, "ts": 1700000000}\n
\n                          ← blank line = end of event
```

- `id:` → browser stores as `lastEventId`, sends as `Last-Event-ID` header on reconnect
- `event:` → maps to `addEventListener('metrics', handler)` in browser
- `data:` → can span multiple lines (each prefixed with `data:`)
- **Retry field:** `retry: 3000\n` → tells client to wait 3s before reconnecting

**Server headers required:**
```
Content-Type: text/event-stream
Cache-Control: no-cache
X-Accel-Buffering: no    ← Nginx specific
```

---

## 4. Redis Pub/Sub for Horizontal Scaling

**The problem:** WebSocket connections are stateful and pinned to one server. Message from Server A must reach a client connected to Server B.

**Solution pattern:**
```
Client A (Server 1) → publishes to Redis channel "room:123"
Redis → delivers to all subscribers
Server 2 (subscribed to "room:123") → broadcasts to its local connections
```

**Key Redis commands:**
```redis
SUBSCRIBE   room:123                       # Subscribe to channel
PUBLISH     room:123  <json_payload>       # Broadcast to all subscribers
PSUBSCRIBE  room:*                         # Subscribe to pattern

# Connection registry
HSET ws:user:alice  server_id "srv-1"  conn_id "conn-xyz"
HGETALL ws:user:alice
HDEL ws:user:alice server_id

# Presence
SETEX presence:online:alice 65 1          # Online — expires after 65s
GET   presence:online:alice               # NULL = offline
TTL   presence:online:alice               # Check remaining TTL
```

**Pub/Sub delivery guarantees:**
- Fire-and-forget — no persistence, no replay
- If no subscriber active when published → message **lost**
- Use Redis Streams (`XADD` / `XREAD`) when you need replay/at-least-once

---

## 5. Presence Detection — Heartbeat TTL Pattern

```
Client                       Server                    Redis
  |--- heartbeat every 30s --->|                          |
  |                            |--- SETEX presence 65 --->|
  |<-- heartbeat_ack ----------|                          |
  |                            |                          |
  |  [client disconnects]      |                          |
  |                            |                          | [TTL expires after 65s]
                                                          | key gone = offline
```

**Formula:** `PRESENCE_TTL = (2 × heartbeat_interval) + small_buffer`

- `heartbeat_interval = 30s` → `PRESENCE_TTL = 65s`
- Gives one missed heartbeat before declaring offline
- `last_seen` timestamp stored separately (non-expiring hash) for "Last seen 2h ago"

**Presence at 500 million users:**
- Don't fan-out presence changes to all followers eagerly
- **Lazy presence:** only check presence of users that are currently visible on screen
- **Coalesce:** batch presence checks (one Redis pipeline for 50 users at once)
- Subscribe to presence changes only for open conversations

---

## 6. Fan-Out Patterns

| Pattern | Mechanism | Write cost | Read cost | Best for |
|---|---|---|---|---|
| Push (fan-out on write) | Publish to each recipient's inbox | O(n) per message | O(1) for read | Small rooms, low follower count |
| Pull (fan-out on read) | Store once, readers fetch latest | O(1) per message | O(n) per read | Celebrity accounts, large channels |
| Hybrid | Push for ≤10K followers, pull for >10K | Mixed | Mixed | Twitter, Slack |

**Slack hybrid decision:**
- Channel with ≤ 5,000 members: push to connected users via Redis Pub/Sub
- Channel with > 5,000 members: pull model — messages fetched when user opens channel
- Subscription laziness: only subscribe to a Redis channel when a user has the channel open

---

## 7. Delivery Receipts (WhatsApp Model)

```
1 tick  ✓  → server received (server ACK of persistence)
2 ticks ✓✓ → delivered to recipient device (device ACK)
Blue    ✓✓ → read by recipient (read receipt)
```

**Implementation:**
```
sender → server: {type:"message", id:"msg-123", content:"..."}
server → DB: persist message, mark status="sent"
server → sender: {type:"ack", msg_id:"msg-123", status:"sent"}    ← 1 tick

server → recipient: {type:"message", id:"msg-123", ...}
recipient → server: {type:"delivered", msg_id:"msg-123"}           ← 2 ticks
server → sender:   {type:"status_update", msg_id:"msg-123", status:"delivered"}

recipient app opens message:
recipient → server: {type:"read", msg_id:"msg-123"}                ← blue ticks
```

---

## 8. Message Ordering and Gap Detection

```python
# Client-side state machine
last_seq = 0

def on_message(msg):
    if msg["seq"] == last_seq + 1:
        process(msg)
        last_seq = msg["seq"]
    elif msg["seq"] > last_seq + 1:
        # Gap detected — fetch missing messages
        fetch_history(after_seq=last_seq, before_seq=msg["seq"])
    # else: duplicate — discard

# On reconnect: sync protocol
ws.send({"type": "sync", "last_seq": last_seq})
# Server responds with all messages after last_seq
```

**Key rules:**
- Monotonic sequence numbers per room/conversation (not global)
- Store seq in the message, not derived from timestamp (clocks skew)
- Server assigns final seq on persist — client's draft seq is provisional

---

## 9. Reconnection Strategy (Exponential Backoff)

```python
import asyncio, websockets, random

async def connect_with_retry(url):
    backoff = 1.0
    while True:
        try:
            async with websockets.connect(url) as ws:
                backoff = 1.0          # Reset on success
                await handle_messages(ws)
        except Exception:
            # Jitter: backoff × (0.5 to 1.5)
            sleep_for = backoff * (0.5 + random.random())
            await asyncio.sleep(min(sleep_for, 30))  # Cap at 30s
            backoff = min(backoff * 2, 30)
```

**Reconnect checklist:**
1. Exponential backoff with jitter (prevent thundering herd)
2. Cap max delay (30s is typical)
3. On reconnect: send `sync` request with `last_seq` to recover missed messages
4. Re-announce presence on reconnect

---

## 10. Key Numbers to Memorise

| Metric | Value | Context |
|---|---|---|
| WebSocket frame overhead | 2–14 bytes | vs 800 bytes for HTTP headers |
| Erlang (WhatsApp) connections/server | ~2 million | Due to BEAM lightweight processes |
| Typical Node.js WebSocket connections/server | ~100K | With tuned file descriptors |
| Heartbeat interval recommendation | 25–30s | Below proxy idle timeout (~60s) |
| Presence TTL | 2× heartbeat + buffer | e.g., 30s → 65s TTL |
| Redis Pub/Sub latency | <1ms local | Sub-millisecond on same network |
| Long poll timeout | 20–30s | Shorter than proxy/load balancer timeout |
| SSE reconnect default delay | 3 seconds | Browser default, overridable via `retry:` |

---

## 11. Scaling Checklist

- [ ] Connection pinning: sticky sessions or Redis Pub/Sub fan-out
- [ ] Heartbeat mechanism to detect zombie connections
- [ ] Presence TTL = 2× heartbeat interval
- [ ] Reconnect with backoff + jitter on client
- [ ] Message ordering: server-assigned sequence numbers
- [ ] Offline delivery: sync endpoint returns messages since `last_seq`
- [ ] Rate limiting per connection to prevent abuse
- [ ] Proxy timeouts: heartbeat interval < proxy idle timeout
- [ ] DLQ / retry for messages that can't be delivered (offline users → store in DB)

---

## 12. Interview Key Phrases

- *"WebSocket is stateful — connections are pinned to one server; Redis Pub/Sub decouples fan-out from connection locality."*
- *"SSE gives you free auto-reconnect and last-event-ID replay from the browser — no client code needed."*
- *"Presence at scale is lazy: subscribe only to users currently visible, check in bulk via Redis pipeline."*
- *"Fan-out hybridises around a threshold — push for small rooms, pull for celebrity/large-channel scenarios."*
- *"The heartbeat TTL pattern means I don't need an explicit disconnect event: if the TTL expires, the user is offline."*
- *"Message ordering uses server-assigned monotonic sequence numbers per room, not timestamps."*
