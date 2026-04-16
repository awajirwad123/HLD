# Service Discovery — Hands-On Exercises

## Exercise 1: In-Memory Service Registry with Health Checks

```python
import time
import threading
import random
from dataclasses import dataclass, field


@dataclass
class ServiceInstance:
    service: str
    host: str
    port: int
    tags: list[str] = field(default_factory=list)
    last_heartbeat: float = field(default_factory=time.time)
    health_status: str = "healthy"   # healthy | unhealthy | unknown

    @property
    def address(self) -> str:
        return f"{self.host}:{self.port}"

    def is_alive(self, ttl_seconds: float = 30.0) -> bool:
        return (time.time() - self.last_heartbeat) < ttl_seconds


class ServiceRegistry:
    """
    In-memory service registry with:
    - TTL-based deregistration (no heartbeat → mark unhealthy)
    - Filtering by health status
    - Round-robin selection
    """

    def __init__(self, ttl_seconds: float = 30.0):
        self._registry: dict[str, dict[str, ServiceInstance]] = {}
        self._lock = threading.RLock()
        self._ttl = ttl_seconds
        self._rr_counters: dict[str, int] = {}

    # ─── Registration ─────────────────────────────────────────

    def register(self, instance: ServiceInstance):
        with self._lock:
            if instance.service not in self._registry:
                self._registry[instance.service] = {}
            key = instance.address
            self._registry[instance.service][key] = instance
            print(f"[registry] Registered {instance.service} @ {instance.address}")

    def deregister(self, service: str, address: str):
        with self._lock:
            if service in self._registry:
                self._registry[service].pop(address, None)
                print(f"[registry] Deregistered {service} @ {address}")

    def heartbeat(self, service: str, address: str):
        with self._lock:
            inst = self._registry.get(service, {}).get(address)
            if inst:
                inst.last_heartbeat = time.time()
                inst.health_status = "healthy"

    # ─── Discovery ────────────────────────────────────────────

    def get_healthy(self, service: str) -> list[ServiceInstance]:
        with self._lock:
            instances = self._registry.get(service, {}).values()
            return [i for i in instances if i.is_alive(self._ttl) and i.health_status == "healthy"]

    def resolve(self, service: str) -> ServiceInstance | None:
        """Round-robin selection among healthy instances."""
        healthy = self.get_healthy(service)
        if not healthy:
            return None
        idx = self._rr_counters.get(service, 0) % len(healthy)
        self._rr_counters[service] = idx + 1
        return healthy[idx]

    # ─── Background TTL Sweeper ───────────────────────────────

    def start_health_sweep(self, interval: float = 10.0):
        def sweep():
            while True:
                time.sleep(interval)
                with self._lock:
                    for service, instances in list(self._registry.items()):
                        for addr, inst in list(instances.items()):
                            if not inst.is_alive(self._ttl):
                                inst.health_status = "unhealthy"
                                print(f"[registry] Marked unhealthy: {service} @ {addr} (no heartbeat)")
        t = threading.Thread(target=sweep, daemon=True)
        t.start()

    def status(self):
        with self._lock:
            for service, instances in self._registry.items():
                print(f"\nService: {service}")
                for addr, inst in instances.items():
                    age = time.time() - inst.last_heartbeat
                    print(f"  {addr:20s}  status={inst.health_status:10s}  last_hb={age:.1f}s ago")


# ─── Demo ─────────────────────────────────────────────────────

def simulate_service_node(registry: ServiceRegistry, service: str, host: str, port: int,
                           stay_alive_seconds: float):
    """Simulates a service instance: registers, sends heartbeats, then dies."""
    inst = ServiceInstance(service=service, host=host, port=port)
    registry.register(inst)
    address = inst.address
    deadline = time.time() + stay_alive_seconds
    while time.time() < deadline:
        time.sleep(5)
        registry.heartbeat(service, address)
    registry.deregister(service, address)


if __name__ == "__main__":
    registry = ServiceRegistry(ttl_seconds=20)
    registry.start_health_sweep(interval=5)

    # Start 3 payment-service instances
    for i, port in enumerate([8080, 8081, 8082]):
        stay = 40 if i < 2 else 12   # Third instance "dies" early
        t = threading.Thread(
            target=simulate_service_node,
            args=(registry, "payment-service", "10.0.1." + str(i+1), port, stay),
            daemon=True
        )
        t.start()

    time.sleep(2)   # Let them register

    print("\n=== Initial Resolution (3 instances) ===")
    for _ in range(6):
        inst = registry.resolve("payment-service")
        if inst:
            print(f"  Resolved: {inst.address}")

    print("\n[Waiting 20s for instance 3 to die and TTL to expire...]\n")
    time.sleep(22)

    print("=== Resolution after instance 3 died ===")
    for _ in range(4):
        inst = registry.resolve("payment-service")
        if inst:
            print(f"  Resolved: {inst.address}")
        else:
            print("  No healthy instance!")

    registry.status()
```

