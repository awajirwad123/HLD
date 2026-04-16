# Video Streaming — Hands-On Exercises

## Exercise 1: Simulate HLS Manifest Generation

```python
"""
Simulate the post-transcoding step: generate HLS manifests
for a multi-resolution video.
"""

from pathlib import Path
from dataclasses import dataclass


@dataclass
class VideoResolution:
    name: str           # "720p"
    width: int
    height: int
    bitrate_kbps: int   # Target bitrate
    segment_duration: int = 4  # seconds


RESOLUTIONS = [
    VideoResolution("360p",  640,  360,   800),
    VideoResolution("720p",  1280, 720,  2800),
    VideoResolution("1080p", 1920, 1080, 5000),
    VideoResolution("1440p", 2560, 1440, 8000),
]


def generate_segment_playlist(
    video_id: str,
    resolution: VideoResolution,
    total_duration_sec: int,
    cdn_base_url: str,
) -> str:
    """Generate a per-resolution HLS playlist (.m3u8)."""
    import math
    num_segments = math.ceil(total_duration_sec / resolution.segment_duration)

    lines = [
        "#EXTM3U",
        "#EXT-X-VERSION:3",
        f"#EXT-X-TARGETDURATION:{resolution.segment_duration}",
        "#EXT-X-PLAYLIST-TYPE:VOD",   # Video on demand (not live)
    ]

    for i in range(1, num_segments + 1):
        # Last segment may be shorter
        seg_duration = min(
            resolution.segment_duration,
            total_duration_sec - (i - 1) * resolution.segment_duration
        )
        seg_url = f"{cdn_base_url}/{video_id}/{resolution.name}/seg_{i:04d}.ts"
        lines.append(f"#EXTINF:{seg_duration:.3f},")
        lines.append(seg_url)

    lines.append("#EXT-X-ENDLIST")
    return "\n".join(lines)


def generate_master_manifest(
    video_id: str,
    available_resolutions: list[VideoResolution],
    cdn_base_url: str,
) -> str:
    """Generate the master HLS manifest that lists all quality tiers."""
    lines = [
        "#EXTM3U",
        "#EXT-X-VERSION:3",
        "",
        "# This file is fetched first by the player.",
        "# Player picks the best quality based on bandwidth.",
        "",
    ]

    for res in sorted(available_resolutions, key=lambda r: r.bitrate_kbps):
        playlist_url = f"{cdn_base_url}/{video_id}/{res.name}/index.m3u8"
        lines.append(
            f"#EXT-X-STREAM-INF:BANDWIDTH={res.bitrate_kbps * 1000},"
            f"RESOLUTION={res.width}x{res.height},"
            f"NAME=\"{res.name}\""
        )
        lines.append(playlist_url)

    return "\n".join(lines)


# ─── Demo ─────────────────────────────────────────────────────────────────

CDN_BASE = "https://cdn.example.com/videos"
VIDEO_ID = "vid-abc123"
DURATION = 127   # seconds (2:07 video)

master = generate_master_manifest(VIDEO_ID, RESOLUTIONS, CDN_BASE)
seg_720p = generate_segment_playlist(VIDEO_ID, RESOLUTIONS[1], DURATION, CDN_BASE)

print("=== Master Manifest ===")
print(master)
print("\n=== 720p Segment Playlist (first 20 lines) ===")
print("\n".join(seg_720p.split("\n")[:20]))

# Segment count
import math
print(f"\nTotal segments (720p): {math.ceil(DURATION / 4)}")
print(f"Total segments × 4 resolutions: {math.ceil(DURATION / 4) * 4}")
```

---

## Exercise 2: Upload API with Multipart and Progress Tracking

