# WebSockets & Real-Time Communication — Hands-On Exercises

## Exercise 1: WebSocket Chat Server with FastAPI + Redis Pub/Sub

```python
"""
Multi-server WebSocket chat room using Redis Pub/Sub for cross-server fan-out.
Each server instance can serve connections independently.
Messages between servers are routed via Redis.

pip install fastapi uvicorn websockets redis asyncio
"""
import asyncio
import json
import uuid
from contextlib import asynccontextmanager
from typing import Optional

import redis.asyncio as aioredis
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import HTMLResponse

REDIS_URL  = "redis://localhost:6379"
SERVER_ID  = str(uuid.uuid4())[:8]   # Unique per server instance

# ── Connection registry (local to this server process) ────────────────────────
class ConnectionManager:
    def __init__(self):
        # room_id → set of WebSocket connections on THIS server
        self._rooms: dict[str, set[WebSocket]] = {}
        # ws → user_id
        self._users: dict[WebSocket, str] = {}

    def connect(self, ws: WebSocket, user_id: str, room_id: str):
        self._rooms.setdefault(room_id, set()).add(ws)
        self._users[ws] = user_id

    def disconnect(self, ws: WebSocket, room_id: str):
        self._rooms.get(room_id, set()).discard(ws)
        self._users.pop(ws, None)

    async def broadcast_local(self, room_id: str, message: str, exclude: WebSocket = None):
        """Send to all local connections in a room (on this server instance)."""
        dead = set()
        for ws in list(self._rooms.get(room_id, set())):
            if ws is exclude:
                continue
            try:
                await ws.send_text(message)
            except Exception:
                dead.add(ws)
        for ws in dead:
            self._rooms.get(room_id, set()).discard(ws)

    def room_members(self, room_id: str) -> list[str]:
        return [self._users[ws] for ws in self._rooms.get(room_id, set())]


mgr = ConnectionManager()
redis_client: aioredis.Redis = None


# ── Redis Pub/Sub listener (runs as background task) ─────────────────────────
async def redis_subscriber():
    """
    Listen to the Redis channel for cross-server messages.
    When a message arrives (from another server), broadcast to local connections.
    """
    sub = redis_client.pubsub()
    await sub.subscribe("ws:broadcast")

    async for msg in sub.listen():
        if msg["type"] != "message":
            continue
        data = json.loads(msg["data"])

        # Only forward if we have local connections for this room
        room_id = data.get("room_id")
        origin  = data.get("server_id")

        if origin == SERVER_ID:
            continue  # This server published it — don't echo back

        if room_id:
            payload = json.dumps(data.get("payload", {}))
            await mgr.broadcast_local(room_id, payload)


@asynccontextmanager
async def lifespan(app: FastAPI):
    global redis_client
    redis_client = aioredis.from_url(REDIS_URL, decode_responses=True)
    # Start Redis subscriber as a background task
    task = asyncio.create_task(redis_subscriber())
    yield
    task.cancel()
    await redis_client.close()


app = FastAPI(lifespan=lifespan)


async def publish_to_all_servers(room_id: str, payload: dict):
    """Publish message to Redis — all servers (including this one) will receive it."""
    await redis_client.publish("ws:broadcast", json.dumps({
        "server_id": SERVER_ID,
        "room_id":   room_id,
        "payload":   payload,
    }))


# ── WebSocket endpoint ────────────────────────────────────────────────────────
@app.websocket("/ws/{room_id}/{user_id}")
async def websocket_endpoint(ws: WebSocket, room_id: str, user_id: str):
    await ws.accept()
    mgr.connect(ws, user_id, room_id)

    # Store presence in Redis (TTL = 60s, renewed by heartbeats)
    await redis_client.setex(f"presence:{user_id}", 60, SERVER_ID)

    # Announce join to room
    join_msg = {"type": "joined", "user": user_id, "room": room_id}
    await mgr.broadcast_local(room_id, json.dumps(join_msg), exclude=ws)
    await publish_to_all_servers(room_id, join_msg)

    try:
        while True:
            raw = await ws.receive_text()
            data = json.loads(raw)

            if data.get("type") == "heartbeat":
                # Renew presence TTL
                await redis_client.setex(f"presence:{user_id}", 60, SERVER_ID)
                await ws.send_text(json.dumps({"type": "heartbeat_ack"}))
                continue

            if data.get("type") == "message":
                msg_id  = str(uuid.uuid4())
                payload = {
                    "type":    "message",
                    "id":      msg_id,
                    "user":    user_id,
                    "room":    room_id,
                    "content": data.get("content", ""),
                }
                # Deliver to local connections on this server
                await mgr.broadcast_local(room_id, json.dumps(payload))
                # Cross-server fan-out via Redis
                await publish_to_all_servers(room_id, payload)

    except WebSocketDisconnect:
        mgr.disconnect(ws, room_id)
        await redis_client.delete(f"presence:{user_id}")
        leave_msg = {"type": "left", "user": user_id, "room": room_id}
        await mgr.broadcast_local(room_id, json.dumps(leave_msg))
        await publish_to_all_servers(room_id, leave_msg)


@app.get("/presence/{user_id}")
async def get_presence(user_id: str):
    server = await redis_client.get(f"presence:{user_id}")
    return {"user_id": user_id, "online": bool(server), "server": server}


@app.get("/")
async def demo_page():
    return HTMLResponse("""
    <script>
      const ws = new WebSocket(`ws://localhost:8000/ws/room1/alice`);
      ws.onmessage = e => console.log(JSON.parse(e.data));
      ws.onopen = () => ws.send(JSON.stringify({type:"message", content:"Hello!"}));
      setInterval(() => ws.send(JSON.stringify({type:"heartbeat"})), 30000);
    </script>
    """)
```

---

## Exercise 2: Server-Sent Events (SSE) — Live Dashboard

```python
"""
SSE endpoint for a live metrics dashboard.
Client opens one connection; server streams updates.
Demonstrates: event IDs for reconnect replay, event types.

