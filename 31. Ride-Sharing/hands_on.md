# Ride-Sharing — Hands-On, Notes, Quick-Fire, Tricky, and Simulator

## hands_on.md

### Exercise 1: Geospatial Driver Matching

```python
"""
Simulate nearby driver lookup using Redis GEO commands.
"""

import asyncio
import random
import math
import redis.asyncio as redis


async def demo_geo_matching():
    rdb = redis.from_url("redis://localhost:6379", decode_responses=True)

    # Seed fake driver locations around San Francisco
    SF_LAT, SF_LNG = 37.7749, -122.4194
    drivers = {}

    print("Adding 100 drivers near San Francisco...")
    for i in range(100):
        lat = SF_LAT + random.uniform(-0.05, 0.05)  # ~5km radius
        lng = SF_LNG + random.uniform(-0.05, 0.05)
        driver_id = f"driver:{i:03d}"
        drivers[driver_id] = (lat, lng)
        await rdb.geoadd("active_drivers", [lng, lat, driver_id])

    # Mark 20 as unavailable (on a trip)
    unavailable = set(list(drivers.keys())[:20])
    for d in unavailable:
        await rdb.sadd("busy_drivers", d)

    # Rider at downtown SF
    rider_lat, rider_lng = 37.7751, -122.4179

    # Find drivers within 3km
    nearby = await rdb.georadius(
        "active_drivers", rider_lng, rider_lat,
        3, "km",
        withcoord=True, withdist=True, sort="ASC", count=20
    )

    # Filter out busy drivers
    available_nearby = [
        (driver_id, dist, coord)
        for driver_id, dist, coord in nearby
        if driver_id not in unavailable
    ]

    print(f"\nTop 5 available drivers within 3km of rider:")
    for driver_id, dist, (lng, lat) in available_nearby[:5]:
        print(f"  {driver_id}: {float(dist):.2f} km away @ ({lat:.4f}, {lng:.4f})")

    # Cleanup
    await rdb.delete("active_drivers", "busy_drivers")
    await rdb.aclose()


asyncio.run(demo_geo_matching())
```

### Exercise 2: Trip State Machine

```python
"""
Implement and drive the ride-sharing trip state machine.
"""

from enum import Enum
from datetime import datetime, timezone
from dataclasses import dataclass, field
from typing import Optional


class TripStatus(str, Enum):
    REQUESTED       = "REQUESTED"
    DRIVER_ACCEPTED = "DRIVER_ACCEPTED"
    EN_ROUTE        = "EN_ROUTE"
    ARRIVED         = "ARRIVED"
    IN_PROGRESS     = "IN_PROGRESS"
    COMPLETED       = "COMPLETED"
    CANCELLED       = "CANCELLED"


VALID_TRANSITIONS = {
    TripStatus.REQUESTED:       {TripStatus.DRIVER_ACCEPTED, TripStatus.CANCELLED},
    TripStatus.DRIVER_ACCEPTED: {TripStatus.EN_ROUTE, TripStatus.CANCELLED},
    TripStatus.EN_ROUTE:        {TripStatus.ARRIVED, TripStatus.CANCELLED},
    TripStatus.ARRIVED:         {TripStatus.IN_PROGRESS, TripStatus.CANCELLED},
    TripStatus.IN_PROGRESS:     {TripStatus.COMPLETED, TripStatus.CANCELLED},
    TripStatus.COMPLETED:       set(),
    TripStatus.CANCELLED:       set(),
}


@dataclass
class Trip:
    trip_id: str
    rider_id: str
    driver_id: Optional[str] = None
    status: TripStatus = TripStatus.REQUESTED
    events: list[dict] = field(default_factory=list)

    def transition(self, new_status: TripStatus, **metadata):
        if new_status not in VALID_TRANSITIONS[self.status]:
            raise ValueError(
                f"Invalid transition: {self.status} → {new_status}. "
                f"Allowed: {VALID_TRANSITIONS[self.status]}"
            )
        old_status = self.status
        self.status = new_status
        self.events.append({
            "from": old_status,
            "to": new_status,
            "ts": datetime.now(timezone.utc).isoformat(),
            **metadata,
        })
        print(f"Trip {self.trip_id}: {old_status} → {new_status}")

    def accept(self, driver_id: str):
        self.driver_id = driver_id
        self.transition(TripStatus.DRIVER_ACCEPTED, driver_id=driver_id)

    def start_pickup(self):
        self.transition(TripStatus.EN_ROUTE)

    def arrive(self):
        self.transition(TripStatus.ARRIVED)

    def start_ride(self):
        self.transition(TripStatus.IN_PROGRESS)

    def complete(self, fare: float):
        self.transition(TripStatus.COMPLETED, fare=fare)

    def cancel(self, reason: str, cancelled_by: str):
        self.transition(TripStatus.CANCELLED, reason=reason, cancelled_by=cancelled_by)


# Demo
trip = Trip(trip_id="trip-001", rider_id="rider-alice")
trip.accept("driver-bob")
trip.start_pickup()
trip.arrive()
trip.start_ride()
trip.complete(fare=18.50)

print("\nTrip event log:")
for event in trip.events:
    print(f"  {event['from']} → {event['to']} at {event['ts']}")

# Test invalid transition
try:
    cancelled_trip = Trip(trip_id="trip-002", rider_id="rider-carol")
    cancelled_trip.complete(fare=10.0)  # Can't complete from REQUESTED
except ValueError as e:
    print(f"\nCorrectly rejected invalid transition: {e}")
```

