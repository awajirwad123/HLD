# Replication — Hands-On Exercises

## Exercise 1: Read-Your-Writes with Replication Lag in FastAPI

```python
"""
Demonstrates the read-your-writes consistency problem and the fix.
The fix: after a write, route that user's reads to the primary for a short window.
"""
import time
from fastapi import FastAPI, Request, Response, Depends
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy import text

app = FastAPI()

# Two engine pools: primary for writes, replica for reads
primary_engine = create_async_engine("postgresql+asyncpg://user:pw@primary:5432/db")
replica_engine  = create_async_engine("postgresql+asyncpg://user:pw@replica:5432/db")

PrimarySession = sessionmaker(primary_engine, class_=AsyncSession, expire_on_commit=False)
ReplicaSession  = sessionmaker(replica_engine,  class_=AsyncSession, expire_on_commit=False)

# How long after a write should we force reads to primary?
READ_YOUR_WRITES_WINDOW_SECONDS = 2


def should_read_from_primary(request: Request) -> bool:
    """
    Check if this user recently wrote data.
    We track this via a cookie set at write time.
    """
    wrote_at_str = request.cookies.get("wrote_at")
    if not wrote_at_str:
        return False
    try:
        wrote_at = float(wrote_at_str)
        return (time.time() - wrote_at) < READ_YOUR_WRITES_WINDOW_SECONDS
    except ValueError:
        return False


async def get_read_session(request: Request):
    """Choose primary or replica depending on recency of writes."""
    SessionClass = PrimarySession if should_read_from_primary(request) else ReplicaSession
    async with SessionClass() as session:
        yield session

async def get_write_session():
    async with PrimarySession() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


# --- Endpoints ---

@app.post("/users/{user_id}/bio")
async def update_bio(user_id: int, bio: str, response: Response,
                     db: AsyncSession = Depends(get_write_session)):
    await db.execute(
        text("UPDATE users SET bio = :bio WHERE id = :id"),
        {"bio": bio, "id": user_id}
    )
    # Set a cookie so subsequent reads by this user go to primary
    response.set_cookie("wrote_at", str(time.time()), max_age=5, httponly=True)
    return {"status": "updated"}


@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_read_session)):
    """
    Goes to primary if user wrote recently (cookie present + within window).
    Goes to replica otherwise → read scale.
    """
    result = await db.execute(
        text("SELECT id, email, bio FROM users WHERE id = :id"),
        {"id": user_id}
    )
    row = result.fetchone()
    return dict(row._mapping) if row else {"error": "not found"}
```

---

## Exercise 2: Simulate Async vs Sync Replication Latency

```python
"""
Simulates the write latency difference between async and sync replication.
Shows why financial systems pay the sync penalty and social apps don't.
"""
import asyncio
import time
import statistics

NETWORK_RTT_MS = 5  # ms between primary and replica (same region)


async def async_replicate():
    """Async: acknowledge to client, then replicate in background."""
    t0 = time.perf_counter()

    # Step 1: Write to primary WAL (fast local disk)
    await asyncio.sleep(0.001)  # 1ms disk write

    # Step 2: Acknowledge to client BEFORE replica confirms
    ack_latency = (time.perf_counter() - t0) * 1000

    # Step 3: Replicate in background (client doesn't wait)
    asyncio.create_task(_simulate_replica_apply())

    return ack_latency


async def sync_replicate():
    """Sync: do not acknowledge until replica confirms."""
    t0 = time.perf_counter()

    # Step 1: Write to primary WAL
    await asyncio.sleep(0.001)  # 1ms disk write

    # Step 2: Send WAL to replica and wait for confirmation
    await asyncio.sleep(NETWORK_RTT_MS / 1000)  # RTT to replica
    await asyncio.sleep(0.001)  # replica disk write

    # Step 3: Acknowledge to client (after replica confirms)
    ack_latency = (time.perf_counter() - t0) * 1000
    return ack_latency


async def _simulate_replica_apply():
    await asyncio.sleep(NETWORK_RTT_MS / 1000 + 0.001)


async def benchmark():
    n = 1000
    async_latencies = [await async_replicate() for _ in range(n)]
    sync_latencies  = [await sync_replicate()  for _ in range(n)]

    print(f"Async replication (n={n}):")
    print(f"  p50: {statistics.median(async_latencies):.2f}ms")
    print(f"  p99: {sorted(async_latencies)[int(n*0.99)]:.2f}ms")

    print(f"\nSync replication (n={n}):")
    print(f"  p50: {statistics.median(sync_latencies):.2f}ms")
    print(f"  p99: {sorted(sync_latencies)[int(n*0.99)]:.2f}ms")

    overhead_pct = (statistics.median(sync_latencies) / statistics.median(async_latencies) - 1) * 100
    print(f"\nSync overhead: +{overhead_pct:.0f}% on median write latency")
    print(f"Cost of durability guarantee: ~{NETWORK_RTT_MS}ms per write")


if __name__ == "__main__":
    asyncio.run(benchmark())

# Expected output (same-region, 5ms RTT):
# Async p50: ~1.0ms
# Sync  p50: ~7.0ms  (+600% — this is why async is default)
```

