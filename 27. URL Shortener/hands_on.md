# URL Shortener — Hands-On Exercises

## Exercise 1: Core URL Shortener Service (FastAPI)

```python
"""
Full-featured URL shortener with:
  - Random token generation
  - Redis caching
  - Click counting (write-behind)
  - Expiration support
  - Custom aliases
"""

import secrets
import string
import asyncio
from datetime import datetime, timezone, timedelta
from typing import Optional

import asyncpg
import redis.asyncio as redis
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import RedirectResponse
from pydantic import BaseModel, HttpUrl, field_validator

app = FastAPI(title="URL Shortener")

# ─── Config ───────────────────────────────────────────────────────────────────

DB_DSN    = "postgresql://user:pass@localhost/shortener"
REDIS_URL = "redis://localhost:6379"
BASE_URL  = "https://short.ly"
ALPHABET  = string.ascii_letters + string.digits  # 62 chars
CODE_LEN  = 7

RESERVED_CODES = {"api", "admin", "static", "health", "metrics", "favicon.ico"}

# ─── Startup ──────────────────────────────────────────────────────────────────

@app.on_event("startup")
async def startup():
    app.state.pool = await asyncpg.create_pool(DB_DSN, min_size=5, max_size=20)
    app.state.redis = redis.from_url(REDIS_URL, decode_responses=True)
    asyncio.create_task(click_flush_worker(app.state.pool, app.state.redis))


@app.on_event("shutdown")
async def shutdown():
    await app.state.pool.close()
    await app.state.redis.aclose()


# ─── Models ───────────────────────────────────────────────────────────────────

class ShortenRequest(BaseModel):
    url: HttpUrl
    alias: Optional[str] = None           # Custom alias
    ttl_days: Optional[int] = None        # Expiration

    @field_validator("alias")
    @classmethod
    def validate_alias(cls, v):
        if v is None:
            return v
        if not v.replace("-", "").isalnum():
            raise ValueError("Alias must be alphanumeric with optional dashes")
        if len(v) > 20:
            raise ValueError("Alias max length is 20 characters")
        if v.lower() in RESERVED_CODES:
            raise ValueError(f"'{v}' is a reserved word")
        return v


class ShortenResponse(BaseModel):
    short_url: str
    short_code: str
    expires_at: Optional[datetime]


# ─── Token generation ─────────────────────────────────────────────────────────

def generate_token(length: int = CODE_LEN) -> str:
    return "".join(secrets.choice(ALPHABET) for _ in range(length))


# ─── Endpoints ────────────────────────────────────────────────────────────────

@app.post("/shorten", response_model=ShortenResponse)
async def shorten_url(req: ShortenRequest):
    pool: asyncpg.Pool = app.state.pool
    long_url = str(req.url)

    expires_at = None
    if req.ttl_days:
        expires_at = datetime.now(timezone.utc) + timedelta(days=req.ttl_days)

    # Use custom alias or generate random code
    if req.alias:
        short_code = req.alias
        try:
            await pool.execute(
                """INSERT INTO urls (short_code, long_url, expires_at)
                   VALUES ($1, $2, $3)""",
                short_code, long_url, expires_at,
            )
        except asyncpg.UniqueViolationError:
            raise HTTPException(409, f"Alias '{short_code}' is already taken")
    else:
        # Retry loop for collision (extremely rare with 7-char random token)
        for attempt in range(5):
            short_code = generate_token()
            try:
                await pool.execute(
                    """INSERT INTO urls (short_code, long_url, expires_at)
                       VALUES ($1, $2, $3)""",
                    short_code, long_url, expires_at,
                )
                break
            except asyncpg.UniqueViolationError:
                if attempt == 4:
                    raise HTTPException(500, "Could not generate unique code")
                continue

    return ShortenResponse(
        short_url=f"{BASE_URL}/{short_code}",
        short_code=short_code,
        expires_at=expires_at,
    )


@app.get("/{short_code}")
async def redirect(short_code: str, request: Request):
    pool: asyncpg.Pool = app.state.pool
    rdb: redis.Redis = app.state.redis

    # ── 1. Redis cache lookup ───────────────────────────────────────────────
    cached = await rdb.hgetall(f"url:{short_code}")
    if cached:
        long_url   = cached["long_url"]
        expires_at = cached.get("expires_at")

        if expires_at and datetime.fromisoformat(expires_at) < datetime.now(timezone.utc):
            await rdb.delete(f"url:{short_code}")
            raise HTTPException(410, "This link has expired")

        # Async click increment (write-behind)
        await rdb.incr(f"clicks:{short_code}")
        return RedirectResponse(long_url, status_code=302,
                                headers={"Cache-Control": "max-age=3600"})

    # ── 2. DB lookup on cache miss ──────────────────────────────────────────
    row = await pool.fetchrow(
        """SELECT long_url, expires_at, is_active
           FROM urls WHERE short_code = $1""",
        short_code,
    )

    if not row:
        raise HTTPException(404, "Short URL not found")
    if not row["is_active"]:
        raise HTTPException(410, "This link has been deactivated")
    if row["expires_at"] and row["expires_at"] < datetime.now(timezone.utc):
        raise HTTPException(410, "This link has expired")

    # ── 3. Warm Redis cache (TTL = 1 hour) ─────────────────────────────────
    cache_ttl = 3600
    pipe = rdb.pipeline()
    pipe.hset(f"url:{short_code}", mapping={
        "long_url": row["long_url"],
        **({"expires_at": row["expires_at"].isoformat()} if row["expires_at"] else {}),
    })
    pipe.expire(f"url:{short_code}", cache_ttl)
    pipe.incr(f"clicks:{short_code}")
    await pipe.execute()

    return RedirectResponse(row["long_url"], status_code=302,
                            headers={"Cache-Control": "max-age=3600"})


@app.get("/api/stats/{short_code}")
async def get_stats(short_code: str):
    pool: asyncpg.Pool = app.state.pool
    rdb: redis.Redis = app.state.redis

    row = await pool.fetchrow(
        "SELECT short_code, long_url, click_count, created_at, expires_at FROM urls WHERE short_code = $1",
        short_code,
    )
    if not row:
        raise HTTPException(404, "Not found")

    # Add in-flight Redis counter to DB count
    pending = int(await rdb.get(f"clicks:{short_code}") or 0)
    return {
        "short_code": row["short_code"],
        "long_url": row["long_url"],
        "click_count": row["click_count"] + pending,
        "created_at": row["created_at"],
        "expires_at": row["expires_at"],
    }


# ─── Write-behind click flush worker ──────────────────────────────────────────

async def click_flush_worker(pool: asyncpg.Pool, rdb: redis.Redis):
    """Every 10 seconds, flush pending click increments to the DB."""
    while True:
        await asyncio.sleep(10)
        try:
            # Scan all keys matching clicks:*
            keys = [k async for k in rdb.scan_iter("clicks:*")]
            if not keys:
                continue

            # Pipeline: GET and DELETE atomically
            pipe = rdb.pipeline()
            for key in keys:
                pipe.getdel(key)
            counts = await pipe.execute()

            # Batch UPDATE
            updates = [
                (int(count), key.split(":")[1])
                for key, count in zip(keys, counts)
                if count and int(count) > 0
            ]
            if updates:
                async with pool.acquire() as conn:
                    await conn.executemany(
                        "UPDATE urls SET click_count = click_count + $1 WHERE short_code = $2",
                        updates,
                    )
        except Exception as e:
            print(f"Click flush error: {e}")
```

