# Notification System — Hands-On Exercises

## Exercise 1: Notification Router with Preference Filtering

Build a notification dispatch engine that:
- Checks user preferences before sending
- Deduplicates within a time window
- Routes to appropriate channel workers

```python
import asyncio
import hashlib
import json
import time
from enum import Enum
from dataclasses import dataclass, field

import redis.asyncio as aioredis


class Channel(str, Enum):
    PUSH = "push"
    EMAIL = "email"
    SMS = "sms"
    IN_APP = "in_app"


class Priority(str, Enum):
    CRITICAL = "critical"   # fraud, OTP
    HIGH = "high"           # transactional
    NORMAL = "normal"       # social
    LOW = "low"             # marketing


@dataclass
class NotificationEvent:
    event_type: str          # "post_liked", "order_shipped", etc.
    recipient_id: int
    actor_id: int | None
    metadata: dict = field(default_factory=dict)
    priority: Priority = Priority.NORMAL


@dataclass
class UserPreferences:
    user_id: int
    enabled_channels: list[Channel]
    quiet_hours_start: int   # hour in UTC, e.g. 23 = 11pm
    quiet_hours_end: int     # hour in UTC, e.g. 7  = 7am
    opted_out_types: set[str] = field(default_factory=set)


# ---------- Simulated user preferences store ----------

USER_PREFS: dict[int, UserPreferences] = {
    101: UserPreferences(
        user_id=101,
        enabled_channels=[Channel.PUSH, Channel.IN_APP, Channel.EMAIL],
        quiet_hours_start=23,
        quiet_hours_end=7,
        opted_out_types={"post_liked"},
    ),
    102: UserPreferences(
        user_id=102,
        enabled_channels=[Channel.PUSH, Channel.EMAIL, Channel.SMS, Channel.IN_APP],
        quiet_hours_start=22,
        quiet_hours_end=8,
        opted_out_types=set(),
    ),
    103: UserPreferences(
        user_id=103,
        enabled_channels=[Channel.IN_APP],  # Only in-app
        quiet_hours_start=0,
        quiet_hours_end=0,
        opted_out_types=set(),
    ),
}

# Notification type → default channels to try (ordered by preference)
TYPE_TO_CHANNELS: dict[str, list[Channel]] = {
    "post_liked":      [Channel.PUSH, Channel.IN_APP],
    "comment":         [Channel.PUSH, Channel.IN_APP, Channel.EMAIL],
    "order_shipped":   [Channel.EMAIL, Channel.SMS, Channel.PUSH],
    "fraud_alert":     [Channel.SMS, Channel.PUSH],
    "otp":             [Channel.SMS],
    "promo":           [Channel.EMAIL, Channel.PUSH],
}

TEMPLATES: dict[str, str] = {
    "post_liked":    "{actor_name} liked your post",
    "comment":       "{actor_name} commented on your post",
    "order_shipped": "Your order #{order_id} has shipped!",
    "fraud_alert":   "Unusual activity detected on your account. Please verify.",
    "otp":           "Your verification code is {code}",
    "promo":         "Special offer just for you: {promo_text}",
}

ACTOR_NAMES: dict[int, str] = {1: "Alice", 2: "Bob", 3: "Carol"}


# ---------- Core logic ----------

def render_template(event: NotificationEvent) -> str:
    template = TEMPLATES.get(event.event_type, "You have a new notification")
    context = {**event.metadata}
    if event.actor_id:
        context["actor_name"] = ACTOR_NAMES.get(event.actor_id, f"User {event.actor_id}")
    # Format known placeholders, leave unknown ones as-is
    for key, val in context.items():
        template = template.replace("{" + key + "}", str(val))
    return template


def is_quiet_hours(prefs: UserPreferences) -> bool:
    """Check if current UTC hour falls within user's quiet window."""
    from datetime import datetime, timezone
    hour = datetime.now(timezone.utc).hour
    qs, qe = prefs.quiet_hours_start, prefs.quiet_hours_end
    if qs == qe == 0:
        return False  # No quiet hours configured
    if qs < qe:
        return qs <= hour < qe
    else:   # Wraps midnight: e.g. 23 → 7
        return hour >= qs or hour < qe


def compute_dedup_key(event: NotificationEvent, channel: Channel) -> str:
    raw = f"{event.event_type}:{event.recipient_id}:{event.actor_id}:{channel}"
    # Include entity ID if present (so "Alice liked post 1" and "Alice liked post 2" are different)
    entity_id = event.metadata.get("post_id") or event.metadata.get("order_id") or ""
    raw += f":{entity_id}"
    return "dedup:" + hashlib.sha256(raw.encode()).hexdigest()[:16]


class NotificationRouter:
    def __init__(self, redis_client: aioredis.Redis, dedup_ttl_seconds: int = 3600):
        self.redis = redis_client
        self.dedup_ttl = dedup_ttl_seconds
        self.dispatched: list[dict] = []  # In-memory log for demo

    async def route(self, event: NotificationEvent) -> list[dict]:
        """Returns list of dispatch results."""
        prefs = USER_PREFS.get(event.recipient_id)
        if not prefs:
            print(f"  [SKIP] No preferences found for user {event.recipient_id}")
            return []

        # 1. Check if user opted out of this notification type
        if event.event_type in prefs.opted_out_types:
            print(f"  [SKIP] User {event.recipient_id} opted out of '{event.event_type}'")
            return []

        # 2. Check quiet hours (only suppress non-critical)
        if event.priority not in (Priority.CRITICAL, Priority.HIGH) and is_quiet_hours(prefs):
            print(f"  [SKIP] User {event.recipient_id} in quiet hours")
            return []

        # 3. Determine target channels
        desired_channels = TYPE_TO_CHANNELS.get(event.event_type, [Channel.IN_APP])
        allowed_channels = [c for c in desired_channels if c in prefs.enabled_channels]

        if not allowed_channels:
            print(f"  [SKIP] User {event.recipient_id} has no matching enabled channels for '{event.event_type}'")
            return []

        # 4. Render message body
        body = render_template(event)

        results = []
        for channel in allowed_channels:
            # 5. Dedup check
            dkey = compute_dedup_key(event, channel)
            already_sent = await self.redis.set(dkey, "1", nx=True, ex=self.dedup_ttl)
            if not already_sent:
                print(f"  [DEDUP] {channel}: duplicate suppressed for user {event.recipient_id}")
                continue

            # 6. Dispatch to channel worker
            result = await self.dispatch(channel, event.recipient_id, body, event.priority)
            results.append({"channel": channel, "body": body, "result": result})

        return results

    async def dispatch(self, channel: Channel, user_id: int, body: str, priority: Priority) -> str:
        """Simulate dispatching. Real impl publishes to Kafka topic per channel."""
        print(f"  [SEND] channel={channel.value:8s} user={user_id}  priority={priority.value:8s}  body='{body}'")
        await asyncio.sleep(0.01)  # Simulated I/O
        return "delivered"


async def main():
    redis = aioredis.from_url("redis://localhost:6379", decode_responses=True)
    router = NotificationRouter(redis)

    events = [
        NotificationEvent("post_liked",  recipient_id=101, actor_id=1),       # user 101 opted out
        NotificationEvent("post_liked",  recipient_id=102, actor_id=1),
        NotificationEvent("post_liked",  recipient_id=102, actor_id=1),       # duplicate → deduped
        NotificationEvent("comment",     recipient_id=102, actor_id=2),
        NotificationEvent("order_shipped", recipient_id=102, actor_id=None,
                          metadata={"order_id": "ORD-999"}, priority=Priority.HIGH),
        NotificationEvent("fraud_alert", recipient_id=103, actor_id=None,
                          priority=Priority.CRITICAL),
        NotificationEvent("promo",       recipient_id=103, actor_id=None,
                          metadata={"promo_text": "50% off today!"}, priority=Priority.LOW),
    ]

    for event in events:
        print(f"\n→ Event: {event.event_type} → user {event.recipient_id}")
        await router.route(event)

    await redis.aclose()


if __name__ == "__main__":
    asyncio.run(main())
```

