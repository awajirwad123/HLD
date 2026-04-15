# Databases — SQL — Interview Simulator

## How to use this file
Set a 35-minute timer. Answer each scenario out loud or in writing before reading the model answer. Score yourself using the rubric at the end.

---

## Scenario 1: Design the DB Layer for an E-Commerce Order System

**Prompt:**  
"We're building an e-commerce platform. Users can browse products, add items to a cart, and place orders. Each order can have multiple line items. Design the database schema, indexes, and scaling strategy for this system. We expect 10K orders/day initially, growing to 1M orders/day in 18 months."

**Time box:** 15 minutes

---

### Model Answer

#### Schema Design

```sql
-- Users
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       TEXT NOT NULL UNIQUE,
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Products
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    price_cents BIGINT NOT NULL,
    stock       INT NOT NULL DEFAULT 0,
    updated_at  TIMESTAMP DEFAULT NOW()
);

-- Orders
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    status      VARCHAR(20) NOT NULL DEFAULT 'pending',
                -- 'pending', 'paid', 'shipped', 'delivered', 'cancelled'
    total_cents BIGINT NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Order line items
CREATE TABLE order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT NOT NULL REFERENCES orders(id),
    product_id  BIGINT NOT NULL REFERENCES products(id),
    quantity    INT NOT NULL,
    unit_price_cents BIGINT NOT NULL  -- snapshot price at time of purchase
);
```

#### Index Strategy

```sql
-- User lookup by email (login, uniqueness check)
-- Already covered by UNIQUE constraint → implicit index

-- Orders by user (show order history)
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC);

-- Orders by status (ops dashboard, fulfilment queue)
-- Partial: most orders will be in terminal states; we mostly query active ones
CREATE INDEX idx_orders_pending ON orders (created_at)
    WHERE status IN ('pending', 'paid');

-- Order items by order (fetch line items for an order)
CREATE INDEX idx_order_items_order ON order_items (order_id);

-- Products: avoid full scan on name search
CREATE INDEX idx_products_name ON products USING gin(to_tsvector('english', name));
```

#### Critical Transaction — Place Order

```sql
BEGIN;
  -- 1. Decrement stock atomically (prevent oversell)
  UPDATE products
  SET stock = stock - :qty
  WHERE id = :product_id AND stock >= :qty;   -- CAS pattern

  -- Check 1 row was updated (if 0, stock was insufficient → rollback)

  -- 2. Create order + line items
  INSERT INTO orders (user_id, status, total_cents)
  VALUES (:user_id, 'pending', :total);

  INSERT INTO order_items (order_id, product_id, quantity, unit_price_cents)
  VALUES (:order_id, :product_id, :qty, :price);
COMMIT;
```

#### Scaling Strategy

```
10K orders/day  = ~0.1 TPS writes          → Single PostgreSQL instance, fine
1M orders/day   = ~12 TPS writes            → Still fine for single PostgreSQL
Peak (flash sale) = 10x spike → ~120 TPS  → Add read replicas for dashboards
                                             Vertical scale if needed (32 vCPU)
```

Key decisions to mention:
1. **Read replicas**: Route order history reads to replica; writes (place order) to primary
2. **PgBouncer**: At 100+ app servers, mandatory to avoid connection exhaustion
3. **Archive old orders**: Orders > 2 years old move to archive table or cold storage. Hot table stays small and fast
4. **Partition by `created_at`**: PostgreSQL table partitioning on orders table once it exceeds ~100M rows — transparent to application code

---

## Scenario 2: Diagnose a Slow Query in Production

**Prompt:**  
"Our product listing page is taking 3–4 seconds instead of < 200ms. It was fine last week. We deployed a schema migration yesterday that added two new columns. The query is: `SELECT * FROM products WHERE category = 'electronics' ORDER BY price_cents ASC LIMIT 50`. The products table has 2 million rows. Diagnose and fix."

**Time box:** 10 minutes

---

### Model Answer

**Step 1: Run EXPLAIN ANALYZE**

```sql
EXPLAIN ANALYZE
SELECT * FROM products
WHERE category = 'electronics'
ORDER BY price_cents ASC
LIMIT 50;
```

**Likely output → the problem:**

```
Sort  (cost=55000.00..55001.25 rows=500 width=256)
  Sort Key: price_cents
  Sort Method: external merge  Disk: 12MB    ← Sorting 500K rows on disk
  ->  Seq Scan on products
        Filter: (category = 'electronics')
        Rows Removed by Filter: 1500000
Planning Time: 0.5ms
Execution Time: 3200ms
```

**Diagnosis:** Full table scan + in-memory (disk) sort. The migration likely:
- Dropped an existing index accidentally (check `\d products` or `pg_indexes`)
- Added a new column with a DEFAULT that triggered a table rewrite, invalidating planner statistics

**Step 2: Verify index existence**

```sql
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'products';
```

**Step 3: Fix — Create composite index**

