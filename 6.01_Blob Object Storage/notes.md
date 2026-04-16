# Blob / Object Storage — Notes & Reference Sheet

## 1. Object Storage vs. Other Storage Types

| Type | Model | Addressable | Max size | Mutability | Example |
|---|---|---|---|---|---|
| Block | Fixed blocks | Disk offset | Volume size | Byte-level edits | EBS, SAN |
| File | Hierarchical files | Path | Filesystem limits | Byte-level edits | NFS, EFS, CIFS |
| **Object** | **Opaque blobs** | **Key (URL)** | **5 TB** | **Replace only** | **S3, GCS, Azure Blob** |

**Object storage rule:** Can't partially update. Must overwrite the entire object.

---

## 2. Core S3 API Cheat Sheet

```python
# Upload (< 100 MB)
s3.put_object(Bucket="b", Key="k", Body=bytes_data)

# Upload large file (handles multipart automatically)
s3.upload_file("/path/file", "bucket", "key")

# Download
obj = s3.get_object(Bucket="b", Key="k")
data = obj["Body"].read()

# Download to file
s3.download_file("bucket", "key", "/path/file")

# Check existence (cheap)
s3.head_object(Bucket="b", Key="k")   # 404 if not found

# List objects
s3.list_objects_v2(Bucket="b", Prefix="folder/", Delimiter="/")

# Delete
s3.delete_object(Bucket="b", Key="k")

# Copy (server-side — no data transfer to client)
s3.copy_object(CopySource={"Bucket":"src","Key":"k"}, Bucket="dst", Key="k2")

# Generate pre-signed URL
url = s3.generate_presigned_url("get_object",
      Params={"Bucket":"b","Key":"k"}, ExpiresIn=3600)
```

---

## 3. Multipart Upload Flow

```
Client                    S3
  │                        │
  ├──create_multipart ────►│  → returns UploadId
  │                        │
  ├──upload_part(1) ──────►│  → returns ETag-1
  ├──upload_part(2) ──────►│  (concurrent)
  ├──upload_part(3) ──────►│
  │                        │
  ├──complete_multipart───►│  (ordered list of [PartNum, ETag])
  │                        │   → object atomically visible
  │◄── ETag of full object─┤
```

**Key numbers:**
- Minimum part size: **5 MB** (except last part)
- Maximum parts: **10,000**
- Maximum object size: **5 TB**
- Recommended threshold: use multipart for files **> 100 MB**
- Incomplete multipart uploads: charged storage until `AbortMultipartUpload` or lifecycle rule

---

## 4. Content Addressing

| Property | Value |
|---|---|
| Hash function | SHA256 (32 bytes → 64 hex chars) |
| Key structure | `objects/{hash[:2]}/{hash[2:]}` — 2-char prefix for directory sharding |
| Deduplication | Same content → same hash → single stored object |
| Integrity | Read-time hash verification |
| Immutability | Content never changes; key is permanent |

**Real-world examples:**
- **Git:** `git cat-file -p <sha1>` — every blob is CAS
- **Docker:** Image layer `digest: sha256:abc123...`
- **IPFS:** Content Identifier (CID) = multihash of content
- **Restic/Borg:** Backup deduplication via CDC + CAS

---

## 5. Chunking Strategies

| Strategy | Chunk boundary | Resistance to insertions | Complexity |
|---|---|---|---|
| Fixed-size | Every N bytes | Poor — all chunks shift | Simple |
| CDC (Rabin) | Rolling hash boundary | Excellent — only changed region | Moderate |
| Semantic | File format boundaries (e.g., video frames) | Good | File-type specific |

**CDC typical parameters:**
- Target chunk size: 1–8 MB average
- Min chunk: 256 KB; Max chunk: 16 MB (prevent unbounded chunks)
- Rabin polynomial evaluatd over sliding window of 64 bytes

---

## 6. S3 Storage Classes (Cost vs. Access Speed)

| Class | Access time | Min storage duration | Use case |
|---|---|---|---|
| Standard | Immediate | None | Active production data |
| Standard-IA | Immediate | 30 days | Backups, accessed monthly |
| One Zone-IA | Immediate | 30 days | Reproducible data, single AZ ok |
| Glacier Instant | Milliseconds | 90 days | Archives retrieved quarterly |
| Glacier Flexible | 1–12 hours | 90 days | Long-term archives |
| Glacier Deep Archive | Up to 48h | 180 days | Compliance, 7+ year retention |
| Intelligent-Tiering | Automatic | 30 days | Unknown/changing access patterns |

**Lifecycle rule pattern:**
```
Standard → (30 days) → Standard-IA → (90 days) → Glacier → (365 days) → Expire
```

---

## 7. Durability & Availability

| S3 Class | Durability | Availability | AZs |
|---|---|---|---|
| Standard | 99.999999999% (11 9s) | 99.99% | ≥ 3 |
| Standard-IA | 11 9s | 99.9% | ≥ 3 |
| One Zone-IA | 11 9s | 99.5% | 1 |

**How 11 nines is achieved:**
- Objects erasure-coded into N data + K parity shards
- Shards distributed across multiple AZs and physical racks
- Reed-Solomon: any N of (N+K) shards can reconstruct the object
- S3 Standard uses 6 data + 3 parity = 1.5× storage overhead (vs 3× for replication)

---

## 8. Pre-Signed URLs — Security Reference

| Property | Detail |
|---|---|
| Who generates | Server (with IAM credentials) |
| Who uses | Anyone with the URL (bearer token) |
| Expiry | Configurable: 1 second to 7 days (default 1 hour) |
| Permission | Derives from the signer's IAM policy |
| Revocation | Revoke the signer's credentials; URL becomes invalid before expiry |
| For upload | `generate_presigned_post()` — includes size and content-type constraints |
| For download | `generate_presigned_url("get_object")` |

**Direct upload pattern (no server proxy):**
```
User → POST /upload-url → App Server → pre-signed POST URL  → User
User → POST directly to S3 (server never sees bytes) ────────────────► S3
User → POST /confirm → App Server → HEAD /key → S3 (verify) → App Server → 200
```

---

## 9. Key Numbers to Memorize

| Metric | Value |
|---|---|
| Max object size | 5 TB |
| Max single PUT size | 5 GB |
| Recommended multipart threshold | 100 MB |
| Min part size (multipart) | 5 MB |
| Max parts (multipart) | 10,000 |
| S3 Standard durability | 99.999999999% (11 nines) |
| S3 consistency model | Strong (since Dec 2020) |
| Erasure coding overhead (vs 3× replication) | 1.5× |
| Pre-signed URL max TTL | 7 days |

---

## 10. Hybrid Storage Pattern (Most Common Production Design)

```
Application Layer
       │
  ┌────▼────┐      ┌──────────────────┐
  │  DB     │      │   Object Store   │
  │(metadata│      │   (S3 / GCS)     │
  │ + key)  │      │                  │
  └─────────┘      │  raw bytes here  │
                   └──────────────────┘

DB row: { user_id, filename, s3_key, content_type, size, created_at }
S3 key: "users/{user_id}/profile/{uuid}.jpg"

- Query metadata from DB (fast, indexed, relational)
- Fetch binary from S3 (cheap, scalable, CDN-cacheable)
- Never store large blobs in the DB
```
