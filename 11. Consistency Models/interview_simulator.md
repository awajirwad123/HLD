# Consistency Models — Interview Simulator

**Instructions:** Set a 35-minute timer. Attempt each scenario without looking at the guidance. A senior interviewer would probe for model selection justification, failure mode awareness, and quantified trade-offs.

---

## Scenario 1: Collaborative Document Editing (Google Docs Style)

**Prompt:**
> "Design the consistency layer for a collaborative document editor. Multiple users can edit the same document simultaneously. Users on the same document must see each other's edits in a consistent order — no one should see a reply to a comment before the comment itself. The system must work globally with users in different regions. What consistency model do you apply, and what are the trade-offs?"

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "Two users simultaneously delete the same paragraph. What happens?"
2. "User A in Sydney types a word at t=100ms. User B in London types at t=110ms. How do you order these?"
3. "What is the replication lag you'd tolerate before the UX degrades?"
4. "Why not just linearizability for everything?"

---

### Guidance (Read After Attempting)

**Model Required:** Causal consistency with CRDT-based conflict resolution.

**Core reasoning:**
- Comments and replies require causal ordering — this rules out pure eventual consistency.
- Text edits are concurrent by nature — this rules out linearizability (too slow for real-time collab).
- Causal consistency gives: "if you see my reply, you've seen my comment" without requiring global coordination.

**Architecture:**

```
User Browser → WebSocket → Regional Server (AWS ap-southeast-2 / eu-west-1 / us-east-1)
                ↓
        Regional Server applies edit to local CRDT state
                ↓
        Async causal broadcast to other regions (vector clock tagged)
                ↓
        Each client merges incoming ops with local state
```

**CRDT Choice:**
- **For text:** LOGOOT or YATA (used by Y.js) — assigns a globally unique position to each character so concurrent inserts never conflict.
- **For comments:** Causal consistency with a happens-before chain. A reply carries the logical clock of the parent comment. A replica delivers the reply only after delivering the parent.

**Conflict handling:**
- Concurrent deletes on the same paragraph: LWW on the paragraph-level operation (last timestamp wins) OR surface conflict to both users for resolution. Most real systems (Google Docs) use OT (Operational Transformation) or CRDTs that silently converge.

**Latency trade-off:**
- Causal broadcast adds 50–150ms cross-region vs 0ms for eventual. For typing, this means a user in London might see a Sydney user's keystrokes 150ms late — imperceptible.
- Linearizability would require a round-trip to a global leader for every keystroke: 200–300ms round-trip → types lag behind by 3–5 seconds. Unacceptable.

**Common mistakes:**
- Saying "use CRDTs" without explaining why causal consistency is the right model.
- Asking for linearizability and not acknowledging the 200ms penalty.
- Neglecting to handle the "reply before post" ordering problem.

---

## Scenario 2: E-Commerce Checkout — Stale Inventory

**Prompt:**
> "You're the tech lead for an e-commerce platform doing 50,000 orders per minute during a flash sale. The inventory service uses Cassandra. During the last sale, 2,000 customers checked out for items that were actually out of stock — the inventory showed '1 unit available' due to a stale read. How do you redesign the consistency model for inventory reads and writes to prevent this? What are the trade-offs?"

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "After your fix, checkout latency goes from 80ms to 350ms. The product team pushes back. How do you respond?"
2. "If you move inventory to PostgreSQL, does that solve the problem?"
3. "Can you get strong consistency without sacrificing availability during the sale?"
4. "What's the over-selling rate you'd accept if you chose a probabilistic approach?"

---

### Guidance (Read After Attempting)

**Root Cause:**
Cassandra with `ONE` consistency: writes go to one replica, reads go to any replica. Under heavy load, replicas fall behind. A replica shows `stock=1` after another node already decremented it to `0`.

**Solution Options:**

**Option A: Cassandra QUORUM**
```
reads:  ConsistencyLevel.QUORUM  (R=2 on N=3)
writes: ConsistencyLevel.QUORUM  (W=2 on N=3)
→ W + R = 4 > N = 3 → always reads the latest write
```
- Latency cost: 80ms → ~150ms (still in checkout SLA)
- Availability: survives 1-node failure
- Caveat: Does not prevent the race condition when two requests _both_ read `stock=1` and _both_ decrement. Need optimistic locking on top.

**Adding Optimistic Locking with Cassandra Lightweight Transactions:**
```python
# Only decrement if current value matches expected
conn.execute(
    "UPDATE inventory SET stock = stock - 1 WHERE product_id = ? IF stock > 0",
    [product_id]
)
# CAS (Compare-And-Swap) — uses Paxos under the hood
# Returns applied: True/False
```
- This adds another ~30–50ms (Paxos round-trip) but prevents oversell completely.
- Total: ~200ms — likely acceptable for checkout.