---

## Exercise 2: Base62 Encoding and ID Generation

```python
"""
Demonstrate:
  - Base62 encoding/decoding
  - Snowflake-style unique ID generation
  - Collision probability calculation
"""

import time
import threading
import math

ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
BASE = len(ALPHABET)  # 62


def base62_encode(n: int) -> str:
    """Encode a positive integer to a Base62 string."""
    if n == 0:
        return ALPHABET[0]
    digits = []
    while n:
        digits.append(ALPHABET[n % BASE])
        n //= BASE
    return "".join(reversed(digits))


def base62_decode(s: str) -> int:
    """Decode a Base62 string back to an integer."""
    result = 0
    for char in s:
        result = result * BASE + ALPHABET.index(char)
    return result


# ── Snowflake ID Generator ─────────────────────────────────────────────────

class SnowflakeGenerator:
    """
    64-bit ID layout:
      [1 sign][41 ms timestamp][10 machine_id][12 sequence]
    Generates ~4096 unique IDs/ms per machine.
    """

    EPOCH = 1_700_000_000_000  # Custom epoch (Nov 2023)
    MACHINE_BITS  = 10
    SEQUENCE_BITS = 12
    MAX_MACHINE   = (1 << MACHINE_BITS) - 1    # 1023
    MAX_SEQUENCE  = (1 << SEQUENCE_BITS) - 1   # 4095

    def __init__(self, machine_id: int):
        assert 0 <= machine_id <= self.MAX_MACHINE
        self.machine_id = machine_id
        self._sequence = 0
        self._last_ms = -1
        self._lock = threading.Lock()

    def next_id(self) -> int:
        with self._lock:
            now_ms = int(time.time() * 1000) - self.EPOCH
            if now_ms == self._last_ms:
                self._sequence = (self._sequence + 1) & self.MAX_SEQUENCE
                if self._sequence == 0:
                    # Sequence exhausted — wait for next millisecond
                    while now_ms <= self._last_ms:
                        now_ms = int(time.time() * 1000) - self.EPOCH
            else:
                self._sequence = 0
                self._last_ms = now_ms

            return (now_ms << (self.MACHINE_BITS + self.SEQUENCE_BITS)) \
                 | (self.machine_id << self.SEQUENCE_BITS) \
                 | self._sequence


# ── Demo ────────────────────────────────────────────────────────────────────

gen = SnowflakeGenerator(machine_id=1)

# Generate 5 IDs and show their Base62 short codes
print("=== Snowflake ID → Base62 short codes ===")
for _ in range(5):
    sf_id = gen.next_id()
    code = base62_encode(sf_id)
    decoded = base62_decode(code)
    print(f"  ID: {sf_id:>20}  →  code: {code:<12}  →  decoded: {decoded}")
    assert decoded == sf_id

# Show ID capacity
max_7_char = BASE ** 7
print(f"\n=== Capacity analysis ===")
print(f"  Max 7-char Base62 value: {max_7_char:,}  (~{max_7_char/1e12:.1f} trillion)")
print(f"  Snowflake timestamp bits → {2**41 / (365*24*3600*1000):.0f} years of headroom")

# Collision probability (birthday paradox)
def birthday_collision_probability(n: int, space: int) -> float:
    """Prob of at least one collision among n random samples from space."""
    log_p_no_collision = sum(math.log(1 - k / space) for k in range(n))
    return 1 - math.exp(log_p_no_collision)

for n_urls in [1_000, 1_000_000, 1_000_000_000]:
    p = birthday_collision_probability(n_urls, max_7_char)
    print(f"  Collision prob with {n_urls:>12,} URLs: {p:.2e}")
```

