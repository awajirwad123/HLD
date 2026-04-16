# Chat System — Hands-On Exercises

## Exercise 1: WebSocket Chat Server with FastAPI

```python
"""
1:1 and group chat demo using WebSockets.
Covers: connection management, message routing, presence tracking.
"""

import asyncio
import json
import uuid
from datetime import datetime, timezone
from typing import Optional

import redis.asyncio as redis
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, HTTPException
from pydantic import BaseModel

app = FastAPI(title="Chat Server")

REDIS_URL = "redis://localhost:6379"
SERVER_ID = "server-1"   # In production: unique per instance


@app.on_event("startup")
async def startup():
    app.state.redis = redis.from_url(REDIS_URL, decode_responses=True)
    app.state.connections: dict[str, WebSocket] = {}   # user_id → WebSocket
    # Subscribe to cross-server message delivery channel
    asyncio.create_task(message_consumer(app.state.redis))


# ─── Presence management ───────────────────────────────────────────────────

async def set_online(rdb: redis.Redis, user_id: str):
    pipe = rdb.pipeline()
    pipe.hset(f"presence:{user_id}", mapping={"server_id": SERVER_ID, "status": "online"})
    pipe.expire(f"presence:{user_id}", 90)   # Heartbeat must refresh within 90s
    await pipe.execute()


async def set_offline(rdb: redis.Redis, user_id: str):
    await rdb.delete(f"presence:{user_id}")


async def get_presence(rdb: redis.Redis, user_id: str) -> Optional[dict]:
    data = await rdb.hgetall(f"presence:{user_id}")
    return data if data else None


# ─── WebSocket endpoint ────────────────────────────────────────────────────

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(ws: WebSocket, user_id: str):
    await ws.accept()
    rdb: redis.Redis = app.state.redis
    connections: dict = app.state.connections

    # Register connection
    connections[user_id] = ws
    await set_online(rdb, user_id)
    print(f"[+] {user_id} connected. Total: {len(connections)}")

    # Notify contacts this user came online
    await rdb.publish(f"status:{user_id}", json.dumps({"user_id": user_id, "status": "online"}))

    try:
        # Heartbeat task
        heartbeat_task = asyncio.create_task(heartbeat(rdb, user_id))

        while True:
            raw = await ws.receive_text()
            msg = json.loads(raw)
            await handle_message(rdb, connections, user_id, msg)

    except WebSocketDisconnect:
        heartbeat_task.cancel()
        connections.pop(user_id, None)
        await set_offline(rdb, user_id)
        await rdb.publish(f"status:{user_id}", json.dumps({"user_id": user_id, "status": "offline"}))
        print(f"[-] {user_id} disconnected")


async def heartbeat(rdb: redis.Redis, user_id: str):
    """Refresh presence TTL every 30 seconds."""
    while True:
        await asyncio.sleep(30)
        await rdb.expire(f"presence:{user_id}", 90)


# ─── Message handling ─────────────────────────────────────────────────────

async def handle_message(rdb: redis.Redis, connections: dict, sender_id: str, msg: dict):
    """
    Route message to recipient. If on this server → direct push.
    If on another server → publish to Redis Pub/Sub cross-server channel.
    """
    msg_type = msg.get("type")

    if msg_type == "direct":
        await handle_direct_message(rdb, connections, sender_id, msg)
    elif msg_type == "group":
        await handle_group_message(rdb, connections, sender_id, msg)
    elif msg_type == "typing":
        await handle_typing_indicator(rdb, connections, sender_id, msg)


async def handle_direct_message(rdb, connections, sender_id, msg):
    recipient_id = msg["to"]
    
    # Assign server-side message ID
    server_msg = {
        "id": str(uuid.uuid4()),
        "type": "direct",
        "from": sender_id,
        "to": recipient_id,
        "content": msg["content"],
        "ts": datetime.now(timezone.utc).isoformat(),
        "client_msg_id": msg.get("client_msg_id"),   # For dedup
    }

    # Send ACK to sender
    if sender_id in connections:
        await connections[sender_id].send_text(json.dumps({
            "type": "ack",
            "client_msg_id": msg.get("client_msg_id"),
            "server_msg_id": server_msg["id"],
        }))

    # Deliver to recipient
    if recipient_id in connections:
        # Local delivery (recipient on this server)
        await connections[recipient_id].send_text(json.dumps(server_msg))
    else:
        # Cross-server: publish to recipient's channel
        presence = await get_presence(rdb, recipient_id)
        if presence:
            # Online on another server
            await rdb.publish(f"deliver:{presence['server_id']}", json.dumps(server_msg))
        else:
            # Offline: queue for push notification
            await rdb.lpush(f"offline_msgs:{recipient_id}", json.dumps(server_msg))
            await rdb.expire(f"offline_msgs:{recipient_id}", 86400)   # 24h TTL


async def handle_group_message(rdb, connections, sender_id, msg):
    group_id = msg["group_id"]
    
    # Get group members from Redis
    members = await rdb.smembers(f"group:{group_id}:members")

    server_msg = {
        "id": str(uuid.uuid4()),
        "type": "group",
        "group_id": group_id,
        "from": sender_id,
        "content": msg["content"],
        "ts": datetime.now(timezone.utc).isoformat(),
    }

    # Fan-out to all members except sender
    for member_id in members:
        if member_id == sender_id:
            continue
        if member_id in connections:
            await connections[member_id].send_text(json.dumps(server_msg))
        else:
            # Queue for delivery or push notification
            await rdb.lpush(f"offline_msgs:{member_id}", json.dumps(server_msg))


async def handle_typing_indicator(rdb, connections, sender_id, msg):
    recipient_id = msg["to"]
    typing_msg = {"type": "typing", "from": sender_id, "to": recipient_id}
    if recipient_id in connections:
        await connections[recipient_id].send_text(json.dumps(typing_msg))
    # Don't persist typing indicators — ephemeral only


# ─── Cross-server message consumer ────────────────────────────────────────

async def message_consumer(rdb: redis.Redis):
    """Subscribe to this server's delivery channel for cross-server messages."""
    sub = rdb.pubsub()
    await sub.subscribe(f"deliver:{SERVER_ID}")

    async for raw_msg in sub.listen():
        if raw_msg["type"] != "message":
            continue
        msg = json.loads(raw_msg["data"])
        recipient_id = msg.get("to") or msg.get("group_id")
        connections: dict = app.state.connections

        if recipient_id in connections:
            try:
                await connections[recipient_id].send_text(json.dumps(msg))
            except Exception:
                pass   # WebSocket may have closed; handled by disconnect handler


# ─── REST: create group ────────────────────────────────────────────────────

class CreateGroupRequest(BaseModel):
    name: str
    member_ids: list[str]


@app.post("/groups")
async def create_group(body: CreateGroupRequest):
    rdb: redis.Redis = app.state.redis
    group_id = str(uuid.uuid4())
    pipe = rdb.pipeline()
    pipe.hset(f"group:{group_id}", mapping={"name": body.name, "created_at": datetime.now(timezone.utc).isoformat()})
    pipe.sadd(f"group:{group_id}:members", *body.member_ids)
    await pipe.execute()
    return {"group_id": group_id}
```

