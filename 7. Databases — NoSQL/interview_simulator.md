# Databases — NoSQL — Interview Simulator

## How to use this file
Set a 35-minute timer. Answer each scenario out loud or in writing before reading the model answer. Score yourself using the rubric at the end.

---

## Scenario 1: Design the Storage Layer for a Chat Application

**Prompt:**
"Design the database layer for a WhatsApp-like chat application. Users can send messages in 1-on-1 and group chats. A message history should be accessible for the last 90 days. The system handles 1 billion messages per day. Choose your storage technology and design the schema."

**Time box:** 15 minutes

---

### Model Answer

**Step 1: Identify access patterns**
```
1. Send a message → write one row
2. Load chat history for chat X, page by time → read sorted by time DESC
3. Search for a message (optional) → full-text
4. Mark messages as read → update a field

→ Core pattern: write-heavy append, read by chat + time range
```

**Step 2: Choose the technology**
```
1 billion messages/day = ~11,600 writes/sec average
                       = ~100,000 writes/sec at peak

PostgreSQL: Handles ~10K TPS writes per node → needs sharding at this scale
DynamoDB: Could work, but message history queries require careful SK design
Cassandra: Purpose-built for this — append-only writes, time-ordered by partition key
           Used by Discord (billions of messages), Apple Messages

→ Cassandra for message storage
→ Redis for presence (online/offline status), unread counts
→ Elasticsearch (optional) for full-text message search
```

**Step 3: Cassandra schema**
```sql
-- 1-on-1 and group messages in same table
CREATE TABLE messages (
    chat_id    UUID,
    bucket     TEXT,          -- 'chat_abc#2024-01' — monthly bucket
    sent_at    TIMESTAMP,
    message_id UUID,
    sender_id  UUID,
    text       TEXT,
    type       TEXT,          -- 'text', 'image', 'audio', 'video'
    status     TEXT,          -- 'sent', 'delivered', 'read'
    PRIMARY KEY ((chat_id, bucket), sent_at, message_id)
) WITH CLUSTERING ORDER BY (sent_at DESC, message_id DESC)
  AND default_time_to_live = 7776000;   -- 90 days auto-expiry

-- User's chat list (which chats does user X participate in?)
CREATE TABLE user_chats (
    user_id    UUID,
    last_msg_at TIMESTAMP,
    chat_id    UUID,
    chat_name  TEXT,
    PRIMARY KEY ((user_id), last_msg_at, chat_id)
) WITH CLUSTERING ORDER BY (last_msg_at DESC, chat_id ASC);
```

**Step 4: Write path**
```
Client → API Server → Cassandra (messages table)
                    → Redis: increment unread count for each recipient
                    → Kafka: fan out notification events to recipients
                    → WebSocket gateway: push message to online recipients
```

**Step 5: Read path (load chat history)**
```python
def load_chat(chat_id: UUID, before: datetime = None, limit: int = 50):
    bucket = get_bucket(before or datetime.utcnow())
    prev_bucket = get_prev_bucket(bucket)

    rows = cassandra.execute("""
        SELECT * FROM messages
        WHERE chat_id = %s AND bucket = %s
        AND sent_at < %s
        LIMIT %s
    """, (chat_id, bucket, before, limit))

    if len(rows) < limit:
        # Cross bucket boundary — fetch remainder from prev bucket
        more = cassandra.execute("""
            SELECT * FROM messages
            WHERE chat_id = %s AND bucket = %s
            LIMIT %s
        """, (chat_id, prev_bucket, limit - len(rows)))
        rows = list(rows) + list(more)

    return rows
```

**Step 6: Capacity estimate**
```
1B messages/day × 500 bytes avg = 500GB/day raw data
With RF=3: 1.5TB/day → ~55TB/month → enable TTL (90 days) → ~165TB steady state
Cassandra nodes with 10TB SSD: ~17 nodes minimum; 20 with headroom
Write throughput: 12K TPS avg → across 20 nodes = 600 TPS/node → well within capacity
```

---

## Scenario 2: MongoDB Schema for a Multi-Tenant SaaS Product

**Prompt:**
"You're building a project management SaaS (think Jira). Tenants are companies. Each company has users, projects, and tasks. Tasks have comments and attachments. Design the MongoDB schema and index strategy. The system has 10,000 tenants at launch. In 2 years, you expect some tenants to have 100K+ tasks."

**Time box:** 10 minutes

---

### Model Answer

**Design principle: embed what grows together, reference what grows unboundedly**

```javascript
// tenants collection
{
  "_id": "tenant_acme",
  "name": "ACME Corp",
  "plan": "enterprise",
  "created_at": ISODate("2024-01-01")
}

// users collection
{
  "_id": ObjectId("..."),
  "tenant_id": "tenant_acme",     // ← tenant isolation
  "email": "alice@acme.com",
  "role": "admin"
}

// projects collection
{
  "_id": ObjectId("..."),
  "tenant_id": "tenant_acme",
  "name": "Website Redesign",
  "owner_id": ObjectId("..."),
  "created_at": ISODate("2024-01-15")
}

// tasks collection — separate from projects because 100K+ tasks per tenant
{
  "_id": ObjectId("..."),
  "tenant_id": "tenant_acme",
  "project_id": ObjectId("..."),   // reference (not embed — unbounded)
  "title": "Design new header",
  "status": "in_progress",
  "assignee_id": ObjectId("..."),
  "priority": "high",
  "created_at": ISODate("2024-01-20"),
  "due_at": ISODate("2024-02-01"),
  // Embed comments: typically small, always read with task
  "comments": [
    {"author_id": ObjectId("..."), "text": "Started", "at": ISODate("...")}
  ],
  // Reference attachments: can be large / many
  "attachment_ids": [ObjectId("..."), ObjectId("...")]
}

// attachments collection (referenced, not embedded)
{
  "_id": ObjectId("..."),
  "tenant_id": "tenant_acme",
  "task_id": ObjectId("..."),
  "filename": "mockup.png",
  "s3_url": "https://...",
  "size_bytes": 204800
}
```

