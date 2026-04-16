# Databases — SQL — Hands-On Exercises

## Exercise 1: EXPLAIN ANALYZE — Index Impact

### Setup (PostgreSQL)

```sql
-- Create a demo table
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    status      VARCHAR(20) NOT NULL,
    total_cents BIGINT NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Insert 500,000 rows
INSERT INTO orders (user_id, status, total_cents)
SELECT
    (random() * 10000)::INT,
    CASE (random() * 3)::INT
        WHEN 0 THEN 'pending'
        WHEN 1 THEN 'paid'
        ELSE 'shipped'
    END,
    (random() * 10000)::INT
FROM generate_series(1, 500000);
```

### Without Index

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 5000 AND status = 'pending';

-- Expected output (slow):
-- Seq Scan on orders
--   (cost=0.00..12500.00 rows=16 width=40)
--   (actual time=12.xxx..52.xxx rows=14 loops=1)
-- Planning Time: 0.3ms
-- Execution Time: ~52ms   ← full table scan on 500K rows
```

### Add Composite Index

```sql
CREATE INDEX idx_orders_user_status ON orders (user_id, status);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 5000 AND status = 'pending';

-- Expected output (fast):
-- Index Scan using idx_orders_user_status on orders
--   (cost=0.42..8.45 rows=14 width=40)
--   (actual time=0.021..0.025 rows=14 loops=1)
-- Planning Time: 0.5ms
-- Execution Time: ~0.05ms  ← 1000x faster
```

### Add Covering Index (Even Faster)

```sql
-- If you SELECT only user_id, status, total_cents
-- → add INCLUDE to avoid heap fetches
CREATE INDEX idx_orders_covering ON orders (user_id, status)
    INCLUDE (total_cents, created_at);

EXPLAIN ANALYZE
SELECT user_id, status, total_cents, created_at
FROM orders WHERE user_id = 5000 AND status = 'pending';

-- Expected output:
-- Index Only Scan using idx_orders_covering on orders
--   (actual time=0.012..0.015 rows=14 loops=1)
-- Heap Fetches: 0     ← no heap I/O at all
```

---

## Exercise 2: ACID Transaction Demo (Python + psycopg2)

```python
"""
Demonstrates ACID atomicity: both updates succeed or both are rolled back.
"""
import psycopg2

def transfer_funds(conn, from_id: int, to_id: int, amount_cents: int):
    """Transfer money between accounts atomically."""
    with conn.cursor() as cur:
        try:
            # Begin is implicit; use explicit savepoint for nested control
            cur.execute(
                "SELECT balance_cents FROM accounts WHERE id = %s FOR UPDATE",
                (from_id,)
            )
            row = cur.fetchone()
            if row is None:
                raise ValueError(f"Account {from_id} not found")

            balance = row[0]
            if balance < amount_cents:
                raise ValueError(f"Insufficient funds: {balance} < {amount_cents}")

            # Debit sender
            cur.execute(
                "UPDATE accounts SET balance_cents = balance_cents - %s WHERE id = %s",
                (amount_cents, from_id)
            )
            # Credit receiver
            cur.execute(
                "UPDATE accounts SET balance_cents = balance_cents + %s WHERE id = %s",
                (amount_cents, to_id)
            )

            conn.commit()  # ← Durability: WAL flush, both rows committed
            print(f"Transferred {amount_cents} from {from_id} to {to_id}")

        except Exception as exc:
            conn.rollback()  # ← Atomicity: neither update persists
            print(f"Transfer failed, rolled back: {exc}")
            raise


# Demonstrate isolation: dirty read prevention
def demonstrate_isolation(conn1, conn2, account_id: int):
    """
    Transaction 1 updates but doesn't commit.
    Transaction 2 reads the same row.
    With Read Committed: Transaction 2 sees the OLD value.
    """
    cur1 = conn1.cursor()
    cur2 = conn2.cursor()

    # Tx1: update balance but don't commit yet
    cur1.execute(
        "UPDATE accounts SET balance_cents = 99999999 WHERE id = %s",
        (account_id,)
    )
    # Don't commit Tx1

    # Tx2: read the same row
    cur2.execute("SELECT balance_cents FROM accounts WHERE id = %s", (account_id,))
    val = cur2.fetchone()[0]
    print(f"Tx2 sees balance: {val}")  # sees ORIGINAL value, not Tx1's dirty write

    conn1.rollback()  # Tx1 rolled back