pip install fastapi uvicorn asyncio
"""
import asyncio
import json
import random
import time
from asyncio import Queue
from typing import AsyncIterator

from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse, HTMLResponse

app = FastAPI()

# ── In-memory event store for replay (production: Redis Streams or DB) ────────
_event_store: list[dict] = []
_event_counter = 0

def store_event(event_type: str, data: dict) -> int:
    global _event_counter
    _event_counter += 1
    event = {"id": _event_counter, "type": event_type, "data": data, "ts": time.time()}
    _event_store.append(event)
    # Keep last 1000 events only
    if len(_event_store) > 1000:
        _event_store.pop(0)
    return _event_counter


def events_since(last_event_id: int) -> list[dict]:
    """Return events with ID > last_event_id (for reconnect replay)."""
    return [e for e in _event_store if e["id"] > last_event_id]


def format_sse(event_id: int, event_type: str, data: dict) -> str:
    """Format a Server-Sent Event according to the spec."""
    lines = [
        f"id: {event_id}",
        f"event: {event_type}",
        f"data: {json.dumps(data)}",
        "",   # Blank line terminates the event
        "",
    ]
    return "\n".join(lines)


async def metrics_generator(request: Request, last_event_id: int = 0) -> AsyncIterator[str]:
    """
    Streams metric updates.
    On reconnect (last_event_id > 0), first replays missed events.
    """
    # Replay missed events on reconnect
    for missed in events_since(last_event_id):
        yield format_sse(missed["id"], missed["type"], missed["data"])

    # Stream new events
    cpu, memory = 45.0, 62.0
    while True:
        if await request.is_disconnected():
            break

        # Simulate changing metrics
        cpu    = max(5, min(95, cpu    + random.uniform(-5, 5)))
        memory = max(10, min(90, memory + random.uniform(-2, 2)))

        metrics = {"cpu": round(cpu, 1), "memory": round(memory, 1),
                   "timestamp": int(time.time())}
        eid = store_event("metrics", metrics)
        yield format_sse(eid, "metrics", metrics)

        # Simulate occasional alerts
        if random.random() < 0.05:
            alert = {"level": "warn", "message": f"CPU spike: {cpu:.1f}%"}
            eid = store_event("alert", alert)
            yield format_sse(eid, "alert", alert)

        await asyncio.sleep(1)


@app.get("/stream/metrics")
async def stream_metrics(request: Request):
    # Extract Last-Event-ID from header (sent by browser EventSource on reconnect)
    last_id = int(request.headers.get("Last-Event-ID", 0))

    return StreamingResponse(
        metrics_generator(request, last_id),
        media_type="text/event-stream",
        headers={
            "Cache-Control":  "no-cache",
            "X-Accel-Buffering": "no",   # Disable Nginx buffering
            "Connection":     "keep-alive",
        },
    )


@app.get("/")
async def dashboard():
    return HTMLResponse("""
    <!DOCTYPE html>
    <html>
    <body>
      <h2>Live Metrics Dashboard</h2>
      <div id="cpu">CPU: --</div>
      <div id="mem">Memory: --</div>
      <div id="alerts" style="color:orange"></div>
      <script>
        const es = new EventSource('/stream/metrics');

        es.addEventListener('metrics', e => {
          const d = JSON.parse(e.data);
          document.getElementById('cpu').textContent = `CPU: ${d.cpu}%`;
          document.getElementById('mem').textContent = `Memory: ${d.memory}%`;
        });

        es.addEventListener('alert', e => {
          const d = JSON.parse(e.data);
          document.getElementById('alerts').textContent = `ALERT: ${d.message}`;
        });

        es.onerror = () => console.log('SSE reconnecting...');
        // Browser automatically reconnects and sends Last-Event-ID header
      </script>
    </body>
    </html>
    """)
```

---

## Exercise 3: Presence Detection System

```python
"""
Presence detection using Redis with heartbeat TTL.
Demonstrates: online/offline detection, last-seen tracking,
bulk presence lookup for a conversation.