```python
"""
Video upload API:
  - Accepts video file (multipart form)
  - Stores to S3 using multipart upload
  - Publishes transcoding job to a queue
  - Tracks upload progress via Redis
"""

import asyncio
import uuid
import hashlib
from pathlib import Path

import asyncpg
import redis.asyncio as redis
import boto3
from botocore.exceptions import ClientError
from fastapi import FastAPI, UploadFile, HTTPException, BackgroundTasks
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI()

DB_DSN    = "postgresql://user:pass@localhost/videos"
REDIS_URL = "redis://localhost:6379"
S3_BUCKET_RAW = "my-raw-videos"
S3_BUCKET_CDN = "my-cdn-videos"
CDN_BASE      = "https://cdn.example.com"

ALLOWED_MIME_TYPES = {
    "video/mp4", "video/quicktime", "video/x-msvideo",
    "video/webm", "video/x-matroska"
}
MAX_VIDEO_SIZE_BYTES = 10 * 1024 * 1024 * 1024  # 10 GB


@app.on_event("startup")
async def startup():
    app.state.db    = await asyncpg.create_pool(DB_DSN)
    app.state.cache = redis.from_url(REDIS_URL, decode_responses=True)
    app.state.s3    = boto3.client("s3")


@app.post("/videos/upload")
async def upload_video(
    file: UploadFile,
    title: str,
    description: str = "",
    background_tasks: BackgroundTasks = None,
    user_id: int = 1,   # In production: from JWT
):
    # ── Validation ────────────────────────────────────────────────────────
    if file.content_type not in ALLOWED_MIME_TYPES:
        raise HTTPException(415, f"Unsupported media type: {file.content_type}")

    video_id = str(uuid.uuid4())

    # ── Create DB record ──────────────────────────────────────────────────
    async with app.state.db.acquire() as conn:
        await conn.execute(
            """INSERT INTO videos (id, user_id, title, description, status)
               VALUES ($1, $2, $3, $4, 'uploading')""",
            video_id, user_id, title, description,
        )

    # ── Stream to S3 in chunks ────────────────────────────────────────────
    s3 = app.state.s3
    rdb: redis.Redis = app.state.cache
    s3_key = f"raw/{video_id}/original{Path(file.filename).suffix}"

    try:
        mpu = s3.create_multipart_upload(Bucket=S3_BUCKET_RAW, Key=s3_key)
        upload_id = mpu["UploadId"]
        parts = []
        part_number = 1
        total_bytes = 0
        sha256 = hashlib.sha256()

        CHUNK_SIZE = 10 * 1024 * 1024  # 10MB chunks

        while True:
            chunk = await file.read(CHUNK_SIZE)
            if not chunk:
                break

            total_bytes += len(chunk)
            if total_bytes > MAX_VIDEO_SIZE_BYTES:
                s3.abort_multipart_upload(Bucket=S3_BUCKET_RAW, Key=s3_key, UploadId=upload_id)
                raise HTTPException(413, "Video exceeds 10GB limit")

            sha256.update(chunk)
            resp = s3.upload_part(
                Bucket=S3_BUCKET_RAW, Key=s3_key,
                PartNumber=part_number, UploadId=upload_id, Body=chunk,
            )
            parts.append({"PartNumber": part_number, "ETag": resp["ETag"]})
            part_number += 1

            # Report progress to Redis
            await rdb.set(f"upload_progress:{video_id}", total_bytes, ex=3600)

        s3.complete_multipart_upload(
            Bucket=S3_BUCKET_RAW, Key=s3_key,
            MultipartUpload={"Parts": parts}, UploadId=upload_id,
        )

    except ClientError as e:
        async with app.state.db.acquire() as conn:
            await conn.execute("UPDATE videos SET status = 'failed' WHERE id = $1", video_id)
        raise HTTPException(500, f"Storage error: {e}")

    # ── Update DB and queue transcoding job ───────────────────────────────
    async with app.state.db.acquire() as conn:
        await conn.execute(
            "UPDATE videos SET status = 'processing', file_size_bytes = $1 WHERE id = $2",
            total_bytes, video_id,
        )

    # Publish transcoding job
    job = {
        "video_id": video_id,
        "s3_key": s3_key,
        "sha256": sha256.hexdigest(),
        "resolutions": ["360p", "720p", "1080p"],
    }
    await rdb.rpush("transcoding_queue", __import__("json").dumps(job))

    return JSONResponse(
        status_code=202,
        content={
            "video_id": video_id,
            "status": "processing",
            "status_url": f"/videos/{video_id}/status",
        },
    )


@app.get("/videos/{video_id}/status")
async def get_video_status(video_id: str):
    async with app.state.db.acquire() as conn:
        row = await conn.fetchrow(
            "SELECT status, manifest_url, view_count FROM videos WHERE id = $1",
            video_id,
        )
    if not row:
        raise HTTPException(404, "Video not found")
    return dict(row)


@app.get("/videos/{video_id}/upload-progress")
async def get_upload_progress(video_id: str):
    rdb: redis.Redis = app.state.cache
    bytes_uploaded = await rdb.get(f"upload_progress:{video_id}")
    return {"bytes_uploaded": int(bytes_uploaded or 0)}
```