**Expected output:**
```
→ Event: post_liked → user 101
  [SKIP] User 101 opted out of 'post_liked'

→ Event: post_liked → user 102
  [SEND] channel=push     user=102  priority=normal    body='Alice liked your post'
  [SEND] channel=in_app   user=102  priority=normal    body='Alice liked your post'

→ Event: post_liked → user 102
  [DEDUP] push: duplicate suppressed for user 102
  [DEDUP] in_app: duplicate suppressed for user 102

→ Event: comment → user 102
  [SEND] channel=push     user=102  priority=normal    body='Bob commented on your post'
  ...

→ Event: fraud_alert → user 103
  [SEND] channel=in_app   user=103  priority=critical  body='Unusual activity detected...'
  # Critical bypasses quiet hours; user 103 only has in_app enabled
```

---

## Exercise 2: Retry Worker with Exponential Backoff + Dead Letter Queue

```python
import asyncio
import json
import random
import time
from dataclasses import dataclass, field

import redis.asyncio as aioredis

RETRY_DELAYS = [0, 30, 120, 600, 3600]  # seconds between retries (5 attempts)


@dataclass
class PendingDelivery:
    notification_id: str
    channel: str
    user_id: int
    payload: dict
    attempt: int = 0
    next_attempt_at: float = field(default_factory=time.time)

    def to_json(self) -> str:
        return json.dumps({
            "notification_id": self.notification_id,
            "channel": self.channel,
            "user_id": self.user_id,
            "payload": self.payload,
            "attempt": self.attempt,
            "next_attempt_at": self.next_attempt_at,
        })

    @classmethod
    def from_json(cls, raw: str) -> "PendingDelivery":
        d = json.loads(raw)
        return cls(**d)


DELIVERY_QUEUE = "notif:retry_queue"     # Redis Sorted Set (score = next_attempt_at)
DEAD_LETTER_QUEUE = "notif:dlq"          # Redis List


async def simulate_delivery(delivery: PendingDelivery) -> bool:
    """Simulate external API call (APNs/FCM/SendGrid/Twilio). 30% random failure."""
    await asyncio.sleep(0.05)
    return random.random() > 0.30


async def enqueue_delivery(redis: aioredis.Redis, delivery: PendingDelivery):
    await redis.zadd(DELIVERY_QUEUE, {delivery.to_json(): delivery.next_attempt_at})
    print(f"  [ENQUEUE] {delivery.notification_id} attempt={delivery.attempt} "
          f"via {delivery.channel} at t+{delivery.next_attempt_at - time.time():.0f}s")


async def retry_worker(redis: aioredis.Redis, worker_id: int = 0):
    """Polls the sorted set for deliveries ready to be sent (score ≤ now)."""
    print(f"Worker {worker_id} started")
    while True:
        now = time.time()
        # Atomically fetch up to 5 items whose score (scheduled_at) ≤ now
        items = await redis.zrangebyscore(DELIVERY_QUEUE, "-inf", now, start=0, num=5)

        if not items:
            await asyncio.sleep(1)
            continue

        for raw in items:
            # Remove from queue first (prevent double-delivery by other workers)
            removed = await redis.zrem(DELIVERY_QUEUE, raw)
            if removed == 0:
                continue  # Another worker already grabbed it

            delivery = PendingDelivery.from_json(raw)
            success = await simulate_delivery(delivery)

            if success:
                print(f"  [OK]     {delivery.notification_id} attempt={delivery.attempt} "
                      f"via {delivery.channel}")
            else:
                delivery.attempt += 1
                if delivery.attempt >= len(RETRY_DELAYS):
                    # Exhausted all retries → DLQ
                    await redis.rpush(DEAD_LETTER_QUEUE, delivery.to_json())
                    print(f"  [DLQ]    {delivery.notification_id} — all {delivery.attempt} attempts failed")
                else:
                    jitter = random.uniform(0, 5)
                    delay = RETRY_DELAYS[delivery.attempt] + jitter
                    delivery.next_attempt_at = time.time() + delay
                    await enqueue_delivery(redis, delivery)
                    print(f"  [RETRY]  {delivery.notification_id} attempt={delivery.attempt} "
                          f"in {delay:.1f}s")


async def main():
    redis = aioredis.from_url("redis://localhost:6379", decode_responses=True)

    # Seed some deliveries
    deliveries = [
        PendingDelivery(
            notification_id=f"notif-{i:03d}",
            channel=random.choice(["push", "email", "sms"]),
            user_id=100 + i,
            payload={"title": "Test notification", "body": f"Message {i}"},
        )
        for i in range(10)
    ]

    for d in deliveries:
        await enqueue_delivery(redis, d)

    # Run worker for a bit (exit after 30s for demo)
    try:
        await asyncio.wait_for(retry_worker(redis), timeout=30.0)
    except asyncio.TimeoutError:
        pass

    # Show DLQ contents
    dlq_size = await redis.llen(DEAD_LETTER_QUEUE)
    print(f"\nDead Letter Queue: {dlq_size} items")

    await redis.aclose()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Exercise 3: Aggregation / Digest Builder

```python
import asyncio
import time
import json
from collections import defaultdict