```

---

## Exercise 3: SQLAlchemy Connection Pool in FastAPI

```python
"""
Configure SQLAlchemy with appropriate pool settings.
Demonstrates transaction-scoped sessions (best for FastAPI).
"""
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy import text

# Connection pool configuration — critical for production
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost:5432/mydb",
    pool_size=10,          # Connections kept open permanently
    max_overflow=20,       # Extra connections allowed under peak load
    pool_timeout=30,       # Seconds to wait for a connection from pool
    pool_recycle=1800,     # Recycle connections every 30 min (prevent stale)
    pool_pre_ping=True,    # Test connection health before using (prevents errors)
    echo=False,            # Set True for development SQL logging
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)

async def get_db():
    """Dependency: yields a session, commits on success, rolls back on error."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        text("SELECT id, email FROM users WHERE id = :id"),
        {"id": user_id}
    )
    row = result.fetchone()
    if row is None:
        return {"error": "not found"}
    return {"id": row.id, "email": row.email}

@app.post("/transfer")
async def transfer(from_id: int, to_id: int, cents: int, db: AsyncSession = Depends(get_db)):
    # Both updates run inside same transaction
    # Commit happens automatically at end of request if no exception
    await db.execute(
        text("UPDATE accounts SET balance_cents = balance_cents - :amt WHERE id = :id"),
        {"amt": cents, "id": from_id}
    )
    await db.execute(
        text("UPDATE accounts SET balance_cents = balance_cents + :amt WHERE id = :id"),
        {"amt": cents, "id": to_id}
    )
    return {"status": "ok"}
```

---

## Exercise 4: N+1 Problem Detection and Fix

```python
"""
Demonstrates the N+1 query problem and the proper fix.
"""
import asyncio
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text
import time

# BAD: N+1 queries
async def get_orders_with_users_bad(db: AsyncSession, limit: int = 100):
    start = time.time()
    query_count = 0

    # Query 1: fetch orders
    result = await db.execute(text("SELECT id, user_id, total_cents FROM orders LIMIT :n"), {"n": limit})
    orders = result.fetchall()
    query_count += 1

    enriched = []
    for order in orders:
        # Query 2..N+1: fetch each user individually
        user_result = await db.execute(
            text("SELECT id, email FROM users WHERE id = :uid"),
            {"uid": order.user_id}
        )
        user = user_result.fetchone()
        query_count += 1
        enriched.append({
            "order_id": order.id,
            "total": order.total_cents,
            "user_email": user.email if user else None
        })

    elapsed = time.time() - start
    print(f"N+1: {query_count} queries in {elapsed:.3f}s")  # 101 queries!
    return enriched


# GOOD: 1 query with JOIN
async def get_orders_with_users_good(db: AsyncSession, limit: int = 100):
    start = time.time()

    result = await db.execute(text("""
        SELECT o.id, o.total_cents, u.email
        FROM orders o
        JOIN users u ON u.id = o.user_id
        LIMIT :n
    """), {"n": limit})
    rows = result.fetchall()

    elapsed = time.time() - start
    print(f"JOIN: 1 query in {elapsed:.3f}s")  # 1 query, typically 10-100x faster

    return [{"order_id": r.id, "total": r.total_cents, "user_email": r.email} for r in rows]


# ALSO GOOD: Bulk load with IN clause (when JOIN is too complex)
async def get_orders_with_users_in(db: AsyncSession, limit: int = 100):
    result = await db.execute(text("SELECT id, user_id, total_cents FROM orders LIMIT :n"), {"n": limit})
    orders = result.fetchall()

    user_ids = list({o.user_id for o in orders})  # unique IDs

    users_result = await db.execute(
        text("SELECT id, email FROM users WHERE id = ANY(:ids)"),
        {"ids": user_ids}
    )
    users_map = {u.id: u.email for u in users_result.fetchall()}
    # 2 queries total regardless of limit size

    return [{"order_id": o.id, "total": o.total_cents, "user_email": users_map.get(o.user_id)} for o in orders]
```

---

## Exercise 5: Deadlock Detection + Prevention

```python
"""
Simulate and prevent deadlocks with consistent lock ordering.
"""
import asyncio
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text

# BAD: Inconsistent lock order causes deadlock
async def transfer_bad(db: AsyncSession, from_id: int, to_id: int, amount: int):
    """
    Thread A: locks account 1, waits for account 2
    Thread B: locks account 2, waits for account 1
    → Deadlock
    """
    await db.execute(text("SELECT * FROM accounts WHERE id = :id FOR UPDATE"), {"id": from_id})
    await asyncio.sleep(0)  # Simulate concurrent interleaving
    await db.execute(text("SELECT * FROM accounts WHERE id = :id FOR UPDATE"), {"id": to_id})
    # ... rest of transfer


# GOOD: Always lock in consistent order (lower ID first)
async def transfer_good(db: AsyncSession, from_id: int, to_id: int, amount: int):
    """
    Both threads always lock in ascending ID order.
    Thread A: locks 1 then 2.
    Thread B: tries 1, waits (can't get 1 while A holds it), eventually proceeds in order.
    → No deadlock possible.
    """
    # Determine lock order
    first_id, second_id = sorted([from_id, to_id])

    await db.execute(text("SELECT * FROM accounts WHERE id = :id FOR UPDATE"), {"id": first_id})
    await db.execute(text("SELECT * FROM accounts WHERE id = :id FOR UPDATE"), {"id": second_id})

    if from_id < to_id:
        debit_id, credit_id = from_id, to_id
    else:
        debit_id, credit_id = from_id, to_id

    await db.execute(
        text("UPDATE accounts SET balance = balance - :amt WHERE id = :id"),
        {"amt": amount, "id": debit_id}
    )
    await db.execute(
        text("UPDATE accounts SET balance = balance + :amt WHERE id = :id"),
        {"amt": amount, "id": credit_id}
    )
```

---

## Exercise 6: Connection Pool Monitoring

```python
"""
Monitor SQLAlchemy pool health — useful for detecting pool exhaustion.
"""
from sqlalchemy import event
from sqlalchemy.pool import Pool
import logging

logger = logging.getLogger(__name__)

def setup_pool_monitoring(engine):
    """Attach event listeners to log pool activity."""

    @event.listens_for(Pool, "connect")
    def on_connect(dbapi_conn, connection_record):
        logger.info(f"New DB connection established: {id(dbapi_conn)}")

    @event.listens_for(Pool, "checkout")
    def on_checkout(dbapi_conn, connection_record, connection_proxy):
        pool = engine.pool
        logger.debug(
            f"Connection checked out | "
            f"size={pool.size()} | "
            f"checked_out={pool.checkedout()} | "
            f"overflow={pool.overflow()} | "
            f"invalid={pool.invalid()}"
        )
        # Alert if pool is nearly exhausted
        if pool.checkedout() >= pool.size() + pool.overflow() - 2:
            logger.warning("Connection pool nearly exhausted!")

    @event.listens_for(Pool, "checkin")
    def on_checkin(dbapi_conn, connection_record):
        logger.debug(f"Connection returned to pool: {id(dbapi_conn)}")


# Expose as FastAPI health endpoint
from fastapi import APIRouter

router = APIRouter()

@router.get("/health/db")
async def db_health():
    from main import engine  # your app's engine
    pool = engine.pool
    return {
        "pool_size": pool.size(),
        "connections_checked_out": pool.checkedout(),
        "overflow": pool.overflow(),
        "connections_invalid": pool.invalid(),
        "status": "healthy" if pool.checkedout() < pool.size() else "stressed"
    }
```

---

## Exercise 7: Partial Index for Soft Deletes

```python
"""
Soft deletes are common (deleted_at IS NULL). 
A partial index on active rows only is more efficient than a full index.
"""

# Schema (SQL)
CREATE_SCHEMA = """
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    email      TEXT NOT NULL UNIQUE,
    deleted_at TIMESTAMP DEFAULT NULL
);

-- FULL index (index ALL rows, including deleted)
CREATE INDEX idx_users_email_full ON users (email);

-- PARTIAL index (index only active rows — much smaller, faster)
CREATE INDEX idx_users_active_email ON users (email)
    WHERE deleted_at IS NULL;

-- The partial index is used for this query (common case):
-- SELECT * FROM users WHERE email = 'alice@example.com' AND deleted_at IS NULL;

-- Key insight: if 10% of users are active, partial index is 10x smaller
--              smaller = fits in buffer pool = faster
"""

# Python: soft delete implementation
async def soft_delete_user(db: AsyncSession, user_id: int):
    await db.execute(
        text("UPDATE users SET deleted_at = NOW() WHERE id = :id"),
        {"id": user_id}
    )

async def get_active_user(db: AsyncSession, email: str):
    result = await db.execute(
        text("SELECT * FROM users WHERE email = :email AND deleted_at IS NULL"),
        {"email": email}
    )
    return result.fetchone()  # Uses partial index → fast
```
