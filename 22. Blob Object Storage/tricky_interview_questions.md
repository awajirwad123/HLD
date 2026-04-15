# Blob / Object Storage — Tricky Interview Questions

---

## Question 1: "Your system stores profile pictures. A user uploads a new picture, but for the next few seconds other users still see the old one. Before December 2020, that was expected S3 behavior — would you build a workaround for it?"

**The trap:** Testing whether you know S3's current consistency model, and whether you'd over-engineer for a problem that no longer exists.

**Strong answer:**

Since December 2020, S3 provides **strong read-after-write consistency** for all operations. A successful PUT is immediately visible to all subsequent GETs — there is no eventual consistency window to work around. I would not build anything special for this.

If this were a pre-2020 system design question, the workaround would be: version the key — `profile/{user_id}/{uuid}.jpg` — so each upload writes to a new key rather than overwriting. The old key remains readable while the metadata record in the DB is atomically updated to point to the new key. Requests using the new metadata key immediately get the new picture, requests with cached old keys still hit the old object. This also naturally provides an audit trail.

But today, simply `PUT users/{user_id}/avatar.jpg` and update the DB metadata atomically — strong consistency ensures no stale reads.

---

## Question 2: "You're storing medical imaging files (DICOM) — some are 2 GB each. You use `put_object` and occasionally uploads fail halfway through with a timeout. The doctor then can't find the file. What's wrong and how do you fix it?"

**The trap:** `put_object` for large files is a single HTTP PUT — no resumability, single point of failure at ~5 GB limit.

**Strong answer:**

Two problems:

1. `put_object` is a single HTTP PUT. At 2 GB, any network hiccup aborts the entire upload. No partial recovery — must restart from zero.
2. S3's single PUT limit is 5 GB, but even 2 GB over a hospital's network is unreliable.

**Fix: multipart upload with resumability:**

```python
# Store the UploadId in the DB when initiating
upload_id = initiate_multipart_upload(bucket, key)
db.save_upload_state(file_id, upload_id)

# Upload 100 MB parts with per-part retry
for part_num, data in split_file(path, part_size=100*MB):
    upload_part_with_retry(upload_id, part_num, data, max_retries=5)
    db.mark_part_done(file_id, part_num)

# On connection loss:
# Re-query DB for upload_id + completed parts
# Resume from the first incomplete part

complete_multipart_upload(upload_id, completed_parts)
```

**Additionally:**
- Set a lifecycle rule: `AbortIncompleteMultipartUpload` after 7 days — prevents abandoned uploads from accumulating storage charges
- Validate integrity: compare the full file's SHA256 against a separately uploaded checksum file or use S3's `ChecksumSHA256` parameter (built-in integrity verification since 2022)

---

## Question 3: "Two engineers debate: Engineer A says 'use content addressing for all our file storage — same hash = deduplicated automatically.' Engineer B says 'content addressing breaks our GDPR deletion requirement — we can't delete a file if another user shares the same content.' Who is right?"

**The trap:** Both are right — they're identifying a genuine tension in content-addressed storage.

**Strong answer:**

Both engineers are identifying real constraints, and the resolution requires a design choice:

**Engineer A is correct about deduplication:** Content-addressed storage naturally deduplicates. Two users uploading the same profile photo → one stored object → significant storage savings at scale.

**Engineer B is correct about GDPR:** GDPR Article 17 grants users the right to erasure. If User A uploads a medical document, and User B happens to have uploaded identical bytes, deleting User A's content cannot delete the underlying object because User B still references it. The object's key is the hash — there's no user identity in it.

**Resolution — reference counting:**

Maintain a mapping table: `{content_hash → [user_ids who reference it, reference_count]}`. On delete request:
1. Decrement the reference count for that user
2. If `reference_count == 0` — no more users reference it → delete the S3 object
3. If `reference_count > 0` — other users still reference it → only remove the mapping for the requesting user

The S3 object survives as long as any user references it, but the deleting user's data is logically erased from their perspective.

**Edge case for GDPR:** If the file is truly unique to one user (count=1 after deletion), it's physically deleted. If it's shared, it's logically unlinked. GDPR typically requires that the user's association with the data is severed — physical deletion of shared bytes may be acceptable under the "technically infeasible" exception, documented in your data processing records.

---

## Question 4: "Your CDN serves S3-backed static assets. You deploy a new frontend build and update index.html and main.js in S3. Users still see the old version for the next 24 hours. Why, and what's the correct fix?"

**The trap:** Candidates say "S3 is eventually consistent" — it's not anymore. The real issue is CDN caching.

**Strong answer:**

S3 is strongly consistent — the objects are updated immediately. The problem is the **CDN's edge cache TTL**. CloudFront (or any CDN) caches `index.html` and `main.js` at edge nodes with the TTL set in `Cache-Control: max-age=86400` (24 hours). When a user in Tokyo requests the file, they get the cached old version from the Tokyo edge — CloudFront doesn't even contact S3 for 24 hours.

**Fixes:**

1. **Cache-busting filenames (preferred):** Name assets with a content hash: `main.abc123.js`. When you deploy, the new hash produces a new filename → CDN sees it as a new object → no stale cache issue. Set long TTL on hashed assets (`max-age=31536000`). Only `index.html` gets a short TTL or `no-cache` since it references the hashed assets by name.

