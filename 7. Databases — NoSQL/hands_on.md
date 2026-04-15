# Databases — NoSQL — Hands-On Exercises

## Exercise 1: Redis — Key-Value Patterns with Python

```python
"""
Demonstrates the most common Redis patterns used in production:
session storage, rate limiting, distributed lock, leaderboard.
"""
import redis
import json
import time
import uuid

r = redis.Redis(host="localhost", port=6379, decode_responses=True)


# --- Pattern 1: Session Store with TTL ---
def save_session(user_id: int, role: str, ttl_seconds: int = 3600) -> str:
    session_id = str(uuid.uuid4())
    payload = json.dumps({"user_id": user_id, "role": role})
    r.set(f"session:{session_id}", payload, ex=ttl_seconds)
    return session_id

def get_session(session_id: str) -> dict | None:
    raw = r.get(f"session:{session_id}")
    return json.loads(raw) if raw else None

def invalidate_session(session_id: str):
    r.delete(f"session:{session_id}")


# --- Pattern 2: Sliding Window Rate Limiter ---
def is_rate_limited(user_id: int, window_seconds: int = 60, max_requests: int = 100) -> bool:
    key = f"rate:{user_id}"
    now = time.time()
    pipe = r.pipeline()

    # Remove entries older than the window
    pipe.zremrangebyscore(key, 0, now - window_seconds)
    # Add current request timestamp
    pipe.zadd(key, {str(now): now})
    # Count requests in window
    pipe.zcard(key)
    # Reset TTL
    pipe.expire(key, window_seconds)

    results = pipe.execute()
    request_count = results[2]
    return request_count > max_requests


# --- Pattern 3: Distributed Lock (Redis SET NX PX) ---
class DistributedLock:
    def __init__(self, resource: str, ttl_ms: int = 5000):
        self.key = f"lock:{resource}"
        self.token = str(uuid.uuid4())
        self.ttl_ms = ttl_ms

    def acquire(self) -> bool:
        # SET key token NX PX ttl — atomic compare-and-set
        result = r.set(self.key, self.token, nx=True, px=self.ttl_ms)
        return result is True  # None if key already existed

    def release(self):
        # Lua script ensures we only delete OUR lock (avoids releasing someone else's)
        script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        r.eval(script, 1, self.key, self.token)

    def __enter__(self):
        if not self.acquire():
            raise RuntimeError(f"Could not acquire lock: {self.key}")
        return self

    def __exit__(self, *args):
        self.release()


# Demonstrate: prevent double-processing of a payment
def process_payment(order_id: str):
    lock = DistributedLock(f"payment:{order_id}", ttl_ms=10000)
    try:
        with lock:
            print(f"Processing payment for {order_id}")
            time.sleep(0.1)  # Simulate payment processor call
            print(f"Payment complete for {order_id}")
    except RuntimeError:
        print(f"Order {order_id} already being processed — skip")


# --- Pattern 4: Real-Time Leaderboard with Sorted Set ---
def update_score(user_id: str, score_delta: int):
    r.zincrby("leaderboard:weekly", score_delta, user_id)

def get_top_n(n: int = 10) -> list[dict]:
    # ZREVRANGE: highest score first. WITHSCORES returns (member, score) pairs
    entries = r.zrevrange("leaderboard:weekly", 0, n - 1, withscores=True)
    return [{"rank": i + 1, "user_id": uid, "score": int(score)}
            for i, (uid, score) in enumerate(entries)]

def get_user_rank(user_id: str) -> int | None:
    rank = r.zrevrank("leaderboard:weekly", user_id)
    return rank + 1 if rank is not None else None  # 0-indexed from Redis → 1-indexed


if __name__ == "__main__":
    # Session demo
    sid = save_session(user_id=42, role="admin")
    print(get_session(sid))  # {"user_id": 42, "role": "admin"}

    # Rate limiter demo
    for i in range(5):
        limited = is_rate_limited(user_id=123, max_requests=3)
        print(f"Request {i+1}: limited={limited}")

    # Leaderboard demo
    for uid, score in [("alice", 150), ("bob", 200), ("charlie", 120), ("alice", 80)]:
        update_score(uid, score)
    print(get_top_n(3))
    print(f"Alice rank: {get_user_rank('alice')}")
```

