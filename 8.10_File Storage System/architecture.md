# File Storage System — Architecture

## Problem Statement

Design a file storage system like Dropbox/Google Drive that:
- Stores files of any size (up to 5GB per file)
- Syncs files across multiple devices
- Supports file versioning and delta sync
- Handles deduplication to save storage
- Supports sharing files/folders with other users

---

## High-Level Architecture

```
Client (desktop/mobile)
    │
    ├── Metadata API (FastAPI/gRPC)    ← file metadata, folder tree, sharing
    ├── Upload API                     ← chunked upload to block storage
    │       ↓
    │   Block Storage (S3/GCS)         ← raw file chunks stored by content hash
    │
    ├── Sync Engine (long-polling/WS)  ← notify client of remote changes
    └── CDN (download acceleration)    ← for frequently accessed files

Metadata DB (PostgreSQL)              ← files, folders, versions, shares
Block DB (Redis / DynamoDB)           ← chunk hash → S3 location mapping
```

---

## Core Data Model

```sql
-- User's virtual file system
CREATE TABLE files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id        BIGINT NOT NULL,
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES files(id),     -- NULL = root folder
    is_folder       BOOLEAN NOT NULL DEFAULT FALSE,
    size_bytes      BIGINT,
    mime_type       TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ                    -- soft delete (trash)
);
CREATE INDEX idx_files_parent ON files (owner_id, parent_id, deleted_at NULLS FIRST);

-- Each version of a file
CREATE TABLE file_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID REFERENCES files(id),
    version_number  INT NOT NULL,
    size_bytes      BIGINT NOT NULL,
    chunk_ids       UUID[] NOT NULL,   -- ordered list of chunk UUIDs
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    created_by      BIGINT,
    UNIQUE (file_id, version_number)
);

-- Content-addressable chunks (dedup happens here)
CREATE TABLE chunks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_hash    TEXT NOT NULL UNIQUE,   -- SHA-256 of chunk bytes
    size_bytes      INT NOT NULL,
    storage_key     TEXT NOT NULL,          -- S3 object key
    ref_count       INT DEFAULT 0           -- how many versions use this chunk
);

-- Sharing permissions
CREATE TABLE shares (
    id              UUID PRIMARY KEY,
    file_id         UUID REFERENCES files(id),
    shared_with     BIGINT,               -- NULL = public link
    permission      TEXT CHECK (permission IN ('view', 'edit', 'admin')),
    share_token     TEXT UNIQUE,          -- for shareable link
    expires_at      TIMESTAMPTZ
);
```

---

## Chunking and Upload Flow

### Why chunk files?

- Resume interrupted uploads (re-upload only failed chunks, not the whole file)
- Deduplication: identical chunks shared across files/users
- Parallel upload (multiple chunks simultaneously)
- Delta sync: on file update, only re-upload changed chunks

### Chunk size

Typical: 4MB chunks. Trade-offs:
- Smaller chunks = better dedup + delta sync, more metadata overhead
- Larger chunks = fewer API calls, less dedup opportunity

### Client-side chunking

```
File (12MB) → split into 3 chunks of 4MB each
 chunk 1: bytes [0,     4MB)  → SHA-256 = abc123
 chunk 2: bytes [4MB,   8MB)  → SHA-256 = def456
 chunk 3: bytes [8MB,  12MB)  → SHA-256 = ghi789
```

### Upload Protocol

```
Step 1: Client calls POST /upload/init
  Request: {file_id, version, chunk_hashes: ["abc123", "def456", "ghi789"]}
  Server: checks which chunks already exist in block store
  Response: {chunks_needed: ["def456"]}  ← "abc123" and "ghi789" already stored!

Step 2: Client uploads only missing chunks
  PUT /upload/chunk/{hash}  ← body = raw chunk bytes

Step 3: Client calls POST /upload/finalize
  Request: {file_id, chunk_hashes: [...], metadata: {name, size, mime_type}}
  Server: creates file_version record, updates file.updated_at
  Response: {version_id, download_url}
```

This is **content-addressed storage**: chunks are keyed by their SHA-256 hash. If two users upload the same file, only one copy is stored.

---

## Delta Sync

When a file changes, only modified chunks need re-uploading:

```
v1: [chunk_A, chunk_B, chunk_C]   ← original file
    user edits the middle section
v2: [chunk_A, chunk_B', chunk_C]  ← only chunk_B changed

Upload v2:
  Client sends hashes: ["sha(A)", "sha(B')", "sha(C)"]
  Server sees sha(A) and sha(C) already exist → only sha(B') needed
  Client uploads only 1 of 3 chunks
```

For large documents (code, text), clients can use smaller chunks (e.g., 1MB or variable-size via content-defined chunking) to maximize dedup across versions.

---

## Sync Engine (Multi-Device)

When File X is updated on Device 1, Device 2 needs to know.

```
Device 1 uploads new version
    → API server writes to metadata DB
    → Publishes "file_updated" event to message queue (Kafka / Redis Pub/Sub)
         event: {user_id, file_id, version_id, device_id: "device-1"}
    ← Sync server notifies all other active devices for user_id
```

Device 2 holds a long-polling connection (or WebSocket):
```
GET /sync/events?since={last_event_id}  ← blocks until new events arrive
```

On receiving `file_updated`:
1. Download changed chunks (only new/modified chunks from delta)
2. Reconstruct file locally

---

## Deduplication

**Block-level dedup:** Two files with the same chunk (same SHA-256) share one stored copy. `ref_count` on each chunk tracks how many versions reference it. When `ref_count` drops to 0, the chunk can be garbage collected from S3.

**File-level dedup:** If a user uploads a file identical to one already in the system, `chunk_hashes` match entirely → server response: "all chunks already present" → no upload needed. File appears in user's drive instantly (zero upload time).

**Cross-user dedup:** Multiple users uploading the same file → stored once. Significant storage saving for common files (installer packages, popular PDFs, etc.).

---

## File Versioning

```
files.id = f001
file_versions:
  version 1: [chunk_A, chunk_B, chunk_C]  (original)
  version 2: [chunk_A, chunk_B', chunk_C] (B changed)
  version 3: [chunk_A', chunk_B', chunk_C'] (major edit)
```

- Keep last 30 versions by default (configurable per plan)
- Version pruning: background job deletes old versions, decrements `ref_count` on chunks
- Restore: update `files.version_id` to point to older version record

---

## Storage Tiers

| Tier | Access pattern | Latency | Cost | Storage backend |
|---|---|---|---|---|
| Hot | Active files (< 90 days) | < 100ms | High | S3 Standard |
| Warm | Older versions (90d–1yr) | < 5s | Medium | S3-IA |
| Cold | Archived / deleted | Minutes | Low | Glacier |

Background lifecycle job: move chunk storage class based on last access time.

---

## Key Numbers

| Metric | Value |
|---|---|
| Average file size | 1MB |
| Chunk size | 4MB |
| Active users per day | 50M |
| Files stored per user | ~2,000 average |
| Total files | 100B |
| Total storage | ~100 PB |
| Daily file uploads | 1B |
| Dedup savings (typical) | 30–60% of raw upload volume |
| S3 PUT cost (us-east-1) | $0.005 per 1,000 requests |
| S3 GET cost (us-east-1) | $0.0004 per 1,000 requests |
| Metadata DB size (PostgreSQL) | ~500 bytes per file row |
