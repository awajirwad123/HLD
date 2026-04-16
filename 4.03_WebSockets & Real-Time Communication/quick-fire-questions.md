# WebSockets & Real-Time Communication — Quick-Fire Q&A

## Protocol & Fundamentals

**Q1. What HTTP status code starts a WebSocket connection?**
101 Switching Protocols. The server responds to the `Upgrade: websocket` request, and the TCP connection is then used as a raw WebSocket channel.

**Q2. What is the minimum WebSocket frame header size?**
2 bytes. Compare to ~800 bytes for a typical HTTP request header.

**Q3. What makes SSE reconnect automatically, and what does the browser re-send?**
The `EventSource` API reconnects automatically after a disconnect. It re-sends the `Last-Event-ID` header with the ID of the last received event, allowing the server to replay missed events.

**Q4. What is the `text/event-stream` MIME type used for?**
Server-Sent Events. It signals to the browser that the response body is an open stream of SSE events, not a complete document.

**Q5. Why do SSE and WebSocket connections time out behind proxies?**
Load balancers and proxies have idle connection timeouts (typically 60s–5min). If no bytes are sent, the proxy closes the connection. Heartbeats (empty frames or keep-alive comments) prevent this.

---

## Scaling & Architecture

**Q6. If Server A holds Alice's WebSocket connection and Server B receives a message for Alice, how does Server B deliver it?**
Via Redis Pub/Sub. Server B publishes the message to the channel for Alice's room. Server A (subscribed to that channel) receives it and forwards to Alice over her WebSocket connection.

**Q7. What information belongs in a WebSocket connection registry?**
At minimum: `user_id → server_id` mapping, so any server can find which server holds a user's connection. Often stored in Redis as `HSET ws:user:{user_id} server_id {srv} conn_id {id}`.

**Q8. What happens to a Redis Pub/Sub message if no server is actively subscribed?**
It is dropped. Redis Pub/Sub has no persistence or replay. Use Redis Streams (`XADD`/`XREAD`) for at-least-once delivery.

**Q9. How do you ensure horizontal scalability for WebSocket servers?**
1. Use Redis Pub/Sub (or similar) for cross-server message routing.
2. Keep a connection registry so any server can route to the correct instance.
3. Put a load balancer with sticky sessions (or no stickiness if using Redis routing) in front.

**Q10. How many WebSocket connections can a single server handle?**
Node.js / Go: ~100K. Erlang (BEAM): ~2 million per server. Bottlenecks are file descriptors, memory (~10–50KB per connection), and CPU for heartbeat processing.

---

## Presence

**Q11. What is the heartbeat TTL formula for presence?**
`PRESENCE_TTL = 2 × heartbeat_interval + small_buffer`
E.g., heartbeat every 30s → TTL = 65s. Allows one missed heartbeat before declaring the user offline.

**Q12. Why store `last_seen` in a separate non-expiring key?**
The expiring `presence:online:{id}` key disappears when the user goes offline. The non-expiring `last_seen` timestamp allows rendering "Last seen 2 hours ago" even after the user disconnects.

**Q13. How do you check presence for 200 users at once without N Redis round-trips?**
Redis pipeline: batch 200 `EXISTS` commands in a single pipeline call. One round-trip, 200 responses.

**Q14. Why is eager presence fan-out impractical at 500 million users?**
If each user going online/offline triggers a notification to all followers, a celebrity with 50M followers would cause 50M writes per status change. Instead, use lazy presence: check status only when a conversation is opened.

---

## SSE vs WebSocket

**Q15. When should you choose SSE over WebSocket?**
When communication is strictly one-directional (server → client): live dashboards, news feeds, order status updates, social activity streams. SSE is simpler, auto-reconnects, and works natively over HTTP/2.

**Q16. Can SSE work over HTTP/2?**
Yes, and better than WebSocket in this case — HTTP/2 multiplexes many SSE streams over a single TCP connection, avoiding the browser's per-domain connection limit (6 for HTTP/1.1).

**Q17. Name one use case where WebSocket is required over SSE.**
Any bidirectional low-latency scenario: chat applications, multiplayer gaming, collaborative document editing (e.g., Google Docs), live trading terminals.

---

## Message Delivery & Ordering

**Q18. Why use server-assigned sequence numbers instead of timestamps for message ordering?**
Clocks on distributed servers can skew. A server-assigned monotonic sequence number per room is the single authoritative ordering. Timestamps are unreliable for strict ordering.

**Q19. What is the gap detection protocol on reconnect?**
Client sends `{"type": "sync", "last_seq": N}`. Server responds with all messages with seq > N for that room. Client processes them in order and updates its local last_seq.

**Q20. What is the "at-least-once delivery" challenge for offline users in WebSocket systems?**
When a user is offline, messages can't be pushed. Solution: persist messages to a database as they arrive, mark delivery status (sent/delivered/read). On reconnect, client sends its `last_seq`; server replays undelivered messages. This separates the transport (WebSocket push) from the storage (persistent DB).
