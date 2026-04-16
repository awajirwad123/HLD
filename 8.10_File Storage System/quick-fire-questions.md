# File Storage System — Quick-Fire Questions

**Q1: Why split files into chunks for storage?**

Three reasons: (1) Resumable uploads — if an upload fails at chunk 7, only re-upload chunk 7, not the whole file. (2) Deduplication — identical chunks (common in document templates, repeated sections) are stored once regardless of how many files reference them. (3) Delta sync — when a file is edited, only changed chunks need to be re-uploaded; unchanged chunks already exist in the block store.

---

**Q2: What is content-addressable storage?**

Each chunk's storage key is its SHA-256 hash. The content determines the address, not the filename or path. Two identical chunks get the same key → stored once → automatic deduplication. Chunks are also immutable — you never update a chunk; you create a new one. References to old chunks persist in older version records.

---

**Q3: How does delta sync work?**

The client computes SHA-256 hashes for all chunks of the new file version. It sends these hashes to the server before uploading anything. The server checks which hashes are already stored and responds with only the missing ones. The client uploads only the missing chunks — often 1 or 2 chunks out of 25 for a minor edit, saving 90%+ bandwidth.

---

**Q4: How do you handle a file upload that is interrupted mid-way?**

Each chunk is independently uploadable. The client tracks which chunks were successfully acknowledged. On resume, it re-sends only unattempted or failed chunks. The `/upload/finalize` call only succeeds after all chunks are present. If the client disconnects before finalize, the partial chunks remain in the block store as orphans (cleanup job runs periodically to remove unreferenced chunks).

---

**Q5: How do you implement file versioning?**

Each `file_version` record stores an ordered list of chunk hashes. When a new version is created, a new `file_version` row is inserted referencing the new/unchanged chunk set. The `files` table may also have a `current_version_id` pointer. To restore: update `current_version_id` to an older version. Storage is efficient because unchanged chunks are shared across versions.

---

**Q6: How do you detect a conflict between two devices editing the same file offline?**

Track a `base_hash` — the hash of the file at the last confirmed sync point. When a device tries to upload, compare: if `local_hash ≠ base_hash` AND `remote_hash ≠ base_hash` AND `local_hash ≠ remote_hash`: conflict. Resolution: rename and keep both copies, similar to Dropbox's "Alice's conflicted copy" approach.

---

**Q7: How does cross-user deduplication work, and is it a privacy risk?**

If user A and user B upload identical files, the same chunk hashes are produced, and the block store returns "chunk already exists" — no data is uploaded twice. Storage efficiency is gained. Privacy consideration: using deduplication, a malicious user could infer that someone else stored the same file (by measuring upload time: instant upload = file exists). This is called a "Convergent Encryption" or "Proof-of-Work" attack. Production systems like Dropbox apply cross-user dedup but audit for this. Mitigation: client-side encryption with user-specific keys (but then cross-user dedup is impossible).

---

**Q8: How do you generate shareable links?**

Create a `shares` record with a cryptographically random `share_token` (UUID v4 or 128-bit random). URL: `https://files.example.com/s/{share_token}`. On access: `SELECT file_id FROM shares WHERE share_token = ? AND (expires_at IS NULL OR expires_at > NOW())`. Generate a signed S3 URL for the actual download. The share_token itself grants no other access — it's an opaque key.

---

**Q9: What's the difference between metadata storage and block storage?**

Metadata (PostgreSQL): file names, folder hierarchy, sharing permissions, version history, access control — structured, queried, indexed. Block storage (S3): raw binary chunks keyed by hash — unstructured, append-only, accessed by exact key, never queried. This separation allows each to be optimized independently: DB for complex queries, object storage for massive scale at low cost.

---

**Q10: How do you handle large file uploads (e.g., 5GB video files)?**

Chunking at 4MB → 1,250 chunks for a 5GB file. Upload chunks in parallel (5–10 concurrent). Use S3 Multipart Upload (minimum 5MB parts). After all parts are uploaded, `CompleteMultipartUpload` assembles the object server-side in S3. Show a progress bar on the client (X of 1,250 chunks uploaded). If upload fails: resume from last successful chunk using the same S3 upload ID.

---

**Q11: How do you clean up orphaned chunks?**

After deleting a file version, ref_count is decremented for each referenced chunk. When `ref_count = 0`, the chunk is eligible for deletion. A background garbage collection job (daily or hourly) queries `SELECT id, storage_key FROM chunks WHERE ref_count = 0` and deletes from S3. To be safe: add a delay (e.g., 24h) before actually deleting, to handle any in-progress uploads that haven't committed yet.

---

**Q12: How do you implement a "trash / recently deleted" feature?**

Soft delete: `files.deleted_at = NOW()`. Files in trash are not shown in normal listings but still exist. Background job: after 30 days, hard-delete by: decrementing chunk ref_counts, deleting file_versions rows, deleting file rows. On restore: `UPDATE files SET deleted_at = NULL`. This pattern allows fast "delete" (just set a timestamp) and safe recovery within the retention window.

---

**Q13: How would you implement full-text search across stored documents?**

Don't query S3 directly — extract text at upload time. Pipeline: upload file → extract text (Apache Tika or similar for PDF/docx) → index text in Elasticsearch keyed by file_id. Search: query Elasticsearch → get file_id results → join with metadata DB for access control. Never return files the user can't access, even if the text search matches. This is a post-processing pipeline, not part of the core upload flow.
