# Consistency Models — Hands-On Exercises

## Exercise 1: Demonstrate Each Consistency Model with Simulated Replicas

```python
"""
Simulates all five key consistency models on a 3-node cluster.
Each model is a different read policy on top of the same underlying store.
"""
import time
import threading
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class VersionedValue:
    value: object
    version: int
    written_at: float = field(default_factory=time.time)


@dataclass
class Node:
    node_id: str
    store: dict = field(default_factory=dict)   # key → VersionedValue
    lag_ms: float = 0.0

    def write(self, key, value, version):
        time.sleep(self.lag_ms / 1000)
        self.store[key] = VersionedValue(value, version)

    def read(self, key) -> Optional[VersionedValue]:
        return self.store.get(key)


class DistributedStore:
    def __init__(self):
        self.primary = Node("primary")
        self.replica_a = Node("replica_a", lag_ms=20)   # 20ms behind
        self.replica_b = Node("replica_b", lag_ms=100)  # 100ms behind
        self._version = 0
        self._client_versions: dict[str, int] = {}  # session_id → min version seen

    def write(self, key, value):
        """Write to primary; async replicate to replicas."""
        self._version += 1
        v = self._version
        self.primary.write(key, value, v)

        # Async replication — starts immediately but arrives later
        for replica in [self.replica_a, self.replica_b]:
            t = threading.Thread(target=replica.write, args=(key, value, v), daemon=True)
            t.start()

        return v  # Return version for session tracking

    # ── Consistency Model Implementations ──────────────────────────────────

    def read_linearizable(self, key):
        """Always read from primary — guaranteed to have the latest write."""
        result = self.primary.read(key)
        return {
            "model": "Linearizable",
            "value": result.value if result else None,
            "version": result.version if result else 0,
            "node": "primary",
            "guarantee": "Always current — no stale reads possible"
        }

    def read_eventual(self, key, prefer_replica="a"):
        """Read from any replica — may be stale but always responds."""
        node = self.replica_a if prefer_replica == "a" else self.replica_b
        result = node.read(key)
        return {
            "model": "Eventual",
            "value": result.value if result else None,
            "version": result.version if result else 0,
            "node": node.node_id,
            "guarantee": "May be stale; will converge eventually"
        }

    def read_your_writes(self, key, session_id: str, written_version: int = 0):
        """
        Return value only from a node that has applied at least `written_version`.
        Falls back to primary if no replica is current enough.
        """
        min_ver = max(self._client_versions.get(session_id, 0), written_version)

        for node in [self.replica_a, self.primary]:
            result = node.read(key)
            if result and result.version >= min_ver:
                self._client_versions[session_id] = result.version
                return {
                    "model": "Read-Your-Writes",
                    "value": result.value,
                    "version": result.version,
                    "node": node.node_id,
                    "guarantee": f"At least as new as client's last write (v{min_ver})"
                }

        return {"model": "Read-Your-Writes", "value": None, "error": "No sufficiently fresh replica available"}

    def read_monotonic(self, key, session_id: str):
        """Never return a version older than what this session has already seen."""
        min_seen = self._client_versions.get(session_id, 0)

        for node in [self.replica_a, self.replica_b, self.primary]:
            result = node.read(key)
            if result and result.version >= min_seen:
                self._client_versions[session_id] = result.version
                return {
                    "model": "Monotonic Reads",
                    "value": result.value,
                    "version": result.version,
                    "node": node.node_id,
                    "guarantee": f"Version ≥ {min_seen} (no time travel)"
                }

        return {"model": "Monotonic Reads", "value": None, "error": "All replicas behind session state"}

    def read_causal(self, key, depends_on_version: int = 0):
        """
        Only return result from a node that has seen `depends_on_version`.
        Simulates: 'only show reply if you've seen the original post'.
        """
        for node in [self.replica_a, self.replica_b, self.primary]:
            result = node.read(key)
            if result and result.version >= depends_on_version:
                return {
                    "model": "Causal",
                    "value": result.value,
                    "version": result.version,
                    "node": node.node_id,
                    "guarantee": f"Causally after v{depends_on_version}"
                }

        return {"model": "Causal", "value": None,
                "warning": f"No replica has applied causal predecessor v{depends_on_version} yet"}


def run_demo():
    store = DistributedStore()

    print("=== Write X=42 to primary ===")
    v = store.write("X", 42)
    print(f"Written version: {v}\n")

    # Wait for replica_a (20ms lag) but NOT replica_b (100ms lag)
    time.sleep(0.025)

    print("=== Read immediately after write (replica_b still not synced) ===")
    print(store.read_linearizable("X"))
    print(store.read_eventual("X", prefer_replica="b"))   # stale — 100ms replica
    print(store.read_your_writes("X", session_id="alice", written_version=v))
    print(store.read_monotonic("X", session_id="alice"))
    print(store.read_causal("X", depends_on_version=v))

    print("\n=== After 200ms (all replicas converged) ===")
    time.sleep(0.2)
    print(store.read_eventual("X", prefer_replica="b"))   # now fresh


if __name__ == "__main__":
    run_demo()
```

