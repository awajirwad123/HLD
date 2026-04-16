# Video Streaming — Quick-Fire Questions

**Q1: What is adaptive bitrate streaming and why is it needed?**
ABR streaming encodes video at multiple quality levels (360p/720p/1080p/4K) and splits each into short segments (2–4s). The player dynamically switches between quality tiers based on available bandwidth. Without ABR, low-bandwidth connections would need a full resolution reduction, and high-bandwidth connections would be stuck at the lowest quality that avoids buffering.

---

**Q2: What's HLS and what does a manifest file look like?**
HLS (HTTP Live Streaming) is Apple's ABR protocol. Videos are split into `.ts` segments. A master manifest (`.m3u8`) lists all available quality tiers. Each tier has its own playlist that lists its segment URLs. The player fetches the master manifest first, then selects a quality playlist and starts downloading segments sequentially.

---

**Q3: Why are video segments served via CDN and not directly from S3?**
S3 is a regional service — serving directly from S3 to global users means every request crosses the internet to one AWS region (high latency). CDN edge nodes are distributed globally (200+ PoPs). User requests are routed to the nearest PoP via GeoDNS. Popular segments are cached at the CDN edge — effectively served < 5ms away from the user.

---

**Q4: What happens to cold (rarely watched) content on a CDN?**
Popular content has a 99%+ CDN hit rate. Cold content is fetched from S3 origin on first access at a CDN PoP and cached for 24 hours. If not accessed again, LRU eviction removes it from the edge. The CDN acts as a lazy cache for long-tail content — it doesn't pre-warm unless you explicitly push content to edges.

---

**Q5: How does transcoding work and why does it take so long?**
Transcoding is re-encoding the video at different resolutions/bitrates. FFmpeg reads every frame, re-scales it, re-encodes with the target codec (H.264/H.265). For a 1-minute 4K video at 60fps: 60×60=3,600 frames must be decoded, resized, and re-encoded. 1 minute of video takes ~5–20 minutes of CPU to transcode. Solution: parallel workers (one per resolution) on GPU instances.

---

**Q6: How do you reduce transcoding time for YouTube-scale uploads?**
Parallel transcoding: each resolution runs on a separate worker simultaneously. GPU-accelerated FFmpeg (`-c:v h264_nvenc`): 10–20× faster than CPU (`libx264`). Also: distributed transcoding — split the video into chunks (by timestamp), transcode each chunk on a separate node, stitch segments. Netflix's video pipeline (Archer/Apollo) transcodes a 2-hour movie in < 30 minutes.

---

**Q7: What is CMAF and why is it important?**
Common Media Application Format (CMAF) standardizes on fragmented MP4 (`.cmfv`/`.m4s`) for both HLS and MPEG-DASH segments. Pre-CMAF: you needed separate `.ts` files for HLS and `.m4s` for DASH — 2× storage. With CMAF: one set of segments serves both protocols. HLS with CMAF and MPEG-DASH both reference the same `.m4s` files — halves storage costs.

---

**Q8: How does DRM (Digital Rights Management) work in video streaming?**
Content is encrypted at rest (segments stored encrypted in S3/CDN). When a user plays a video, their player requests a license from a DRM License Server (authenticated by subscription status). The license contains the decryption key valid for a time window. Player decrypts segments in a trusted execution environment (DRM decode hardware) — the plaintext video never exists in accessible memory. Widevine (Google) / FairPlay (Apple) / PlayReady (Microsoft) are the main DRM systems.

---

**Q9: How do you implement resume playback (continue watching)?**
Store playback position in DB: `user_resume: {user_id, video_id, position_seconds, updated_at}`. Update every ~30 seconds during playback (frontend sends heartbeat). On next play, fetch position → seek player to stored timestamp before loading the manifest. CDN serves the right segment for that timestamp (no re-download of earlier segments).

---

**Q10: How do you handle a 500-hour-per-minute upload rate at YouTube?**
500 hr/min × 60 = 30,000 hr/hr. At ~5GB/hr per video: 150TB of raw uploads per hour. Design: distributed upload service (Client → CDN upload accelerator → Regional S3). Transcode farm autoscales (spot instances + GPU). Kafka absorbs bursty job submissions. Each job is independent — horizontally scalable with no coordination needed.

---

**Q11: What's the difference between live streaming and VOD (video on demand)?**
VOD: complete file is transcoded and stored before playback. HLS playlist type = `VOD` — includes `#EXT-X-ENDLIST`. Segments are immutable; long CDN cache TTL.

Live: stream is transcoded in real-time as the broadcast happens. HLS playlist type = `LIVE` — playlist is continuously updated with new segments; no `#EXT-X-ENDLIST`. Very short CDN TTL (5–10s) on the playlist file. Buffer is small (10–30s). The challenge: transcoding latency + segment upload + CDN propagation = end-to-end stream delay (typically 6–30s for HLS, 2–6s for CMAF LHLS).