---

## Exercise 2: MongoDB — Document Queries and Aggregation

```python
"""
Demonstrates MongoDB document operations, compound indexes, and aggregation pipeline.
"""
from pymongo import MongoClient, ASCENDING, DESCENDING, TEXT
from datetime import datetime, timedelta
import random

client = MongoClient("mongodb://localhost:27017")
db = client["ecommerce"]
orders = db["orders"]
products = db["products"]


# --- Setup: Create indexes ---
def setup_indexes():
    # Compound index for order history query
    orders.create_index([("user_id", ASCENDING), ("created_at", DESCENDING)])

    # Partial index: only index pending orders (smaller, faster for ops queue)
    orders.create_index(
        [("created_at", ASCENDING)],
        partialFilterExpression={"status": "pending"},
        name="idx_pending_orders"
    )

    # Text index for product search
    products.create_index([("name", TEXT), ("description", TEXT)])

    # TTL index: auto-delete cart sessions after 2 hours
    db["cart_sessions"].create_index(
        [("created_at", ASCENDING)],
        expireAfterSeconds=7200
    )

    print("Indexes created")


# --- CRUD with schema-flexibility demo ---
def insert_orders():
    """Demonstrates flexible schema: different orders can have different fields."""
    docs = [
        {
            "user_id": "user_123",
            "status": "paid",
            "total_cents": 4999,
            "created_at": datetime.utcnow() - timedelta(days=2),
            "items": [{"sku": "BOOK-001", "qty": 1, "price": 4999}],
            "payment_method": "card",
        },
        {
            "user_id": "user_123",
            "status": "pending",
            "total_cents": 1299,
            "created_at": datetime.utcnow(),
            "items": [{"sku": "PEN-005", "qty": 3, "price": 433}],
            # No payment_method yet — schema-less allows this
        },
        {
            "user_id": "user_456",
            "status": "shipped",
            "total_cents": 9999,
            "created_at": datetime.utcnow() - timedelta(days=5),
            "items": [{"sku": "LAPTOP-010", "qty": 1, "price": 9999}],
            "payment_method": "paypal",
            "gift_wrap": True,  # This field only exists on some orders
        },
    ]
    result = orders.insert_many(docs)
    print(f"Inserted {len(result.inserted_ids)} orders")


# --- Query patterns ---
def get_user_orders(user_id: str, limit: int = 10):
    """Uses compound index: (user_id, created_at DESC)"""
    return list(orders.find(
        {"user_id": user_id},
        {"_id": 0, "status": 1, "total_cents": 1, "created_at": 1}
    ).sort("created_at", DESCENDING).limit(limit))


def search_products(query: str):
    """Uses text index for full-text search with relevance score."""
    return list(products.find(
        {"$text": {"$search": query}},
        {"score": {"$meta": "textScore"}, "name": 1, "price_cents": 1}
    ).sort([("score", {"$meta": "textScore"})]).limit(20))


# --- Aggregation pipeline: total revenue by status ---
def revenue_by_status():
    """
    MongoDB aggregation pipeline — equivalent to:
    SELECT status, COUNT(*), SUM(total_cents) FROM orders GROUP BY status
    """
    pipeline = [
        {"$group": {
            "_id": "$status",
            "count": {"$sum": 1},
            "total_revenue": {"$sum": "$total_cents"}
        }},
        {"$sort": {"total_revenue": DESCENDING}},
        {"$project": {
            "status": "$_id",
            "count": 1,
            "total_revenue_dollars": {"$divide": ["$total_revenue", 100]},
            "_id": 0
        }}
    ]
    return list(orders.aggregate(pipeline))


# --- Update: atomic array operations ---
def add_item_to_cart(cart_id: str, item: dict):
    """$push appends to an array field atomically."""
    db["carts"].update_one(
        {"_id": cart_id},
        {
            "$push": {"items": item},
            "$inc": {"total_cents": item["price"]},
            "$setOnInsert": {"created_at": datetime.utcnow()}
        },
        upsert=True  # Create cart if doesn't exist
    )

def remove_item_from_cart(cart_id: str, sku: str):
    """$pull removes matching elements from array."""
    db["carts"].update_one(
        {"_id": cart_id},
        {"$pull": {"items": {"sku": sku}}}
    )


if __name__ == "__main__":
    setup_indexes()
    insert_orders()
    print("User orders:", get_user_orders("user_123"))
    print("Revenue by status:", revenue_by_status())
```