---

## Exercise 2: Causal Consistency — Post and Reply Ordering

```python
"""
Demonstrates causal consistency with vector clocks.
Ensures a reply is never visible without its parent post.
"""
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class VectorClock:
    counters: dict[str, int] = field(default_factory=dict)

    def increment(self, node_id: str) -> "VectorClock":
        new = VectorClock(dict(self.counters))
        new.counters[node_id] = new.counters.get(node_id, 0) + 1
        return new

    def merge(self, other: "VectorClock") -> "VectorClock":
        all_keys = set(self.counters) | set(other.counters)
        return VectorClock({k: max(self.counters.get(k, 0), other.counters.get(k, 0))
                            for k in all_keys})

    def happened_before(self, other: "VectorClock") -> bool:
        """Returns True if self causally precedes other."""
        all_keys = set(self.counters) | set(other.counters)
        return (
            all(self.counters.get(k, 0) <= other.counters.get(k, 0) for k in all_keys)
            and any(self.counters.get(k, 0) < other.counters.get(k, 0) for k in all_keys)
        )

    def __repr__(self):
        return str(self.counters)


@dataclass
class Message:
    msg_id: str
    content: str
    author: str
    clock: VectorClock
    parent_id: Optional[str] = None   # None = top-level post


class CausalMessageStore:
    """
    A message store that enforces causal consistency:
    a reply is only visible if the message it replies to has been delivered.
    """
    def __init__(self):
        self.delivered: dict[str, Message] = {}   # msg_id → Message
        self.pending: list[Message] = []           # waiting for causal predecessors

    def receive(self, msg: Message):
        """Accept a message; buffer it if its causal dependencies aren't met yet."""
        if msg.parent_id is None or msg.parent_id in self.delivered:
            self._deliver(msg)
            self._try_deliver_pending()
        else:
            print(f"  🔁 BUFFERING reply '{msg.msg_id}' — parent '{msg.parent_id}' not yet delivered")
            self.pending.append(msg)

    def _deliver(self, msg: Message):
        self.delivered[msg.msg_id] = msg
        print(f"  ✅ DELIVERED [{msg.author}]: {msg.content!r} (clock={msg.clock})")

    def _try_deliver_pending(self):
        """Re-check buffered messages after each delivery."""
        changed = True
        while changed:
            changed = False
            for msg in list(self.pending):
                if msg.parent_id in self.delivered:
                    self.pending.remove(msg)
                    self._deliver(msg)
                    changed = True


def demo_causal():
    print("=== Causal Consistency Demo: Post + Reply ===\n")
    store = CausalMessageStore()

    vc_alice = VectorClock()
    vc_bob = VectorClock()

    # Alice posts
    vc_alice = vc_alice.increment("alice")
    post = Message("post_1", "Anyone going to the meetup?", "alice", vc_alice)

    # Bob reads and replies
    vc_bob = vc_bob.merge(vc_alice).increment("bob")
    reply = Message("reply_1", "Yes, I'll be there!", "bob", vc_bob, parent_id="post_1")

    print("Scenario A: Reply arrives BEFORE post (out-of-order delivery)")
    print("→ Causal store BUFFERS reply until post arrives\n")
    store.receive(reply)   # Reply arrives first
    store.receive(post)    # Post arrives second → triggers buffer flush

    print("\nScenario B: Normal order (post first, then reply)")
    store2 = CausalMessageStore()
    store2.receive(post)
    store2.receive(reply)


if __name__ == "__main__":
    demo_causal()
```

