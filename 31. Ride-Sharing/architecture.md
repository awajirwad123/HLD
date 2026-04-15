# Ride-Sharing (Uber) — Architecture

## Problem Statement

Design a ride-sharing platform like Uber:
- Riders request a ride from location A to B
- System finds nearby available drivers in real-time
- Matches rider to optimal driver
- Tracks driver location during ride
- Calculates dynamic pricing (surge)

**Scale:** ~5M trips/day, 4M drivers globally, peak concurrency ~500K active trips

---

## Capacity Estimation

```
Trips/day:        5M → ~58/sec average, ~200/sec peak
Driver locations: 4M drivers × 1 update/4s → 1M location updates/sec peak
Active trips:     500K concurrent → 500K location streams
Location database reads: matching queries ~10/sec per requester
```

**Dominant challenge: real-time location updates from millions of drivers.**

---

## Location Tracking (The Core Problem)

Every driver sends GPS updates every 4 seconds. The platform must:
1. Store current location of all active drivers
2. Find all drivers within X km of a rider in < 300ms
3. Update as drivers move

### What Doesn't Work

- **PostgreSQL with a regular index:** Full table scan for each nearby query at 4M rows with coordinate matching = 100ms+ per query
- **In-memory unstructured hash:** O(N) scan to find nearby drivers

### Geospatial Indexing: S2 / QuadTree / Geohash

**Approach 1: Geohash**

Divide the Earth's surface into a grid. Each cell gets a string prefix:
```
Precision 6: ~1.2km × 0.6km cells → cell ID = "9q8yy"
Precision 7: ~153m × 153m cells → cell ID = "9q8yyz"

Driver at (37.7749, -122.4194) → geohash = "9q8yy6"
```

To find nearby drivers: query current cell + 8 adjacent cells (all share the same prefix except one character). This is an indexed string lookup.

**Redis implementation:**
```
GEOADD drivers_active {lng} {lat} {driver_id}
GEODIST drivers_active driver_1 driver_2 km
GEORADIUS drivers_active {rider_lng} {rider_lat} 5 km ASC COUNT 20
```

Redis GEO commands use Geohash internally. `GEORADIUS` with 5km radius returns all drivers within 5km, sorted by distance. O(N+log(M)) where N = results, M = total drivers in the geo index.

**Approach 2: Google S2 Library (Uber's actual approach)**

S2 divides the sphere into cells with hierarchical levels (0=Earth to 30=~1cm²). Each cell is a 64-bit integer — fast integer comparison. Uber uses S2 cells to partition drivers into cells and maintain a ring of cells around a point.

### Location Storage Architecture

```
Driver App (GPS) → Location Service (FastAPI)
                         ↓ write
                   Redis Geo Index
                   (active drivers: driver_id → {lng, lat})
                         ↓ async
                   Kafka (location events)
                         ↓ consume
                   Cassandra (location history per trip)
```

**Redis for real-time:** sub-millisecond read, `GEORADIUS` queries for nearby drivers
**Kafka:** fan-out location to trip tracking service, analytics, ETA recalculation
**Cassandra:** historical GPS trail per trip (for receipts, dispute resolution, analytics)

---

## Ride Matching Flow

```
Rider requests ride at (lat, lng)
      ↓
1. Query Redis GEORADIUS → 20 nearest available drivers
2. Score each driver: distance + rating + vehicle type match
3. Send ride request to top 3 drivers sequentially:
   a. Notify Driver 1 → wait 10 seconds for acceptance
   b. If decline/timeout → Notify Driver 2
   c. If decline/timeout → Notify Driver 3
4. On acceptance → create Trip record in DB
5. Notify rider: driver matched, ETA, driver info
```

**Why sequential, not broadcast?**
Broadcasting to all 20 nearby drivers creates a race condition. Multiple drivers accept simultaneously — only one wins, others are wasted. Sequential matching respects driver preference while minimizing waste. Uber does use a more sophisticated optimization (Hungarian algorithm / matching solver) at scale, but sequential is the interview-level answer.

---

## Trip State Machine

```
REQUESTED → DRIVER_ACCEPTED → DRIVER_EN_ROUTE → ARRIVED → IN_PROGRESS → COMPLETED
                ↑                                                              ↑
            CANCELLED                                                     CANCELLED
```

Stored in:
- **PostgreSQL:** `trips` table with all fields (authoritative state)
- **Redis:** current state of active trips (fast lookup for real-time queries)

---

## Database Schema

```sql
CREATE TABLE trips (
    id              BIGSERIAL PRIMARY KEY,
    rider_id        BIGINT NOT NULL,
    driver_id       BIGINT,
    status          TEXT NOT NULL,               -- State machine value
    pickup_lat      DOUBLE PRECISION,
    pickup_lng      DOUBLE PRECISION,
    dropoff_lat     DOUBLE PRECISION,
    dropoff_lng     DOUBLE PRECISION,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    accepted_at     TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    estimated_fare  NUMERIC(10,2),
    actual_fare     NUMERIC(10,2),
    surge_multiplier NUMERIC(4,2) DEFAULT 1.00
);

CREATE TABLE driver_locations (
    driver_id   BIGINT PRIMARY KEY,
    lat         DOUBLE PRECISION,
    lng         DOUBLE PRECISION,
    heading     SMALLINT,           -- 0-359 degrees
    speed_kmh   SMALLINT,
    is_available BOOLEAN DEFAULT TRUE,
    updated_at  TIMESTAMPTZ DEFAULT now()
);
```

---

## Surge Pricing

**Trigger:** `demand / supply ratio` in a geographic cell exceeds a threshold.

```python
def compute_surge(cell_id: str, timewindow_min: int = 5) -> float:
    active_requests = count_pending_requests_in_cell(cell_id, timewindow_min)
    available_drivers = count_available_drivers_in_cell(cell_id)
    
    ratio = active_requests / max(available_drivers, 1)
    
    if ratio < 1.5:   return 1.0
    if ratio < 2.5:   return 1.5
    if ratio < 4.0:   return 2.0
    if ratio < 6.0:   return 2.5
    return 3.0   # Cap at 3×
```

Surge multiplier is computed per S2/geohash cell and updates every 30 seconds. Riders see the surge before confirming. Uber's Marketplace team uses ML models for more sophisticated surge predictions.

---

## ETA Calculation

Real-time ETA =  routing (roads, current traffic) + historical averages.

**External dependency:** Google Maps Platform / Mapbox Directions API — provides turn-by-turn ETA considering current traffic. This is a read-through cache:
- `GEOROUTE:origin_hash:dest_hash` → Cached for 30 seconds
- On miss → call Maps API → cache → return

---

## Architecture Overview

```
[Rider App]──────────→ [API Gateway]──────────→ [Ride Service]──────→ [Postgres]
[Driver App GPS]──→ [Location Service]──→ [Redis Geo Index]
                              │                      ↑
                              ▼                  queries
                         [Kafka]──→ [Matching Service]
                              │         │
                              │         ↓ notify driver
                         [Cassandra] [WebSocket / Push]
                         (trip history)
```
