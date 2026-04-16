# URL Shortener — Quick-Fire Questions

**Q1: How many characters do you need for a 7-char Base62 short code?**
62^7 ≈ 3.5 trillion unique codes. Sufficient for hundreds of years of URL shortening at 100M URLs/day.

---

**Q2: Why use 302 instead of 301 for redirects?**
302 is a temporary redirect — browsers don't cache it permanently, so every click hits the server and can be counted for analytics. 301 is permanent — browsers cache it forever, so subsequent clicks never reach the server.

---

**Q3: What's the main risk of using sequential auto-increment IDs as short codes?**
Sequential IDs are guessable. An attacker can enumerate `/0000001`, `/0000002`, etc., to discover all shortened URLs. Use random tokens or Snowflake IDs (which have a time-sorted but non-sequential appearance) to prevent enumeration.

---

**Q4: How do you handle 115,000 click increments per second without killing the DB?**
Write-behind counter: use `INCR clicks:{code}` in Redis on each redirect. A background worker periodically (every 10s) reads and resets the Redis counter with `GETDEL`, then batches a single `UPDATE urls SET click_count = click_count + N` to the DB.

---

**Q5: What happens if two users try to create the same custom alias simultaneously?**
The DB `UNIQUE` constraint on `short_code` prevents duplicates. The second INSERT raises a `UniqueViolationError` which the service catches and returns HTTP 409 Conflict to the user.

---

**Q6: How do you implement URL expiration?**
Store `expires_at` timestamp in the DB. On every redirect, check `if now > expires_at → 410 Gone`. A background cleanup job soft-deletes expired rows, then hard-deletes them 24h later (after CDN TTL has expired). Also set Redis TTL to the expiration time so cache entries auto-evict.

---

**Q7: How do you prevent malicious URLs (phishing, malware)?**
On shorten: synchronously check `long_url` against Google Safe Browsing API or VirusTotal. Reject URLs on known bad-domain blocklists. For operator control: maintain a deactivation flag (`is_active = false`) to disable a code without deleting it.

---

**Q8: What's the read:write ratio and why does it matter?**
100:1 (10B redirects vs 100M writes/day). This means the system is dominated by reads — the redirect path must be extremely fast while write latency can be higher. Design optimimes for read (CDN + Redis caching) over write (simple DB INSERT is fine).

---

**Q9: What's the difference between the write API and redirect API?**
They should be separate services. Redirect API: stateless, read-only, sits behind CDN, scales horizontally. Write API: handles DB writes, validation, user auth — smaller scale (1,160 wps). Mixing them means scaling the entire service for redirect traffic which is wasteful.

---

**Q10: What's the birthday paradox and how does it apply here?**
The birthday paradox says you need approximately √(space) random samples before a collision becomes likely. For 62^7 ≈ 3.5T possibilities, you'd need ~1.87 million URLs before collision probability reaches 50%. With 100M URLs/day you'd hit significant collision rates within ~2 weeks. But the collision is handled by the `UNIQUE` constraint + retry, so it's not a failure — just a retry. Snowflake IDs eliminate collisions entirely.

---

**Q11: How do you store analytics per click (user agent, country, referrer)?**
Stream each redirect event to Kafka (short_code, timestamp, ip, user_agent, referer). Consumer writes to ClickHouse or BigQuery for analytics queries. Never write detailed click events to the PostgreSQL primary — it's meant for URL storage, not analytics.

---

**Q12: How do you handle the "same long URL submitted twice" case?**
You can deduplicate (same long URL → same short code) or not (each submission gets its own short code, enabling separate analytics streams). Bitly deduplicates by URL+user. TinyURL creates a new code per submission. Dedup: `SELECT short_code FROM urls WHERE long_url = $1` before INSERT (but this query doesn't use an index well at scale — needs index on `long_url` which is TEXT = expensive).
