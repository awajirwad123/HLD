# Ride-Sharing — Notes & Reference

## Geospatial Index Options

| Option | Query type | Performance | Notes |
|---|---|---|---|
| Redis GEOADD/GEORADIUS | Radius search | O(N+log M) | Best for real-time; Geohash internally |
| PostGIS (Postgres extension) | ST_DWithin, ST_Distance | Index needed | Great for persistence + SQL joins |
| Google S2 library | Cell-based hierarchical | Very fast | Uber's actual approach; 64-bit cell IDs |
| Quadtree / R-tree | Bounding box | O(log N) | Used in spatial DBs (MySQL spatial) |

---

## Real-Time Location Update Flow

```
Driver GPS update (every 4s)
  ↓
Location Service API
  ├── GEOADD drivers_active {lng} {lat} {driver_id}  (Redis — real-time current position)
  └── Publish to Kafka (location topic)
         ├── Matching Service (nearby queries)
         ├── Trip Tracker (ETA recalculation for active trip)
         └── Analytics (heatmap, surge calculation per cell)
```

---

## Trip State Machine

```
REQUESTED → DRIVER_ACCEPTED → EN_ROUTE → ARRIVED → IN_PROGRESS → COMPLETED
    ↓              ↓              ↓          ↓            ↓
CANCELLED     CANCELLED     CANCELLED   CANCELLED    CANCELLED
```

Valid transitions enforced server-side. Each transition records a timestamp (for billing/SLA analytics).

---

## Matching Algorithm (Interview Level)

1. `GEORADIUS active_drivers {rider_lng} {rider_lat} 5km ASC COUNT 20` → 20 nearest candidates
2. Filter: `driver_id NOT IN busy_drivers SET`
3. Score each: `score = α × (1/distance) + β × rating + γ × acceptance_rate`
4. Sequential offer: offer to highest-score driver → 10s timeout → next

**More advanced (Uber production):** Global optimization using the Hungarian algorithm — match all outstanding requests to all available drivers simultaneously to minimize total pick-up time across the entire city. Runs every 5 seconds.

---

## Surge Pricing Design

```
Divide city into S2 cells (~100m × 100m)
Each cell tracks:
  - Pending requests in last 5 minutes
  - Available drivers currently in cell

surge_ratio = pending_requests / max(available_drivers, 1)

Multiplier table:
  ratio < 1.5  → 1.0×
  ratio < 2.5  → 1.5×
  ratio < 4.0  → 2.0×
  ratio < 6.0  → 2.5×
  ratio ≥ 6    → 3.0×  (cap)

Updated: every 30s
Stored: Redis hash per cell
```

---

## Geohash vs S2 vs What vs Quadtree

| | Geohash | S2 (Google) | Quadtree |
|---|---|---|---|
| Representation | Base32 string prefix | 64-bit integer | Tree structure |
| Shape | Rectangle | Spherical cap (no distortion) | Rectangle |
| Hierarchy | String prefix = parent | Integer level | Tree levels |
| Distortion | Yes (worse near poles) | No | Yes |
| Uber usage | Not primary | Yes (S2 cells) | No |
| Redis built-in | Yes (internal) | No (use int index) | No |

---

## ETA Calculation

```
ETA = routing_time(origin, destination, current_traffic)
    = Maps API (Google/Mapbox/HERE) response

Cache: Redis key = hash(origin_cell + destination_cell)
  TTL = 30s (traffic changes)
  
On cache miss: call Maps API → parse duration in seconds → store → return
Cost: ~$5 per 1000 API calls → cache is critical at 500K active trips
```

---

## Key Numbers

| Metric | Value |
|---|---|
| Trips/day | ~5M |
| Driver location updates/sec | ~1M |
| Active trips peak | ~500K |
| GEORADIUS query time (Redis) | < 1ms |
| Driver location update interval | 4 seconds |
| Surge update interval | 30 seconds |
| Matching timeout per driver | 10 seconds |
| ETA cache TTL | 30 seconds |
| GPS history retention (Cassandra) | ~6 months |
| Location update size | ~50 bytes |

---

## Common Design Mistakes

1. **Storing driver locations in PostgreSQL without PostGIS** → full table scan for nearby query
2. **Broadcasting ride request to all N nearby drivers** → N competing accepts → wasted effort
3. **No atomic booking** → two riders book the same driver simultaneously
4. **Polling for driver location** instead of streaming → stale positions
5. **No surge cap** → 10× surge during disaster causes PR crisis
6. **Trusting client-side GPS entirely** → GPS spoofing fraud (fake location to get surge area fares)
