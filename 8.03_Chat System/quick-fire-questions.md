# Chat System — Quick-Fire Questions

**Q1: Why WebSocket over HTTP long polling for a chat system?**
WebSocket is a persistent bidirectional TCP connection. Once established, frames are sent in both directions with minimal overhead (~2 byte header vs HTTP's 200+ byte headers). Long polling simulates server push but requires a new HTTP request for each message delivery — high overhead and latency. WebSocket supports typing indicators, delivery receipts, and messages simultaneously on one connection.

---

**Q2: How do you route a message from User A (on Server 1) to User B (on Server 7)?**
Presence service (Redis) maps each user_id to the server they're connected to. Server 1 looks up User B → "server-7" → publishes message to a cross-server channel (Redis Pub/Sub or Kafka). Server 7 subscribes and delivers to User B's WebSocket. If User B is offline, message is queued for push notification.

---

**Q3: How do you handle message delivery when the recipient is offline?**
Store the message in the database (Cassandra). Queue a push notification via APNs/FCM to wake the user's device. When the user opens the app, they reconnect via WebSocket and fetch undelivered messages from the DB. The client ACKs each message after display — server marks as "delivered."

---

**Q4: Why use Cassandra for message storage over PostgreSQL?**
Chat messages are append-only, high-volume (1M/sec), queried by conversation in time-range (last 50 messages). Cassandra's LSM tree handles append-only writes efficiently; partitioning by conversation_id co-locates messages for fast range scans. PostgreSQL's B-tree index requires random writes and doesn't scale to 1M writes/sec without massive hardware.

---

**Q5: How do you prevent duplicate message delivery?**
Clients generate a `client_msg_id` (UUID4) before sending. Server stores this in Cassandra with a UNIQUE constraint. If the client retries (network timeout), the second INSERT is rejected by the unique constraint — message is not saved twice. Client matches `client_msg_id` in the server's ACK to know the message was accepted.

---

**Q6: How does the single-tick (✓) vs double-tick (✓✓) work?**
Single tick: server received and persisted the message, sends ACK to sender. Double gray tick: recipient's device received the message (recipient's app sends a "delivered" event back to server). Double blue tick: recipient opened the conversation (recipient's app sends a "read" event).

---

**Q7: How do you scale to 350M concurrent WebSocket connections?**
Each chat server handles ~100K connections. 350M ÷ 100K = 3,500 servers. Connections are load-balanced at the gateway layer (sticky sessions based on user_id hash). Chat servers are stateless except for in-memory WebSocket state — they read/write to Redis and Kafka.

---

**Q8: What's the challenge with group message fan-out at scale?**
A message to a 1,000-member group requires delivering to 1,000 connections potentially across hundreds of servers. Naive synchronous fan-out from the sender's server adds latency proportional to group size. Solution: publish one message to Kafka; a fan-out service asynchronously dispatches to each member's server. This decouples the sender's write path from delivery.

---

**Q9: How do you implement "typing..." indicators?**
Client sends a `{type: "typing", to: "user_id"}` WebSocket frame. Server routes to recipient's server which pushes to recipient's WebSocket. Typing events are ephemeral — NOT stored in DB. If recipient is offline, typing indicator is dropped (no point in notifying). Client stops sending after 5 seconds of inactivity.

---

**Q10: How does end-to-end encryption work architecturally?**
Each client generates a public/private key pair. Public keys are stored on the server. When A wants to send to B: A fetches B's public key → encrypts message locally → sends ciphertext to server → server stores ciphertext (cannot read it) → server delivers ciphertext to B → B decrypts with private key. Server never has the plaintext. Uses Signal Protocol (Double Ratchet) for forward secrecy.

---

**Q11: How do you handle message ordering in group chats?**
Each message gets a Snowflake ID (time-ordered) at the server. Clients sort messages by Snowflake ID. Minor clock skew across servers is acceptable — Snowflake embeds the server's timestamp so messages from different servers sort correctly as long as clocks are within ~1 second (NTP synchronized).

---

**Q12: How do you implement message search?**
Full-text search of messages requires an inverted index. Options: Elasticsearch (index message content, recipient_id) or Cassandra SASI indexes (limited). For E2EE chats, search must happen client-side (server has ciphertext, can't index). For non-encrypted chats: stream messages to Elasticsearch via Kafka consumer; support per-conversation search.