---

## Exercise 3: Read-Your-Writes in FastAPI with Session Tokens

```python
"""
Production-ready read-your-writes implementation using version tokens.
After a write, the client receives a version token.
Subsequent reads include the token: server routes to the first replica
that has applied at least that version.
"""
import asyncio
import time
from fastapi import FastAPI, Header, Response
from typing import Optional
import asyncpg

app = FastAPI()

PRIMARY_DSN  = "postgresql://user:pw@primary:5432/db"
REPLICA_DSNS = [
    "postgresql://user:pw@replica1:5432/db",
    "postgresql://user:pw@replica2:5432/db",
]

# Version token: client receives the WAL LSN after a write
# Subsequent reads include it in X-Read-After-Version header
# We route to a replica whose WAL has advanced past that LSN

async def get_replica_for_version(required_lsn: Optional[str]) -> asyncpg.Connection:
    """
    Return a connection to the first replica that has applied `required_lsn`.
    Fallback to primary if no replica qualifies.
    """
    if not required_lsn:
        # No requirement — use first available replica
        conn = await asyncpg.connect(REPLICA_DSNS[0])
        return conn, "replica"

    for dsn in REPLICA_DSNS:
        try:
            conn = await asyncpg.connect(dsn)
            # Check if this replica has applied the required LSN
            row = await conn.fetchrow(
                "SELECT pg_last_wal_apply_lsn() >= $1::pg_lsn AS is_current",
                required_lsn
            )
            if row["is_current"]:
                return conn, "replica"
            await conn.close()
        except Exception:
            continue

    # Fallback: primary is always current
    conn = await asyncpg.connect(PRIMARY_DSN)
    return conn, "primary"


@app.put("/users/{user_id}/bio")
async def update_bio(user_id: str, bio: str, response: Response):
    conn = await asyncpg.connect(PRIMARY_DSN)
    try:
        await conn.execute(
            "UPDATE users SET bio = $1 WHERE id = $2",
            bio, user_id
        )
        # Capture the current WAL LSN — this is our version token
        lsn = await conn.fetchval("SELECT pg_current_wal_lsn()::text")
        # Return the version token to the client as a response header
        response.headers["X-Written-At-Version"] = lsn
        return {"status": "updated", "version_token": lsn}
    finally:
        await conn.close()


@app.get("/users/{user_id}")
async def get_user(user_id: str, x_read_after_version: Optional[str] = Header(None)):
    """
    x_read_after_version: LSN token from previous write.
    If provided, only serve from a replica that has applied that LSN.
    """
    conn, source = await get_replica_for_version(x_read_after_version)
    try:
        row = await conn.fetchrow(
            "SELECT id, email, bio FROM users WHERE id = $1", user_id
        )
        result = dict(row) if row else {}
        result["_served_from"] = source
        result["_version_required"] = x_read_after_version
        return result
    finally:
        await conn.close()
```

---

## Exercise 4: Monotonic Reads — Detect and Prevent Time-Travel

