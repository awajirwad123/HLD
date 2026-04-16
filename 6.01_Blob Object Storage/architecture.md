# Blob / Object Storage — Architecture & Concepts

## 1. What Is Object Storage?

Object storage treats data as **opaque blobs** (binary large objects) addressed by a globally unique key, rather than files in a hierarchy or records in tables.

| Storage Type | Model | Address | Best For |
|---|---|---|---|
| Block storage (EBS) | Fixed-size blocks | Disk offset | Databases, OS volumes |
| File storage (NFS, EFS) | Files in directories | Path | Shared file systems |
| **Object storage (S3)** | Arbitrary bytes + metadata | Key (URL) | Media, backups, data lakes |

**Key properties of object storage:**
- Flat namespace: `{bucket}/{key}` — no real directory hierarchy (prefix `/` is convention)
- Write-once-read-many: objects are replaced atomically, not partially updated
- Unlimited scale: store petabytes without pre-provisioning
- HTTP/HTTPS API: `PUT`, `GET`, `DELETE`, `HEAD` per object
- Durability: AWS S3 provides 11 nines (99.999999999%) via erasure coding across AZs

---

## 2. Core Object Storage Concepts

### Buckets
- Namespace container; globally unique name within a region
- Policies: IAM, bucket policies, ACLs, block public access
- Versioning: keep all versions of an object; `?versionId=` to access a specific version
- Lifecycle rules: auto-transition to cheaper storage tiers or expire old objects

### Objects
- **Key:** Arbitrary string, often `/`-delimited for logical grouping: `videos/2024/01/abc.mp4`
- **Value:** Byte stream, up to 5 TB per object
- **Metadata:** System metadata (Content-Type, ETag, Last-Modified) + user-defined key-value pairs
- **ETag:** MD5 hash of the object content (for integrity verification and conditional requests)

### Storage Classes (S3 example)
| Class | Retrieval | Durability | Use |
|---|---|---|---|
| Standard | Immediate | 11 9s | Active data |
| Standard-IA | Immediate | 11 9s | Infrequent access (monthly) |
| Glacier Instant Retrieval | Milliseconds | 11 9s | Archive, retrieved < 1×/quarter |
| Glacier Flexible | Minutes–hours | 11 9s | Long-term archive |
| Glacier Deep Archive | Hours | 11 9s | Compliance, years |

---

## 3. Multipart Upload

Large objects (>100 MB) should use multipart upload:

```
1. InitiateMultipartUpload → returns UploadId
2. UploadPart(UploadId, PartNumber=1, data[0:5MB])  → ETag-1
   UploadPart(UploadId, PartNumber=2, data[5MB:10MB]) → ETag-2
   ...parallel uploads...
3. CompleteMultipartUpload(UploadId, [(1, ETag-1), (2, ETag-2), ...])
   → Single atomic object committed
4. (or) AbortMultipartUpload(UploadId) → reclaim partial storage
```

**Benefits:**
- **Parallelism:** Upload N parts simultaneously → throughput = N × single-connection speed
- **Resumability:** Network error → retry only the failed part, not the entire file
- **AWS limits:** Minimum part size 5 MB (except last part); max 10,000 parts; max object size 5 TB
- **Recommended:** Use multipart for anything > 100 MB

**Atomic completion:** `CompleteMultipartUpload` atomically assembles the parts; the object becomes visible only after completion — no partial reads.

---

## 4. Content Addressing

Objects addressed by the **hash of their content** rather than by a semantic name.

```
content_hash = SHA256(file_bytes)
key = f"objects/{content_hash}"

s3.put_object(Bucket="my-store", Key=key, Body=file_bytes)
# Key is deterministic — same content = same key forever
```

**Properties:**
- **Deduplication:** Store the file once regardless of how many users upload identical content
- **Integrity:** Any modification changes the key — you always know what you fetched is what was stored
- **Immutability:** Content-addressed objects never change — the key IS the content

**Real-world uses:**
- **Git:** Every blob, tree, commit is addressed by SHA1/SHA256 of its content
- **Docker images:** Each layer's `digest` is `sha256:<hash>` — layer deduplication across all nodes
- **IPFS:** Entire distributed filesystem built on content addressing (CIDs)
- **CDN cache keys:** Some CDNs optionally use content hash as cache key to maximize cache hit rate
- **Deduplication in backup systems:** Restic, Borg — store identical chunks once across all backups

---

## 5. Chunking

Splitting large files into fixed-size or variable-size chunks before storage:

### Fixed-size chunking
```
file → [chunk_0: 4MB][chunk_1: 4MB][chunk_2: 4MB][chunk_3: 1.2MB]
keys: sha256(chunk_0), sha256(chunk_1), ...
manifest: { "chunks": [hash0, hash1, hash2, hash3], "chunk_size": 4MB }
```

