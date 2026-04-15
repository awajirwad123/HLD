# Ride-Sharing — Quick-Fire Questions

**Q1: How do you find nearby drivers in real-time?**
Use Redis GEO commands: `GEOADD active_drivers {lng} {lat} {driver_id}` on each location update. `GEORADIUS active_drivers {rider_lng} {rider_lat} 5 km ASC COUNT 20` to find the 20 nearest drivers within 5km. Redis stores coordinates as a Geohash-encoded integer in a Sorted Set — O(1) update, O(log M + N) radius query, < 1ms latency.

---

**Q2: Why Redis for driver locations instead of a relational DB?**
Driver locations update every 4 seconds × 4M drivers = 1M writes/sec. PostgreSQL at 1M writes/sec requires massive hardware and doesn't natively support fast radius queries. Redis handles 1M ops/sec on a single node; GEORADIUS returns nearby drivers in < 1ms. PostgreSQL (with PostGIS) is better for persistent trip history and analytics queries.

---

**Q3: How does ride matching work at a high level?**
1. Rider requests → `GEORADIUS` returns N nearest available drivers
2. Score candidates: distance + rating + acceptance rate + vehicle type
3. Offer sequentially to top-scored driver, wait 10s
4. On decline/timeout → move to next candidate
5. On accept → atomically create Trip record + mark driver unavailable

---

**Q4: How do you prevent double-booking a driver?**
Database-level: `UPDATE drivers SET is_available = false WHERE driver_id = $1 AND is_available = true RETURNING id`. If 0 rows updated → driver already booked → select next candidate. This is an atomic check-and-set. Alternative: use SELECT FOR UPDATE with a transaction.

---

**Q5: How do you calculate surge pricing?**
Divide the map into geospatial cells. In each cell, compute `pending_requests / available_drivers` ratio over the last 5 minutes. Above threshold ratios apply multipliers (1.5×, 2×, etc.), capped at 3×. Update every 30 seconds. Rider sees the current multiplier before confirming.

---

**Q6: What's in the trip state machine?**
REQUESTED → DRIVER_ACCEPTED → EN_ROUTE → ARRIVED → IN_PROGRESS → COMPLETED. Any non-terminal state can transition to CANCELLED. Server validates transitions — client cannot jump states. Each transition records a timestamp for billing (per-minute fare) and SLA analytics.

---

**Q7: How does ETA get computed and updated during a ride?**
ETA = call to Maps API (Google Maps / Mapbox) with current driver location and rider/destination coordinates. Result cached in Redis (TTL = 30s). During an active trip, the trip tracking service recalculates ETA every GPS update by querying the cache. On cache miss, a fresh Maps API call is made. Driver GPS trail is streamed via Kafka to the trip tracker.

---

**Q8: How do you store the GPS trail during a trip for fare calculation?**
Each driver GPS update (every 4s) during an active trip is written to Cassandra partitioned by `trip_id`. At trip completion, the fare service reads all GPS points, calculates total distance using the Haversine formula between consecutive points, applies the per-km rate, and adds the per-minute component.

---

**Q9: How does the driver app communicate location in real-time?**
Driver app sends HTTP POST (or WebSocket frames) to the Location Service every 4 seconds with current GPS coordinates. HTTPS + JWT authentication ensures the update is tied to the correct driver account. Location service writes to Redis GEO index and publishes to Kafka for downstream consumers.

---

**Q10: What happens if a driver's GPS signal is lost during a trip?**
Hold the last known position in Redis. Use dead reckoning (last heading × speed × elapsed time) to estimate current position. Don't charge extra distance during the signal gap. Flag the trip for review if the gap exceeds 60 seconds. Resume normal GPS tracking when signal returns.

---

**Q11: How do you handle a driver who GPS-spoofs their location to appear in a high-surge area?**
Cross-reference GPS coordinates with accelerometer/gyroscope data from the driver app (real movement has consistent sensor patterns). Compare position delta to maximum physically plausible speed. Require driver to actually arrive at a pickup location before a trip starts (can't fake the arrival photo/confirmation). Flag accounts with statistically anomalous GPS patterns for fraud review.

---

**Q12: Why Cassandra for GPS history instead of PostgreSQL?**
GPS updates: every 4 seconds × 500K active trips = 125K writes/sec for trip GPS history. Cassandra handles high-throughput append-only writes. Data model: `PRIMARY KEY (trip_id, timestamp)` — natural for time-series per trip. PostgreSQL could work at smaller scale but struggles with 125K writes/sec on a single node.