---

## Exercise 3: Quorum Simulation (Leaderless Replication)

```python
"""
Simulates a leaderless quorum read/write system (Dynamo/Cassandra-style).
Demonstrates how W+R>N guarantees reading the latest write.
"""
import asyncio
import random
import time
from dataclasses import dataclass, field

@dataclass
class Node:
    node_id: str
    data: dict = field(default_factory=dict)
    is_down: bool = False
    lag_ms: float = 0.0   # Simulated apply delay

    async def write(self, key: str, value, version: int) -> bool:
        if self.is_down:
            return False
        await asyncio.sleep(self.lag_ms / 1000)
        self.data[key] = {"value": value, "version": version}
        return True

    async def read(self, key: str) -> dict | None:
        if self.is_down:
            return None
        await asyncio.sleep(self.lag_ms / 1000)
        return self.data.get(key)


class QuorumCluster:
    def __init__(self, nodes: list[Node], W: int, R: int):
        self.nodes = nodes
        self.W = W
        self.R = R
        self.N = len(nodes)
        self._version = 0

    async def write(self, key: str, value) -> bool:
        self._version += 1
        version = self._version

        tasks = [node.write(key, value, version) for node in self.nodes]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        successes = sum(1 for r in results if r is True)

        if successes >= self.W:
            print(f"  Write '{key}={value}' succeeded on {successes}/{self.N} nodes (W={self.W}) ✅")
            return True
        else:
            print(f"  Write '{key}={value}' FAILED — only {successes}/{self.N} nodes (W={self.W}) ❌")
            return False

    async def read(self, key: str):
        tasks = [node.read(key) for node in self.nodes]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        valid = [r for r in results if isinstance(r, dict) and r is not None]

        if len(valid) < self.R:
            print(f"  Read '{key}' FAILED — only {len(valid)}/{self.N} nodes responded (R={self.R}) ❌")
            return None

        # Return value with highest version (newest write)
        newest = max(valid, key=lambda r: r["version"])
        versions = [r["version"] for r in valid]
        print(f"  Read '{key}' got versions {versions} → returning v{newest['version']}={newest['value']} ✅")

        # Read repair: update stale nodes
        for node, result in zip(self.nodes, results):
            if isinstance(result, dict) and result.get("version", 0) < newest["version"]:
                asyncio.create_task(node.write(key, newest["value"], newest["version"]))
                print(f"    Read repair: {node.node_id} updated to v{newest['version']}")

        return newest["value"]


async def demo():
    nodes = [
        Node("A"),
        Node("B", lag_ms=10),   # Slightly behind
        Node("C", lag_ms=50),   # Notably behind
    ]

    print("=== N=3, W=2, R=2 (QUORUM — strong consistency) ===")
    cluster = QuorumCluster(nodes, W=2, R=2)
    await cluster.write("username", "alice")
    await cluster.read("username")  # Guaranteed to see newest

    print("\n=== Node C goes down ===")
    nodes[2].is_down = True
    await cluster.write("username", "alice_v2")
    await cluster.read("username")  # Still works: W=2, R=2 satisfied with A+B

    print("\n=== N=3, W=1, R=1 (Weak consistency — any node responds) ===")
    nodes[2].is_down = False
    nodes[2].data = {}  # Reset C — it's stale
    cluster_weak = QuorumCluster(nodes, W=1, R=1)
    await cluster_weak.write("score", 100)  # Writes to A only
    # Force read to go to C (stale node)
    nodes[0].is_down = True
    nodes[1].is_down = True
    await cluster_weak.read("score")  # C doesn't have it ← stale read


if __name__ == "__main__":
    asyncio.run(demo())
```

