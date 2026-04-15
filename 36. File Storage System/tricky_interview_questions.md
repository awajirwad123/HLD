# File Storage System — Tricky Interview Questions

## Q1: "Two users upload the same 1GB file simultaneously. What happens, and how do you ensure it's only stored once without a race condition?"

**What they're testing:** Content-addressed storage, race conditions in block store writes, atomic chunk existence check.

**Scenario:**
- User A chunks the file: [hash1, hash2, ..., hash250]
- User B chunks the same file: [hash1, hash2, ..., hash250]
- Both call `/upload/init` at the same time: server checks block store, both get "missing = ALL 250 hashes"
- Both start uploading → same chunks uploaded twice
- No data loss, but wasted bandwidth

**Race condition analysis:**
- The "check then upload" pattern has a TOCTOU (time of check to time of use) gap.
- Solution: use S3's conditional `PUT` (object-level locking isn't native in S3) OR use a database lock per hash.

**Production solution:**

```python
# Before accepting upload for a chunk:
result = await db.execute("""
    INSERT INTO chunks (content_hash, size_bytes, storage_key, ref_count)
    VALUES ($1, $2, $3, 1)
    ON CONFLICT (content_hash) DO NOTHING
    RETURNING id
""", content_hash, size_bytes, storage_key)

if result:
    # This INSERT won — upload to S3
    await s3.put_object(Key=storage_key, Body=chunk_bytes)
else:
    # Lost the race — chunk already stored (or being stored)
    # Verify the S3 object exists before using it
    await verify_chunk_exists(content_hash)
```