---

## Exercise 3: Expiration Cleanup Job

```python
"""
Background job to:
  1. Soft-delete expired URLs (is_active = false)
  2. Hard-delete rows that have been soft-deleted for > 1 day
  3. Invalidate Redis cache for expired entries
"""

import asyncio
import asyncpg
import redis.asyncio as redis
from datetime import datetime, timezone


DB_DSN    = "postgresql://user:pass@localhost/shortener"
REDIS_URL = "redis://localhost:6379"


async def cleanup_expired_urls():
    pool = await asyncpg.create_pool(DB_DSN)
    rdb  = redis.from_url(REDIS_URL, decode_responses=True)

    try:
        while True:
            now = datetime.now(timezone.utc)

            async with pool.acquire() as conn:
                # Step 1: Soft-delete newly expired URLs
                expired = await conn.fetch(
                    """UPDATE urls SET is_active = false
                       WHERE expires_at < $1 AND is_active = true
                       RETURNING short_code""",
                    now,
                )
                if expired:
                    codes = [r["short_code"] for r in expired]
                    print(f"[{now}] Soft-deleted {len(codes)} expired URLs")

                    # Invalidate Redis cache for these codes
                    pipe = rdb.pipeline()
                    for code in codes:
                        pipe.delete(f"url:{code}")
                    await pipe.execute()

                # Step 2: Hard-delete rows soft-deleted > 24h ago
                # (CDN TTL has expired by now — safe to remove)
                deleted = await conn.fetchval(
                    """DELETE FROM urls
                       WHERE is_active = false
                         AND (expires_at IS NOT NULL AND expires_at < $1 - INTERVAL '1 day')
                       RETURNING count(*)""",
                    now,
                )
                if deleted:
                    print(f"[{now}] Hard-deleted {deleted} rows")

            await asyncio.sleep(3600)  # Run hourly

    finally:
        await pool.close()
        await rdb.aclose()


if __name__ == "__main__":
    asyncio.run(cleanup_expired_urls())
```
