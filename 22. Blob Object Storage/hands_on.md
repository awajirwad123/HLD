# Blob / Object Storage — Hands-On Exercises

## Exercise 1: Multipart Upload with Resumability

Implement a resumable multipart uploader that can continue after network failure:

```python
"""
Goal: Upload a large file to S3 using multipart upload.
      - Split into 10 MB parts
      - Upload parts in parallel (concurrently)
      - Retry failed parts individually
      - Resume an in-progress upload if UploadId is already known
      - Clean up aborted uploads
"""

import asyncio
import hashlib
import os
from pathlib import Path
from typing import Optional
import aioboto3
from botocore.exceptions import ClientError

PART_SIZE = 10 * 1024 * 1024   # 10 MB (min is 5 MB for non-last parts)
MAX_CONCURRENCY = 4
BUCKET = "my-upload-bucket"


def split_file(file_path: str, part_size: int = PART_SIZE) -> list[tuple[int, bytes]]:
    """Split file into numbered (1-based) parts."""
    parts = []
    with open(file_path, "rb") as f:
        part_number = 1
        while chunk := f.read(part_size):
            parts.append((part_number, chunk))
            part_number += 1
    return parts


async def upload_part(
    s3,
    bucket: str,
    key: str,
    upload_id: str,
    part_number: int,
    data: bytes,
    semaphore: asyncio.Semaphore,
    max_retries: int = 3,
) -> dict:
    """Upload a single part with retry on transient errors."""
    async with semaphore:
        for attempt in range(max_retries):
            try:
                response = await s3.upload_part(
                    Bucket=bucket,
                    Key=key,
                    UploadId=upload_id,
                    PartNumber=part_number,
                    Body=data,
                )
                etag = response["ETag"]
                print(f"  Part {part_number:3d} uploaded (ETag: {etag[:12]}...)")
                return {"PartNumber": part_number, "ETag": etag}

            except ClientError as e:
                if attempt < max_retries - 1:
                    delay = 2 ** attempt
                    print(f"  Part {part_number} failed (attempt {attempt+1}), retrying in {delay}s: {e}")
                    await asyncio.sleep(delay)
                else:
                    raise


async def multipart_upload(
    file_path: str,
    key: str,
    existing_upload_id: Optional[str] = None,
) -> str:
    """
    Upload a file using multipart upload.
    Pass existing_upload_id to resume a previous upload.
    Returns the ETag of the completed object.
    """
    session = aioboto3.Session()
    async with session.client("s3") as s3:

        # ─── Initiate or resume ───────────────────────────────────────────
        if existing_upload_id:
            upload_id = existing_upload_id
            print(f"Resuming upload {upload_id[:8]}...")

            # Retrieve already-uploaded parts to skip them
            paginator = s3.get_paginator("list_parts")
            completed_parts = {}
            async for page in paginator.paginate(
                Bucket=BUCKET, Key=key, UploadId=upload_id
            ):
                for part in page.get("Parts", []):
                    completed_parts[part["PartNumber"]] = part["ETag"]
            print(f"  {len(completed_parts)} parts already uploaded")
        else:
            response = await s3.create_multipart_upload(
                Bucket=BUCKET,
                Key=key,
                ContentType="application/octet-stream",
            )
            upload_id = response["UploadId"]
            completed_parts = {}
            print(f"Initiated multipart upload: {upload_id[:8]}...")

        # ─── Split and upload ─────────────────────────────────────────────
        parts = split_file(file_path)
        print(f"File: {file_path} ({len(parts)} parts × {PART_SIZE // 1024 // 1024} MB)")

        semaphore = asyncio.Semaphore(MAX_CONCURRENCY)
        tasks = []
        for part_number, data in parts:
            if part_number in completed_parts:
                print(f"  Part {part_number:3d} already done — skipping")
                tasks.append(asyncio.create_task(
                    asyncio.coroutine(lambda pn=part_number, et=completed_parts[part_number]:
                        {"PartNumber": pn, "ETag": et})()
                ))
            else:
                tasks.append(asyncio.create_task(
                    upload_part(s3, BUCKET, key, upload_id, part_number, data, semaphore)
                ))

        uploaded_parts = await asyncio.gather(*tasks)

        # ─── Complete ─────────────────────────────────────────────────────
        sorted_parts = sorted(uploaded_parts, key=lambda p: p["PartNumber"])
        response = await s3.complete_multipart_upload(
            Bucket=BUCKET,
            Key=key,
            UploadId=upload_id,
            MultipartUpload={"Parts": sorted_parts},
        )

        etag = response["ETag"]
        print(f"Upload complete! ETag: {etag}")
        return etag


async def abort_upload(key: str, upload_id: str):
    """Clean up an incomplete multipart upload."""
    session = aioboto3.Session()
    async with session.client("s3") as s3:
        await s3.abort_multipart_upload(
            Bucket=BUCKET, Key=key, UploadId=upload_id
        )
        print(f"Aborted upload {upload_id[:8]}")


# ─── Test ─────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    # Create a ~50 MB test file
    test_file = "/tmp/test_large_file.bin"
    if not Path(test_file).exists():
        with open(test_file, "wb") as f:
            f.write(os.urandom(50 * 1024 * 1024))
        print(f"Created {test_file}")

    asyncio.run(multipart_upload(test_file, "uploads/test_large_file.bin"))
```