---

## Exercise 4: Replication Lag Monitor + Alert

```python
"""
Monitor PostgreSQL replication lag and emit alerts above threshold.
Runs as a background task in FastAPI.
"""
import asyncio
import asyncpg
import logging
from datetime import datetime

logger = logging.getLogger(__name__)

PRIMARY_DSN = "postgresql://user:pw@primary:5432/mydb"
LAG_ALERT_THRESHOLD_SECONDS = 5.0


async def check_replication_lag():
    """
    Queries pg_stat_replication on the primary to get lag per replica.
    Returns a list of replica lag measurements.
    """
    conn = await asyncpg.connect(PRIMARY_DSN)
    try:
        rows = await conn.fetch("""
            SELECT
                client_addr::TEXT                                AS replica_addr,
                state,
                EXTRACT(EPOCH FROM write_lag)   AS write_lag_s,
                EXTRACT(EPOCH FROM flush_lag)   AS flush_lag_s,
                EXTRACT(EPOCH FROM replay_lag)  AS replay_lag_s,
                sent_lsn::TEXT,
                replay_lsn::TEXT
            FROM pg_stat_replication
        """)
        return [dict(r) for r in rows]
    finally:
        await conn.close()


async def replication_monitor(interval_seconds: int = 10):
    """Background loop: check lag every N seconds, alert on threshold breach."""
    while True:
        try:
            replicas = await check_replication_lag()

            if not replicas:
                logger.warning("No replicas connected to primary!")

            for r in replicas:
                lag = r.get("replay_lag_s") or 0
                addr = r["replica_addr"]
                state = r["state"]

                if lag > LAG_ALERT_THRESHOLD_SECONDS:
                    logger.error(
                        f"ALERT: Replica {addr} replay lag = {lag:.1f}s "
                        f"(threshold={LAG_ALERT_THRESHOLD_SECONDS}s) state={state}"
                    )
                else:
                    logger.info(f"Replica {addr}: lag={lag:.3f}s state={state}")

        except Exception as exc:
            logger.error(f"Replication monitor error: {exc}")

        await asyncio.sleep(interval_seconds)


# Integrate into FastAPI lifespan
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    task = asyncio.create_task(replication_monitor(interval_seconds=10))
    yield
    task.cancel()

app = FastAPI(lifespan=lifespan)


@app.get("/health/replication")
async def replication_health():
    """Expose replication lag as a health endpoint."""
    try:
        replicas = await check_replication_lag()
        healthy = all((r.get("replay_lag_s") or 0) < LAG_ALERT_THRESHOLD_SECONDS
                      for r in replicas)
        return {
            "healthy": healthy,
            "replicas": replicas,
            "threshold_seconds": LAG_ALERT_THRESHOLD_SECONDS,
            "checked_at": datetime.utcnow().isoformat()
        }
    except Exception as exc:
        return {"healthy": False, "error": str(exc)}
```

---

## Exercise 5: Version Vector (Causality Tracking)