```python
"""
Monotonic reads implementation at the client level.
Tracks the minimum version seen per key and refuses reads that would go backward.
"""
import time
import random
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class Versioned:
    value: object
    version: int


class Replica:
    def __init__(self, name: str, initial_version: int = 0):
        self.name = name
        self.store: dict[str, Versioned] = {}
        self._current_version = initial_version

    def apply_up_to(self, version: int, key: str, value):
        """Simulate applying writes up to a given version."""
        if version > (self.store.get(key, Versioned(None, 0)).version):
            self.store[key] = Versioned(value, version)
            self._current_version = max(self._current_version, version)

    def read(self, key) -> Optional[Versioned]:
        return self.store.get(key)


class MonotonicReadSession:
    """
    A client session that enforces monotonic reads.
    Never returns a version older than what was seen in this session.
    """
    def __init__(self, replicas: list[Replica]):
        self.replicas = replicas
        self._min_versions: dict[str, int] = {}
        self.violations_prevented = 0

    def read(self, key: str):
        min_ver = self._min_versions.get(key, 0)

        # Shuffle to simulate load balancing (requests may hit different replicas)
        candidates = list(self.replicas)
        random.shuffle(candidates)

        for replica in candidates:
            result = replica.read(key)
            if result is None:
                continue
            if result.version >= min_ver:
                # This replica is fresh enough
                self._min_versions[key] = result.version
                return {
                    "value": result.value,
                    "version": result.version,
                    "from": replica.name,
                    "monotonic": "✅ guaranteed"
                }
            else:
                # This replica would time-travel — skip it
                self.violations_prevented += 1
                print(f"  ⚠️ Skipping {replica.name}: version {result.version} < session minimum {min_ver}")

        # No replica was fresh enough — escalate to most-up-to-date replica
        best = max(candidates, key=lambda r: (r.read(key) or Versioned(None, -1)).version)
        result = best.read(key)
        if result:
            self._min_versions[key] = result.version
            return {
                "value": result.value,
                "version": result.version,
                "from": best.name,
                "monotonic": "✅ escalated to freshest replica"
            }

        return {"value": None, "from": "none", "monotonic": "❌ read failed"}


def demo():
    # Three replicas at different versions
    r1 = Replica("replica_1")
    r2 = Replica("replica_2")
    r3 = Replica("replica_3")

    # Simulate writes that have propagated to some replicas but not all
    r1.apply_up_to(5, "username", "alice_v5")
    r2.apply_up_to(3, "username", "alice_v3")   # behind
    r3.apply_up_to(5, "username", "alice_v5")

    session = MonotonicReadSession([r1, r2, r3])

    print("=== Monotonic Read Demo ===\n")
    print("Read 1:", session.read("username"))

    # Advance r1 to version 6
    r1.apply_up_to(6, "username", "alice_v6")

    print("Read 2:", session.read("username"))  # must see ≥ version 5

    # Simulate r1 being slow — force r2 to respond first
    print("\nForcing r2 (stale) to respond first:")
    session2 = MonotonicReadSession([r2, r1, r3])  # r2 first in order
    result = session2.read("username")
    print("Read from r2-biased order:", result)

    print(f"\nTime-travel violations prevented: {session.violations_prevented + session2.violations_prevented}")


if __name__ == "__main__":
    demo()
```

---

## Exercise 5: Eventual Consistency — Conflict Resolution Strategies