---

## Exercise 3: Cassandra Table Design — Message History

```python
"""
Cassandra data modeling demo: design for WhatsApp-style message history.
Shows correct partition key selection and query patterns.

Note: Uses cassandra-driver. Install: pip install cassandra-driver
"""
from cassandra.cluster import Cluster
from cassandra.query import SimpleStatement
from datetime import datetime, timezone
import uuid

cluster = Cluster(["localhost"])
session = cluster.connect()

# Setup keyspace
session.execute("""
    CREATE KEYSPACE IF NOT EXISTS messenger
    WITH REPLICATION = {'class': 'SimpleStrategy', 'replication_factor': 3}
""")
session.set_keyspace("messenger")


# --- Schema design for message history ---
# Query: "Give me the last 50 messages in chat X ordered by time"
# → Partition key: chat_id (all messages for a chat on same node)
# → Clustering key: (sent_at DESC, message_id) for time-ordered retrieval

session.execute("""
    CREATE TABLE IF NOT EXISTS messages (
        chat_id    UUID,
        sent_at    TIMESTAMP,
        message_id UUID,
        sender_id  UUID,
        text       TEXT,
        type       TEXT,       -- 'text', 'image', 'audio'
        PRIMARY KEY ((chat_id), sent_at, message_id)
    ) WITH CLUSTERING ORDER BY (sent_at DESC, message_id DESC)
      AND default_time_to_live = 7776000  -- 90 days TTL
""")


# --- Time-bucketed partition key (prevents hot partition at scale) ---
# Problem: A popular chat group with millions of messages → huge, hot partition
# Solution: Bucket by month
session.execute("""
    CREATE TABLE IF NOT EXISTS messages_bucketed (
        chat_id    UUID,
        bucket     TEXT,        -- 'chat_abc#2024-01', 'chat_abc#2024-02'
        sent_at    TIMESTAMP,
        message_id UUID,
        sender_id  UUID,
        text       TEXT,
        PRIMARY KEY ((chat_id, bucket), sent_at, message_id)
    ) WITH CLUSTERING ORDER BY (sent_at DESC, message_id DESC)
""")


def get_bucket(ts: datetime) -> str:
    return ts.strftime("%Y-%m")


def send_message(chat_id: uuid.UUID, sender_id: uuid.UUID, text: str):
    now = datetime.now(timezone.utc)
    bucket = get_bucket(now)
    msg_id = uuid.uuid4()

    session.execute("""
        INSERT INTO messages_bucketed (chat_id, bucket, sent_at, message_id, sender_id, text)
        VALUES (%s, %s, %s, %s, %s, %s)
    """, (chat_id, bucket, now, msg_id, sender_id, text))
    return msg_id


def get_recent_messages(chat_id: uuid.UUID, limit: int = 50) -> list:
    """Fetch recent messages from current and previous month buckets."""
    now = datetime.now(timezone.utc)
    buckets = [get_bucket(now)]

    # Also check previous month for page-across-boundary reads
    from datetime import timedelta
    prev_month = now - timedelta(days=32)
    buckets.append(get_bucket(prev_month))

    messages = []
    for bucket in buckets:
        rows = session.execute("""
            SELECT sent_at, sender_id, text
            FROM messages_bucketed
            WHERE chat_id = %s AND bucket = %s
            LIMIT %s
        """, (chat_id, bucket, limit))
        messages.extend(rows)
        if len(messages) >= limit:
            break

    return sorted(messages, key=lambda r: r.sent_at, reverse=True)[:limit]


if __name__ == "__main__":
    chat = uuid.UUID("00000000-0000-0000-0000-000000000001")
    user_a = uuid.UUID("00000000-0000-0000-0000-000000000002")
    user_b = uuid.UUID("00000000-0000-0000-0000-000000000003")

    send_message(chat, user_a, "Hey!")
    send_message(chat, user_b, "Hi there!")
    send_message(chat, user_a, "How are you?")

    msgs = get_recent_messages(chat, limit=10)
    for m in msgs:
        print(f"[{m.sent_at}] {m.text}")
```