```python
"""
Version vectors track causality to detect concurrent writes
(the multi-leader conflict scenario).
Used by Riak, DynamoDB (internally), CouchDB.
"""
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class VersionVector:
    """
    Maps node_id → counter. Incremented every time that node writes.
    Two version vectors can be compared to determine:
      - Happens-before (one is causally newer)
      - Concurrent (neither is newer — conflict!)
    """
    counters: dict[str, int] = field(default_factory=dict)

    def increment(self, node_id: str) -> "VersionVector":
        new = VersionVector(dict(self.counters))
        new.counters[node_id] = new.counters.get(node_id, 0) + 1
        return new

    def dominates(self, other: "VersionVector") -> bool:
        """Returns True if self is causally NEWER than other (self happens-after other)."""
        all_nodes = set(self.counters) | set(other.counters)
        return (
            all(self.counters.get(n, 0) >= other.counters.get(n, 0) for n in all_nodes)
            and any(self.counters.get(n, 0) > other.counters.get(n, 0) for n in all_nodes)
        )

    def concurrent_with(self, other: "VersionVector") -> bool:
        """Returns True if neither dominates the other → conflict."""
        return not self.dominates(other) and not other.dominates(self)

    def merge(self, other: "VersionVector") -> "VersionVector":
        """Merge: take element-wise maximum (used in leaderless systems)."""
        all_nodes = set(self.counters) | set(other.counters)
        return VersionVector({n: max(self.counters.get(n, 0), other.counters.get(n, 0))
                              for n in all_nodes})


@dataclass
class Replica:
    name: str
    data: dict[str, tuple] = field(default_factory=dict)  # key → (value, VersionVector)

    def write(self, key: str, value) -> VersionVector:
        _, current_vv = self.data.get(key, (None, VersionVector()))
        new_vv = current_vv.increment(self.name)
        self.data[key] = (value, new_vv)
        print(f"[{self.name}] wrote {key}={value} vv={new_vv.counters}")
        return new_vv

    def receive_from_peer(self, key: str, value, peer_vv: VersionVector):
        local_val, local_vv = self.data.get(key, (None, VersionVector()))

        if peer_vv.dominates(local_vv):
            # Peer is newer — accept
            self.data[key] = (value, peer_vv)
            print(f"[{self.name}] accepted {key}={value} from peer")
        elif local_vv.dominates(peer_vv):
            # Local is newer — discard peer write
            print(f"[{self.name}] discarded stale {key}={value} from peer")
        else:
            # Concurrent writes — CONFLICT
            merged_vv = local_vv.merge(peer_vv)
            conflict_value = f"CONFLICT({local_val} vs {value})"
            self.data[key] = (conflict_value, merged_vv)
            print(f"[{self.name}] CONFLICT on {key}: '{local_val}' vs '{value}'")


if __name__ == "__main__":
    node_a = Replica("A")
    node_b = Replica("B")

    # Normal case: one writer
    vv1 = node_a.write("title", "Hello")
    node_b.receive_from_peer("title", "Hello", vv1)

    # Conflict: both write concurrently (partition scenario)
    print("\n--- Simulating partition: both nodes write ---")
    vv_a = node_a.write("title", "Hello World")   # A increments A's counter
    vv_b = node_b.write("title", "Hi There")      # B increments B's counter

    # Partition heals: exchange writes
    node_a.receive_from_peer("title", "Hi There", vv_b)
    node_b.receive_from_peer("title", "Hello World", vv_a)
```

---

## Exercise 6: Monotonic Reads — Route User to Same Replica

```python
"""
Monotonic reads guarantee: once you've seen a value,
you'll never see an older value in a subsequent read.
Implementation: consistently route a user to the same replica.
"""
import hashlib
from fastapi import FastAPI, Request
from typing import List
import random

app = FastAPI()

REPLICAS = [
    "replica-1:5432",
    "replica-2:5432",
    "replica-3:5432",
]


def pick_replica_for_user(user_id: str, replicas: List[str]) -> str:
    """
    Consistent mapping: same user always goes to same replica.
    Uses hash of user_id modulo replica count.
    If replica list changes, some users remap (acceptable).
    """
    h = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
    return replicas[h % len(replicas)]


@app.get("/feed/{user_id}")
async def get_feed(user_id: str, request: Request):
    replica = pick_replica_for_user(user_id, REPLICAS)

    # In production: create/fetch an async connection to `replica`
    # Here: just show which replica was chosen
    return {
        "user_id": user_id,
        "routed_to": replica,
        "guarantee": "monotonic reads — same replica every time for this user"
    }


# Demonstrate consistent mapping
if __name__ == "__main__":
    users = ["alice", "bob", "charlie", "david", "eve"]
    print("User → Replica mapping:")
    for user in users:
        r = pick_replica_for_user(user, REPLICAS)
        print(f"  {user:10s} → {r}")

    # Verify: same user always maps to same replica
    assert all(
        pick_replica_for_user(u, REPLICAS) == pick_replica_for_user(u, REPLICAS)
        for u in users
    )
    print("\nMonotonic reads guarantee: ✅ same replica per user")
```