```python
"""
Demonstrates three conflict resolution strategies for eventual consistency:
1. Last Write Wins (LWW)
2. Multi-value (conflict surfaces to application)
3. CRDT G-Counter (conflict-free by design)
"""
import time
from dataclasses import dataclass, field
from typing import Union


# ── Strategy 1: Last Write Wins ─────────────────────────────────────────────
@dataclass
class LWWRegister:
    """Last Write Wins — highest timestamp wins. Simple, but loses data silently."""
    value: object = None
    timestamp: float = 0.0

    def write(self, value, ts: float = None):
        ts = ts or time.time()
        if ts > self.timestamp:
            self.value = value
            self.timestamp = ts

    def merge(self, other: "LWWRegister"):
        if other.timestamp > self.timestamp:
            self.value = other.value
            self.timestamp = other.timestamp

    def read(self):
        return self.value


# ── Strategy 2: Multi-Value Register (conflict visible) ─────────────────────
@dataclass
class MVRegister:
    """Surfaces conflicts to the application. Client must resolve."""
    versions: list = field(default_factory=list)  # list of (value, vector_clock)

    def write(self, value, clock: dict):
        # Discard dominated values; keep concurrent ones
        surviving = [(v, c) for v, c in self.versions
                     if not self._dominates(clock, c)]  # keep if clock doesn't dominate
        surviving.append((value, clock))
        self.versions = surviving

    def _dominates(self, a: dict, b: dict) -> bool:
        all_keys = set(a) | set(b)
        return (all(a.get(k, 0) >= b.get(k, 0) for k in all_keys)
                and any(a.get(k, 0) > b.get(k, 0) for k in all_keys))

    def read(self) -> Union[object, list]:
        if len(self.versions) == 1:
            return self.versions[0][0]
        elif len(self.versions) == 0:
            return None
        else:
            values = [v for v, _ in self.versions]
            print(f"  ⚠️ CONFLICT: {values} — application must resolve")
            return values  # Surface conflict


# ── Strategy 3: CRDT G-Counter (conflict-free) ───────────────────────────────
@dataclass
class GCounter:
    """
    Grow-only counter CRDT.
    Each node tracks its own increment count.
    Merge = element-wise max.
    No conflicts possible — convergence guaranteed.
    """
    counts: dict[str, int] = field(default_factory=dict)  # node_id → count

    def increment(self, node_id: str, by: int = 1):
        self.counts[node_id] = self.counts.get(node_id, 0) + by

    def value(self) -> int:
        return sum(self.counts.values())

    def merge(self, other: "GCounter") -> "GCounter":
        all_keys = set(self.counts) | set(other.counts)
        merged = {k: max(self.counts.get(k, 0), other.counts.get(k, 0))
                  for k in all_keys}
        return GCounter(merged)


def demo():
    print("=== Conflict Resolution Strategy Demo ===\n")

    # 1. LWW
    print("--- Last Write Wins ---")
    r_a = LWWRegister()
    r_b = LWWRegister()
    r_a.write("alice", ts=1000.0)
    r_b.write("bob",   ts=1001.0)  # Concurrent write on partition
    r_a.merge(r_b)
    r_b.merge(r_a)
    print(f"  Node A sees: {r_a.read()}")  # bob (later timestamp wins)
    print(f"  Node B sees: {r_b.read()}")  # bob
    print(f"  ⚠️ 'alice' write SILENTLY LOST\n")

    # 2. Multi-Value
    print("--- Multi-Value Register ---")
    mv_a = MVRegister()
    mv_b = MVRegister()
    mv_a.write("alice", {"A": 1, "B": 0})
    mv_b.write("bob",   {"A": 0, "B": 1})   # Concurrent on partition
    # After reconciliation: both values survive as conflicting siblings
    for val, clk in mv_b.versions:
        mv_a.write(val, clk)
    print(f"  Merged state: {mv_a.read()}")  # Application must pick winner
    print()

    # 3. G-Counter CRDT
    print("--- CRDT G-Counter (like counts) ---")
    counter_a = GCounter()
    counter_b = GCounter()
    counter_a.increment("node_a", 5)   # 5 likes on replica A
    counter_b.increment("node_b", 3)   # 3 likes on replica B (partition)
    merged = counter_a.merge(counter_b)
    print(f"  Node A count: {counter_a.value()}")   # 5
    print(f"  Node B count: {counter_b.value()}")   # 3
    print(f"  Merged count: {merged.value()}")       # 8 — no conflict!
    print(f"  ✅ G-Counter always converges to correct total")


if __name__ == "__main__":
    demo()
```
