# WebSockets & Real-Time Communication — Interview Simulator

## Scenario 1: WhatsApp-Style Chat System

**Prompt:**
> "Design a real-time chat system for 2 billion users. Support 1-on-1 and group chats (up to 256 members). Messages must show delivery receipts (sent / delivered / read). Offline users must receive messages when they reconnect."

**Time limit:** 35 minutes

---

### Clarifying Questions You Should Ask

1. Max group size? (256 was given — important for fan-out strategy)
2. Message persistence duration? (WhatsApp: 30 days on server, deleted after delivery)
3. Media messages in scope? (If yes, adds blob storage path)
4. End-to-end encryption required? (Changes server-side capabilities)
5. Expected QPS? (2B users × avg 5 messages/day = ~115K messages/sec peak)

---

### Expected Architecture Walkthrough

**Client Connection Layer:**
- Persistent WebSocket connection per active device
- WebSocket Gateway servers: ~50K connections per server → ~40,000 gateway servers at 2B peak concurrently active 10%
- Connection registry: `HSET ws:user:{user_id} server_id {id} conn_id {id}` in Redis cluster

**Message Flow (1-on-1):**
```
Sender ──WebSocket──► Gateway A
Gateway A ──► Chat Service (validates, assigns msg_id + seq)
Chat Service ──► Cassandra (persist message, status=sent)
Chat Service ──► Gateway A: ACK to sender (1 tick)
Chat Service ──► connection registry: find recipient's gateway
Chat Service ──► Redis Pub/Sub ──► Gateway B (recipient's server)
Gateway B ──► Recipient WebSocket: deliver
Recipient ──► Gateway B: delivered ACK
Gateway B ──► Chat Service ──► update status=delivered
Chat Service ──► Gateway A ──► Sender: 2 ticks
```

**Database Choice:**
- Messages: Cassandra — `(conversation_id, seq)` as partition + clustering key. Write-heavy, time-series access pattern, no joins needed.
- User metadata: PostgreSQL
- Media: S3 + CDN

**Offline Delivery:**
- Messages persisted to Cassandra with `status=pending`
- On reconnect: client sends `{"type": "sync", "conversation_id": "...", "last_seq": N}`
- Server queries: `SELECT * FROM messages WHERE conversation_id=? AND seq > ? LIMIT 100`

**Group Fan-out (256 members):**
- At 256 members, iterate the member list and push individually — not a full fan-out problem at this scale
- Fetch group members from Redis cache (TTL 5min), deliver to each active member's gateway

**Delivery Receipt State Machine:**
```
sent (persisted) → delivered (device ACK) → read (user opens chat)
```

---

### Key Numbers

| Metric | Value |
|---|---|
| Peak concurrent connections | ~200M |
| Gateway servers needed | ~4,000 (50K conn each) |
| Messages/sec | ~120K |
| Cassandra write throughput | ~120K writes/sec |
| Redis Pub/Sub channels | 1 per gateway server |

---

### Follow-up Questions

**"What if a user has 3 devices?"**
Each device has its own WebSocket connection pointing to the connection registry. Message delivered to all devices independently. Read receipt requires all devices to send read ACK; the "read" tick shows when any device opens the message.

**"How do you handle the 30-day message retention without storing forever?"**
Cassandra TTL: `INSERT INTO messages (...) USING TTL 2592000` (30 days in seconds). On delivery to recipient, delete early or let TTL expire naturally.

**"Why Cassandra over PostgreSQL for messages?"**
Write throughput (120K/sec), linear scalability, no single leader bottleneck. Read pattern is always time-ordered per conversation — matches Cassandra's partition + clustering key model.

---

## Scenario 2: Slack-Style Workspace Messaging

**Prompt:**
> "Design a team messaging platform (Slack-like). Features: channels with up to 100,000 members, presence indicators, typing indicators, edit/delete message history (7 years), search."

**Time limit:** 35 minutes

---

### Clarifying Questions You Should Ask

1. How many workspaces / total users? (Slack: 20M DAU)
2. Is search full-text or just recent messages? (Affects index strategy)
3. Real-time presence for all channel members or just online ones?
4. Per-workspace data isolation required? (Multi-tenancy model)

---

### Expected Architecture Walkthrough

**Connection Layer:**
- WebSocket Gateway (one persistent connection per client)
- On join: `SUBSCRIBE room:{channel_id}` via Redis Pub/Sub — but only for channels the user has open (lazy subscription)

**Fan-out Strategy (channel with 100K members):**
- Small channels (≤5K members): Redis Pub/Sub push — deliver to all online members' gateways
- Large channels (>5K members): Pull model — publish message to channel; members fetch when they open the channel
- Threshold decision: monitor read amplification vs write amplification cost

**Presence at 20M DAU:**
- Heartbeat every 30s → `SETEX presence:online:{user_id} 65 1`
- Typing indicators: `SETEX typing:{channel_id}:{user_id} 5 1`
- Grace period on disconnect: broadcast "offline" only after 10s without reconnect
- Lazy subscription: `SUBSCRIBE presence:changes:{workspace_id}` only for workspaces the user has open

**Message Storage:**
- 7-year retention with edit/delete: PostgreSQL with `messages` table + `message_edits` audit table
- Soft delete: `deleted_at TIMESTAMP NULL` — never hard-delete for compliance
- Message edits: `INSERT INTO message_edits (msg_id, content, edited_at)` + update `messages.content`
- Partition by `(workspace_id, channel_id)` → separate table per large workspace (Slack uses tenant-isolated DB shards)