---

## Exercise 3: Simulated Transcoding Worker

```python
"""
Transcoding worker that consumes jobs from a queue,
simulates FFmpeg transcoding, generates HLS segments.
"""

import asyncio
import json
import os
import subprocess
from pathlib import Path


RESOLUTIONS = {
    "360p":  {"width": 640,  "height": 360,  "bitrate": "800k"},
    "720p":  {"width": 1280, "height": 720,  "bitrate": "2800k"},
    "1080p": {"width": 1920, "height": 1080, "bitrate": "5000k"},
}

OUTPUT_DIR = Path("/tmp/transcoded")
OUTPUT_DIR.mkdir(exist_ok=True)


def transcode_to_hls(input_path: str, video_id: str, resolution: str) -> Path:
    """
    Use FFmpeg to transcode video to HLS segments.
    In production this runs on GPU-accelerated instances.
    """
    res = RESOLUTIONS[resolution]
    output_dir = OUTPUT_DIR / video_id / resolution
    output_dir.mkdir(parents=True, exist_ok=True)

    cmd = [
        "ffmpeg", "-i", input_path,
        "-vf", f"scale={res['width']}:{res['height']}",
        "-b:v", res["bitrate"],
        "-c:v", "libx264",
        "-c:a", "aac",
        "-b:a", "128k",
        "-hls_time", "4",           # 4-second segments
        "-hls_playlist_type", "vod",
        "-hls_segment_filename", str(output_dir / "seg_%04d.ts"),
        str(output_dir / "index.m3u8"),
        "-y",                        # Overwrite
    ]

    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        raise RuntimeError(f"FFmpeg failed: {result.stderr}")

    return output_dir


async def process_transcoding_job(job: dict):
    video_id = job["video_id"]
    s3_key   = job["s3_key"]
    resolutions = job.get("resolutions", ["720p"])

    print(f"[Worker] Transcoding {video_id}")

    # In real system: download from S3 to temp storage
    # For demo: assume input file exists locally
    input_path = f"/tmp/raw/{video_id}/original.mp4"

    for resolution in resolutions:
        print(f"  Transcoding → {resolution}...")
        try:
            output_dir = transcode_to_hls(input_path, video_id, resolution)
            # Upload segments to S3/CDN origin
            for segment_file in sorted(output_dir.glob("*.ts")):
                # s3.upload_file(str(segment_file), CDN_BUCKET, ...)
                print(f"    Uploaded {segment_file.name}")
            print(f"  ✓ {resolution} complete")
        except Exception as e:
            print(f"  ✗ {resolution} failed: {e}")

    # Generate master manifest and upload
    print(f"  Generating master manifest...")
    # update DB: status = 'ready', manifest_url = CDN URL


async def worker_loop():
    """Poll the transcoding queue and process jobs."""
    import redis.asyncio as redis
    rdb = redis.from_url("redis://localhost:6379", decode_responses=True)

    print("[Worker] Waiting for transcoding jobs...")
    while True:
        # BRPOP: blocking pop — waits up to 5s for a job
        item = await rdb.brpop("transcoding_queue", timeout=5)
        if item:
            _, raw = item
            job = json.loads(raw)
            await process_transcoding_job(job)
```
