# Ride-Sharing — Tricky Interview Questions

## Q1: "A driver is at the edge of two geohash cells. Nearby queries from either cell might miss them. How do you fix this?"

**The edge problem:** Geohash divides the world into rectangular cells. A driver sitting exactly on a cell boundary is technically in one cell but very close to a rider in the adjacent cell. A naive "give me all drivers in cell X" query misses them.

**Why this doesn't apply to GEORADIUS:**
`GEORADIUS` (and its successor `GEOSEARCH`) use **true Euclidean/Haversine distance** — not cell boundaries. The geohash is just an index used internally to narrow the candidate set. The final filtering is always by actual distance. So a driver 0.1km from a rider is returned regardless of which geohash cell they're in,as long as the search radius covers 0.1km.

**Where cell problems DO occur:** If you build custom cell-based matching (e.g., `SMEMBERS cell:{geohash}`) without the surrounding cells, you get boundary misses. Fix: always query the center cell + 8 adjacent cells. In Geohash, neighbors share a prefix — you can compute all 8 neighbors from the center cell prefix.

**The insight they're testing:** Understand that Redis GEORADIUS is distance-based, not cell-based. Custom geohash implementations need neighbor cell queries.

---

## Q2: "On New Year's Eve, 50× the normal ride requests come in at midnight. Your location service Redis node can't handle the load. How do you scale?"

**Why Redis struggles at 50× load:**
Normal: 1M location updates/sec. NYE peak: 50M/sec — even Redis Cluster would need careful design.

**Solutions:**

1. **Geographic sharding of Redis.** Partition the GEO index by geographic region (city or country level). Each city has its own Redis instance (or Redis Cluster shard). `active_drivers:new_york`, `active_drivers:london`, etc. A location update goes to only one shard based on city. `GEORADIUS` queries a single shard. This eliminates cross-shard coordination.

2. **Client-side batching.** Instead of every driver sending an update every 4 seconds, aggregate updates on the client side: send only if driver moved > 50 meters. Drivers waiting at a curb don't update at all. Reduces update rate by 30–60% during peak (many drivers are parked waiting).

3. **Pre-scaling.** Provision 5× normal capacity ahead of NYE (auto-scaling + pre-warming Redis Cluster).

4. **Rate throttling of location updates.** Drop to 8-second intervals during extreme load (slight increase in location staleness, acceptable trade-off).

5. **In-memory aggregation on Location Service.** Location Service instances buffer last known positions in local memory. Redis writes happen every 2 seconds (throttled), not per update. GEORADIUS accuracy drops by ≤ 50m (one car length on surface streets) — acceptable for matching.

---

## Q3: "Two riders request a ride at the same time. They're 200m apart, and there's only one available nearby driver. How do you ensure only one gets the driver?"

**Race condition:** 
- Server A processes Rider 1's request: finds Driver D available → sending offer
- Server B processes Rider 2's request: also finds Driver D available → sending offer
- Driver D accepts both → double booking

**Solution: Optimistic lock with atomic DB state change**

```sql
-- Atomic booking attempt
UPDATE drivers
SET is_available = false, current_trip_id = $1
WHERE driver_id = $2
  AND is_available = true     -- Condition: still available
RETURNING id;
```

If the UPDATE returns 0 rows → another server already booked this driver → try the next candidate.

**More rigorous: Distributed lock (Redis)**

```
SET booking_lock:{driver_id} {trip_id} NX EX 30
```

- `NX`: only sets if not already set
- If SET returns nil → another server holds the lock → skip this driver
- On accept: lock is kept for 30s (long enough to complete DB write)
- On driver decline: `DEL booking_lock:{driver_id}` → driver becomes bookable again

**Why not just rely on DB constraint?** The DB constraint catches the race at the DB level, but means concurrent requests both go to DB, generating unnecessary contention. Redis lock catches it earlier, at the application layer, sparing DB load.

---

## Q4: "Compute the fare for a completed trip. The driver's GPS was intermittent — you have gaps in the location trail. How do you handle fair fare calculation?"

**Problem setup:**
GPS trail for a 20-minute trip: 
- Min 0–5: 60 points (15-second intervals) — normal
- Min 5–8: 0 points (GPS outage, e.g., tunnel)
- Min 8–20: 72 points — normal

If you naively sum Haversine distances between adjacent GPS points, the 3-minute gap produces zero distance for that segment — undercharging the driver.

**Solutions:**

1. **Dead reckoning for gap:** Use the last known heading (bearing) and speed. Estimate distance during gap: `gap_distance = avg_speed_kmh × gap_duration_hours`. Fill the gap segment with this estimate.

2. **Detect gap and apply time-based fare for the gap window:** `fare_for_gap = per_minute_rate × 3 minutes`. This is fair — the driver was still providing service; the meter still runs on time.

3. **Use predicted route for gap segment:** At trip start, the route was computed (Google Maps API). GPS points should follow this route. During a GPS gap, interpolate along the predicted route for the gap duration.

4. **Flag for review:** If gap is > 5 minutes and gap-estimated distance > 20km, flag for manual review (possible GPS spoofing or data corruption).

**The insight:** Fare calculation must account for GPS unreliability. Production fare engines always combine time + distance + a fallback estimate for gaps. Pure distance-only fare would be gameable (driver takes a long route but with weak GPS reporting a short route).

---

## Q5: "Driver says their rating dropped unfairly because of a ride they didn't accept. How does your system protect drivers from bad-faith low ratings?"

**The problem stated:** Rider gave 1 star because the driver went to the wrong pickup location. But the wrong-location address was provided by the rider's app due to a GPS error — not driver negligence.

**System design angles:**

1. **Separate component-level ratings.** Instead of one overall trip rating, rate each component: pick-up accuracy, driving quality, vehicle cleanliness, communication. A GPS pickup error affects pick-up accuracy only; it's contextualized, not a full trip penalty.

2. **Contextual rating analysis.** Automatically detect if the pickup latitude/longitude shown in the app was > 100m from the actual driver pickup (GPS error). If so, filter the rating from the driver's score (or apply lower weight).

3. **Statistical outlier filtering.** One 1-star rating among 500 five-star ratings: apply Bayesian average — weight extreme outliers less. `adjusted_rating = (prior_mean × C + sum(ratings)) / (C + N)` where C is a confidence constant.

4. **Dispute resolution UI.** Driver can flag a specific rating as inappropriate. Human review team checks GPS trail, rider's address, and contextual data before making a decision.

5. **Floor the impact.** A single bad rating can move the average by at most 0.1 points. Compute rolling 30-day average over last 500 trips — more resistant to occasional noise.