**Search:**
- Elasticsearch cluster: index `{workspace_id, channel_id, content, author, ts}`
- Write path: async indexing via Kafka consumer (don't block message send on index write)
- Query: `workspace_id` as a routing key ensures search stays within tenant

**Typing Indicator:**
- Client sends `{type: "typing"}` on each keypress (debounced to 1 per 500ms client-side)
- Server: `SETEX typing:{channel_id}:{user_id} 5 1` — expires automatically
- `GET /channels/{id}/typing` → Redis `SCAN typing:{channel_id}:*` → list of usernames

---

### Follow-up Questions

**"How do you prevent a thundering herd when 10,000 members open a Slack channel simultaneously after an outage?"**
Request coalescing at the channel-level cache: first reader fetches from DB, caches the recent messages list (last 50 messages) for 10s. Subsequent readers hit the cache. After 10s, cache is warm and DB pressure drops.

**"How do you handle timezone-aware "last seen" in presence?"**
The `last_seen` timestamp is stored in UTC. Client-side formatting converts to local timezone using the client's offset. Never store timezone-local times on the server.

**"A channel has 100K members. One message triggers 100K push notifications. How do you not melt APNs?"**
APNs has per-app rate limits (~1M/sec total, but shared). Batch and coalesce notifications: if a user hasn't opened the app in >1 hour, send one "X new messages in Y channels" notification, not one per message. Use APNs priority 5 (low) for background channels vs priority 10 (immediate) for direct mentions.

---

## Scenario 3: Live Sports Score Dashboard

**Prompt:**
> "Design a live sports score system. 50 million concurrent viewers watching the same 10 active matches. Scores update every 5–30 seconds. Minimize latency from score event to user display."

**Time limit:** 25 minutes

---

### Clarifying Questions You Should Ask

1. Is data source real-time (manual entry, official feed)? (Kafka or webhook from data provider)
2. Tolerance for slight staleness? (5–10s acceptable → SSE over WebSocket)
3. Mobile and web clients? (SSE works on both; WebSocket adds complexity for no benefit here — read-only)
4. Personalisation needed? (If no → same payload for all viewers of the same match → ideal for CDN)

---

### Expected Architecture Walkthrough

**Observation**: 50M concurrent users all reading the same 10 matches. This is the **fan-out read** problem — identical data being delivered to millions. The answer is not 50M WebSocket connections to origin servers.

**Transport choice: SSE (not WebSocket)**
- Unidirectional (server → client only) — WebSocket's bidirectionality adds no value here
- SSE auto-reconnects, supports `Last-Event-ID` replay
- Works with HTTP/2 multiplexing and CDN edge caching

**Architecture:**

```
Official Score Feed (Kafka / webhook)
  ↓
Score Ingestion Service
  ↓
Publishes to Redis Pub/Sub: "match:{match_id}:scores"
  ↓
SSE Edge Workers (at CDN PoPs — e.g., Cloudflare Workers)
  ↓  (subscribe permanently to Redis channel per active match)
50M SSE clients (connected to nearest CDN edge)
```

**Key insight**: CDN edge workers hold a single Redis subscription per match at each PoP. One Redis publish → CDN PoP → fans out locally to all connected clients at that edge location. Total Redis fan-out = number of PoPs (hundreds), not number of users.

**Score update flow:**
```
Score event → Kafka topic "score_updates"
Score Service → consumes → publishes to Redis Pub/Sub "match:123:scores"
  AND → persists to "current_state" Redis key: SET match:123:state {json} EX 3600
  AND → appends to Redis Stream: XADD match:123:history * score {json}

CDN Worker (subscribed to "match:123:scores"):
  → receives update
  → formats as SSE: "id: {event_id}\nevent: score\ndata: {json}\n\n"
  → broadcasts to all local SSE connections for this match

New client connects (or reconnects with Last-Event-ID):
  → GET match:123:state (instant current score, cached at edge)
  → Then subscribe to SSE stream for live updates
```

**Race condition on connect:**
New client gets current state from cache, then subscribes to SSE. Gap between the two steps could miss an update. Fix: subscribe before reading cache, apply deduplification if any event arrives with seq ≤ cached seq.

**Scale numbers:**
| Component | Count |
|---|---|
| Active matches | 10 |
| CDN PoPs | 300 |
| Redis subscriptions | 10 × 300 = 3,000 |
| Origin SSE servers | ~100 (origin connection to Redis only) |
| Users per PoP | ~50M / 300 ≈ 167K per PoP |

---

### Follow-up Questions

**"What if a match has a sudden burst — a goal scored at 90+4 minutes — and 1M users are refreshing simultaneously?"**
Cache stampede risk on the current state. Solution: Redis `SET match:123:state {json} EX 3600` is a single atomic write. Reads go to Redis or edge-cached copy. The CDN SSE worker doesn't need to call origin for each new connection — it has the state in memory from the last update. No stampede possible if state is cached at the edge.

**"Why not WebSocket here?"**
WebSocket is a stateful persistent connection that requires server-side connection management, heartbeats, reconnect logic, and cannot be cached at the CDN edge. SSE is just an HTTP response with `Transfer-Encoding: chunked` — standard HTTP that CDN infrastructure was built for.

**"How do you handle a match going to extra time past the scheduled end? Your TTL expires."**
The ingestion service owns the lifecycle. On extra time event, it extends the Redis key TTL and publishes a `match_extended` SSE event. Clients continue receiving. On final whistle, publish `match_ended` event and allow clients to close the connection. `EventSource.close()` on the `match_ended` event type in JavaScript.
