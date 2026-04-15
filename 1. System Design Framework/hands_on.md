# System Design Framework — Hands-On

## Goal

Practice the interview framework on a real problem end-to-end using Python/FastAPI examples where relevant.

---

## Exercise 1: Practice Requirement Clarification

**Prompt:** *"Design a notification system."*

Work through these questions yourself before looking at the answers:

### Questions You Should Ask

```
Functional:
  1. What types of notifications? (push, email, SMS, in-app)
  2. Who triggers them? (users, system events, admin campaigns)
  3. Is delivery order important?
  4. Do we need delivery confirmation / receipts?

Non-Functional:
  5. Scale — how many notifications per day?
  6. Latency — real-time or near-real-time?
  7. What delivery guarantee? (at-least-once OK? exactly-once needed?)
  8. Multi-region?
```

### Scoped Answer (Example)
- Core: push + email notifications, triggered by system events
- Scale: 10M notifications/day
- Latency: < 5 seconds for delivery
- At-least-once delivery acceptable
- Out of scope: SMS, analytics dashboard

---

## Exercise 2: Capacity Estimation — Practice

**Problem:** Design a photo-sharing app (Instagram-like).

Work through this:

```python
# Capacity estimation helper — reasoning in code comments

DAU = 50_000_000          # 50 million daily active users
PHOTOS_PER_USER_PER_DAY = 0.1  # 1 in 10 users uploads a photo

# Write QPS
write_per_day = DAU * PHOTOS_PER_USER_PER_DAY   # 5,000,000 writes/day
write_qps = write_per_day / 86_400              # ~58 QPS

# Read QPS (assume 100:1 read-write ratio)
read_qps = write_qps * 100                      # ~5,800 QPS

# Storage (per photo = 200KB avg compressed)
photo_size_bytes = 200 * 1024
storage_per_day = write_per_day * photo_size_bytes   # ~1 TB/day
storage_5_years = storage_per_day * 365 * 5          # ~1.8 PB

# Bandwidth
bandwidth_bytes_per_sec = read_qps * photo_size_bytes  # ~1.16 GB/s
```

**Conclusion:**
- Need object storage (S3-style) for photos
- Need CDN — 1.16 GB/s outbound is served globally
- DB just stores metadata, not the photo bytes

---

## Exercise 3: Sketch an API Layer — FastAPI

A minimal skeleton of what the "API Gateway layer" looks like in practice:

```python
from fastapi import FastAPI, Request, HTTPException, Depends
from pydantic import BaseModel
import time

app = FastAPI()

# --- Rate limiting (simple in-memory, conceptual) ---
request_log: dict[str, list[float]] = {}
RATE_LIMIT = 10  # max 10 requests per 60 seconds

def rate_limiter(request: Request):
    client_ip = request.client.host
    now = time.time()
    window = 60

    history = request_log.get(client_ip, [])
    history = [t for t in history if now - t < window]

    if len(history) >= RATE_LIMIT:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")

    history.append(now)
    request_log[client_ip] = history

# --- Endpoints ---
class TweetRequest(BaseModel):
    user_id: str
    content: str

@app.post("/tweet", dependencies=[Depends(rate_limiter)])
def post_tweet(body: TweetRequest):
    # In real system: validate → write to DB → publish to MQ
    return {"status": "queued", "tweet_id": "abc123"}

@app.get("/timeline/{user_id}", dependencies=[Depends(rate_limiter)])
def get_timeline(user_id: str):
    # In real system: read from cache → fallback to DB
    return {"user_id": user_id, "tweets": []}
```

> This simulates the API Gateway + rate-limiter layer in any real HLD.

---

## Exercise 4: Draw Your Own HLD

**Task:** Sketch the HLD for a URL shortener. Spend 5 minutes, then compare.

**Reference answer:**

```
User
  │
  ▼
[ DNS: short.ly ]
  │
  ▼
[ CDN ] ◄── cache popular short URLs
  │ (miss)
  ▼
[ Load Balancer ]
  │
  ├──► [ Redirect Service ] ──► [ Redis Cache ] ──► [ PostgreSQL ]
  │         (GET /:code)           (code → url)       (source of truth)
  │
  └──► [ Write Service ] ──► [ PostgreSQL ]
            (POST /shorten)      (store mapping)
              │
              └──► [ ID Generator (Snowflake/UUID) ]
```

**Key decisions to articulate:**
- Why Redis? Reads are 100x writes; cache hit rate > 95%
- Why PostgreSQL? Mappings are relational; consistency matters (no duplicate codes)
- Why CDN? Top 1% of short URLs get 80% of traffic — cache them at edge

---

## Exercise 5: Trade-off Practice

**Scenario:** You're designing the timeline for a social network with 100M users.