---

## Exercise 2: Content-Addressed Storage

Build a local content-addressed object store — same semantics as Git's object store:

```python
"""
Goal: Implement a content-addressed store where:
  - Objects are stored by SHA256 hash of their content
  - Uploading the same content twice is a no-op (deduplication)
  - Any corruption is detectable on read
  - A manifest file maps a logical filename → list of chunk hashes
"""

import hashlib
import json
import os
from pathlib import Path
from dataclasses import dataclass, field
from typing import Optional


STORE_DIR = Path("/tmp/cas_store")
CHUNK_SIZE = 4 * 1024 * 1024   # 4 MB chunks


class ContentAddressedStore:

    def __init__(self, root: Path = STORE_DIR):
        self.root = root
        self.objects_dir = root / "objects"
        self.manifests_dir = root / "manifests"
        self.objects_dir.mkdir(parents=True, exist_ok=True)
        self.manifests_dir.mkdir(parents=True, exist_ok=True)

    def _object_path(self, digest: str) -> Path:
        """Two-level directory sharding: objects/ab/cd1234..."""
        return self.objects_dir / digest[:2] / digest[2:]

    def put_chunk(self, data: bytes) -> str:
        """
        Store a bytes chunk by its SHA256 hash.
        Returns the hex digest. Duplicate = no-op.
        """
        digest = hashlib.sha256(data).hexdigest()
        path = self._object_path(digest)

        if path.exists():
            return digest   # Already stored — deduplication

        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_bytes(data)
        return digest

    def get_chunk(self, digest: str) -> bytes:
        """
        Retrieve a chunk by its SHA256 hash.
        Raises ValueError if content is corrupted.
        """
        path = self._object_path(digest)
        if not path.exists():
            raise KeyError(f"Object {digest[:12]}... not found")

        data = path.read_bytes()

        # Integrity check — always verify on read
        actual_digest = hashlib.sha256(data).hexdigest()
        if actual_digest != digest:
            raise ValueError(
                f"Integrity violation! Expected {digest[:12]}..., "
                f"got {actual_digest[:12]}..."
            )
        return data

    def exists(self, digest: str) -> bool:
        return self._object_path(digest).exists()

    def store_file(self, file_path: str, name: Optional[str] = None) -> dict:
        """
        Chunk a file, store all chunks content-addressed,
        and write a manifest. Returns the manifest dict.
        """
        path = Path(file_path)
        name = name or path.name
        chunk_hashes = []
        total_size = 0

        with open(path, "rb") as f:
            while chunk := f.read(CHUNK_SIZE):
                digest = self.put_chunk(chunk)
                chunk_hashes.append(digest)
                total_size += len(chunk)

        manifest = {
            "name": name,
            "size": total_size,
            "chunk_size": CHUNK_SIZE,
            "chunk_count": len(chunk_hashes),
            "chunks": chunk_hashes,
        }

        # Store manifest by its own content hash
        manifest_bytes = json.dumps(manifest, indent=2).encode()
        manifest_digest = hashlib.sha256(manifest_bytes).hexdigest()
        manifest_path = self.manifests_dir / f"{manifest_digest}.json"
        manifest_path.write_bytes(manifest_bytes)

        print(f"Stored '{name}': {len(chunk_hashes)} chunks, manifest={manifest_digest[:12]}...")
        return manifest, manifest_digest

    def restore_file(self, manifest_digest: str, output_path: str) -> None:
        """Reconstruct a file from its manifest hash."""
        manifest_path = self.manifests_dir / f"{manifest_digest}.json"
        manifest = json.loads(manifest_path.read_bytes())

        print(f"Restoring '{manifest['name']}': {manifest['chunk_count']} chunks...")
        with open(output_path, "wb") as f:
            for i, chunk_hash in enumerate(manifest["chunks"]):
                chunk = self.get_chunk(chunk_hash)
                f.write(chunk)
                print(f"  Chunk {i+1}/{manifest['chunk_count']} restored ({chunk_hash[:8]}...)")

        print(f"Restored to {output_path} ({manifest['size']:,} bytes)")

    def stats(self) -> dict:
        """Count objects and total storage used."""
        total_objects = 0
        total_bytes = 0
        for obj_file in self.objects_dir.rglob("*"):
            if obj_file.is_file():
                total_objects += 1
                total_bytes += obj_file.stat().st_size
        return {"objects": total_objects, "bytes": total_bytes}


# ─── Demo ─────────────────────────────────────────────────────────────────────

def demo():
    store = ContentAddressedStore()

    # Create test files
    file_a = "/tmp/file_a.bin"
    file_b = "/tmp/file_b.bin"  # Same first half as file_a → chunk reuse

    import os
    shared_data = os.urandom(8 * 1024 * 1024)   # 8 MB shared content

    with open(file_a, "wb") as f:
        f.write(shared_data + os.urandom(4 * 1024 * 1024))   # 12 MB

    with open(file_b, "wb") as f:
        f.write(shared_data + os.urandom(4 * 1024 * 1024))   # 12 MB, same first 8 MB

    # Store both files
    _, manifest_a = store.store_file(file_a, "file_a.bin")
    _, manifest_b = store.store_file(file_b, "file_b.bin")

    stats = store.stats()
    print(f"\nStore stats: {stats['objects']} objects, {stats['bytes'] / 1024 / 1024:.1f} MB")
    print("(Expected ~4 unique 4MB chunks from file_a + 2 unique from file_b = 6 total)")
    print("(Shared 8MB content = 2 chunks stored ONCE despite appearing in both files)")

    # Restore file_a
    store.restore_file(manifest_a, "/tmp/file_a_restored.bin")

    # Verify integrity
    import hashlib
    orig_hash = hashlib.sha256(open(file_a, "rb").read()).hexdigest()
    rest_hash = hashlib.sha256(open("/tmp/file_a_restored.bin", "rb").read()).hexdigest()
    print(f"\nIntegrity check: {orig_hash == rest_hash}")


demo()
```

