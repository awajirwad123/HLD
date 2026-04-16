# Blob / Object Storage — Interview Simulator

---

## Scenario 1: Design a Video Upload and Streaming Platform (YouTube-Like)

**Interviewer prompt:**

> "Design the storage and delivery layer for a video platform. Users upload videos (up to 10 GB), videos are transcoded into multiple resolutions, and served to millions of viewers globally. Focus on the object storage architecture."

---

### Strong Answer

**Requirements clarification:**
- Write throughput: say 100,000 uploads/day = ~1.2 uploads/sec peak
- Object size: up to 10 GB per raw video; transcoded outputs smaller
- Global audience: CDN delivery is critical
- Access pattern: write-once, read-many (millions of plays per popular video)

**Upload flow (raw video):**

Never proxy bytes through the app server. Use pre-signed POST:

```
1. Client → App Server: POST /videos/upload-url {filename, size, content_type}
2. App Server → S3: generate_presigned_post(
       Key=f"raw/{video_id}/original.mp4",
       Conditions=[content-length-range 1..10GB, Content-Type video/mp4]
   )
3. App Server → Client: {upload_url, fields, video_id}
4. Client → S3: POST directly (server never sees bytes)
   - For files > 100MB (most videos): multipart upload
   - S3 SDK handles multipart automatically with upload_file()
5. S3 fires event notification → SQS queue: "raw/video_id/original.mp4 uploaded"
6. Client → App Server: POST /videos/{video_id}/confirm
7. App Server → DB: UPDATE videos SET status='processing'
```

**Multipart upload resilience:**
- SDK splits into 100 MB parts, uploads 4 concurrently
- Failed parts retry individually — 10 GB upload survives intermittent mobile network
- If upload is abandoned, `AbortIncompleteMultipartUpload` lifecycle rule (7 days) prevents ghost storage charges

**Transcoding pipeline:**

```
SQS message ──► Transcoding Worker (EC2 / Lambda)
                  ├── Fetch raw/video_id/original.mp4 from S3
                  ├── Transcode to 360p, 720p, 1080p (using FFmpeg)
                  └── Upload outputs to:
                        videos/{video_id}/360p.mp4
                        videos/{video_id}/720p.mp4
                        videos/{video_id}/1080p.mp4
                        videos/{video_id}/thumbnail.jpg
```

Use HLS (HTTP Live Streaming) for adaptive bitrate: output `.m3u8` playlist + `.ts` segments instead of monolithic MP4. Enables mid-stream quality switching.

**Storage layout:**
```
S3 bucket: my-video-platform
├── raw/{video_id}/original.mp4      ← transcoded from, eventually archived
├── videos/{video_id}/360p/          ← HLS segments
│       ├── playlist.m3u8
│       └── segment_000.ts, segment_001.ts ...
├── videos/{video_id}/720p/
├── videos/{video_id}/1080p/
└── thumbnails/{video_id}/thumb.jpg
```

**CDN delivery:**
- CloudFront distribution with S3 `videos/` as origin
- Long TTL on video segments: `Cache-Control: max-age=31536000, immutable` (content-addressed segments never change)
- Short TTL on `playlist.m3u8`: `max-age=60` (changes if segments are added/removed for live)
- Pre-warm edge caches for viral videos by prefetching segments via CloudFront

**Cost optimization:**
- Raw originals: lifecycle rule → Glacier after 30 days (most re-transcodes never happen)
- Old resolution outputs: expire 360p after 1 year if 4K becomes standard
- Intelligent-Tiering for long-tail videos with unpredictable access patterns

**Access control:**
- Public videos: no auth on CloudFront, public S3 policy on `videos/` prefix
- Private/paid videos: CloudFront signed cookies (token valid for session) — more efficient than per-object pre-signed URLs for multi-segment HLS streams

---

## Scenario 2: Design a Distributed Backup System (Restic/Dropbox-Like)

**Interviewer prompt:**

> "Design a backup system that: (1) backs up files from millions of client machines, (2) uses deduplication to avoid re-uploading unchanged data, (3) allows restoring a specific file or a point-in-time snapshot. Use object storage as the backend."

---

### Strong Answer

**Core insight: content-addressed chunked storage**

A client's files are chunked with CDC (Rabin-fingerprint-based), each chunk stored by its SHA256 hash. Two key properties:
- Same chunk on two different machines → stored once
- Client changes one file → only modified chunks re-uploaded

**Storage layout in S3:**
```
S3 bucket: backup-store
├── objects/{hash[:2]}/{hash[2:]}     ← raw encrypted chunks (content-addressed)
├── manifests/{file_hash}.json        ← chunk list for a specific file version
└── snapshots/{client_id}/{timestamp}.json  ← list of {path → manifest_hash}
```

