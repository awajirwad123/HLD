# Video Streaming — Notes & Reference

## HLS vs MPEG-DASH

| | HLS (HTTP Live Streaming) | MPEG-DASH |
|---|---|---|
| Origin | Apple | Industry standard |
| Segment format | `.ts` (MPEG-2 Transport Stream) or `.fmp4` | `.m4s` (fragmented MP4) |
| Manifest format | `.m3u8` (plain text) | `.mpd` (XML) |
| DRM | FairPlay | Widevine, PlayReady, ClearKey |
| Latency (live) | 6–30s default | 2–6s (CMAF low-latency) |
| Browser support | Native on Safari/iOS; HLS.js for others | Shaka Player / Dash.js |
| Netflix/YouTube | Both support DASH | Both support DASH |

**Production:** Both HLS and DASH are generated from the same transcoded segments. CMAF (Common Media Application Format) uses the same segments for both, reducing storage by 50%.

---

## Adaptive Bitrate (ABR) Algorithm Overview

```
Buffer Level:      [                    ] ← monitor this
Segment Download:  [  ] per segment
Available Quality: [360p] [720p] [1080p] [1440p] [4K]

ABR decision (simplified):
  bandwidth = last_segment_bits / last_segment_download_time
  
  if bandwidth > bitrate_1440p: switch to 1440p
  elif bandwidth > bitrate_1080p: switch to 1080p  
  elif bandwidth > bitrate_720p:  switch to 720p
  else:                           drop to 360p

  Also check buffer level:
    buffer < 10s: prefer quality drop over playback stall
    buffer > 30s: allow quality increase
```

**Netflix BOLA algorithm:** Buffer Occupancy based Lyapunov Algorithm — optimizes for maximum quality given buffer constraints without stalling. More sophisticated than pure bandwidth-based ABR.

---

## Video Transcoding Pipeline

```
Raw video upload (H.264 or HEVC source)
      ↓
Scene detection → keyframe placement every 4s (for segment alignment)
      ↓
Parallel transcoding workers (one per resolution):
  FFmpeg: -c:v libx264 -b:v 2800k -vf scale=1280:720
      ↓
HLS segmentation: -hls_time 4 → seg_0001.ts, seg_0002.ts, ...
      ↓
Per-resolution playlist (index.m3u8) + master manifest
      ↓
Upload all .ts files + .m3u8 files to S3 (CDN origin)
      ↓
Database: status = 'ready', manifest_url = CDN URL
```

**FFmpeg key flags:**
```bash
-hls_time 4              # 4-second segments
-hls_playlist_type vod   # VOD (not live) — full playlist
-hls_segment_type mpegts # Output format (ts files)
-hls_segment_filename "seg_%04d.ts"
-preset fast             # Speed vs compression tradeoff
-crf 23                  # Constant rate factor (quality-based encoding)
```

---

## CDN Architecture Summary

```
Origin (S3) → CDN Pull Zone → CDN Edge PoPs (200+ globally)
                                    ↑
                              User requests routed by GeoDNS 
                              to nearest PoP
```

**Cache-Control for video:**
- Segments (`.ts`, `.m4s`): `max-age=31536000` (1 year — immutable)
- Playlist (`.m3u8`): `max-age=30` (refreshed frequently for ABR)
- Thumbnail: `max-age=86400` (1 day)

**Long-tail problem:** 80% of content is cold (never or rarely requested). CDN never pre-warms these — they're fetched from origin on first request. CDN TTL keeps them cached as long as there's demand; LRU eviction removes them if space is needed.

---

## Storage Tiers

| Content Age | Storage | Access Pattern | Cost |
|---|---|---|---|
| New / popular (< 3 months) | CDN + hot S3 | Frequent | High |
| Catalog (3 months – 2 years) | S3 Standard-IA | Occasional | Medium |
| Archive (> 2 years) | S3 Glacier | Rare | Very low |

Netflix keeps the complete library on hot storage because ANY content can be requested at any time by any subscriber globally.

---

## View Count at Scale

At 500M views/day = ~5.8K views/sec globally on popular content:

| Approach | Write load | Accuracy |
|---|---|---|
| `UPDATE view_count = view_count + 1` | 5.8K DB writes/sec | Exact |
| Redis INCR + batch flush | ~0 DB load | ~1 second delay |
| Kafka events → ClickHouse | 0 DB writes | Real-time analytics |

**Production:** Redis `INCR` per video, batch flush to DB every 60 seconds. ClickHouse for analytics (who watched, when, from where, completion rate).

---

## Key Numbers

| Metric | Value |
|---|---|
| YouTube: hours uploaded per minute | ~500 hours |
| Transcoding ratio | 1 min upload → ~20 min processing |
| 1 hour 1080p compressed | ~3 GB |
| Netflix peak: share of internet traffic | ~40% |
| CDN hit ratio for popular content | 99%+ |
| HLS segment size (4s at 720p) | ~1.4 MB |
| Typical ABR buffer target | 30 seconds ahead |
| Netflix playback start target | < 1 second |