---

## Exercise 3: Pre-Signed URL Service (FastAPI)

Build an upload/download service using S3 pre-signed URLs so the server never proxies file bytes:

```python
"""
Goal: API that:
  1. Client asks server for an upload URL → server generates pre-signed POST URL
  2. Client uploads DIRECTLY to S3 (server never sees the bytes)
  3. Server records the metadata (key, user, size) in PostgreSQL
  4. Client asks for download URL → server generates pre-signed GET URL

Security: Server validates file size and content-type constraints
          in the pre-signed POST conditions.
"""

import uuid
import boto3
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from datetime import datetime

app = FastAPI()
s3 = boto3.client("s3", region_name="us-east-1")
BUCKET = "my-media-bucket"
MAX_FILE_SIZE = 500 * 1024 * 1024  # 500 MB
ALLOWED_CONTENT_TYPES = ["image/jpeg", "image/png", "video/mp4", "application/pdf"]


class UploadRequest(BaseModel):
    filename: str
    content_type: str
    size_bytes: int


class UploadResponse(BaseModel):
    upload_url: str           # S3 pre-signed POST endpoint
    upload_fields: dict       # Form fields to include with the POST
    object_key: str           # S3 key assigned to this upload
    expires_in: int           # Seconds until upload URL expires


class DownloadResponse(BaseModel):
    download_url: str
    expires_in: int


# In-memory metadata store (substitute real DB)
_metadata_store: dict[str, dict] = {}


@app.post("/upload/initiate", response_model=UploadResponse)
async def initiate_upload(req: UploadRequest):
    # Validate
    if req.content_type not in ALLOWED_CONTENT_TYPES:
        raise HTTPException(400, f"Content-Type {req.content_type} not allowed")
    if req.size_bytes > MAX_FILE_SIZE:
        raise HTTPException(400, f"File too large: {req.size_bytes} > {MAX_FILE_SIZE}")

    # Generate a unique key
    object_key = f"uploads/{uuid.uuid4()}/{req.filename}"

    # Generate pre-signed POST — conditions enforced by S3, not the client
    response = s3.generate_presigned_post(
        Bucket=BUCKET,
        Key=object_key,
        Fields={
            "Content-Type": req.content_type,
        },
        Conditions=[
            {"Content-Type": req.content_type},
            ["content-length-range", 1, req.size_bytes],
        ],
        ExpiresIn=3600,
    )

    # Record metadata
    _metadata_store[object_key] = {
        "filename": req.filename,
        "content_type": req.content_type,
        "size_bytes": req.size_bytes,
        "status": "pending",
        "created_at": datetime.utcnow().isoformat(),
    }

    return UploadResponse(
        upload_url=response["url"],
        upload_fields=response["fields"],
        object_key=object_key,
        expires_in=3600,
    )


@app.post("/upload/confirm/{object_key:path}")
async def confirm_upload(object_key: str):
    """
    Called after client finishes uploading to S3.
    Server verifies the object exists and updates status.
    """
    if object_key not in _metadata_store:
        raise HTTPException(404, "Unknown object key")

    # Verify object actually exists in S3
    try:
        head = s3.head_object(Bucket=BUCKET, Key=object_key)
    except s3.exceptions.ClientError:
        raise HTTPException(400, "Object not found in S3 — upload may have failed")

    _metadata_store[object_key]["status"] = "uploaded"
    _metadata_store[object_key]["etag"] = head["ETag"]
    _metadata_store[object_key]["s3_size"] = head["ContentLength"]

    return {
        "object_key": object_key,
        "status": "uploaded",
        "size": head["ContentLength"],
    }


@app.get("/download/{object_key:path}", response_model=DownloadResponse)
async def get_download_url(object_key: str):
    meta = _metadata_store.get(object_key)
    if not meta or meta["status"] != "uploaded":
        raise HTTPException(404, "Object not found or not yet uploaded")

    url = s3.generate_presigned_url(
        "get_object",
        Params={
            "Bucket": BUCKET,
            "Key": object_key,
            "ResponseContentDisposition": f'attachment; filename="{meta["filename"]}"',
        },
        ExpiresIn=900,   # 15-minute download URL
    )

    return DownloadResponse(download_url=url, expires_in=900)


# ─── Client-side upload flow (httpx) ─────────────────────────────────────────

"""
import httpx, pathlib

async def upload_file(file_path: str):
    path = pathlib.Path(file_path)
    size = path.stat().st_size

    async with httpx.AsyncClient() as client:
        # 1. Initiate — get pre-signed URL
        r = await client.post("http://localhost:8000/upload/initiate", json={
            "filename": path.name,
            "content_type": "image/jpeg",
            "size_bytes": size,
        })
        data = r.json()
        object_key = data["object_key"]

        # 2. Upload DIRECTLY to S3 (server never receives bytes)
        with open(file_path, "rb") as f:
            files = {"file": (path.name, f, "image/jpeg")}
            r = await httpx.AsyncClient().post(
                data["upload_url"],
                data=data["upload_fields"],
                files=files,
            )
        assert r.status_code == 204, f"Upload failed: {r.text}"

        # 3. Confirm with server
        r = await client.post(f"http://localhost:8000/upload/confirm/{object_key}")
        print(r.json())

        # 4. Get download URL
        r = await client.get(f"http://localhost:8000/download/{object_key}")
        print("Download URL:", r.json()["download_url"])
"""
```