```
Option A: Fan-out on Write (Push model)
  - When user posts, push to all followers' timelines immediately
  - Read is O(1) — pre-computed
  - Problem: celebrity with 10M followers → 10M writes per post

Option B: Fan-out on Read (Pull model)
  - When user opens app, merge feeds of followed users
  - Write is O(1)
  - Problem: slow reads for users following 1000+ people

Option C: Hybrid (Twitter's actual approach)
  - Push model for normal users
  - Pull model for celebrities (> threshold)
  - Merge at read time
```

**Practice script:** Explain Option C out loud in 60 seconds, covering what, why, and trade-offs.

---

## Exercise 6: JWT Refresh Token Rotation (FastAPI)

Access tokens are short-lived; refresh tokens enable silent re-auth without re-login.

```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
import time
import uuid

app = FastAPI()
security = HTTPBearer()

SECRET_KEY = "use-env-var-in-production"
ALGORITHM = "HS256"

# Simulated refresh token store (use Redis in production)
valid_refresh_tokens: set[str] = set()

def create_access_token(user_id: str) -> str:
    return jwt.encode({
        "sub": user_id,
        "exp": int(time.time()) + 900,       # 15 minutes
        "type": "access"
    }, SECRET_KEY, algorithm=ALGORITHM)

def create_refresh_token(user_id: str) -> str:
    jti = str(uuid.uuid4())
    token = jwt.encode({
        "sub": user_id,
        "jti": jti,
        "exp": int(time.time()) + 86400 * 7,  # 7 days
        "type": "refresh"
    }, SECRET_KEY, algorithm=ALGORITHM)
    valid_refresh_tokens.add(jti)             # store in Redis in prod
    return token

@app.post("/login")
def login(username: str, password: str):
    if username != "alice" or password != "secret":
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return {
        "access_token": create_access_token(username),
        "refresh_token": create_refresh_token(username),
    }

@app.post("/refresh")
def refresh(creds: HTTPAuthorizationCredentials = Depends(security)):
    try:
        payload = jwt.decode(creds.credentials, SECRET_KEY, algorithms=[ALGORITHM])
        if payload.get("type") != "refresh":
            raise HTTPException(status_code=401, detail="Not a refresh token")
        jti = payload["jti"]
        if jti not in valid_refresh_tokens:
            raise HTTPException(status_code=401, detail="Refresh token revoked")
        # Rotate: invalidate old refresh token, issue new pair
        valid_refresh_tokens.discard(jti)
        user_id = payload["sub"]
        return {
            "access_token": create_access_token(user_id),
            "refresh_token": create_refresh_token(user_id),
        }
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Refresh token expired")

@app.post("/logout")
def logout(creds: HTTPAuthorizationCredentials = Depends(security)):
    try:
        payload = jwt.decode(creds.credentials, SECRET_KEY, algorithms=[ALGORITHM])
        valid_refresh_tokens.discard(payload.get("jti", ""))
        return {"message": "Logged out"}
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

**Key points to say in interview:**
- Access token: 15 min TTL, verified by signature only (no DB hit)
- Refresh token: 7 days, stored server-side (Redis) so it can be revoked
- Rotation: each refresh issues a new refresh token and invalidates the old one — prevents token theft replay

---

## Exercise 7: Distributed Tracing with Correlation IDs (FastAPI)

Every request gets a `trace_id` that flows through all services — makes debugging distributed systems possible.

```python
from fastapi import FastAPI, Request, Response
import uuid
import logging
import time

app = FastAPI()
logging.basicConfig(level=logging.INFO, format="%(message)s")
logger = logging.getLogger(__name__)

# Middleware: inject/propagate trace ID
@app.middleware("http")
async def trace_middleware(request: Request, call_next):
    trace_id = request.headers.get("X-Trace-ID") or str(uuid.uuid4())
    start = time.time()

    # Make trace_id available to route handlers
    request.state.trace_id = trace_id

    response: Response = await call_next(request)
    duration_ms = (time.time() - start) * 1000

    # Structured log — queryable in CloudWatch/Datadog/Grafana Loki
    logger.info({
        "trace_id": trace_id,
        "method": request.method,
        "path": request.url.path,
        "status": response.status_code,
        "duration_ms": round(duration_ms, 2),
    })

    # Propagate trace ID back to client and downstream services
    response.headers["X-Trace-ID"] = trace_id
    return response

@app.get("/feed")
def get_feed(request: Request):
    trace_id = request.state.trace_id
    # When calling downstream services, pass this header:
    # headers = {"X-Trace-ID": trace_id}
    # httpx.get("http://recommendation-service/recs", headers=headers)
    return {"trace_id": trace_id, "feed": []}
```

**Why this matters in interviews:**
- Shows you understand observability beyond just "add logging"
- Correlation ID ties together logs across 10 microservices for one user request
- Without it, debugging production issues in distributed systems is nearly impossible
