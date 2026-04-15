# Load Balancing — Hands-On

## Exercise 1: Simulate Load Balancing Algorithms in Python

Compare Round Robin, Weighted Round Robin, and Least Connections side-by-side:

```python
from dataclasses import dataclass, field
from collections import deque
import random

@dataclass
class Server:
    id: str
    weight: int = 1
    active_connections: int = 0
    request_count: int = 0

    def handle(self, req_id: int) -> str:
        self.request_count += 1
        return f"[{self.id}] handled req #{req_id}"


# ── Algorithm 1: Round Robin ──────────────────────────────────────────────────
class RoundRobinLB:
    def __init__(self, servers: list[Server]):
        self.servers = servers
        self._idx = 0

    def route(self, req_id: int) -> str:
        s = self.servers[self._idx % len(self.servers)]
        self._idx += 1
        return s.handle(req_id)


# ── Algorithm 2: Weighted Round Robin ────────────────────────────────────────
class WeightedRoundRobinLB:
    def __init__(self, servers: list[Server]):
        # Expand pool: server with weight=3 appears 3 times
        self._pool = [s for s in servers for _ in range(s.weight)]
        self._idx = 0

    def route(self, req_id: int) -> str:
        s = self._pool[self._idx % len(self._pool)]
        self._idx += 1
        return s.handle(req_id)


# ── Algorithm 3: Least Connections ───────────────────────────────────────────
class LeastConnectionsLB:
    def __init__(self, servers: list[Server]):
        self.servers = servers

    def route(self, req_id: int) -> str:
        s = min(self.servers, key=lambda x: x.active_connections)
        s.active_connections += 1
        result = s.handle(req_id)
        # Simulate request completion (random duration, connection freed)
        if random.random() > 0.5:
            s.active_connections = max(0, s.active_connections - 1)
        return result


# ── Demo ──────────────────────────────────────────────────────────────────────
def print_distribution(label: str, servers: list[Server]):
    print(f"\n{label}")
    total = sum(s.request_count for s in servers)
    for s in servers:
        pct = s.request_count / total * 100
        print(f"  {s.id}: {s.request_count} reqs ({pct:.1f}%)")

N = 100

# Round Robin
servers_rr = [Server("A"), Server("B"), Server("C")]
lb = RoundRobinLB(servers_rr)
for i in range(N): lb.route(i)
print_distribution("Round Robin (equal weight)", servers_rr)

# Weighted Round Robin (A gets 2x traffic)
servers_wrr = [Server("A", weight=2), Server("B", weight=1), Server("C", weight=1)]
lb = WeightedRoundRobinLB(servers_wrr)
for i in range(N): lb.route(i)
print_distribution("Weighted RR (A=2, B=1, C=1)", servers_wrr)

# Least Connections
servers_lc = [Server("A"), Server("B"), Server("C")]
lb = LeastConnectionsLB(servers_lc)
for i in range(N): lb.route(i)
print_distribution("Least Connections", servers_lc)
```

---

## Exercise 2: Active Health Check Simulator

Simulates how an LB monitors backend health and removes/reinstates servers:

```python
import time
import random
import threading
from dataclasses import dataclass, field

@dataclass
class Backend:
    id: str
    healthy: bool = True
    consecutive_failures: int = 0
    consecutive_successes: int = 0

    def ping(self) -> bool:
        """Simulate health check — 30% chance of failure when marked unhealthy."""
        if not self.healthy:
            return random.random() > 0.7  # 30% chance health probe succeeds
        return random.random() > 0.05     # 5% false-negative rate when healthy

class LoadBalancer:
    def __init__(
        self,
        backends: list[Backend],
        check_interval: float = 2.0,
        unhealthy_threshold: int = 3,
        healthy_threshold: int = 2,
    ):
        self.backends = backends
        self.check_interval = check_interval
        self.unhealthy_threshold = unhealthy_threshold
        self.healthy_threshold = healthy_threshold
        self._active: list[Backend] = list(backends)
        self._idx = 0

    def route(self) -> str | None:
        healthy = [b for b in self.backends if b in self._active]
        if not healthy:
            return None
        b = healthy[self._idx % len(healthy)]
        self._idx += 1
        return b.id

    def _health_check_loop(self):
        while True:
            for b in self.backends:
                ok = b.ping()
                if ok:
                    b.consecutive_failures = 0
                    b.consecutive_successes += 1
                    if b not in self._active and b.consecutive_successes >= self.healthy_threshold:
                        self._active.append(b)
                        print(f"  ✅ {b.id} reinstated (healthy threshold reached)")
                else:
                    b.consecutive_successes = 0
                    b.consecutive_failures += 1
                    if b in self._active and b.consecutive_failures >= self.unhealthy_threshold:
                        self._active.remove(b)
                        print(f"  ❌ {b.id} removed (unhealthy threshold reached)")
            time.sleep(self.check_interval)

    def start_health_checks(self):
        t = threading.Thread(target=self._health_check_loop, daemon=True)
        t.start()


# Demo
backends = [Backend("Server-A"), Backend("Server-B"), Backend("Server-C")]
lb = LoadBalancer(backends, check_interval=1.0)
lb.start_health_checks()

# Simulate Server-B going down
time.sleep(0.5)
backends[1].healthy = False
print("Server-B set to unhealthy...")

time.sleep(5)
# Route some traffic — B should be gone by now
print("\nRouting 10 requests:")
for i in range(10):
    dest = lb.route()
    print(f"  Request {i+1} → {dest}")

# Bring Server-B back
backends[1].healthy = True
print("\nServer-B recovering...")
time.sleep(4)
print("\nRouting 5 more requests:")
for i in range(5):
    print(f"  Request {i+1} → {lb.route()}")
```

