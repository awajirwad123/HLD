# Blob / Object Storage — Quick-Fire Interview Q&A

---

**Q1. What is the difference between block storage, file storage, and object storage?**

Block storage (EBS) exposes raw disk blocks — the OS manages the filesystem. File storage (NFS/EFS) provides a POSIX hierarchical filesystem over a network. Object storage (S3) stores arbitrary blobs addressed by a flat key with no directory structure — accessed via HTTP, write-once-replace semantics, no in-place editing.

---

**Q2. Why can't you partially update an S3 object? What are the implications?**

S3 objects are immutable — to change one byte, you must write the entire object. This simplifies replication and erasure coding (a single version is always consistent). Implication: don't store frequently-updated small records in S3 (use a database). Good fit for write-once data: media, archives, logs, immutable artifacts.

---

**Q3. When should you use multipart upload? What is the minimum part size?**

Use multipart for files > ~100 MB. Benefits: parallelism (simultaneous part uploads), resumability (retry only failed parts), ability to upload the object before you know its full size (streaming). Minimum part size is 5 MB except for the final part. Maximum 10,000 parts, so for a 5 TB file, minimum part size = 500 MB.

---

**Q4. What is content addressing and why is it useful?**

Content addressing stores objects by the hash of their content rather than a semantic name. Same content = same hash = one stored copy, regardless of how many clients have uploaded it. Provides deduplication and integrity verification — if you fetch an object and the hash doesn't match, it was corrupted. Used by Git, Docker image layers, IPFS, and backup systems.

---

**Q5. How does S3 achieve 99.999999999% (11 nines) durability?**

S3 uses Reed-Solomon erasure coding. An object is split into N data shards + K parity shards. Any N of the N+K total shards can reconstruct the object. Shards are distributed across at least 3 Availability Zones. S3 Standard uses 6 data + 3 parity shards = 1.5× storage overhead (more efficient than 3× full replication), tolerating simultaneous loss of any 3 shards.

---

**Q6. What is a pre-signed URL and what security guarantees does it provide?**

A pre-signed URL is an S3 URL with an embedded signature granting time-limited access to a specific operation (GET or PUT) on a specific object. It inherits the signer's IAM permissions; if those permissions are revoked, the URL immediately stops working. No AWS credentials are needed by the client. For uploads, `generate_presigned_post` allows enforcing size limits and content-type constraints server-side.

---

**Q7. Why should your application use pre-signed URLs for uploads instead of proxying bytes through the server?**

Proxying creates a bottleneck: the server must receive the full upload, hold it in memory/disk, then re-upload to S3. Pre-signed URLs allow the client to upload directly to S3 — the server only generates the URL (fast) and later confirms completion. This eliminates server bandwidth costs, removes the server as a bottleneck, and allows S3 to handle all the multipart/chunking logic.

---

**Q8. What is a lifecycle rule in S3 and give a typical use case?**

A lifecycle rule automatically transitions objects to cheaper storage classes or deletes them based on age or tags. Typical use: application logs stored in Standard on day 1 → Standard-IA after 30 days → Glacier after 90 days → deleted after 365 days. Another critical use: `AbortIncompleteMultipartUpload` after 7 days to clean up storage from abandoned uploads.

---

**Q9. What is the S3 consistency model? Has it always been this way?**

Since December 2020, S3 provides strong consistency for all operations. A successful PUT is immediately visible to subsequent GET and LIST requests from any client. Before that, S3 was eventually consistent for some operations (overwrite PUTs and DELETEs) — applications had to account for stale reads.

---

**Q10. You have 10 GB of server logs generated daily. After 30 days they're rarely accessed, after 90 days almost never, and after 1 year can be deleted. Design the storage strategy.**

Upload logs to S3 Standard on creation. Configure a lifecycle rule: transition to Standard-IA after 30 days (reduces cost ~60%), transition to Glacier Flexible after 90 days (reduces cost a further 70%), expire after 365 days. Add an `AbortIncompleteMultipartUpload` rule (7 days) for failed uploads. For ad-hoc queries on the Standard/IA tier, use S3 Select or Athena without downloading full files.

---

**Q11. Explain the ETag field on an S3 object. Is it always an MD5 hash?**

For objects uploaded in a single PUT, the ETag is the MD5 of the entire object content. For multipart uploads, the ETag is `MD5(MD5(part1) + MD5(part2) + ...) + "-" + num_parts` — NOT a single MD5 of the content. This is a common gotcha: you can't verify a multipart-uploaded object's integrity by comparing the ETag to a locally computed MD5 of the full file.

---

**Q12. What is S3 Intelligent-Tiering and when would you use it?**

Intelligent-Tiering monitors access patterns and automatically moves objects between Standard and IA tiers based on access frequency (30-day window). Use it when access patterns are unknown or change over time and you don't want to manage lifecycle rules. It carries a small per-object monitoring charge (~$0.0025/1000 objects/month), so it's not cost-effective for millions of very small objects.

---

**Q13. How would you implement cross-region replication for compliance?**

Enable S3 Cross-Region Replication (CRR) on the source bucket. Configure a destination bucket in the target region and an IAM role with permissions to replicate. S3 asynchronously copies new objects (and optionally existing objects via S3 Batch Operations) to the destination. CRR adds latency but provides geo-redundancy and data sovereignty compliance. Note: the destination bucket is not a real-time mirror — it's eventually consistent (seconds to minutes lag for new objects).

---

**Q14. What is S3 Select and when would you use it?**

S3 Select allows you to run SQL queries against structured data (CSV, JSON, Parquet) stored in S3, returning only the matching rows — without downloading the entire file. Use when you need to query large log files or datasets and only need a subset of records. Reduces data transfer cost and latency. For complex queries, Athena is more appropriate (full SQL engine on top of S3).

---

**Q15. You're building a video platform. Users upload videos, videos are transcoded into multiple resolutions, and served globally. Sketch the architecture involving object storage.**

1. **Upload:** Client gets a pre-signed POST URL from app server → uploads raw video to `raw/{video_id}/original.mp4` in S3.
2. **Trigger:** S3 event notification fires a Lambda (or SQS message) on upload completion.
3. **Transcode:** Worker fetches raw video from S3, transcodes to 360p/720p/1080p, uploads output to `videos/{video_id}/720p.mp4` etc.
4. **CDN:** CloudFront distribution with S3 `videos/` prefix as origin. Serves transcoded files from edge caches globally.
5. **Metadata:** PostgreSQL stores `{video_id, title, s3_keys, status, created_at}`.
6. **Access control:** Pre-signed URLs for private videos; public URLs for public content. Lifecycle rules to archive raws after 90 days.