---

## Exercise 4: DynamoDB Single-Table Design

```python
"""
DynamoDB single-table design pattern using boto3.
Models users, their orders, and order items in one table.
"""
import boto3
from datetime import datetime
import uuid

dynamodb = boto3.resource(
    "dynamodb",
    region_name="us-east-1",
    endpoint_url="http://localhost:8000"  # Local DynamoDB for development
)

TABLE_NAME = "AppTable"

# Access patterns this schema supports:
# 1. Get user profile by user_id
# 2. Get all orders for a user (sorted by date)
# 3. Get a specific order + its items
# 4. Get pending orders (using GSI)


def create_table():
    table = dynamodb.create_table(
        TableName=TABLE_NAME,
        KeySchema=[
            {"AttributeName": "PK", "KeyType": "HASH"},   # Partition key
            {"AttributeName": "SK", "KeyType": "RANGE"},  # Sort key
        ],
        AttributeDefinitions=[
            {"AttributeName": "PK", "AttributeType": "S"},
            {"AttributeName": "SK", "AttributeType": "S"},
            {"AttributeName": "GSI1PK", "AttributeType": "S"},  # For GSI
            {"AttributeName": "GSI1SK", "AttributeType": "S"},
        ],
        GlobalSecondaryIndexes=[{
            "IndexName": "GSI1",
            "KeySchema": [
                {"AttributeName": "GSI1PK", "KeyType": "HASH"},
                {"AttributeName": "GSI1SK", "KeyType": "RANGE"},
            ],
            "Projection": {"ProjectionType": "ALL"},
        }],
        BillingMode="PAY_PER_REQUEST",
    )
    table.wait_until_exists()
    return table


def get_table():
    return dynamodb.Table(TABLE_NAME)


def put_user(user_id: str, email: str, name: str):
    """
    PK = USER#user_123
    SK = PROFILE
    """
    get_table().put_item(Item={
        "PK": f"USER#{user_id}",
        "SK": "PROFILE",
        "email": email,
        "name": name,
        "created_at": datetime.utcnow().isoformat(),
        "entity_type": "USER",
    })


def put_order(user_id: str, order_id: str, total_cents: int, status: str = "pending"):
    """
    PK  = USER#user_123
    SK  = ORDER#2024-01-15T10:00#ord_abc   (range queryable by date)
    GSI = STATUS#pending | ORDER#2024-01-15T10:00#ord_abc  (query pending orders)
    """
    now = datetime.utcnow().isoformat()
    get_table().put_item(Item={
        "PK": f"USER#{user_id}",
        "SK": f"ORDER#{now}#{order_id}",
        "order_id": order_id,
        "status": status,
        "total_cents": total_cents,
        "created_at": now,
        "entity_type": "ORDER",
        # GSI for querying by status
        "GSI1PK": f"STATUS#{status}",
        "GSI1SK": f"ORDER#{now}#{order_id}",
    })


def get_user_with_orders(user_id: str) -> dict:
    """
    Single query fetches BOTH the user profile AND all orders.
    PK = USER#user_123, SK begins_with either 'PROFILE' or 'ORDER#'
    This is the power of single-table design.
    """
    from boto3.dynamodb.conditions import Key

    response = get_table().query(
        KeyConditionExpression=Key("PK").eq(f"USER#{user_id}")
        # No SK filter → returns PROFILE + all ORDER# items
    )

    items = response["Items"]
    profile = next((i for i in items if i["SK"] == "PROFILE"), None)
    orders = [i for i in items if i["SK"].startswith("ORDER#")]
    orders.sort(key=lambda o: o["SK"], reverse=True)  # newest first

    return {"profile": profile, "orders": orders}


def get_pending_orders(limit: int = 20) -> list:
    """Uses GSI to find all pending orders across all users."""
    from boto3.dynamodb.conditions import Key

    response = get_table().query(
        IndexName="GSI1",
        KeyConditionExpression=Key("GSI1PK").eq("STATUS#pending"),
        Limit=limit,
        ScanIndexForward=False,  # newest first
    )
    return response["Items"]


if __name__ == "__main__":
    put_user("user_123", "alice@example.com", "Alice")
    put_order("user_123", str(uuid.uuid4()), 4999, "pending")
    put_order("user_123", str(uuid.uuid4()), 1299, "paid")

    result = get_user_with_orders("user_123")
    print("Profile:", result["profile"])
    print("Orders:", len(result["orders"]))

    pending = get_pending_orders()
    print("Pending orders:", len(pending))
```