---

## notes.md

### Geospatial Index Options

| Option | Query type | Performance | Notes |
|---|---|---|---|
| Redis GEOADD/GEORADIUS | Radius search | O(N+log M) | Best for real-time; Geohash internally |
| PostGIS (Postgres extension) | ST_DWithin, ST_Distance | Index needed | Great for persistence + SQL joins |
| Google S2 library | Cell-based hierarchical | Very fast | Uber's actual approach; 64-bit cell IDs |
| Quadtree / R-tree | Bounding box | O(log N) | Used in spatial DBs (MySQL spatial) |

### Real-Time Location Update Flow

```
Driver GPS update (every 4s)
  ↓
Location Service API
  ├── GEOADD drivers_active {lng} {lat} {driver_id}  (Redis — real-time)
  └── Publish to Kafka (location topic)
         ├── Matching Service (nearby queries)
         ├── Trip Tracker (ETA recalculation for active trip)
         └── Analytics (heatmap, surge calculation)
```

### Surge Pricing Cells

- Divide city into ~100m × 100m S2 cells
- Each cell tracks: `active_requests / available_drivers` ratio in last 5 minutes
- Surge multiplier computed per cell, updated every 30s
- Cells aggregate upward: if block is calm but neighborhood is busy, neighborhood surge applies

### Key Numbers

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

---

## quick-fire-questions.md

**Q1: How do you find nearby drivers in real-time?**
Use Redis GEO commands: `GEOADD active_drivers {lng} {lat} {driver_id}` on each location update. `GEORADIUS active_drivers {rider_lng} {rider_lat} 5 km ASC COUNT 20` to find 20 nearest drivers within 5km. Redis stores coordinates as Geohash (52-bit integer in a Sorted Set) — O(1) update, O(log M + N) radius query.

---

**Q2: Why Redis for driver locations instead of a relational DB?**
Driver locations update every 4 seconds × 4M drivers = 1M writes/sec. PostgreSQL at 1M writes/sec requires massive hardware and doesn't natively support fast radius queries. Redis handles 1M ops/sec on a single node; GEORADIUS returns nearby drivers in < 1ms. PostgreSQL (with PostGIS) is better for persistent trip history and analytics.

---

**Q3: How does ride matching work?**
1. Rider requests → find N nearest available drivers via GEORADIUS
2. Score and rank drivers (distance, rating, vehicle type)
3. Notify top driver → wait 10s → if no accept, notify second driver
4. On accept → create Trip record, notify rider

---

**Q4: How do you calculate surge pricing?**
Divide the map into geospatial cells. In each cell, compute `pending_requests / available_drivers` over the last 5 minutes. Above threshold ratios apply multipliers (1.5×, 2×, etc.). Update every 30 seconds. Rider sees the current multiplier before confirming the ride.

---

**Q5: What's the trip state machine?**
REQUESTED → DRIVER_ACCEPTED → EN_ROUTE → ARRIVED → IN_PROGRESS → COMPLETED
Any non-terminal state can go to CANCELLED. Transitions are validated server-side (reject invalid transitions). Stored in PostgreSQL with timestamps for billing/analytics.

---