pip install redis asyncio fastapi
"""
import asyncio
import time
from datetime import datetime, UTC

import redis.asyncio as aioredis
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

REDIS_URL      = "redis://localhost:6379"
HEARTBEAT_INTERVAL = 30    # Client sends heartbeat every 30s
PRESENCE_TTL       = 65    # 2× heartbeat + buffer → user appears offline after 65s

app = FastAPI()
r: aioredis.Redis = None


@app.on_event("startup")
async def startup():
    global r
    r = aioredis.from_url(REDIS_URL, decode_responses=True)


class PresenceService:
    """Manages user online/offline status in Redis."""

    @staticmethod
    async def set_online(user_id: str, metadata: dict = None):
        pipe = r.pipeline()
        # Expiring key: user appears online while key exists
        pipe.setex(f"presence:online:{user_id}", PRESENCE_TTL, "1")
        # Last-seen: persists after going offline (for "last seen 2h ago")
        pipe.hset(f"presence:meta:{user_id}", mapping={
            "last_seen": datetime.now(UTC).isoformat(),
            "status":    "online",
            **(metadata or {})
        })
        await pipe.execute()

    @staticmethod
    async def set_offline(user_id: str):
        pipe = r.pipeline()
        pipe.delete(f"presence:online:{user_id}")
        pipe.hset(f"presence:meta:{user_id}", mapping={
            "last_seen": datetime.now(UTC).isoformat(),
            "status":    "offline",
        })
        await pipe.execute()
        # Notify subscribers that this user went offline
        await r.publish(f"presence:changes", f"{user_id}:offline")

    @staticmethod
    async def is_online(user_id: str) -> bool:
        return bool(await r.exists(f"presence:online:{user_id}"))

    @staticmethod
    async def get_last_seen(user_id: str) -> dict:
        meta = await r.hgetall(f"presence:meta:{user_id}")
        online = await r.exists(f"presence:online:{user_id}")
        return {
            "user_id":   user_id,
            "online":    bool(online),
            "last_seen": meta.get("last_seen"),
            "status":    "online" if online else "offline",
        }

    @staticmethod
    async def bulk_presence(user_ids: list[str]) -> dict[str, bool]:
        """
        Check presence for many users in one Redis round-trip.
        Uses pipeline to batch EXISTS calls.
        """
        pipe = r.pipeline()
        for uid in user_ids:
            pipe.exists(f"presence:online:{uid}")
        results = await pipe.execute()
        return {uid: bool(online) for uid, online in zip(user_ids, results)}


presence = PresenceService()


@app.websocket("/ws/{user_id}")
async def ws_endpoint(ws: WebSocket, user_id: str):
    await ws.accept()
    await presence.set_online(user_id)
    # Notify room members this user came online
    await r.publish("presence:changes", f"{user_id}:online")

    try:
        while True:
            try:
                data = await asyncio.wait_for(ws.receive_json(), timeout=HEARTBEAT_INTERVAL + 5)
                if data.get("type") == "heartbeat":
                    # Renew TTL — user is still active
                    await r.setex(f"presence:online:{user_id}", PRESENCE_TTL, "1")
                    await ws.send_json({"type": "heartbeat_ack", "ts": time.time()})
            except asyncio.TimeoutError:
                # Client missed heartbeat — connection may be stale
                await ws.send_json({"type": "ping"})

    except WebSocketDisconnect:
        await presence.set_offline(user_id)


@app.get("/presence/{user_id}")
async def get_presence(user_id: str):
    return await presence.get_last_seen(user_id)


@app.post("/presence/bulk")
async def bulk_presence(user_ids: list[str]):
    """Get presence for a list of users — used when opening a DM conversation."""
    return await presence.bulk_presence(user_ids)


# ── Presence change subscriber (notifies connected clients) ──────────────────
async def presence_broadcaster():
    """
    Subscribes to presence change events and notifies affected WebSocket rooms.
    In production: this logic lives in the WebSocket handler per room.
    """
    sub = r.pubsub()
    await sub.subscribe("presence:changes")
    async for msg in sub.listen():
        if msg["type"] != "message":
            continue
        # Format: "{user_id}:{online|offline}"
        user_id, status = msg["data"].rsplit(":", 1)
        print(f"[Presence] {user_id} is now {status}")
        # In production: find all rooms this user is in → notify members
```

---

## Exercise 4: Long Polling — Simple Implementation

```python
"""
Long polling endpoint: client hangs until new data is available or timeout.
Simulates a notification inbox.