---

## Exercise 4: S3 Lifecycle Policy Automation

Programmatically set lifecycle rules to auto-tier objects:

```python
"""
Goal: Configure an S3 bucket to automatically:
  1. Move objects with prefix 'logs/' to Standard-IA after 30 days
  2. Move them to Glacier after 90 days
  3. Expire (delete) them after 365 days
  4. Clean up incomplete multipart uploads after 7 days
"""

import boto3

s3 = boto3.client("s3")
BUCKET = "my-data-bucket"


def configure_lifecycle(bucket: str) -> None:
    lifecycle_config = {
        "Rules": [
            {
                "ID": "logs-tiering",
                "Status": "Enabled",
                "Filter": {"Prefix": "logs/"},
                "Transitions": [
                    {
                        "Days": 30,
                        "StorageClass": "STANDARD_IA",
                    },
                    {
                        "Days": 90,
                        "StorageClass": "GLACIER",
                    },
                ],
                "Expiration": {
                    "Days": 365,
                },
            },
            {
                "ID": "abort-incomplete-multipart",
                "Status": "Enabled",
                "Filter": {"Prefix": ""},   # applies to all keys
                "AbortIncompleteMultipartUpload": {
                    "DaysAfterInitiation": 7,
                },
            },
            {
                "ID": "expire-tmp-uploads",
                "Status": "Enabled",
                "Filter": {"Prefix": "tmp/"},
                "Expiration": {"Days": 1},
            },
        ]
    }

    s3.put_bucket_lifecycle_configuration(
        Bucket=bucket,
        LifecycleConfiguration=lifecycle_config,
    )
    print(f"Lifecycle rules configured for {bucket}")


def print_lifecycle(bucket: str) -> None:
    response = s3.get_bucket_lifecycle_configuration(Bucket=bucket)
    for rule in response["Rules"]:
        print(f"\nRule: {rule['ID']} [{rule['Status']}]")
        print(f"  Filter: {rule.get('Filter', 'all')}")
        for t in rule.get("Transitions", []):
            print(f"  After {t['Days']} days → {t['StorageClass']}")
        if "Expiration" in rule:
            print(f"  Expire after {rule['Expiration']['Days']} days")
        if "AbortIncompleteMultipartUpload" in rule:
            print(f"  Abort incomplete multipart after "
                  f"{rule['AbortIncompleteMultipartUpload']['DaysAfterInitiation']} days")


configure_lifecycle(BUCKET)
print_lifecycle(BUCKET)
```