**S3 races:**
- Even if two clients both win a DB INSERT (which can't happen with `ON CONFLICT`), uploading the same bytes to the same S3 key twice is safe — S3 is eventually consistent but the final state is the same data.
- S3 `PUT` is idempotent for the same content at the same key.

**Finalize is safe:** At finalize time, the server creates a `file_version` record referencing existing chunk hashes. If chunk A was uploaded by User A and chunk hash1 is shared with User B's version, both versions independently reference the same chunk. Correct outcome: one S3 object, two version references.

---

## Q2: "A user has 1M files in Dropbox. How do you efficiently list just their root folder contents (~50 files)?"

**What they're testing:** DB query design, indexing, hierarchical data modeling.

**Naive approach:** `SELECT * FROM files WHERE owner_id = 42` → scans all 1M files. Very slow.

**Better approach:** Query by `parent_id`:
```sql
SELECT id, name, is_folder, size_bytes, updated_at
FROM files
WHERE owner_id = 42
  AND parent_id IS NULL
  AND deleted_at IS NULL
ORDER BY is_folder DESC, name ASC
LIMIT 100 OFFSET 0;
```

**Index:** `CREATE INDEX idx_files_parent ON files (owner_id, parent_id, deleted_at NULLS FIRST)` — this index covers exactly this query pattern. Result: ~1ms even with 1M files.

**Pagination:** Use cursor-based pagination keyed on `(name, id)` rather than OFFSET — OFFSET scans all rows before the offset position.

```sql
WHERE owner_id = 42
  AND parent_id IS NULL
  AND deleted_at IS NULL
  AND (name, id) > ($cursor_name, $cursor_id)  -- cursor pagination
ORDER BY name, id
LIMIT 100;
```

**Recursive folder traversal:** "Select all files in a subtree" requires a recursive CTE:
```sql
WITH RECURSIVE folder_tree AS (
    SELECT id FROM files WHERE id = $folder_id
    UNION ALL
    SELECT f.id FROM files f
    INNER JOIN folder_tree ft ON f.parent_id = ft.id
    WHERE f.deleted_at IS NULL
)
SELECT f.* FROM files f
INNER JOIN folder_tree ft ON f.id = ft.id;
```

For deep hierarchies or "select all shared files in a subtree" queries, consider a Materialized Path or Closure Table: store the full path string `"root/projects/2024/"`, then `WHERE path LIKE 'root/projects/2024/%'`. Fast with a prefix index but hard to update on rename.

---

## Q3: "How do you implement a sharing feature where a folder is shared with an entire team, and all files inside are accessible?"

**What they're testing:** Permission modeling, inheritance, performance trade-offs.

**Challenge:** If folder `/Engineering` is shared with team "Backend", all files inside should be accessible. When a new file is added to `/Engineering`, it should immediately inherit access. Checking "does user X have access to file Y?" must be fast.

**Option 1: Permission check with ancestor traversal**

On access check for file F:
```sql
-- Walk up the folder tree; check if any ancestor is shared with this user
WITH RECURSIVE parents AS (
    SELECT id, parent_id, owner_id FROM files WHERE id = $file_id
    UNION ALL
    SELECT f.id, f.parent_id, f.owner_id FROM files f
    INNER JOIN parents p ON f.id = p.parent_id
)
SELECT s.permission FROM shares s
INNER JOIN parents p ON p.id = s.file_id
WHERE s.shared_with = $user_id OR s.shared_with IS NULL (public)
LIMIT 1;
```

Pros: No data duplication, immediate inheritance. Cons: Recursive query on every access check — expensive.

**Option 2: Denormalized access table (ACL)**

Maintain an `access_list` table: `(user_id, file_id, permission)`. When a folder is shared, fan-out to all current children and insert rows. When a new file is added inside a shared folder, add it to `access_list` at creation time.

```sql
CREATE TABLE file_access (
    user_id  BIGINT,
    file_id  UUID,
    permission TEXT,
    PRIMARY KEY (user_id, file_id)
);

-- Access check (1ms):
SELECT permission FROM file_access WHERE user_id=$1 AND file_id=$2;
```

Fan-out on new file inside shared folder: insert access rows for all team members. Fan-out when new team member is added: insert access rows for all files in all folders shared with that team.

Cons: Large fan-out for big teams and deep folder trees. Requires sync when users are added/removed from teams.

**Production approach (Google Drive style):** Option 1 for correctness, with an LRU permission cache per `(user_id, file_id)` pair (TTL 5 minutes). Most files are accessed repeatedly; cache eliminates the recursive query overhead in steady state.

---

## Q4: "A user deletes a file but then wants it back 45 days later. Your trash retention is 30 days. What should you have done, and how do you design recovery?"

**What they're testing:** Soft delete, tiered retention, data recovery design.

**The problem:** After 30 days, the hard-delete job ran: `file_versions` deleted, chunk `ref_counts` decremented to 0, chunks purged from S3, then chunk rows deleted. The data is gone.

**Better design:**

**Tier 1: Trash (30 days)** — soft delete, easily restorable, same hot storage.
**Tier 2: Extended retention (30–365 days)** — moved to Glacier, restorable within hours (not instant), costs less.
**Tier 3: Permanent deletion** — after 1 year.

Implementation:
```
At day 30 soft-delete job:
  → Don't delete yet
  → Move S3 objects to Glacier: s3.copy_object(StorageClass='GLACIER')
  → Update chunks.storage_class = 'GLACIER'
  → files.status = 'archived'

At day 365 final delete:
  → Purge from Glacier
  → Decrement ref_counts, GC orphan chunks
```

**Restore from Glacier:**
```
User requests restore (days 31–365):
  → S3 Glacier restore: s3.restore_object(Days=3, GlacierJobParameters={'Tier': 'Standard'})
  → File status = 'restoring'
  → 3–5 hours later: SNS/SQS notification "restore complete"
  → File status → 'restored'; user notified via email
```

**UI:** Show files in "recently deleted" for 30 days (instant restore), and "archived" for 31–365 days (hours to restore). After 365 days: "no longer recoverable."

---

## Q5: "How would you detect that a chunk was corrupted in S3?"

**What they're testing:** Data integrity, checksums, defense in depth.

**The problem:** S3 is durable (11 nines) but not zero-rate corrupt. Bit-rot, silent data corruption from hardware failures, GCS/S3 bugs. If a chunk is corrupted, files that reference it will be silently wrong.

**Detection mechanisms:**

1. **Verify at upload time:** After writing chunk to S3, immediately do a `GET` of the first 32 bytes and re-hash. Compare against expected hash. If mismatch: re-upload.

2. **Content-addressed verify at download:** Before serving a download, hash the retrieved chunk and compare to the stored `content_hash`. If mismatch: serve error + trigger re-upload.
   ```python
   chunk = await s3.get_object(key)
   actual_hash = sha256(chunk.body)
   if actual_hash != expected_hash:
       raise ChunkCorruptionError(f"Chunk {expected_hash} is corrupt")
   ```

3. **S3 ETag validation:** S3 stores an ETag (MD5 for single-part uploads). When uploading, compute MD5 locally and pass via `Content-MD5` header. S3 rejects the upload if the server-computed MD5 differs. Provides integrity guarantee at upload time.

4. **Periodic integrity audit (scrubbing):** Background job: for each chunk, download it and verify hash. Daily sample of 1% of chunks → full verification every 100 days. On corruption detected: attempt recovery from other S3 regions (cross-region replication) or warn the user.

5. **Cross-region replication:** Enable S3 CRR to a second region. If primary region loses a chunk, recover from replica. Also protects against regional disasters.

**At-rest encryption:** Use S3 SSE-S3 or SSE-KMS. Doesn't prevent corruption, but ensures unauthorized reads of S3 objects yield garbage — reduces exposure from S3 misconfigurations.