pip install fastapi asyncio
"""
import asyncio
import time
import uuid
from collections import defaultdict

from fastapi import FastAPI

app = FastAPI()

# Per-user event queues (in-memory; use Redis in production)
_queues: dict[str, asyncio.Queue] = defaultdict(asyncio.Queue)


async def wait_for_event(user_id: str, timeout: float = 30.0) -> list[dict]:
    """
    Blocks until an event is available or timeout expires.
    Returns list of pending events.
    """
    q = _queues[user_id]

    try:
        # Block until first event arrives or timeout
        first = await asyncio.wait_for(q.get(), timeout=timeout)
        events = [first]

        # Drain any additional events that arrived simultaneously
        while not q.empty():
            events.append(q.get_nowait())

        return events

    except asyncio.TimeoutError:
        return []   # Empty list = no new events, client should re-poll


@app.get("/poll/{user_id}")
async def long_poll(user_id: str, since: float = 0):
    """
    Long polling endpoint.
    Client calls this and waits up to 30 seconds for events.
    On timeout: returns empty list → client immediately re-polls.
    On events: returns events → client processes → re-polls.
    """
    events = await wait_for_event(user_id, timeout=29.0)
    return {
        "events":    events,
        "timestamp": time.time(),
        "has_more":  not _queues[user_id].empty(),
    }


@app.post("/notify/{user_id}")
async def push_notification(user_id: str, message: str):
    """Simulate pushing a notification to a user (would be called by other services)."""
    event = {
        "id":      str(uuid.uuid4()),
        "type":    "notification",
        "content": message,
        "ts":      time.time(),
    }
    await _queues[user_id].put(event)
    return {"status": "queued", "event_id": event["id"]}


# Client-side simulation:
# import httpx, asyncio
# async def client():
#     while True:
#         resp = await httpx.AsyncClient().get("http://localhost:8000/poll/alice")
#         data = resp.json()
#         if data["events"]:
#             print("New events:", data["events"])
#         # Re-poll immediately (whether timeout or events received)
```

---

## Exercise 5: Typing Indicator with Debounce

```python
"""
Typing indicator: "Alice is typing..." that disappears after 3s of inactivity.
Demonstrates: ephemeral real-time state, debounce via Redis TTL, fan-out.

