# File Storage System — Notes & Reference

## Chunking Strategies

| Strategy | Chunk size | Best for |
|---|---|---|
| Fixed-size | 4MB | Simple implementation, good parallelism |
| Variable (CDC) | 1KB–10MB | Maximum dedup (handles insertions/deletions in middle of file) |
| Line-based | Variable | Text files, code repos (Git uses this) |

**Content-Defined Chunking (CDC):** Uses a rolling hash (Rabin fingerprint) to find natural chunk boundaries. Inserting text at the beginning of a file doesn't change all subsequent chunk boundaries (as it would with fixed-size). Better dedup ratio for frequently edited documents. Used by rsync, Bup, Restic.

---

## Storage Architecture: Separation of Metadata and Blocks

```
Metadata Service (PostgreSQL)          Block Store (S3)
─────────────────────────────          ───────────────
files, folders, versions, shares       Raw chunk bytes, keyed by SHA-256
  → Fast queries (folder listings)       → Highly durable, cheap at scale
  → Access control                       → Content-addressable (immutable)
  → Change history                       → Chunks never updated, only added
  → User-facing operations               → Lifecycle policies (hot→warm→cold)
```

---

## Chunked Upload Protocol

```
Client side:
  1. Split file into 4MB chunks
  2. SHA-256 each chunk

Protocol:
  POST /upload/init
    → body: {file_id, chunk_hashes: [...]}
    ← response: {missing_hashes: [...]}  # Server tells client which to upload

  PUT /upload/chunk/{hash}             # Repeat for each missing hash
    → body: raw chunk bytes
    ← 200 OK

  POST /upload/finalize
    → body: {file_id, chunk_hashes: [...], metadata: {...}}
    ← response: {version_id}

Resumable upload: if connection drops at chunk #4, resume from chunk #4
Parallel upload: upload chunks 1,2,3 simultaneously (up to N concurrent)
```

---

## Content-Addressable Storage

Every chunk is stored exactly once, keyed by its SHA-256 hash:

```
chunk bytes → SHA-256 → "3d4f2..."
storage key in S3: blocks/3d/4f/3d4f2...

Two files sharing content:
  user_a/report.pdf version 1: [chunk:3d4f2, chunk:7abc1, chunk:9def3]
  user_b/same_report.pdf:      [chunk:3d4f2, chunk:7abc1, chunk:9def3]
  → S3 only stores ONE copy of each chunk
```

This is why chunk deletion requires reference counting:
- `chunks.ref_count` incremented when a version references the chunk
- `ref_count` decremented when a version is deleted
- When `ref_count = 0`: chunk can be GC'd from S3

---

## Delta Sync Explained

```
Original file (v1):     [A, B, C, D, E]   ← 5 chunks
User edits middle:
Modified file (v2):     [A, B', C, D, E]  ← only chunk B changed to B'

Client computes hashes for v2: [SHA(A), SHA(B'), SHA(C), SHA(D), SHA(E)]
POST /upload/init with these hashes
Server responds: missing = [SHA(B')]   ← A, C, D, E already stored

Client uploads only 1 chunk instead of 5 → 80% bandwidth saving
```

---

## Multi-Device Sync via Event Queue

```
Device 1 saves file
    ↓
Metadata DB updated: files.version_id = v2, files.updated_at = now
    ↓
Publish event: {type: "file_updated", user_id: 42, file_id: "f001", version_id: "v2"}
    ↓ Kafka / Redis Pub/Sub
Sync Server (per-user channel subscription)
    ↓
Long-polling response sent to Device 2, Device 3 (both connected)
    ↓
Device 2 & 3 request delta: "what changed in file f001 since v1?"
    ↓
Download only changed chunks
```

Long-polling endpoint:
```http
GET /sync/changes?since={last_cursor}&device_id={device_id}
  ← Blocks until new events; returns after max 30s (then client reconnects)
```

---

## Conflict Resolution

```
Conflict happens when two devices both edit the same file while offline.

Device 1 was on a plane:                Device 2 at home:
  file v1 → edits → local v2              file v1 → edits → syncs as v2

When Device 1 reconnects:
  local_hash != remote_hash
  AND both differ from base_hash (v1)
  → CONFLICT detected

Resolution options:
  1. Last-write-wins     (simplest, user may lose work)
  2. Rename and keep both  (Dropbox: "file (conflicted copy 2024-01-15).pdf")
  3. Manual merge UI     (Google Docs: shows tracked changes)
  4. Three-way merge     (Git-style: merge v1+local+remote)
```

Dropbox's approach: keep both — rename the remote version adding "(username's conflicted copy date)" suffix. User resolves manually.

---

## Sharing and Permissions

```sql
-- View: select f.* FROM files f WHERE f.owner_id = ? OR EXISTS sharing grant
-- Edit: same check with permission = 'edit'

-- Public link: share_token is a random UUID in the URL
-- https://drive.example.com/s/{share_token}
-- Resolve: SELECT file_id FROM shares WHERE share_token = ? AND expires_at > NOW()
```

Permission inheritance: a folder share grants access to all files inside it. Check parents recursively (or use a Closure Table for the folder tree).

---

## Virus Scanning

For user-uploaded content: never serve a file until it's been scanned.

```
Upload chunk/finalize → Write chunks to S3 "quarantine/" prefix
                       → Publish SQS event
                       → Antivirus lambda reads from SQS, scans file
                       → On clean: move to "files/" prefix, mark file as "available"
                       → On infected: delete, notify user, log incident
```

Files in quarantine are not accessible via download URLs.

---

## Key Numbers

| Metric | Value |
|---|---|
| Chunk size | 4MB (configurable) |
| SHA-256 length | 64 hex chars |
| S3 max object size | 5TB |
| S3 multipart upload min part size | 5MB |
| Typical dedup ratio | 30–60% (for user files) |
| Metadata DB per file | ~500 bytes |
| Per chunk metadata | ~200 bytes (hash + storage_key + ref_count) |
| Max parallel chunk uploads | 5–10 (bandwidth dependent) |
| Sync long-poll timeout | 30 seconds |
| Conflict detection window | Defined by base_hash at last sync |
| File download URL TTL (signed S3) | 1–24 hours |
| Version retention (default) | 30 versions |
