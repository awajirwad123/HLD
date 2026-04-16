# Chat System — Tricky Interview Questions

## Q1: "A user sends a message, gets a ✓ (server ACK), but the recipient never receives it. How do you investigate and fix this?"

**The investigation path:**

1. Check Kafka: Is the message in the `messages` topic? (Kafka consumer group lag metrics)
2. Check Cassandra: Is the message persisted? (`SELECT * FROM messages WHERE conversation_id=X AND message_id=Y`)
3. Check the fan-out service: Did it publish to the recipient's server's delivery channel?
4. Check presence: Was the recipient online or offline at delivery time?
5. If offline: Was a push notification queued and sent?
6. Check the recipient's device: APNs/FCM delivery logs

**Root cause in this scenario:** The fan-out service published to `deliver:server-7` (Redis Pub/Sub), but Server 7 had just crashed. The Pub/Sub message was lost — Redis Pub/Sub is fire-and-forget with no persistence.

**Fix:** Don't use Redis Pub/Sub for cross-server delivery. Use **Kafka** with consumer groups. Each chat server is a consumer in its own group. Kafka retains messages until the server ACKs them. Server 7 restarts → resumes consuming from last committed offset → delivers buffered messages.

**Or:** Use a direct DB pull model. When User B's new connection arrives at Server 12, Server 12 queries Cassandra for all messages received while User B was offline. No cross-server push needed — pull on reconnect is simpler and more reliable.

---

## Q2: "How do you prevent a single hot Cassandra partition from becoming a bottleneck for a very active group chat (10M messages/month)?"

**The problem:** All messages for group G are stored in partition `conversation_id = G`. At 10M messages/month, this partition grows large. Write hotspot on one Cassandra node.

**Solutions:**

1. **Time-bucket partitioning (Composite partition key):**
```sql
CREATE TABLE messages (
    conversation_id  TEXT,
    bucket           TEXT,        -- e.g., "2026-04" (year-month)
    message_id       TIMEUUID,
    ...
    PRIMARY KEY ((conversation_id, bucket), message_id)
)
```
Partition key = `(conversation_id, bucket)`. Writes in April 2026 go to `(G, "2026-04")` partition. Large group chat no longer has a single hot partition. Query modification: fetch by month, then merge (or always start from current month and page backward).

2. **Separate high-traffic conversations to dedicated Cassandra keyspace/cluster.** Identify top 1% conversations by message volume; route their writes to a hot-data Cassandra ring while regular conversations use the standard ring.

3. **Write amplification trade-off:** For the top 100 group chats globally, consider writing to a dedicated Redis Stream per conversation (fast append-only, max 7-day retention) alongside Cassandra (long-term). Real-time delivery always from Redis Stream; historical queries from Cassandra.

**The insight they're testing:** You should know that Cassandra partitions have practical size limits (~100MB–1GB) and hot partitions cause write amplification and read latency on the owning node.

---

## Q3: "How do you implement message reactions ('❤️' on a message) efficiently?"

**Naïve approach:** Store a reaction as a new message row. Problem: for a message with 10,000 emoji reactions, the conversation history becomes polluted with reaction events, and counting reactions requires scanning many rows.

**Better design: Separate reactions table**

```sql
CREATE TABLE message_reactions (
    message_id  TIMEUUID,
    user_id     TEXT,
    emoji       TEXT,
    PRIMARY KEY (message_id, user_id)   -- One reaction per user per message
)
```

Query all reactions for a message: `SELECT emoji, count(*) FROM message_reactions WHERE message_id = X GROUP BY emoji` — but Cassandra GROUP BY is limited.

**Better with Redis:**

```redis
ZADD reactions:{message_id}:{emoji} 1 user_id   # Track who reacted
HINCRBY reaction_counts:{message_id} ❤️ 1       # Aggregate count
```

Read path: `HGETALL reaction_counts:{message_id}` → O(1). When user removes reaction: `HINCRBY reaction_counts:{message_id} ❤️ -1`.

For reactions display (who reacted): `ZRANGE reactions:{message_id}:❤️ 0 9` (first 10 users).

**Delivery of reaction events:** React event is published via WebSocket to all conversation participants (like a mini-message). Since reactions are ephemeral enough, fail-safe delivery (Redis Pub/Sub) with TTL-based reconciliation from DB is acceptable.

---

## Q4: "The design stores messages in Cassandra. How do you implement 'last message preview' in the conversation list without reading the full message table?"

**Problem:** On app open, user sees their conversation list with the last message preview for each conversation. Querying Cassandra for `SELECT * FROM messages WHERE conversation_id = X ORDER BY message_id DESC LIMIT 1` for each of 100 conversations = 100 Cassandra reads. Expensive at scale.

**Solution: Denormalize `last_message` into a separate table**

```sql
CREATE TABLE conversation_meta (
    user_id            TEXT,
    conversation_id    UUID,
    last_message_ts    TIMESTAMP,
    last_message_text  TEXT,
    last_sender_id     TEXT,
    unread_count       INT,
    PRIMARY KEY (user_id, last_message_ts)
) WITH CLUSTERING ORDER BY (last_message_ts DESC);
```

On every new message:
1. Write to `messages` table (durable storage)
2. Write to `conversation_meta` for ALL participants (denormalized update)

User opens app → single query: `SELECT * FROM conversation_meta WHERE user_id = X LIMIT 50` → conversation list with previews in one read.

**Trade-off:** Every message requires N writes (N = conversation members) to `conversation_meta` in addition to the `messages` write. For group chats with 1,000 members, one message → 1,000 meta updates. This is the fan-out on write trade-off for fast reads.

**Or (simpler for small scale):** Cache last message per conversation in Redis: `SET last_msg:{conv_id} {preview_json} EX 3600`. Conversation list reads from Redis; stale by up to 1 hour which is acceptable for the preview.

---

## Q5: "Users complain that message order is wrong — messages appear out of order on the screen. How do you fix this?"

**Root cause analysis:**

1. **Client-side ordering by receive time.** Client displays messages in the order they're received over the WebSocket. Network jitter means message B (sent after A) arrives before A. Fix: use `server_message_id` (Snowflake) for display ordering, not arrival time.

2. **Clock skew between chat servers.** Two messages sent 100ms apart from different servers may get Snowflake IDs with inverted order if Server 2's clock is 200ms behind Server 1. Fix: NTP sync + NTP drift monitoring alert. Also: use local sequence number per conversation (Redis INCR) as a fine-grained tiebreaker.

3. **Client timestamp used.** App displays client-generated timestamps before the server assigns message IDs. User A's phone clock is wrong. Fix: display message order by server-assigned ID, not client timestamp. Show client time for user convenience but sort by server ID.

4. **Duplicate message IDs.** Two messages both get ID X (bug in Snowflake config — duplicate machine_id). Fix: machine_id must be unique per server instance, assigned from a coordinated source (ZooKeeper, Consul) at startup.

**The production fix:** Client maintains a local sort buffer. After receiving messages from WebSocket, sort by Snowflake message_id before rendering. On pagination (scrolling up), merge incoming historical messages into the same sorted order.
