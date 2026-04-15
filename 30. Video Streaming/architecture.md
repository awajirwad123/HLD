# Video Streaming (Netflix / YouTube) — Architecture

## Problem Statement

Design a video streaming platform that:
- Supports uploading, transcoding, and streaming videos
- Serves ~1B DAU (YouTube scale) or 200M subscribers (Netflix scale)
- Video playback starts in < 2 seconds (Netflix target: < 1 second)
- Supports adaptive bitrate streaming (480p / 720p / 1080p / 4K)
- Handles ~500 hours of video uploaded per minute (YouTube)

---

## Capacity Estimation

```
Netflix scale:
  200M subscribers, 2h avg watch/day → 400M hours/day
  Peak bandwidth: 40% of internet traffic globally

YouTube scale:
  500 hours uploaded per minute
  1 min raw video → 20min transcoding (processing per minute uploaded)

Storage per 1 hour of video:
  Raw 4K:    ~100 GB
  Transcoded (all bitrates): ~6 GB (1080p=3GB, 720p=1.5GB, 480p=0.8GB, 360p=0.5GB)
  
YouTube total storage: 500hr/min × 60min × 24hr × ~6GB = 25.9 PB/day (all resolutions)
```

---

## Upload and Transcoding Pipeline

```
User uploads raw video
       ↓
Upload Service (FastAPI / S3 multipart)
       ↓ Stores raw file
Raw Video Bucket (S3 / GCS)
       ↓ Publishes event to Kafka
Transcoding Job Queue (Kafka)
       ├─ Worker 1: transcode → 360p
       ├─ Worker 2: transcode → 720p
       ├─ Worker 3: transcode → 1080p
       └─ Worker 4: transcode → 4K (if source quality allows)
       ↓ Each worker stores output segments
CDN Origins (Object Storage)
       ↓
CDN Edge Nodes (Global PoPs)
       ↓
User's video player
```

**Why parallel transcoding?** A 10-minute 4K video at 30fps takes ~20 minutes to transcode serially. Parallel workers = 4 resolutions × parallel = 5 minutes.

---

## Adaptive Bitrate Streaming (ABR)

**Problem:** User's bandwidth changes (mobile → WiFi → 3G). One fixed bitrate is either too low quality or buffers constantly.

**Solution: HLS (HTTP Live Streaming) or MPEG-DASH**

Both work the same way:
1. Video is split into **2–4 second segments** (`.ts` chunks for HLS, `.m4s` for DASH)
2. For each segment, multiple quality versions are stored
3. A **manifest file** lists all available quality tiers and segment URLs

**HLS manifest example:**
```m3u8
#EXTM3U
#EXT-X-VERSION:3

#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
https://cdn.example.com/video/v123/360p/index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720
https://cdn.example.com/video/v123/720p/index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
https://cdn.example.com/video/v123/1080p/index.m3u8
```

**Per-resolution playlist:**
```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:4
#EXTINF:4.000,
seg_001.ts
#EXTINF:4.000,
seg_002.ts
...
```

**Player adaptive logic:** ABR algorithm monitors download speed. If `segment_download_time < segment_duration / 2` → buffer is filling → bump quality. If `download_time > segment_duration` → buffer draining → drop quality. Target: 30 seconds of buffer ahead.

---

## CDN Architecture

Videos are served from CDN, never directly from origin storage:

```
User Request → DNS → CDN PoP (nearest)
               CDN HIT → stream directly ← 99% of traffic for popular content
               CDN MISS → Pull from Origin (S3) → cache segment → stream
```

**Content popularity distribution follows power law:** Top 1% of videos = 90%+ of traffic. CDN cache hit ratio for these is ~100%.

**Long-tail content:** A video uploaded 3 years ago with 50 total views might not be cached on any CDN PoP. User watches it → CDN pulls from S3 origin → cached for 24h at that PoP.

**Cache TTL:** Video segments are immutable (transcoded video never changes). `Cache-Control: max-age=31536000` (1 year). Manifest files have short TTL (30s) to support quality updates or DRM key rotation.

---

## Database Design

### Metadata DB (PostgreSQL / MySQL)

```sql
CREATE TABLE videos (
    id           BIGSERIAL PRIMARY KEY,
    user_id      BIGINT NOT NULL REFERENCES users(id),
    title        TEXT NOT NULL,
    description  TEXT,
    duration_sec INT,
    status       TEXT CHECK (status IN ('uploading','processing','ready','failed')),
    manifest_url TEXT,          -- CDN URL to master manifest
    thumbnail_url TEXT,
    created_at   TIMESTAMPTZ DEFAULT now(),
    view_count   BIGINT DEFAULT 0
);

CREATE TABLE video_resolutions (
    video_id     BIGINT REFERENCES videos(id),
    resolution   TEXT,           -- '360p', '720p', '1080p'
    bitrate_kbps INT,
    storage_path TEXT,           -- S3 path for this resolution
    PRIMARY KEY (video_id, resolution)
);
```

### View Count Design

Same problem as URL shortener click counts: 500M views/day = ~5,800 increments/sec per popular video.

Solution: Redis counter + periodic flush, or approximate counting with HyperLogLog for unique viewers.

---

## Video Recommendations

Not a pure streaming problem but always asked:
- **Collaborative filtering:** "Users who watched A also watched B"
- **Content-based filtering:** Video metadata similarity
- **Two-tower neural network (YouTube's approach):** Candidate retrieval (billions → hundreds) + Ranking (hundreds → 10 shown)
- **Real-time vs batch:** Candidate generation runs nightly (batch ML); ranking runs real-time (fast model inference)

---

## Authentication and DRM

**Free content:** Public CDN URLs. No auth required for segments.

**Premium content (Netflix):**
- **DRM (Digital Rights Management):** Segments are encrypted at rest and in transit
- **Widevine (Google), FairPlay (Apple), PlayReady (Microsoft)** — DRM systems
- **License server:** Player requests a license (decryption key) for each video. License expires after 24h or when subscription lapses.

**CDN signed URLs:** Even for "public" content, CDN signed URLs prevent direct hotlinking (stealing bandwidth):
```
https://cdn.example.com/video/v123/720p/seg_001.ts?
  Signature=HMAC-SHA256&Expires=1713178800&KeyId=k1
```

---

## Architecture Diagram

```
                      ┌───────────────────────────────────────────────────┐
Upload                │                                                   │
  ↓ raw file          │              Upload & Processing Pipeline          │
  ↓ multipart         │                                                   │
┌──────────────┐      │ ┌──────────────┐   Kafka   ┌───────────────────┐ │
│ Upload API   │──────┼→│ Raw Storage  │──────────→│ Transcoding Farm  │ │
│ (FastAPI/S3) │      │ │ (S3)         │           │ (FFmpeg workers)  │ │
└──────────────┘      │ └──────────────┘           └────────┬──────────┘ │
                      │                                      │            │
                      └──────────────────────────────────────┼────────────┘
                                                             │ segments + manifests
                                                             ▼
                                                     ┌───────────────┐
                                                     │ CDN Origin    │
                                                     │ (S3 + CF/Akamai)│
                                                     └───────┬───────┘
                                                             │ cache pull
                                                             ▼
Playback                                             ┌───────────────┐
  Client ──GET manifest→ CDN Edge ─────────────────→│ CDN Edge PoP  │
  Client ──GET segment→  CDN Edge                   │ (globally     │
                                                     │  distributed) │
                                                     └───────────────┘
```
