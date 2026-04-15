# Video Streaming — Tricky Interview Questions

## Q1: "Netflix shows < 1 second startup time. How is this possible at scale with adaptive bitrate?"

**The trap:** "Fast CDN." That's necessary but not sufficient.

**The real answer — multiple techniques:**

1. **Predictive pre-fetching of the manifest.** When a user hovers over a thumbnail (or the app predicts they'll click Next Episode), Netflix pre-fetches the master manifest and the first segment of the most-likely-chosen quality. By the time the user clicks play, segment 1 is already downloading.

2. **HEVC/AV1 codec efficiency.** Higher compression ratio → smaller segments → faster download → buffer fills faster at lower bandwidth.

3. **Fast start quality selection.** On click, the ABR algorithm starts at a conservative quality (720p or 480p depending on recent bandwidth probe) rather than the lowest quality. It doesn't start at 360p and ramp up. This avoids the visual quality shock.

4. **CDN anycast + intelligent traffic management.** Netflix's Open Connect appliances are physically installed in ISP data centers. For large ISPs, the CDN is literally on the same LAN as the user → < 1ms RTT. First byte latency is near zero.

5. **Client-side bandwidth probe.** The player pings the CDN edge before a user clicks play (background request) to measure bandwidth. By click time, the ABR already knows which starting quality to use.

6. **Startup optimization in HLS:** Start buffering before the full manifest loads using MP4 `moov` box pre-fetching, which lets decoding begin with fewer round trips.

---

## Q2: "A popular video is causing massive CDN cache misses — 30% of requests hit S3 origin. You'd expect 99% cache hit for popular content. What's wrong?"

**Root cause possibilities:**

1. **Cache Busting query parameters.** Request URL has a tracking parameter: `https://cdn.example.com/video/seg001.ts?user_id=12345&t=1713175200`. Every user generates a unique URL → every request is a miss. Fix: strip tracking parameters at the CDN edge; route them to analytics only.

2. **Short CDN TTL.** Manifest file has `Cache-Control: no-store` or `max-age=1`. Each request refreshes from origin. Fix: playlist TTL = 30s, segments = 1 year.

3. **Geographic CDN miss.** The video just went viral in a new region. CDN PoP in Southeast Asia has never cached it. First viewers in that region all miss. Fix: CDN push (proactively push content to all PoPs for viral/scheduled-release content).

4. **Cache fragmentation by byte range.** Player is requesting byte ranges of the segment files (HTTP Range requests). CDN doesn't support byte-range caching correctly — each range request is treated as a separate cache entry → all miss. Fix: serve whole segments, not byte ranges. HLS is already segment-based; verify your CDN config doesn't break it.

5. **Cache invalidation bug.** A deployment script is calling `CloudFront CreateInvalidation` on the video segments (not just the manifest). This evicts cached segments. Fix: invalidate only the manifest; segments are immutable and should never be invalidated.

---

## Q3: "You want to implement skip-ahead seeking in a VOD video efficiently. A user jumps from minute 5 to minute 47. How does your system serve this?"

**How seeking works with HLS:**

Each segment has a start timestamp encoded in the playlist:
```m3u8
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:4.000,
seg_0001.ts  ← 0–4s
#EXTINF:4.000,
seg_0002.ts  ← 4–8s
...
#EXTINF:4.000,
seg_0706.ts  ← 2824–2828s (47:04–47:08)
```

Player seeking to 47:00:
1. Calculate segment number: `floor(47*60 / 4) + 1 = 706`
2. Request `seg_0706.ts` directly — no need to download seg_0002 through seg_0705
3. CDN serves `seg_0706.ts` directly from edge cache (it may already be cached if other users reached that point)

**Keyframe alignment:** Seeking snaps to the nearest keyframe. HLS segments start at keyframes. So seeking to 47:00 snaps to the start of `seg_0706.ts` (the closest 4-second boundary).

**For trick-play thumbnails** (progress bar preview images while scrubbing):
Generate a "thumbnail sprite" — one image strip with a frame every 10 seconds. Store as a single JPEG (faster to load than N individual images). Serve via CDN. No database needed.

---

## Q4: "A new season of a popular Netflix show drops at midnight. Hundreds of millions of users click play simultaneously. How do you prevent the infrastructure from falling over?"

**The thundering herd at video scale:**

500M users × 2 requests (manifest + first segment) in the first 60 seconds = 1B requests/min = ~16.7M rps. This is extreme.

**Strategies:**

1. **Pre-push to CDN (origin shield warm-up).**
   6 hours before release: push all segments for S01E01 to every CDN PoP globally. When users click play, all requests are cache hits. S3 origin receives ZERO traffic. This is the primary technique Netflix uses for new releases.

2. **Rate-based release (geo-staggered).**
   Don't release to all regions simultaneously. Eastern Australia gets it first (midnight there = noon in the US). Stagger by timezone → traffic extends over 24 hours rather than a 1-hour spike.

3. **CDN capacity pre-scaling.**
   Negotiate bandwidth guarantees with CDN provider for the expected spike. CDN provider pre-scales edge capacity.

4. **Manifest caching with short TTL stagger.**
   All manifests set a random `max-age` between 28–32s (not exactly 30s). Avoids a situation where 500M manifest caches expire at the exact same second and all revalidate simultaneously.

5. **Origin shield (CDN mid-tier).**
   CDN edge misses → hit a regional origin shield (also CDN, closer to origin) → only the shield hits S3. Shields absorb CDN-level cache misses; S3 only gets traffic from N shields, not 500M users.

---

## Q5: "How do you handle video copyright — detecting if a user is uploading copyrighted content?"

**Content ID / fingerprinting (YouTube's approach):**

1. **Fingerprint generation.** Rights holder uploads their content. System generates a **perceptual hash** (fingerprint) of the video: sample frames every N seconds → compute a compact hash that's robust to re-encoding and slight modifications.

2. **New upload matching.** When a user uploads a new video, generate its fingerprint. Compare against the rights holder database. If match above threshold → copyright claim.

3. **Audio fingerprinting.** Separate audio track fingerprint (like Shazam). Catches videos where only the audio is copyrighted (e.g., background music).

4. **Actions on match:**
   - Block video
   - Mute audio
   - Allow with monetization going to rights holder (YouTube's Content ID model)
   - Allow if rights holder pre-approved

5. **Evasion resistance.** Users try to evade: mirroring, pitch shift, speed change. Perceptual hashing algorithms are designed to be robust to these transformations (DCT-based hashing, chromaprint for audio).

**Technical implementation:** Video fingerprint service runs as a separate pipeline stage after transcoding. Result stored in a fingerprint DB (approximate nearest-neighbor index, e.g., FAISS or purpose-built like Google's Rovi/Vobile). Most comparisons complete in < 1 second.