Uses the WebSocket server from Exercise 1 as the base.
"""
import asyncio
import json
import time
import redis.asyncio as aioredis

REDIS_URL       = "redis://localhost:6379"
TYPING_TTL      = 3    # Seconds before "typing" expires automatically
r: aioredis.Redis = None


class TypingIndicator:
    """
    Tracks typing state per user per room.
    Uses Redis TTL to auto-expire "is typing" state without requiring
    an explicit "stopped typing" message from the client.
    """

    @staticmethod
    async def start_typing(user_id: str, room_id: str):
        """
        Called when user types a keystroke.
        Resets the TTL on each keystroke — expires after TYPING_TTL seconds of silence.
        """
        key = f"typing:{room_id}:{user_id}"
        await r.setex(key, TYPING_TTL, "1")

        # Publish to room channel so other servers can notify their connections
        await r.publish(f"room:{room_id}:typing", json.dumps({
            "user_id":   user_id,
            "room_id":   room_id,
            "is_typing": True,
            "ts":        time.time(),
        }))

    @staticmethod
    async def stop_typing(user_id: str, room_id: str):
        """Explicit stop (e.g., message sent or input cleared)."""
        await r.delete(f"typing:{room_id}:{user_id}")
        await r.publish(f"room:{room_id}:typing", json.dumps({
            "user_id":   user_id,
            "room_id":   room_id,
            "is_typing": False,
            "ts":        time.time(),
        }))

    @staticmethod
    async def who_is_typing(room_id: str) -> list[str]:
        """Returns list of user_ids currently typing in this room."""
        pattern = f"typing:{room_id}:*"
        keys = []
        async for key in r.scan_iter(pattern):
            keys.append(key)
        # Extract user_id from key: "typing:{room_id}:{user_id}"
        return [k.split(":")[-1] for k in keys]


# ── Integration with WebSocket handler ───────────────────────────────────────
async def handle_ws_message(data: dict, user_id: str, room_id: str,
                             ws, broadcast_fn):
    typing = TypingIndicator()

    if data["type"] == "typing":
        # Client sends this on every keypress (debounced client-side to ~500ms)
        await typing.start_typing(user_id, room_id)

    elif data["type"] == "stop_typing":
        await typing.stop_typing(user_id, room_id)

    elif data["type"] == "message":
        # Sending a message implicitly stops typing
        await typing.stop_typing(user_id, room_id)
        # ... handle message delivery ...

    elif data["type"] == "who_typing":
        # Client can request current typing state on join
        typers = await typing.who_is_typing(room_id)
        await ws.send_json({"type": "typing_state", "users": typers})


# ── Demo: simulate typing ─────────────────────────────────────────────────────
async def demo():
    global r
    r = aioredis.from_url(REDIS_URL, decode_responses=True)
    typing = TypingIndicator()

    print("Alice starts typing...")
    await typing.start_typing("alice", "room_1")
    print("Typing in room_1:", await typing.who_is_typing("room_1"))

    print("Bob also starts typing...")
    await typing.start_typing("bob", "room_1")
    print("Typing in room_1:", await typing.who_is_typing("room_1"))

    print("Waiting 4s (TTL expires for both)...")
    await asyncio.sleep(4)
    print("Typing in room_1 after TTL:", await typing.who_is_typing("room_1"))

    print("Alice sends message (explicit stop)...")
    await typing.start_typing("alice", "room_1")
    await typing.stop_typing("alice", "room_1")
    print("Typing in room_1:", await typing.who_is_typing("room_1"))

    await r.close()


if __name__ == "__main__":
    asyncio.run(demo())
```