**Option B: Move to PostgreSQL with Serializable Isolation**
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT stock FROM inventory WHERE product_id = $1 FOR UPDATE;
-- If stock > 0, decrement
UPDATE inventory SET stock = stock - 1 WHERE product_id = $1;
COMMIT;
```
- Eliminates race: `FOR UPDATE` row-level lock blocks concurrent checkers.
- Latency: 50–120ms on a tuned primary.
- Downside: Single primary is a bottleneck at 50K orders/min.

**Option C: Redis atomic counter (best throughput)**
```python
# Atomic decrement — returns new value
new_stock = redis.decr(f"inventory:{product_id}")
if new_stock < 0:
    redis.incr(f"inventory:{product_id}")  # undo
    raise OutOfStockError
```
- `DECR` is atomic on Redis (single-threaded command execution).
- Throughput: 100K+ ops/sec per Redis primary.
- Risk: Redis async replication — if primary fails, may lose last few writes (acknowledged but not replicated). Use `WAIT 1 100` to reduce this window.
- Best for: inventory counts that can be reconstructed from the database in a failover.

**Recommended Architecture:**
- Redis for hot inventory counter (atomic DECR pattern).
- PostgreSQL as source of truth (write final deduction with SERIALIZABLE).
- Re-sync Redis counter from PostgreSQL every N seconds.

**Latency trade-off discussion:**
- 80ms → 200ms is a 2.5× increase. For a checkout flow (which users accept taking 1–2s), this is acceptable.
- Frame it as: the cost of preventing 2,000 oversells per sale that result in refunds, customer service tickets, and reputational damage.

---

## Scenario 3: Social DMs — Message Ordering

**Prompt:**
> "You're building direct messaging for a 500M user social platform (think Instagram DMs). Messages must be displayed in the correct causal order — a reply must always appear after the message it replies to. Messages should never 'disappear' after being seen. The system must handle 5M messages per second globally. Design the consistency model and storage layer."

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "Two users in different regions send a message at the same millisecond. How do you order them?"
2. "A user sends a message and instantly navigates to the conversation. They don't see their message. What model guarantees this and how do you implement it?"
3. "Why not just use a single global sequence number for all messages?"
4. "What's your write path and what consistency level does it target?"

---

### Guidance (Read After Attempting)

**Requirements Mapping:**
- "Correct causal order" → Causal consistency minimum (reply seen after parent)
- "Never disappear" → Monotonic reads (session never time-travels)
- "User sees their own message" → Read-Your-Writes
- "5M msgs/second globally" → Not linearizable (too expensive at this scale)

**Storage Layer:**
Cassandra with the following schema:
```
partition key: (conversation_id, bucket)     -- bucket = week number
clustering key: (sequence_id, message_id)   -- sequence_id = Snowflake ID (monotonic per worker)
columns: sender_id, content, reply_to_id, client_timestamp
```

**Consistency Levels:**
- **Writes:** `LOCAL_QUORUM` — ensures message is on majority of nodes in local DC before ack. Cross-DC replication is async.
- **Reads:** `LOCAL_QUORUM` — reads always see most recent local write.

**Causal Ordering:**
Each message carries `reply_to_id` and `reply_to_sequence`. Client-side (and server-side on delivery):
```python
def can_display(msg: Message, delivered: set[str]) -> bool:
    if msg.reply_to_id is None:
        return True  # Top-level message, no dependency
    return msg.reply_to_id in delivered  # Only display if parent is visible
```
Messages with unresolved parents are buffered client-side. Server pre-fetches the parent if the client doesn't have it.

**Read-Your-Writes:**
After a write to Cassandra (`LOCAL_QUORUM`), return the Cassandra `WriteTime` as a version token in the API response. Client attaches `If-Min-Version: {token}` header on the next read. Server routes to a replica that has applied at least that version, or falls back to the coordinator node.

**Monotonic Reads:**
Client stores the highest sequence_id seen per conversation. Subsequent reads request `WHERE sequence_id >= {last_seen}`. This prevents messages from "disappearing" even if load-balanced to a different replica.

**Ordering ties:**
Two messages sent at the same millisecond get unique Snowflake IDs (different worker nodes generate different IDs). At the receiving end, sort by `(client_timestamp, sequence_id)` — ties broken by Snowflake ID. This is deterministic and every client converges to the same order.

**Why not a global sequence number?**
A single global sequence counter would be a single-node bottleneck, handling 5M/sec would require a dedicated sequencer service with complex failover. Snowflake IDs (time-prefix + worker + sequence) distribute the generation and are monotonic enough for our ordering requirements without coordination.

**Global Replication:**
- Primary DC: `LOCAL_QUORUM` writes (2 of 3 nodes).
- Async replication to secondary DCs with ~50–500ms lag.
- Users reading from their closest DC may see their own write immediately (LOCAL_QUORUM ensures it's on 2 local nodes), but a user in another region may have up to 500ms of cross-DC replication lag.
- This is acceptable: cross-region eventual convergence for DMs is fine — it's not a financial transaction.

**Scale sanity check:**
- 5M msgs/sec × 500 bytes avg = 2.5 GB/sec write throughput
- Cassandra cluster sized for this: ~30–50 nodes across 3 DCs with 10 Gbps NIC
- Read path: 10× write volume = 50M reads/sec → requires extensive caching of recent messages (Redis, write-through on delivery)