---

## Exercise 3: Nginx Load Balancer Config (Reference)

Know how to describe this in interviews or system diagrams:

```nginx
# /etc/nginx/nginx.conf

upstream api_servers {
    # Round robin (default)
    server 10.0.1.10:8000;
    server 10.0.1.11:8000;
    server 10.0.1.12:8000;

    # Or: least connections
    # least_conn;

    # Or: weighted
    # server 10.0.1.10:8000 weight=3;
    # server 10.0.1.11:8000 weight=1;

    # Health checks (Nginx Plus feature; use passive in open source)
    keepalive 32;  # persistent connections to backends
}

server {
    listen 443 ssl;
    server_name api.example.com;

    # TLS termination here
    ssl_certificate /etc/ssl/certs/api.crt;
    ssl_certificate_key /etc/ssl/private/api.key;
    ssl_protocols TLSv1.3;

    location / {
        proxy_pass http://api_servers;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;

        # Timeouts
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;

        # Retry on failure
        proxy_next_upstream error timeout http_500 http_502 http_503;
        proxy_next_upstream_tries 2;
    }

    # Static assets → separate upstream or CDN
    location /static/ {
        proxy_pass http://cdn_origin;
        proxy_cache_valid 200 1d;
    }
}
```

---

## Exercise 4: Path-Based Routing (Layer 7 — AWS ALB style)

Layer 7 LBs route requests based on URL path. Here's a FastAPI simulation:

```python
from fastapi import FastAPI, Request
import httpx

# Simulated API gateway that routes based on path prefix
gateway = FastAPI(title="API Gateway")

SERVICE_MAP = {
    "/api/users":   "http://user-service:8001",
    "/api/feed":    "http://feed-service:8002",
    "/api/media":   "http://media-service:8003",
}

@gateway.api_route("/{path:path}", methods=["GET", "POST", "PUT", "DELETE"])
async def route_request(path: str, request: Request):
    """
    Layer 7 routing: match request path to backend service.
    In production: AWS ALB listener rules do this natively.
    """
    full_path = f"/{path}"
    target_base = None

    for prefix, service_url in SERVICE_MAP.items():
        if full_path.startswith(prefix):
            target_base = service_url
            break

    if not target_base:
        return {"error": "No route found", "path": full_path}, 404

    # Forward request to the matched service
    target_url = f"{target_base}{full_path}"
    body = await request.body()
    headers = dict(request.headers)
    headers["X-Forwarded-For"] = request.client.host

    # In production: LB does this at network level, not in application code
    print(f"Routing {request.method} {full_path} → {target_url}")
    return {"routed_to": target_url, "method": request.method}
```

---

## Exercise 5: Connection Draining — Why It Matters

When removing a server (deploy, scale-in), in-flight requests must complete:

```
Without draining:
  Server B is being terminated
  LB immediately stops sending NEW requests ✅
  But 200 existing requests are killed mid-flight ❌ → 500 errors for clients

With draining (deregistration delay):
  Server B is being terminated
  LB stops sending NEW requests ✅
  Existing 200 requests complete normally ✅
  After 30s (deregistration delay), server B is forcibly terminated
```

**AWS ALB default:** 300 seconds deregistration delay. Reduce to 30s for fast deployments if requests are short-lived.

**FastAPI graceful shutdown:**
```python
import signal
import asyncio
from fastapi import FastAPI

app = FastAPI()
_shutdown = False

@app.on_event("shutdown")
async def shutdown_event():
    global _shutdown
    _shutdown = True
    # Wait for in-flight requests to complete
    # FastAPI/uvicorn handles SIGTERM gracefully by default
    print("Graceful shutdown: waiting for in-flight requests...")
```

**Interview answer:**
> *"I'd configure a 30-second connection draining window on the LB. When a server is deregistered (during a deploy or scale-in), the LB stops sending new requests but lets existing ones finish cleanly. Without this, rolling deployments cause 500 errors for users whose requests happen to be in-flight during the switch."*