---

## Exercise 2: DNS-Based Discovery Simulator

```python
import time
import random
from collections import defaultdict

class DnsRegistry:
    """Simulates DNS-based service discovery with TTL caching."""

    def __init__(self):
        self._records: dict[str, list[str]] = {}    # name → [ip, ...]
        self._cache: dict[str, tuple[list[str], float]] = {}  # name → (ips, expiry)

    def register(self, name: str, ip: str):
        self._records.setdefault(name, []).append(ip)

    def deregister(self, name: str, ip: str):
        self._records[name] = [i for i in self._records[name] if i != ip]

    def resolve(self, name: str, ttl: float = 10.0) -> list[str]:
        """Returns cached IPs if not expired, else re-queries."""
        cached = self._cache.get(name)
        if cached and time.time() < cached[1]:
            return cached[0]   # Cache hit

        # "DNS query"
        ips = list(self._records.get(name, []))
        if ips:
            self._cache[name] = (ips, time.time() + ttl)
        return ips


def demo_dns_stale_cache():
    dns = DnsRegistry()
    dns.register("order-service.svc", "10.0.1.1")
    dns.register("order-service.svc", "10.0.1.2")
    dns.register("order-service.svc", "10.0.1.3")

    print("Initial DNS lookup:")
    ips = dns.resolve("order-service.svc", ttl=30)
    print(f"  {ips}")

    # Simulate 10.0.1.3 dying
    dns.deregister("order-service.svc", "10.0.1.3")
    print("\nInstance 10.0.1.3 deregistered (but cache TTL not expired):")
    ips = dns.resolve("order-service.svc", ttl=30)
    print(f"  {ips}  ← 10.0.1.3 still returned (stale cache)")

    # Force cache expiry
    dns._cache.clear()
    print("\nAfter cache expiry:")
    ips = dns.resolve("order-service.svc", ttl=30)
    print(f"  {ips}  ← 10.0.1.3 gone")

    print("\nKey insight: short TTL = fresher data but more DNS queries")
    for ttl, qps_factor in [(5, "HIGH"), (30, "MEDIUM"), (300, "LOW")]:
        print(f"  TTL={ttl:3d}s  DNS query rate: {qps_factor}")


demo_dns_stale_cache()
```

---

## Exercise 3: Consistent Hashing for Service Discovery

Client-side load balancing using consistent hashing (ideal for cache-affinity routing — sticky routing of the same user to the same backend):

```python
import hashlib
from bisect import bisect, insort

class ConsistentHashRouter:
    """
    Routes a key (user_id, session_id) consistently to the same
    backend instance across scale-up/scale-down events.
    """

    def __init__(self, virtual_nodes: int = 100):
        self._ring: list[int] = []
        self._map: dict[int, str] = {}
        self._vnodes = virtual_nodes

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        for i in range(self._vnodes):
            h = self._hash(f"{node}:{i}")
            insort(self._ring, h)
            self._map[h] = node

    def remove_node(self, node: str):
        for i in range(self._vnodes):
            h = self._hash(f"{node}:{i}")
            if h in self._map:
                self._ring.remove(h)
                del self._map[h]

    def get_node(self, key: str) -> str | None:
        if not self._ring:
            return None
        h = self._hash(key)
        idx = bisect(self._ring, h) % len(self._ring)
        return self._map[self._ring[idx]]

    def distribution(self, keys: list[str]) -> dict[str, int]:
        counts: dict[str, int] = {}
        for k in keys:
            node = self.get_node(k)
            if node:
                counts[node] = counts.get(node, 0) + 1
        return counts


def demo_router():
    router = ConsistentHashRouter(virtual_nodes=150)
    nodes = ["backend-1", "backend-2", "backend-3"]
    for n in nodes:
        router.add_node(n)

    keys = [f"user:{i}" for i in range(10_000)]
    dist = router.distribution(keys)
    print("Distribution across 3 nodes:")
    for node, count in sorted(dist.items()):
        bar = "█" * (count // 100)
        print(f"  {node}: {count:5d} {bar}")

    # Add a 4th node
    router.add_node("backend-4")
    new_dist = router.distribution(keys)
    moved = sum(1 for k in keys if router.get_node(k) != [
        n for n, c in dist.items() for _ in range(c)
        if k in [f"user:{i}" for i in range(c)]
    ])

    print("\nAfter adding backend-4:")
    for node, count in sorted(new_dist.items()):
        bar = "█" * (count // 100)
        print(f"  {node}: {count:5d} {bar}")
    print(f"  ~{10000 // 4} keys remapped (expected ~25% = 1/N)")


demo_router()
```