```sql
-- Index on (category, price_cents) satisfies both WHERE and ORDER BY
-- Query planner can use index scan + limit, no sort needed
CREATE INDEX CONCURRENTLY idx_products_category_price
    ON products (category, price_cents ASC);
-- CONCURRENTLY: no table lock, safe to run in production
```

**After index:**

```
Limit  (cost=0.56..12.43 rows=50 width=256)
  ->  Index Scan using idx_products_category_price on products
        Index Cond: (category = 'electronics')
Execution Time: 0.8ms   ← down from 3200ms
```

**Key interview point:** Always use `CREATE INDEX CONCURRENTLY` in production to avoid locking the table. Takes longer to build, but zero downtime.

**Follow-up prevention:**
- Run `ANALYZE products;` after any schema migration that touches data
- Add the slow-query-detect migration step to your CI pipeline: run `EXPLAIN` on the top-20 critical queries after each migration

---

## Scenario 3: PostgreSQL from 1K to 100K QPS

**Prompt:**  
"You're running a SaaS product on a single PostgreSQL instance. It handles 1K QPS today. Your product is going viral and you need to scale to 100K QPS within 3 months. Walk me through your scaling roadmap."

**Time box:** 10 minutes

---

### Model Answer

**First: characterize the load**

```
Is it 100K reads, 100K writes, or mixed?
Typical SaaS: 80% reads, 20% writes → 80K read QPS, 20K write QPS
```

**Phase 1: Vertical Scale + Query Optimization (Week 1–2)**

```
Current bottleneck? Run EXPLAIN ANALYZE on top queries.
- Add missing indexes (often gets 5–10x improvement for free)
- Fix N+1 queries (can cut total query count by 10x)
- Upgrade instance: 4→32 vCPU, 32GB→256GB RAM
  → larger buffer pool = more hot pages in memory = faster reads

Expected result: 5–20x improvement, reaches 10–20K QPS
```

**Phase 2: Read Replicas + PgBouncer (Week 2–4)**

```
Add 2–3 read replicas to distribute read QPS
Route:
  - Dashboard queries  → replica 1
  - Historical data    → replica 2
  - Writes + critical reads → primary

Add PgBouncer (transaction mode) in front of each DB node
→ Eliminates connection overhead
→ Enables 200+ app servers without overwhelming DB connections

Expected result: 3–4x read scale → 40–60K read QPS
```

**Phase 3: Application-Level Caching (Ongoing)**

```
Hot data cached in Redis:
- Product catalog, user profiles, config → TTL cache (5–60min)
- Session data → Redis
- Counters → Redis INCR (avoid DB writes for view counts, likes)

Expected cache hit rate: 60–80%
→ Reduces DB read QPS by 60–80%: 80K → 16K reads to hit DB
```

**Phase 4: Table Partitioning (Month 2, if needed)**

```
For large append-only tables (events, logs, orders):
CREATE TABLE events (
    id         BIGINT,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

→ Queries with date filter hit only one partition
→ Old partitions can be detached and archived
→ Transparent to application, no code change needed
```

**Phase 5: Write Scale (CQRS pattern, Month 3 if write bottleneck persists)**

```
If writes are the bottleneck at 20K TPS:
- Separate write model (OLTP, normalized schema) from read model (denormalized, cached)
- Event sourcing: writes go to event log (append-only, very fast)
- Projections rebuild read models async

Still not enough? → This is when you evaluate sharding.
Most SaaS products never reach this point within 3 months of going viral.
```

**Final architecture diagram:**

```
App Servers (many)
    │
    ├─→ PgBouncer (Primary) → PostgreSQL Primary (writes)
    │
    ├─→ PgBouncer (Replica 1) → PostgreSQL Replica 1 (reads)
    ├─→ PgBouncer (Replica 2) → PostgreSQL Replica 2 (reads)
    │
    └─→ Redis (hot reads: profiles, catalog, sessions)
```

---

## Self-Assessment Scorecard

Rate yourself 0–5 on each criterion:

| Criterion                                              | Score /5 |
|--------------------------------------------------------|----------|
| Correctly identified ACID guarantees and their costs   |          |
| Schema had appropriate FKs and normalization           |          |
| Named specific index types (composite, partial, covering) |       |
| Mentioned connection pooling (PgBouncer) spontaneously |          |
| Identified replication (read replicas) as read scale solution |   |
| Used EXPLAIN ANALYZE correctly in diagnosis            |          |
| Knew to use CONCURRENTLY for production index creation |          |
| Gave phased scaling roadmap (didn't jump to sharding)  |          |

**Total: /40**

| Score | Interpretation                                        |
|-------|-------------------------------------------------------|
| 35–40 | Excellent — ready for senior DB design questions      |
| 25–34 | Good foundation — review tricky_interview_questions   |
| 15–24 | Revisit architecture.md, redo Scenario 2 especially  |
| < 15  | Full re-read of architecture.md + hands_on.md first  |