---

## Exercise 2: Message History with Cassandra (Simulated)

```python
"""
Simulate Cassandra message storage with sorted message retrieval.
Real implementation uses cassandra-driver; this uses SQLite for portability.
"""

import sqlite3
import uuid
import time
from dataclasses import dataclass
from typing import Optional


@dataclass
class Message:
    conversation_id: str
    message_id: str      # Time-ordered UUID (sortable by creation time)
    sender_id: str
    content: str
    created_at: float    # Unix timestamp


def timeuuid() -> str:
    """Generate a time-ordered UUID (simulating Cassandra TIMEUUID)."""
    ts_100ns = int(time.time() * 1e7)   # 100-nanosecond intervals
    rand = uuid.uuid4().int & 0xFFFFFF
    return f"{ts_100ns:016x}-{rand:06x}"


class MessageStore:
    def __init__(self, db_path: str = ":memory:"):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self._create_schema()

    def _create_schema(self):
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS messages (
                conversation_id TEXT NOT NULL,
                message_id      TEXT NOT NULL,
                sender_id       TEXT NOT NULL,
                content         TEXT NOT NULL,
                client_msg_id   TEXT UNIQUE,   -- Deduplication
                created_at      REAL NOT NULL,
                PRIMARY KEY (conversation_id, message_id)
            )
        """)
        self.conn.execute(
            "CREATE INDEX IF NOT EXISTS idx_conv_time ON messages(conversation_id, message_id DESC)"
        )
        self.conn.commit()

    def save_message(self, msg: Message, client_msg_id: Optional[str] = None) -> bool:
        """Returns False if duplicate (client_msg_id already exists)."""
        try:
            self.conn.execute(
                """INSERT INTO messages
                   (conversation_id, message_id, sender_id, content, client_msg_id, created_at)
                   VALUES (?, ?, ?, ?, ?, ?)""",
                (msg.conversation_id, msg.message_id, msg.sender_id,
                 msg.content, client_msg_id, msg.created_at),
            )
            self.conn.commit()
            return True
        except sqlite3.IntegrityError:
            return False   # Duplicate

    def get_messages(self, conversation_id: str, limit: int = 50,
                     before_message_id: Optional[str] = None) -> list[Message]:
        """Paginated fetch of most recent messages (newest first)."""
        if before_message_id:
            rows = self.conn.execute(
                """SELECT conversation_id, message_id, sender_id, content, created_at
                   FROM messages
                   WHERE conversation_id = ? AND message_id < ?
                   ORDER BY message_id DESC LIMIT ?""",
                (conversation_id, before_message_id, limit),
            ).fetchall()
        else:
            rows = self.conn.execute(
                """SELECT conversation_id, message_id, sender_id, content, created_at
                   FROM messages
                   WHERE conversation_id = ?
                   ORDER BY message_id DESC LIMIT ?""",
                (conversation_id, limit),
            ).fetchall()

        return [Message(*row) for row in rows]


# ─── Demo ─────────────────────────────────────────────────────────────────

store = MessageStore()
conv_id = "conv-alice-bob"

# Simulate message exchange
participants = [("alice", "Hey Bob!"), ("bob", "Hey Alice!"), ("alice", "How's it going?"),
                ("bob", "Great, thanks!"), ("alice", "Nice to hear")]

for sender, content in participants:
    client_id = str(uuid.uuid4())
    msg = Message(
        conversation_id=conv_id,
        message_id=timeuuid(),
        sender_id=sender,
        content=content,
        created_at=time.time(),
    )
    saved = store.save_message(msg, client_msg_id=client_id)
    time.sleep(0.001)   # Ensure distinct timestamps

# Fetch last 3 messages
messages = store.get_messages(conv_id, limit=3)
print("=== Last 3 messages (newest first) ===")
for m in messages:
    print(f"  [{m.message_id[:16]}] {m.sender_id}: {m.content}")

# Test deduplication
duplicate_client_id = client_id   # Reuse last client_id
dup_result = store.save_message(
    Message(conv_id, timeuuid(), "alice", "Duplicate!", time.time()),
    client_msg_id=duplicate_client_id,
)
print(f"\nDuplicate message rejected: {not dup_result}")  # Should be True
```