**Backup flow (client-side):**
```python
def backup(client_id, local_path):
    snapshot = {}

    for file_path in walk(local_path):
        chunks = cdc_chunk(file_path)           # Variable-size CDC chunking
        chunk_hashes = []

        for chunk in chunks:
            h = sha256(chunk)
            if not s3.exists(f"objects/{h[:2]}/{h[2:]}"):
                encrypted = encrypt(chunk, client_key)   # Encrypt before upload
                s3.put(f"objects/{h[:2]}/{h[2:]}", encrypted)
            chunk_hashes.append(h)

        manifest = {"path": file_path, "chunks": chunk_hashes, "size": file_size}
        manifest_hash = sha256(manifest_bytes)
        s3.put(f"manifests/{manifest_hash}.json", manifest_bytes)
        snapshot[file_path] = manifest_hash

    snapshot_hash = sha256(json.dumps(snapshot))
    s3.put(f"snapshots/{client_id}/{timestamp}.json", json.dumps(snapshot))
    return snapshot_hash
```

**Deduplication explanation for interviewer:**
- If a 10 GB file is backed up and only 100 bytes change, CDC detects ~2 changed chunks → uploads ~8 MB, not 10 GB
- If the same file exists on two different client machines → all chunks shared → second client uploads zero new data

**Restore flow:**
```
SELECT snapshot WHERE client_id=X AND timestamp <= target_date ORDER BY timestamp DESC LIMIT 1
→ Load snapshot JSON: { path → manifest_hash }
→ For each file: load manifest → get chunk hashes → download + decrypt chunks → reconstruct file
```

**Encryption (critical for backup systems):**
- Chunks encrypted client-side before upload (server never sees plaintext)
- Key management: client holds the key; loss of key = data unrecoverable
- S3 bucket has SSE-S3 as additional layer (defense in depth, not primary)

**Garbage collection:**
When old snapshots are deleted, their manifest/chunk references must be freed. The GC process:
1. Walk all active snapshots → collect all referenced chunk hashes
2. List all chunks in S3 → find hashes with no reference
3. Delete unreferenced chunks

**Multipart upload for large chunks:**
Chunk size config: average 4 MB, max 16 MB. No multipart needed for individual chunks (< 100 MB threshold). But the backup initiator uses multipart for very large individual files that can't be pre-chunked in memory.

---

## Scenario 3: Design a HIPAA-Compliant Medical Document Storage

**Interviewer prompt:**

> "A healthcare company stores patient medical records (PDFs, DICOM imaging files) in S3. Patients can request deletion under GDPR/HIPAA. Documents may be referenced by multiple patients in shared appointments. Requirements: audit every access, enforce deletion properly, handle files up to 2 GB."

---

### Strong Answer

**Core challenges:**
1. HIPAA: PHI (Protected Health Information) must be encrypted at rest and in transit, access logged
2. GDPR: right to erasure — deletion must be real, within 30 days of request
3. Shared documents: one file may be referenced by multiple patients
4. Large files: robust upload

**Data model:**
```
DB: documents table
  { doc_id, s3_key, content_type, size_bytes, sha256, created_at, deleted_at }

DB: document_access table (reference counting + audit)
  { doc_id, patient_id, granted_at, revoked_at, relationship }

S3: medical-records-bucket / {doc_id}/{filename}
```

**Encryption:**
- S3 SSE-KMS: each object encrypted with a KMS-managed key
- For maximum control: client-side encryption with per-patient KMS keys — deleting the KMS key cryptographically erases all data encrypted with it ("crypto shredding")
- TLS enforced via S3 bucket policy: `"Condition": {"Bool": {"aws:SecureTransport": "false"}}` → deny

**Upload (2 GB files):**
- Use multipart upload exclusively for all files (never `put_object` for large medical files)
- Store `UploadId` and completed parts in DB for resumability
- On completion, verify SHA256 client-computed vs S3 `head_object` response checksum
- `AbortIncompleteMultipartUpload` lifecycle rule: 7 days

**Access control and audit:**
- No public access; bucket policy denies all public access
- Pre-signed GET URLs (15-minute TTL) generated only after authorization check
- Every URL generation logged: `{doc_id, patient_id, actor, ip_address, timestamp}` (S3 server access logs + CloudTrail + application audit log)
- S3 Object Lock (WORM) for immutable records required by HIPAA retention rules (e.g., 6–10 years)

**GDPR deletion with shared documents:**
```python
def delete_patient_document_access(patient_id, doc_id):
    with db.transaction():
        db.execute("UPDATE document_access SET revoked_at = now()
                    WHERE patient_id = $1 AND doc_id = $2", patient_id, doc_id)

        active_refs = db.count("SELECT COUNT(*) FROM document_access
                                WHERE doc_id = $1 AND revoked_at IS NULL", doc_id)

        if active_refs == 0:
            # Last reference removed — physically delete from S3
            s3.delete_object(Bucket=BUCKET, Key=get_key(doc_id))
            db.execute("UPDATE documents SET deleted_at = now() WHERE doc_id = $1", doc_id)
            log_audit("PHYSICAL_DELETE", doc_id, triggered_by=patient_id)
        else:
            log_audit("ACCESS_REVOKED", doc_id, remaining_refs=active_refs)
```

For HIPAA audit trail: the deletion log itself is retained for 6 years in a separate locked S3 bucket (Object Lock compliance mode).

**What interviewers look for:**
- Crypto-shredding as a GDPR deletion technique
- Reference counting for shared documents
- S3 Object Lock for compliance (WORM — Write Once Read Many)
- Audit logging at every access point, not just on delete
- Multipart upload for large files with resumability