import redis.asyncio as aioredis

BUFFER_KEY_PREFIX = "notif:buffer:"   # notif:buffer:{user_id}:{type}
BUFFER_TTL = 300                       # 5-minute aggregation window
AGGREGATE_THRESHOLD = 3                # "A, B, and 2 others"


async def buffer_event(redis: aioredis.Redis, user_id: int, event_type: str, actor_name: str):
    """Add actor to a user's buffer list for this event type."""
    key = f"{BUFFER_KEY_PREFIX}{user_id}:{event_type}"
    await redis.rpush(key, actor_name)
    await redis.expire(key, BUFFER_TTL)  # Reset TTL on each new event
    print(f"  [BUFFER] user={user_id} type={event_type} actor={actor_name}")


async def flush_user_digest(redis: aioredis.Redis, user_id: int, event_type: str) -> str | None:
    """Build and clear digest message for this user+type buffer."""
    key = f"{BUFFER_KEY_PREFIX}{user_id}:{event_type}"
    actors = await redis.lrange(key, 0, -1)
    if not actors:
        return None
    await redis.delete(key)  # Clear after reading

    if len(actors) == 1:
        msg = f"{actors[0]} liked your post"
    elif len(actors) == 2:
        msg = f"{actors[0]} and {actors[1]} liked your post"
    else:
        others = len(actors) - 2
        msg = f"{actors[0]}, {actors[1]}, and {others} others liked your post"

    return msg