---

## Exercise 5: NoSQL vs SQL — Benchmark Mental Model

```python
"""
Not a runnable benchmark — a structured thought experiment to internalize
when each store wins. Run through this before any system design interview.
"""

SCENARIOS = [
    {
        "scenario": "Store and retrieve user session (100M active sessions)",
        "winner": "Redis",
        "why": "O(1) GET by session ID, built-in TTL, sub-millisecond latency",
        "sql_problem": "100M rows, index lookup still requires disk I/O path",
    },
    {
        "scenario": "Find all orders for user_123, sorted by date",
        "winner": "PostgreSQL or DynamoDB",
        "why": "Relational join between users and orders. DynamoDB works if single-table design used.",
        "nosql_problem": "Cassandra: needs partition key (user_id) + clustering key (date) — correct design",
    },
    {
        "scenario": "Store 1 million IoT temperature readings per minute",
        "winner": "Cassandra or TimescaleDB",
        "why": "Append-only write pattern. Cassandra LSM tree handles this at scale.",
        "sql_problem": "B-tree index maintenance on high-write tables causes write amplification",
    },
    {
        "scenario": "Product catalog with varying attributes (electronics vs clothing)",
        "winner": "MongoDB",
        "why": "Electronics have voltage/wattage, clothing has size/color — flexible schema per doc",
        "sql_problem": "EAV pattern in SQL is unmaintainable; many NULLable columns is messy",
    },
    {
        "scenario": "Debit account A, credit account B atomically",
        "winner": "PostgreSQL",
        "why": "Multi-entity ACID transaction with single commit",
        "nosql_problem": "DynamoDB TransactWrite: 2 items, works but limited. Cassandra: LWT single-partition only",
    },
    {
        "scenario": "Friend-of-friend recommendations (2-hop graph query)",
        "winner": "Neo4j",
        "why": "Traverse edges natively. In SQL: 2 self-JOINs → O(N²) at scale",
        "note": "For 1-hop (direct friends), SQL with FK index is fine",
    },
]

for s in SCENARIOS:
    print(f"\nScenario: {s['scenario']}")
    print(f"  Winner: {s['winner']}")
    print(f"  Why:    {s['why']}")
```