2. **CDN invalidation (for files that can't be renamed):** Explicitly invalidate `index.html` in CloudFront: `aws cloudfront create-invalidation --paths "/*"`. Propagates to all edge nodes within ~5 minutes. **Cost:** AWS charges for invalidations beyond the first 1,000/month.

3. **Versioned S3 prefixes:** Deploy to `app/v42/` prefix, update DNS/CDN origin to point to new prefix, keep old prefix for 24 hours before deletion.

**Correct architecture:** Use immutable, content-hashed filenames for all static assets. Set `Cache-Control: max-age=31536000, immutable`. Only `index.html` gets `Cache-Control: no-cache, no-store`. This eliminates stale cache issues entirely.

---

## Question 5: "You're designing a file storage service like Dropbox. A user has a 100 GB file. They edit one line in the middle of it. Do you re-upload the 100 GB file? How do you handle this efficiently?"

**The trap:** Naive approach re-uploads everything. CDC chunking is the right answer.

**Strong answer:**

Re-uploading 100 GB for a single-line edit is wasteful and impractical. The solution is **CDC (Content-Defined Chunking)** — the same approach used by Dropbox, Restic, and rsync:

**How Dropbox handles this:**

1. The file is chunked using a rolling hash (Rabin fingerprint) — chunk boundaries are determined by content, not fixed offsets.
2. Each chunk is stored content-addressed: `SHA256(chunk) → S3 key`.
3. A manifest file records the ordered list of chunk hashes.

**What happens on edit:**

```
Before edit:
  Chunk 1: [bytes 0–4MB]      hash: aaa...
  Chunk 2: [bytes 4MB–8MB]    hash: bbb...   ← contains the edited line
  Chunk 3: [bytes 8MB–12MB]   hash: ccc...
  ...etc (100 GB / ~4 MB avg = ~25,000 chunks)

After edit:
  Chunk 1: same content → hash aaa... (already stored — SKIP)
  Chunk 2: changed → new hash ddd... → upload only this chunk (~4 MB)
  Chunk 3: same content → hash ccc... (already stored — SKIP)
  ...
```

**Result:** Only ~4 MB of the 100 GB file is re-uploaded — the one changed chunk.

**Why CDC is better than fixed-size chunking here:** If you insert a few bytes at offset 0, fixed chunking shifts ALL subsequent chunks → all hashes change → full re-upload. CDC detects natural boundaries that survive insertions, so only the modified region's chunks change.

**Implementation note:** The manifest (list of chunk hashes in order) is itself stored as a small object in S3 and versioned/updated on each sync.

---

## Question 6: "An engineer says: 'S3 is just a giant key-value store. We should store all our data there, including user profiles and transaction records.' Critique this proposal."

**The trap:** Tests ability to recognize when object storage is the wrong tool.

**Strong answer:**

S3 is an excellent key-value store for **blobs**, but using it for structured records has significant drawbacks:

1. **No queries:** You cannot do `SELECT * FROM users WHERE email = 'x'`. You can only fetch by exact key. Storing user profiles as `users/{user_id}.json` means you must know the `user_id` to retrieve — no secondary index, no range scans.

2. **No transactions:** You cannot atomically update a user's email and a related record. S3 has no multi-object atomicity or rollback.

3. **No secondary indexes:** Finding all users in a city requires listing ALL user objects and filtering in the client — O(N) full scan.

4. **Latency:** S3 GET latency is ~10–100ms (vs <1ms for a local key-value store or in-memory cache). Fine for media, poor for user profile lookups on critical paths.

5. **Consistency window for lists:** While individual GETs are strongly consistent, `list_objects_v2` is eventually consistent with respect to very recent writes in some edge cases.

**Correct pattern:**
- User profiles and transaction records → PostgreSQL or DynamoDB (queryable, transactional, indexed)
- Profile pictures, attachments, raw exports → S3 (blob storage)
- S3 key stored as a field in the database row: `{ user_id, name, email, avatar_s3_key }`

---

## Question 7: "Your system generates 1 million small files per day (~1 KB each). You store them in S3. Storage cost is minimal, but S3 costs are surprisingly high. Why, and how do you fix it?"

**The trap:** S3 charges per-request, not just per-byte-stored. Small files amplify request costs.

**Strong answer:**

S3 pricing has two components: storage (per GB/month) and **requests** (per 1,000 PUT/GET). 1 million files/day = 1 million PUT requests = $5/day in PUT costs alone, regardless of tiny file size.

**Root cause:** S3's request cost is amortized over object size — 1 million 1 KB files has the same request cost as 1 million 1 GB files, but the 1 KB variant stores only 1 GB total.

**Solutions:**

1. **Aggregate before storing:** Combine many small records into larger batches. Instead of 1 file per event, batch 10,000 events into a single Parquet file uploaded once. 1 million events = 100 files instead of 1 million → 100 PUTs/day.

2. **Use a different store for small objects:** Write small records to a database (DynamoDB, Redis) or a message queue. Periodically flush to S3 in bulk.

3. **Compression + batching:** GZIP or Snappy compress batches of records before uploading — reduces both storage and request count.

4. **S3 class selection:** For write-once analytics data, write to Standard initially, transition with lifecycle rules; doesn't reduce request count but reduces storage cost.

**Key rule:** S3 is optimized for large objects. The cost sweet spot is objects ≥ 1 MB. For millions of small objects, aggregate them first.