async def digest_scheduler(redis: aioredis.Redis):
    """Simulates a scheduled digest flush (would run as a cron job)."""
    # In reality, you'd query all buffer keys and send digests
    # Here we manually flush for demo users
    demo_users = [(101, "post_liked"), (102, "post_liked"), (102, "comment")]

    for user_id, event_type in demo_users:
        msg = await flush_user_digest(redis, user_id, event_type)
        if msg:
            print(f"  [DIGEST] → user {user_id}: '{msg}'")
        else:
            print(f"  [DIGEST] → user {user_id} type={event_type}: no events buffered")


async def main():
    redis = aioredis.from_url("redis://localhost:6379", decode_responses=True)

    # Simulate 5 likes coming in over 2 seconds
    print("=== Buffering Events ===")
    await buffer_event(redis, 101, "post_liked", "Alice")
    await buffer_event(redis, 101, "post_liked", "Bob")
    await buffer_event(redis, 101, "post_liked", "Carol")
    await buffer_event(redis, 101, "post_liked", "Dave")
    await buffer_event(redis, 102, "post_liked", "Eve")
    await buffer_event(redis, 102, "comment", "Frank")

    print("\n=== Running Digest Flush ===")
    await digest_scheduler(redis)

    await redis.aclose()


if __name__ == "__main__":
    asyncio.run(main())
```

**Expected output:**
```
=== Buffering Events ===
  [BUFFER] user=101 type=post_liked actor=Alice
  [BUFFER] user=101 type=post_liked actor=Bob
  [BUFFER] user=101 type=post_liked actor=Carol
  [BUFFER] user=101 type=post_liked actor=Dave
  [BUFFER] user=102 type=post_liked actor=Eve
  [BUFFER] user=102 type=comment actor=Frank

=== Running Digest Flush ===
  [DIGEST] → user 101: 'Alice, Bob, and 2 others liked your post'
  [DIGEST] → user 102: 'Eve liked your post'
  [DIGEST] → user 102 type=comment: 'Frank commented on your post'
```