---

## Exercise 5: Content Hash Deduplication Check

Before uploading, check if identical content already exists — save bandwidth and API calls:

```python
"""
Goal: Implement client-side deduplication by:
  1. Computing SHA256 of the local file
  2. Checking if S3 already has an object at the content-addressed key
  3. Only uploading if not already present (skip duplicate uploads)
"""

import hashlib
import json
import boto3
from pathlib import Path


s3 = boto3.client("s3")
BUCKET = "my-cas-bucket"


def sha256_file(path: str) -> str:
    """Stream-hash a file — handles files larger than RAM."""
    h = hashlib.sha256()
    with open(path, "rb") as f:
        while chunk := f.read(8 * 1024 * 1024):
            h.update(chunk)
    return h.hexdigest()


def object_exists(bucket: str, key: str) -> bool:
    """HEAD request — cheap existence check, no data transfer."""
    try:
        s3.head_object(Bucket=bucket, Key=key)
        return True
    except s3.exceptions.ClientError as e:
        if e.response["Error"]["Code"] == "404":
            return False
        raise


def upload_if_not_exists(
    file_path: str,
    content_type: str = "application/octet-stream",
) -> tuple[str, bool]:
    """
    Upload file using content addressing.
    Returns (object_key, was_uploaded).
    was_uploaded=False means the file was already stored — bandwidth saved.
    """
    digest = sha256_file(file_path)
    key = f"objects/{digest[:2]}/{digest[2:]}"

    if object_exists(BUCKET, key):
        print(f"[SKIP] {Path(file_path).name} already stored as {digest[:12]}...")
        return key, False

    file_size = Path(file_path).stat().st_size

    if file_size > 100 * 1024 * 1024:
        # Large file — use multipart upload (simplified here)
        s3.upload_file(file_path, BUCKET, key, ExtraArgs={"ContentType": content_type})
    else:
        with open(file_path, "rb") as f:
            s3.put_object(Bucket=BUCKET, Key=key, Body=f, ContentType=content_type)

    print(f"[UPLOAD] {Path(file_path).name} → {digest[:12]}... ({file_size:,} bytes)")
    return key, True


# ─── Demo: Upload the same file twice ────────────────────────────────────────

import os

test_file = "/tmp/dedup_test.bin"
with open(test_file, "wb") as f:
    f.write(os.urandom(1024 * 1024))  # 1 MB

key, uploaded1 = upload_if_not_exists(test_file)
key, uploaded2 = upload_if_not_exists(test_file)   # Same file — should be skipped

print(f"\nFirst upload:  {'uploaded' if uploaded1 else 'skipped'}")
print(f"Second upload: {'uploaded' if uploaded2 else 'skipped'}")
print(f"Bandwidth saved on second attempt: 1 MB")
```
