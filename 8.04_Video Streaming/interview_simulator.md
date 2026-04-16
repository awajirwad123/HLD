# Video Streaming — Interview Simulator

## Scenario 1: "Design YouTube"

**Interviewer:** "Design YouTube. Walk me through upload, storage, playback, and scaling."

---

### Strong Answer

**Step 1 — Clarify**
> "YouTube-scale: 2B users, 500 hours uploaded per minute, ~1B hours watched per day. Focus on: upload pipeline, transcoding, CDN delivery, and recommendations?"

**Step 2 — Estimate**
- Uploads: 500hr/min → ~8.3 hours/sec raw video
- Transcoded storage: each hour of video ≈ 6GB across resolutions → ~8.3 × 6 = ~50 GB/sec new storage
- Reads: 1B hours watched/day = ~11.6M hours/hour = ~3.2K hours/sec being played = ~5.4 Tbps of bandwidth

**Step 3 — Upload Pipeline**
```
Client → Upload Service (multipart, S3 via resumable upload protocol)
       → Raw video to S3
       → Kafka event: {video_id, s3_key, user_id}
       → Transcoding workers (auto-scaled, GPU instances)
         ├── 360p worker
         ├── 720p worker
         └── 1080p worker
       → Segments + manifests uploaded to CDN origin (S3)
       → DB updated: status = ready, manifest_url set
       → User notified: "Your video is live"
```

**Step 4 — Playback**
```
User clicks video:
1. GET /videos/{id}/manifest → manifest_url from PostgreSQL (cached in Redis)
2. GET CDN manifest.m3u8 → list of quality tiers
3. Player selects quality based on bandwidth
4. GET CDN seg_0001.ts → cached at CDN edge → stream
5. ABR switches quality as bandwidth changes
```

**Step 5 — Scale challenges addressed**
- Thumbnail rendering on CDN (generated during transcoding)
- View count via Redis INCR + flush
- Search: Elasticsearch index on title/description/tags, updated via Kafka consumer
- Recommendations: Two-tower model, candidate retrieval + ranking

---

## Scenario 2: "Netflix starts buffering during peak evening hours. Diagnose and fix."

**Interviewer:** "Users report buffering from 8pm–10pm. CDN hit rate is still 95%. What could cause this?"

---

### Investigation

Even with 95% CDN hit rate, buffering can occur if:

1. **CDN edge bandwidth exhausted.** 8pm = peak viewing. CDN PoP serving a city is at 100% network capacity → packets queued → high latency → ABR drops to 360p → buffering below 360p download speed. Fix: CDN overprovisioning contract; geographic load balancing (route to less-loaded PoPs).

2. **ABR selected too high a quality.** Users on 50Mbps home connections are watching 4K. At 8pm, ISP backhaul traffic is congested → effective bandwidth drops to 10Mbps → ABR hasn't adapted fast enough → 4K segment downloads slower than 4s = stall. Fix: ABR algorithm must be responsive (buffer-based + bandwidth estimation); Netflix's BOLA algorithm is designed for this.

3. **ISP peering congestion.** Netflix and the ISP may have a peering agreement that's saturated during peak hours. Netflix sends traffic via a congested interconnect. Fix: Netflix Open Connect — place appliances physically inside the ISP's network to eliminate the congested peering link entirely.

4. **Upstream bandwidth exhaustion at CDN origin.** If the 5% cache misses are going to S3 origin, and peak traffic is huge, even 5% of 5.4 Tbps = 270 Gbps of origin traffic. If S3 origin bandwidth is limited → misses are slow → video stalls until segment downloads. Fix: Origin shield (mid-tier CDN cache) between edge PoPs and S3.

---

## Scenario 3: "Design a live streaming platform (Twitch-like)"

**Interviewer:** "Design a live streaming platform for gaming. Streamers broadcast, viewers watch with < 5 second delay."

---

### Key Differences from VOD

| Aspect | VOD (Netflix/YouTube) | Live (Twitch) |
|---|---|---|
| Content ready before playback? | Yes | No |
| Manifest type | `VOD` (end known) | `LIVE` (updated continuously) |
| Segment availability | All segments exist | New segment every 2s |
| CDN TTL for playlist | 30s | 2s |
| Buffer strategy | 30s pre-buffer | 10s live buffer |
| Seeking | Full range | Only backward to start of DVR window |

**Architecture:**

```
Streamer (OBS) → RTMP ingest server
                      ↓
              Transcoding farm (real-time)
                 ├── 720p@30fps
                 └── 1080p@30fps
                      ↓
              CDN origin: new segments pushed every 2s
                      ↓
              CDN edge: serves live playlist to all viewers
                      ↓
              Viewers: HLS player, 2s segment polling
```

**RTMP ingest:** Streamers' OBS sends RTMP (Real-Time Messaging Protocol) to ingest servers. Ingest servers transcode in real-time using FFmpeg (GPU). Segments uploaded to CDN origin using `ffmpeg -f hls` output.

**Edge PoP selection:** Viewer routed to CDN PoP nearest to them. CDN PoP fetches the live manifest from origin every 2s (or CDN uses origin push via WebSockets). Each new segment is pull-fetched on first viewer request.

**Latency optimization:** Standard HLS = 6–30s latency (segment size + CDN propagation). LL-HLS (Low-Latency HLS) = 1–2s latency using partial segments (0.5s chunks pushed incrementally) and `#EXT-X-PRELOAD-HINT` to notify the CDN edge before the segment is fully written.

**DVR window:** Store the last 60 minutes of live stream segments. Allows viewers to seek back up to 60 minutes. Older segments expire (deleted or moved to VOD).

**Chat (Twitch chat):** WebSocket connection to chat system (separate from video streaming). Messages routed via pub/sub (Redis Streams or Kafka). Fan-out to all connected viewers for that stream. At peak (100K viewers per stream), even simple fan-out is 100K WebSocket pushes per message.