**Index strategy:**
```javascript
// All multi-tenant queries include tenant_id → every compound index starts with it

// Tasks by project
db.tasks.createIndex({ tenant_id: 1, project_id: 1, created_at: -1 })

// Tasks by assignee (my tasks view)
db.tasks.createIndex({ tenant_id: 1, assignee_id: 1, status: 1 })

// Partial index: open tasks only (avoids indexing completed/archived)
db.tasks.createIndex(
  { tenant_id: 1, due_at: 1 },
  { partialFilterExpression: { status: { $in: ["open", "in_progress"] } } }
)

// Text search on task titles
db.tasks.createIndex({ title: "text", "comments.text": "text" })
```

**Why cap comments inside the task document?**
- If a task can have unlimited comments → document grows without bound → 16MB limit hit, performance degrades on updates
- Add a cap: if `comments.length > 50`, overflow to a separate `task_comments` collection

---

## Scenario 3: Choose the Right NoSQL Store for a Real-Time Game Leaderboard

**Prompt:**
"You're building a multiplayer mobile game. Players earn points during matches. After each match, update scores. Display: (a) global top 100, (b) friends leaderboard (user's friends, sorted by score), (c) user's current rank globally. Scores update 1,000 times per second. Design the storage for this."

**Time box:** 10 minutes

---

### Model Answer

**Technology choice: Redis Sorted Sets**
```
Requirements mapped to Redis:
  "Update score after match"     → ZINCRBY O(log N)
  "Global top 100"               → ZREVRANGE 0 99 WITHSCORES O(log N + 100)
  "User's global rank"           → ZREVRANK user_id O(log N)
  "Friends leaderboard"          → ZINTERSTORE (intersect global leaderboard with friend set)

Why not SQL?
  1,000 updates/sec to a top-N query with ORDER BY score → index rebuilds under write load
  Top-N from 10M players → full index scan vs O(log N) in Redis

Why not Cassandra?
  No native sorted-by-value structure
  Would need to pull all rows and sort in app = not viable
```

**Redis data structures:**
```python
import redis
r = redis.Redis(decode_responses=True)

GLOBAL_LB  = "leaderboard:global"
FRIENDS_PREFIX = "friends"   # sets of friend user_ids per user

def update_score(user_id: str, points: int):
    """Atomic increment — safe under concurrent updates."""
    r.zincrby(GLOBAL_LB, points, user_id)

def get_top_100() -> list[dict]:
    entries = r.zrevrange(GLOBAL_LB, 0, 99, withscores=True)
    return [{"rank": i+1, "user_id": uid, "score": int(s)}
            for i, (uid, s) in enumerate(entries)]

def get_user_rank(user_id: str) -> int | None:
    rank = r.zrevrank(GLOBAL_LB, user_id)
    return rank + 1 if rank is not None else None

def get_friends_leaderboard(user_id: str) -> list[dict]:
    """Intersect global leaderboard with user's friend set."""
    friends_key = f"{FRIENDS_PREFIX}:{user_id}"
    dest_key = f"lb:friends:{user_id}"

    # ZINTERSTORE: keep scores from GLOBAL_LB for keys that exist in friends set
    r.zinterstore(dest_key, [GLOBAL_LB, friends_key], aggregate="MAX")
    r.expire(dest_key, 60)  # Cache result for 60 seconds

    entries = r.zrevrange(dest_key, 0, 49, withscores=True)
    return [{"rank": i+1, "user_id": uid, "score": int(s)}
            for i, (uid, s) in enumerate(entries)]
```

**Persistence and durability:**
```
Redis AOF (Append-Only File) with fsync=everysec:
  → At most 1 second of score updates lost on crash
  → Acceptable for a game leaderboard

For zero data loss:
  → Redis + PostgreSQL dual write
  → Redis for read performance; PostgreSQL as source of truth
  → On Redis restart: reload leaderboard from PostgreSQL
```

**Scale:**
```
10M players × avg 64 bytes per sorted set entry = 640MB
→ Fits comfortably in a Redis instance with 2–4GB RAM allocation
→ Redis handles 100K+ ops/sec easily on a single node
→ For multi-region: Redis Global Cluster, or replicate to nearest region's Redis
```

---

## Self-Assessment Scorecard

| Criterion                                                         | Score /5 |
|-------------------------------------------------------------------|----------|
| Identified correct NoSQL family for each scenario                 |          |
| Cassandra schema used partition key matching the access pattern   |          |
| Applied time-bucketing to prevent hot/unbounded partitions        |          |
| MongoDB schema used embed vs reference distinction correctly       |          |
| Redis sorted set commands named correctly (ZADD/ZREVRANGE etc.)   |          |
| Mentioned eventual consistency trade-off and how to handle it     |          |
| Included capacity estimate (at least once)                        |          |
| Did NOT jump to sharding/NoSQL without justifying the need        |          |

**Total: /40**

| Score | Interpretation                                               |
|-------|--------------------------------------------------------------|
| 35–40 | Excellent — deep NoSQL design ready for senior interviews    |
| 25–34 | Good — review architecture.md Cassandra + DynamoDB sections  |
| 15–24 | Revisit partition key design rules and Redis data structures  |
| < 15  | Full re-read of architecture.md, then redo Scenario 1        |
