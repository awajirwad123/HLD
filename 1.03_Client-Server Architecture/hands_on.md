# Client-Server Architecture — Hands-On

## Exercise 1: Stateless API — JWT-Based Identity

A fully stateless FastAPI server. No server-side session — identity travels in the JWT.

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt  # pip install pyjwt
import time

app = FastAPI()
security = HTTPBearer()

SECRET_KEY = "super-secret-change-in-prod"  # use env var in real code
ALGORITHM = "HS256"

def create_token(user_id: str) -> str:
    payload = {
        "sub": user_id,
        "iat": int(time.time()),
        "exp": int(time.time()) + 3600,  # 1 hour
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def get_current_user(creds: HTTPAuthorizationCredentials = Depends(security)) -> str:
    try:
        payload = jwt.decode(creds.credentials, SECRET_KEY, algorithms=[ALGORITHM])
        return payload["sub"]
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.post("/login")
def login(username: str, password: str):
    # In reality: validate credentials against DB
    if username == "alice" and password == "secret":
        return {"token": create_token(user_id=username)}
    raise HTTPException(status_code=401, detail="Invalid credentials")

@app.get("/profile")
def get_profile(user_id: str = Depends(get_current_user)):
    # No session lookup needed — user_id came from the token
    return {"user_id": user_id, "message": "Fully stateless - any server can serve this"}
```

**Key insight:** Any server in the cluster can verify this JWT without talking to a central session store — that's what makes it stateless and horizontally scalable.

---

## Exercise 2: Stateful Problem Demo — Why Sticky Sessions Break Scaling

```python
# BAD: in-memory session state (stateful server - DON'T do this in production)
from fastapi import FastAPI, Request, Response
import uuid

app = FastAPI()
sessions: dict[str, dict] = {}  # stored in THIS server's RAM

@app.post("/login-bad")
def login_bad(username: str, response: Response):
    session_id = str(uuid.uuid4())
    sessions[session_id] = {"user": username, "cart": []}  # ❌ only on this server
    response.set_cookie("session_id", session_id)
    return {"message": "logged in"}

@app.get("/cart-bad")
def get_cart(request: Request):
    session_id = request.cookies.get("session_id")
    session = sessions.get(session_id)  # ❌ fails if routed to different server
    if not session:
        return {"error": "session not found - you hit a different server!"}
    return session["cart"]
```

```python
# GOOD: externalized session state (stateless server - Redis-backed)
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

@app.post("/login-good")
def login_good(username: str, response: Response):
    session_id = str(uuid.uuid4())
    r.setex(f"session:{session_id}", 3600, json.dumps({"user": username, "cart": []}))
    response.set_cookie("session_id", session_id)
    return {"message": "logged in"}

@app.get("/cart-good")
def get_cart_good(request: Request):
    session_id = request.cookies.get("session_id")
    data = r.get(f"session:{session_id}")  # ✅ works from any server
    if not data:
        return {"error": "session expired"}
    return json.loads(data)["cart"]
```

---

## Exercise 3: Horizontal Scaling Simulation

Simulate how a load balancer distributes requests across multiple stateless servers:

```python
import random
from dataclasses import dataclass, field

@dataclass
class Server:
    id: str
    request_count: int = 0

    def handle(self, request_id: int) -> str:
        self.request_count += 1
        return f"Server {self.id} handled request #{request_id}"

class RoundRobinLoadBalancer:
    def __init__(self, servers: list[Server]):
        self.servers = servers
        self._index = 0

    def route(self, request_id: int) -> str:
        server = self.servers[self._index % len(self.servers)]
        self._index += 1
        return server.handle(request_id)

    def stats(self):
        for s in self.servers:
            print(f"  {s.id}: {s.request_count} requests")

# Simulate 100 requests
servers = [Server("A"), Server("B"), Server("C")]
lb = RoundRobinLoadBalancer(servers)

for i in range(1, 101):
    lb.route(i)

print("Load distribution:")
lb.stats()
# Output: A: 34, B: 33, C: 33 — perfectly balanced
```

---

## Exercise 4: Retry with Exponential Backoff + Jitter

Preventing thundering herd when a server recovers:

```python
import time
import random
import httpx  # pip install httpx

def fetch_with_retry(url: str, max_retries: int = 5) -> dict:
    """
    Exponential backoff with jitter to avoid thundering herd.
    Wait: 1s, 2s, 4s, 8s, 16s (with ±25% jitter)
    """
    for attempt in range(max_retries):
        try:
            response = httpx.get(url, timeout=5.0)
            response.raise_for_status()
            return response.json()
        except (httpx.HTTPError, httpx.TimeoutException) as e:
            if attempt == max_retries - 1:
                raise

            base_wait = 2 ** attempt          # 1, 2, 4, 8, 16
            jitter = random.uniform(0.75, 1.25)
            wait = base_wait * jitter
            print(f"Attempt {attempt + 1} failed ({e}). Retrying in {wait:.2f}s...")
            time.sleep(wait)

# Usage:
# data = fetch_with_retry("https://api.example.com/feed")
```

**Why jitter?** Without it, all retrying clients fire at the exact same time (thundering herd), overwhelming the recovering server. Jitter spreads them out.

---

## Exercise 5: Health Check Endpoint (FastAPI)

Every horizontally scaled server needs a health check for the load balancer:

```python
from fastapi import FastAPI
import redis
import time

app = FastAPI()
start_time = time.time()

r = redis.Redis(host="localhost", port=6379)

@app.get("/health")
def health_check():
    """
    Load balancers poll this endpoint.
    Return 200 = healthy, include to load balancer pool.
    Return 503 = unhealthy, remove from pool.
    """
    checks = {}

    # Check Redis connectivity
    try:
        r.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {e}"

    # Overall status
    all_ok = all(v == "ok" for v in checks.values())
    status_code = 200 if all_ok else 503

    return {
        "status": "healthy" if all_ok else "unhealthy",
        "uptime_seconds": int(time.time() - start_time),
        "checks": checks,
    }, status_code
```

**Real-world:** AWS ALB, Nginx, Kubernetes liveness probes all poll `/health`. If it returns non-200, the instance is removed from rotation automatically.

---

## Exercise 6: Circuit Breaker Pattern in Python

Prevents cascading failures when a downstream service is unhealthy.

```python
import time
import random
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"         # normal — requests flow through
    OPEN = "open"             # failing — fast-fail all requests
    HALF_OPEN = "half_open"   # recovery probe — let one request through

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_threshold = failure_threshold  # failures before opening
        self.recovery_timeout = recovery_timeout    # seconds before trying again
        self.failure_count = 0
        self.last_failure_time: float = 0
        self.state = CircuitState.CLOSED

    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                print("  [CIRCUIT] Half-open — sending probe request")
            else:
                raise Exception("Circuit OPEN — fast failing (not calling service)")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            print("  [CIRCUIT] Probe succeeded — closing circuit")
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
            print(f"  [CIRCUIT] Opened after {self.failure_count} failures")


# Simulate a flaky downstream service
def flaky_service_call(fail: bool):
    if fail:
        raise ConnectionError("Downstream service unavailable")
    return {"data": "ok"}

# Demo
cb = CircuitBreaker(failure_threshold=3, recovery_timeout=5)

# Simulate 5 failures → circuit opens
for i in range(5):
    try:
        cb.call(flaky_service_call, fail=True)
    except Exception as e:
        print(f"  Request {i+1}: {e}")

# Simulate fast-fail while open
try:
    cb.call(flaky_service_call, fail=False)
except Exception as e:
    print(f"  Fast-failed: {e}")

# Wait for recovery, then probe succeeds
time.sleep(6)
try:
    result = cb.call(flaky_service_call, fail=False)
    print(f"  Recovered: {result}")
except Exception as e:
    print(f"  Still failing: {e}")
```

**Expected output:**
```
  Request 1: Downstream service unavailable
  Request 2: Downstream service unavailable
  Request 3: [CIRCUIT] Opened after 3 failures
             Downstream service unavailable
  Fast-failed: Circuit OPEN — fast failing (not calling service)
  [CIRCUIT] Half-open — sending probe request
  [CIRCUIT] Probe succeeded — closing circuit
  Recovered: {'data': 'ok'}
```

**Interview answer:**
> *"I'd wrap every inter-service call in a circuit breaker. When a downstream service fails N times, we stop calling it and fast-fail immediately — this prevents thread pool exhaustion and cascading failures. After a timeout, we send a probe request. If it succeeds, we close the circuit and resume normal traffic."*

---

## Exercise 7: Readiness vs Liveness Probes (Kubernetes Context)

Two distinct health checks — know the difference for system design:

```python
from fastapi import FastAPI
import redis
import time

app = FastAPI()
r = redis.Redis(host="localhost", port=6379)
start_time = time.time()
_ready = False  # set to True after startup tasks complete

@app.on_event("startup")
async def startup():
    global _ready
    # Simulate warm-up: load config, prime cache, run migrations check
    time.sleep(2)
    _ready = True

@app.get("/healthz/live")
def liveness():
    """
    Kubernetes liveness probe.
    Question: Is the process alive and not deadlocked?
    If this fails → Kubernetes RESTARTS the container.
    Keep it cheap: just return 200. Don't check dependencies.
    """
    return {"status": "alive"}

@app.get("/healthz/ready")
def readiness():
    """
    Kubernetes readiness probe.
    Question: Is the app ready to serve traffic?
    If this fails → Kubernetes REMOVES from load balancer (no restart).
    Check dependencies here.
    """
    if not _ready:
        return {"status": "not ready", "reason": "still warming up"}, 503
    try:
        r.ping()
    except Exception:
        return {"status": "not ready", "reason": "redis unavailable"}, 503
    return {"status": "ready"}
```

**Key distinction:**
| Probe      | Failure action             | What to check                      |
|------------|----------------------------|------------------------------------|
| Liveness   | Restart container          | Is process alive? (cheap check)    |
| Readiness  | Remove from load balancer  | Can it serve traffic? (deps check) |

> "Never put dependency checks in the liveness probe — if Redis is down, you don't want ALL your pods to restart in a loop."