### Variable-size chunking (CDC — Content-Defined Chunking)
Uses a rolling hash (Rabin fingerprint) to find natural boundaries in the data. When the local hash matches a pattern (e.g., lowest N bits = 0), a chunk boundary is cut there.

**Why CDC?** Inserting content in the middle of a file shifts all fixed chunks → all new hashes → full re-upload. CDC detects the insertion and only new/modified chunks get new hashes — unchanged content before and after the insertion reuses existing stored chunks.

**Use in backup systems:**
- Restic: CDC chunking, content-addressed storage → cross-snapshot deduplication
- Tarsnap, Borg: similar approach
- Typical chunk size: average 1–8 MB depending on config

### Manifest file
The manifest (or index) maps a logical file to its ordered list of chunk hashes:
```json
{
  "filename": "video.mp4",
  "size": 1073741824,
  "chunk_size": 4194304,
  "chunks": [
    "e3b0c44298fc1c149afb...",
    "6b86b273ff34fce19d6b...",
    ...
  ]
}
```
To reconstruct the file: fetch each chunk by hash and concatenate in order.

---

## 6. S3 Architecture (How AWS S3 Works Internally)

S3's internal design (from Amazon's papers):

```
Client → S3 Front-End (HTTP) → Metadata Service → Storage Nodes
                              ↕
                         Placement Service
```

**Storage nodes:** Flat key-value stores on commodity hardware. Objects are erasure-coded across multiple physical disks and nodes within an AZ.

**Erasure coding:** S3 Standard uses Reed-Solomon erasure coding — divides an object into N data shards + K parity shards. Any N shards can reconstruct the object. More storage-efficient than 3× replication for large objects.

```
Object → 6 data shards + 3 parity shards = 9 total
Any 6 of the 9 shards can reconstruct the object
Storage overhead: 9/6 = 1.5× (vs 3× for triple replication)
Tolerate: loss of any 3 shards simultaneously
```

**Cross-AZ replication (S3 Standard):**
- Shards are spread across at least 3 AZs
- Total durability: 11 nines

**Strong consistency (since December 2020):**
- All S3 operations are strongly consistent
- A successful PUT is immediately visible to subsequent GETs — no eventual consistency window

---

## 7. Pre-Signed URLs

Delegate time-limited access to objects without exposing credentials:

```python
import boto3
s3 = boto3.client("s3")

# Generate a URL that allows anyone to GET this object for 1 hour
url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "my-bucket", "Key": "videos/abc.mp4"},
    ExpiresIn=3600,   # seconds
)
# URL contains: signature, expiry, bucket, key
# Anyone with the URL can GET the object until expiry — no AWS credentials needed
```

**Pre-signed POST (for direct browser upload):**
```python
response = s3.generate_presigned_post(
    Bucket="my-bucket",
    Key="uploads/${filename}",
    Fields={"Content-Type": "video/mp4"},
    Conditions=[
        ["content-length-range", 1, 500_000_000],  # 500 MB max
    ],
    ExpiresIn=3600,
)
# Browser POSTs directly to S3 — server never proxies the upload stream
```

**Security note:** Pre-signed URL authority comes from the IAM credentials used to sign. If the signing role's permissions are revoked, the URL stops working regardless of expiry.

---

## 8. Use Cases and When to Use Object Storage

| Use Case | Why Object Storage | Notes |
|---|---|---|
| Static assets (images, JS, CSS) | Unlimited scale, CDN-friendly, HTTP URL per object | CloudFront in front of S3 |
| Video storage & streaming | Multipart upload, byte-range GETs for streaming | Pre-signed URLs for access control |
| Data lake / analytics | Columnar files (Parquet, ORC), S3 Select for query pushdown | Athena, Spark on S3 |
| Backup & archival | Lifecycle rules → Glacier, versioning for recovery | Glacier Deep Archive for compliance |
| Log aggregation | Ship logs to S3, query with Athena | Cost-effective vs hot storage |
| ML training data | Immutable datasets, content hashing for reproducibility | Mount with s3fs or use S3A connector |
| Docker registry | OCI image layers = content-addressed blobs in object storage | Registry API on top of S3 |

---

## 9. Object Storage vs. Database: Decision Guide

```
Should I use object storage?

Is the data a large, unstructured blob (image, video, document, archive)?
  └─ YES → Object storage

Do I need to query/filter on field values within the data?
  └─ YES → Database (or search index), with object storage for the raw document

Do I need sub-object updates (e.g., update one field without rewriting)?
  └─ YES → Database or file storage; object storage is write-once

Is the data > 1 MB per item?
  └─ YES → Object storage almost certainly; databases perform poorly at this scale
```

**Hybrid pattern (most common):**
- Store raw binary in S3: `s3://my-bucket/users/{user_id}/avatar.jpg`
- Store metadata + S3 key in relational DB: `{user_id, s3_key, content_type, size, uploaded_at}`
- Query the DB; fetch the actual file from S3 when needed