**Q6: How do you prevent double-booking a driver?**
Database-level constraint: `UPDATE drivers SET is_available = false WHERE driver_id = $1 AND is_available = true`. If the UPDATE affects 0 rows, the driver was already booked — retry with next driver. Use PostgreSQL optimistic locking or a SELECT FOR UPDATE to atomically check and book.

---

**Q7: How do you calculate fare?**
`fare = base_fare + (distance_km × per_km_rate) + (duration_min × per_min_rate)`. Applied with `surge_multiplier`. Distance and duration are measured from GPS trail stored in Cassandra. Final fare only calculated on trip completion.

---

**Q8: How does the driver app communicate location in real-time?**
Driver app sends HTTP POST (or WebSocket frames) to the Location Service every 4 seconds with current GPS coordinates. HTTPS /websocket ensures the update is authenticated (driver JWT). Location service writes to Redis and publishes to Kafka.

---

## tricky_interview_questions.md

**Q1: "A driver is at the edge of two geohash cells. Their location straddles the border and nearby queries from different cells don't return them. How do you fix this?"**

The "edge problem" in geohash: if a driver is at lat/lng that falls on a cell boundary, a `GEORADIUS` query centered in the adjacent cell with radius R might miss them if R exactly matches the cell size.

Fix: Query 8 adjacent cells in addition to the center cell (Redis `GEORADIUS` handles this automatically with a radius larger than cell size). Alternatively, use a `GEORADIUS` radius 20% larger than needed, then filter results by actual distance. Redis `GEORADIUS` with a proper km radius handles border cases correctly — it doesn't use cell boundaries for filtering, it computes actual distances.

**The insight:** `GEORADIUS` is distance-based, not cell-based. The internal Geohash is just for indexing efficiency. The returned results are filtered by actual Haversine distance, so border artifacts don't occur.

---

**Q2: "Driver location updates stop for 30 seconds (GPS signal lost in a tunnel). What should the system do?"**

Options:
1. **Dead reckoning:** Use the last known heading and speed to estimate current position during the outage. Show estimated position on the map with an "estimated" indicator.
2. **Freeze last known position:** Display last known location. Don't remove driver from the active index until TTL expires (e.g., 60 seconds).
3. **Pause ETA updates:** The ETA recalculation service skips updates for trips where position is stale.

Redis GEO key for the driver remains until explicitly updated or TTL-expired. Driver's active status should not be changed based on a 30-second gap.

**What NOT to do:** Remove the driver from the active index after 30 seconds — this would wrongly appear as if they went offline.

---

## interview_simulator.md

**Scenario 1: "Design Uber"**

*Step 1 — Clarify:* "5M trips/day, 4M drivers globally, real-time matching in < 300ms, GPS tracking every 4s?"

*Step 2 — Estimate:* 1M location updates/sec. 500K active trips peak.

*Step 3 — Components:*
- Location Service → Redis GEOADD
- Ride Service → PostgreSQL trips table + Redis trip state
- Matching Service → GEORADIUS + driver scoring + sequential offer
- Notification Service → WebSocket (driver) + push notification (rider)
- Surge Service → S2 cell demand/supply ratios
- Trip Tracker → ETA from Maps API, Cassandra GPS trail

*Step 4 — Data flow:* Walk through ride request → matching → acceptance → en route → completion.

*Step 5 — Scaling:* Location updates 1M/sec → Redis Cluster (shard by geospatial region). Trip state → PostgreSQL with read replicas. Matching → stateless service, scales horizontally. Kafka decouples services.

---

**Scenario 2: "New Year's Eve surge — every user tries to request a ride at midnight simultaneously. Design for this."**

Pre-scale driver count: incentivize drivers with guaranteed earnings during 11pm–1am. Surge pricing rises steeply to balance supply/demand (economic mechanism). Pre-warm Redis GEO index. Scale Location Service horizontally (expect 5× normal location update rate). Rate limit ride requests (max N retries per rider per minute). Queue overflow requests (FIFO + timeout).

---

**Scenario 3: "A driver claims they were incorrectly charged for tolls. How do you dispute-resolve this?"**

GPS trail stored in Cassandra for every trip. Replay the GPS points → reconstruct the route → check if toll roads were traversed. Cross-reference with toll database. If GPS signal was weak at the toll location (< 10m accuracy), flag for manual review.
