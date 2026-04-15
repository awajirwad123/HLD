# Ride-Sharing — Interview Simulator

## Scenario 1: "Design Uber"

**Interviewer:** "Design a ride-sharing platform like Uber. Cover driver tracking, ride matching, and real-time updates."

---

### Strong Answer Walkthrough

**Step 1 — Clarify (2 min)**
> "5M trips/day, 4M active drivers, real-time matching, GPS tracking every 4s, surge pricing, global deployment?"

**Step 2 — Estimate**
- Location updates: 4M drivers × 1/4s = 1M writes/sec (peak)
- Active trips: ~500K concurrent
- Matching latency target: < 300ms

**Step 3 — Core services**

| Service | Responsibility |
|---|---|
| Location Service | Receive GPS updates → write Redis GEO index + Kafka |
| Ride Service | Handle ride requests, manage trip state |
| Matching Service | Query Redis GEORADIUS, score drivers, send offers |
| Notification Service | Push to driver (WebSocket), rider (WebSocket + push) |
| Trip Tracker | Monitor active trips, ETA updates from Maps API |
| Surge Service | Compute demand/supply ratio per cell every 30s |

**Step 4 — Data flow: ride request**
```
1. Rider POST /rides {pickup_lat, pickup_lng, dropoff_lat, dropoff_lng}
2. Ride Service: create Trip(status=REQUESTED) in PostgreSQL
3. Matching Service:
   a. GEORADIUS active_drivers 5km → 20 candidates
   b. Filter busy drivers (Redis SET)
   c. Score candidates
   d. Offer to driver 1: WebSocket push → wait 10s
   e. Accept → atomic DB UPDATE driver is_available=false
   f. Trip status → DRIVER_ACCEPTED → notify rider
4. Driver app: update status via WebSocket frames
```

**Step 5 — Location tracking**
- Driver sends GPS every 4s → Location Service
- Location Service: `GEOADD active_drivers {lng} {lat} {driver_id}`
- Also publishes to Kafka for trip tracker and analytics

**Step 6 — Surge pricing**
- Surge Service reads pending requests + available drivers per S2 cell
- Computes multiplier → stores in Redis per cell_id
- Rider sees multiplier before confirming trip

**Step 7 — Scale**
- Location writes: Redis Cluster sharded by city
- Trip history: PostgreSQL (500K trips ≈ 50K rows/sec at peak — manageable with 10 replicas)
- GPS trail: Cassandra (append-only, time-partitioned by trip_id)
- Matching: stateless, horizontal scale behind a load balancer

---

## Scenario 2: "Uber Eats — Design a food delivery service"

**Interviewer:** "Extend the Uber design to food delivery. What changes?"

---

### Key Differences

| Aspect | Uber Rides | Uber Eats |
|---|---|---|
| Matching | Rider ↔ Driver | Restaurant + Driver + Customer (3-way) |
| Time constraints | Pickup immediately | Restaurant prep time (15–30 min) |
| Multi-stop | No | Driver picks up from restaurant, delivers to customer |
| Order tracking | Driver real-time position | Driver position + restaurant prep status |
| Cancellation window | Before driver accepts | Before restaurant starts preparing |

**New components:**

```
Order Service:
  - Customer places order
  - Routes to Restaurant POS system (accept/reject)
  - Computes expected ready_time (restaurant ETA)
  
Dispatch Service:
  - Waits until ready_time - 10min, THEN matches a driver
  - Driver matched just before food is ready (minimize driver wait at restaurant)
  
Restaurant Dashboard:
  - WebSocket to restaurant-side app
  - Update order status: RECEIVED → PREPARING → READY
  - Real-time ETA to driver arrival
```

**The key insight: Don't match driver immediately on order placement.** If you match a driver when the customer orders, the driver arrives at the restaurant 5 minutes later but the food takes 25 more minutes → driver waits 20 minutes → inefficient. Match driver at `ready_time - driver_ETA_to_restaurant`. Requires knowing restaurant prep time (historical average per dish).

---

## Scenario 3: "Uber during an earthquake — all areas turn into high-surge zones simultaneously. Design for this disaster scenario."

**Interviewer:** "Assume a major earthquake. 100× normal requests. Drivers go offline (battery, roads blocked). What does the system need to do differently?"

---

### Design Adaptations

**Problem 1: Requests flood the system at 100× normal rate.**
- Rate limiting per rider: max 1 pending request every 60 seconds
- Request queuing: place new requests in a priority queue (medical emergencies first, then standard by wait time)
- Graceful degradation: display "very high demand, estimated wait 45 min" instead of matching immediately

**Problem 2: Most drivers go offline.**
- Surge pricing ceases (no incentive with no drivers)
- System switches to "queue mode": requests are queued and matched when drivers come online
- Drivers who re-enter availability get matched to the oldest queued request in their cell

**Problem 3: Infrastructure strain.**
- Location update interval relaxed from 4s to 30s (reduces Redis write load by 7.5×)
- Non-critical services (recommendations, marketing) shed load automatically
- Use chaos runbook: pre-written procedures for each failure mode

**Problem 4: Safety mode.**
- Activate "Safety Routing": prefer police/hospital routes for critical rides
- Partnership API: if hospital in the destination, route to emergency entrance
- Integration with emergency services: batch of coordinates sent to local emergency coordinators every 5 minutes
