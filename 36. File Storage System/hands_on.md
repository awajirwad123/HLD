# File Storage System — Hands-On Exercises

## Exercise 1: Content-Addressable Chunk Store

Simulates the core storage engine: split files into chunks, store by SHA-256 hash, deduplicate.

```python
import hashlib
import io
import os
from dataclasses import dataclass, field
from pathlib import Path

CHUNK_SIZE = 1024 * 1024  # 1MB for demo (4MB in production)


@dataclass
class Chunk:
    content_hash: str   # SHA-256 hex digest
    size_bytes: int
    storage_key: str    # Simulated S3 object key


@dataclass
class FileVersion:
    version_id: str
    file_id: str
    chunk_hashes: list[str]
    size_bytes: int


class ChunkStore:
    """
    Content-addressable block store.
    In production: storage_key maps to S3 object.
    Here: stored in a dict keyed by hash.
    """

    def __init__(self):
        self._chunks: dict[str, bytes] = {}    # hash → raw bytes
        self._chunk_meta: dict[str, Chunk] = {}  # hash → Chunk metadata
        self.upload_count = 0
        self.dedup_count = 0

    def _hash(self, data: bytes) -> str:
        return hashlib.sha256(data).hexdigest()

    def store_chunk(self, data: bytes) -> str:
        """Store a chunk by content hash. Returns the hash."""
        h = self._hash(data)
        if h not in self._chunks:
            self._chunks[h] = data
            self._chunk_meta[h] = Chunk(
                content_hash=h,
                size_bytes=len(data),
                storage_key=f"blocks/{h[:2]}/{h[2:4]}/{h}",
            )
            self.upload_count += 1
        else:
            self.dedup_count += 1
        return h

    def get_chunk(self, content_hash: str) -> bytes | None:
        return self._chunks.get(content_hash)

    def check_exists(self, hashes: list[str]) -> list[str]:
        """Returns list of hashes NOT yet stored (client should upload these)."""
        return [h for h in hashes if h not in self._chunks]

    @property
    def stats(self) -> dict:
        total_bytes = sum(c.size_bytes for c in self._chunk_meta.values())
        return {
            "unique_chunks": len(self._chunks),
            "total_stored_bytes": total_bytes,
            "uploads": self.upload_count,
            "dedup_hits": self.dedup_count,
        }


class FileStore:
    """
    Manages files and their version history using the ChunkStore.
    """

    def __init__(self, chunk_store: ChunkStore):
        self.chunks = chunk_store
        self._versions: dict[str, list[FileVersion]] = {}  # file_id → list[FileVersion]
        self._version_counter: dict[str, int] = {}
        self._raw_upload_bytes = 0

    def _split_into_chunks(self, data: bytes) -> list[bytes]:
        chunks = []
        stream = io.BytesIO(data)
        while True:
            chunk = stream.read(CHUNK_SIZE)
            if not chunk:
                break
            chunks.append(chunk)
        return chunks or [b""]  # At least one chunk for empty file

    def upload_file(self, file_id: str, data: bytes) -> FileVersion:
        """
        Upload a file (or new version of a file).
        Returns the created FileVersion.
        """
        raw_chunks = self._split_into_chunks(data)
        chunk_hashes = []

        self._raw_upload_bytes += len(data)

        # Step 1: Compute hashes
        hashes = [hashlib.sha256(c).hexdigest() for c in raw_chunks]

        # Step 2: Check which chunks are already stored
        missing = set(self.chunks.check_exists(hashes))

        # Step 3: Upload only missing chunks
        for chunk_data, h in zip(raw_chunks, hashes):
            self.chunks.store_chunk(chunk_data)  # No-op if already exists

        # Step 4: Create version record
        version_num = self._version_counter.get(file_id, 0) + 1
        self._version_counter[file_id] = version_num

        version = FileVersion(
            version_id=f"{file_id}_v{version_num}",
            file_id=file_id,
            chunk_hashes=hashes,
            size_bytes=len(data),
        )

        if file_id not in self._versions:
            self._versions[file_id] = []
        self._versions[file_id].append(version)

        new_chunks = len(missing)
        total_chunks = len(hashes)
        print(f"  Upload {file_id} v{version_num}: {len(data):,} bytes, "
              f"{total_chunks} chunks → uploaded {new_chunks}, deduped {total_chunks - new_chunks}")

        return version

    def download_file(self, file_id: str, version: int = -1) -> bytes:
        """Download a specific version of a file (default: latest)."""
        versions = self._versions.get(file_id, [])
        if not versions:
            raise FileNotFoundError(f"File {file_id} not found")

        v = versions[version]  # -1 = last
        parts = [self.chunks.get_chunk(h) for h in v.chunk_hashes]
        return b"".join(p for p in parts if p is not None)

    def list_versions(self, file_id: str) -> list[FileVersion]:
        return self._versions.get(file_id, [])

    def delta_chunks(self, file_id: str, new_data: bytes) -> tuple[int, int]:
        """Returns (chunks_to_upload, total_chunks) for a proposed new version."""
        current_versions = self._versions.get(file_id, [])
        if not current_versions:
            return len(self._split_into_chunks(new_data)), len(self._split_into_chunks(new_data))

        new_chunks = self._split_into_chunks(new_data)
        new_hashes = set(hashlib.sha256(c).hexdigest() for c in new_chunks)
        missing = self.chunks.check_exists(list(new_hashes))
        return len(missing), len(new_chunks)

    def storage_efficiency(self) -> dict:
        stored = self.chunks.stats["total_stored_bytes"]
        raw = self._raw_upload_bytes
        savings = (1 - stored / raw) * 100 if raw > 0 else 0
        return {
            "raw_bytes_received": raw,
            "actual_stored_bytes": stored,
            "dedup_savings_pct": round(savings, 1),
            **self.chunks.stats,
        }


def demo():
    store = FileStore(ChunkStore())

    # 1MB = realistic for demo
    ONE_MB = 1024 * 1024

    print("=== File Storage Deduplication Demo ===\n")

    # User A uploads report.pdf (3MB)
    file_a_data = b"HEADER" + b"A" * (ONE_MB * 3 - 10) + b"FOOTER"
    print("[User A] Uploading report.pdf (3MB)")
    store.upload_file("user_a:report.pdf", file_a_data)

    # User B uploads the SAME file (should deduplicate entirely)
    print("\n[User B] Uploading identical report.pdf (3MB)")
    store.upload_file("user_b:report.pdf", file_a_data)

    # User A makes a small edit (only last chunk changes)
    print("\n[User A] Editing report.pdf (small change at end)")
    edited_data = file_a_data[:-100] + b"EDITED" * 16 + b"NEWFOOTER"
    store.upload_file("user_a:report.pdf", edited_data)

    # User C uploads a different file
    print("\n[User C] Uploading notes.txt (5MB with some shared content)")
    different_data = b"HEADER" + b"A" * ONE_MB + b"C" * (ONE_MB * 4) + b"END"
    store.upload_file("user_c:notes.txt", different_data)

    # Download check
    print("\n=== Verifying downloads ===")
    downloaded = store.download_file("user_a:report.pdf")
    print(f"[Check] report.pdf v1 size: {len(file_a_data):,} == {len(downloaded):,}? "
          f"{'✓' if downloaded == file_a_data else '✗'}")

    downloaded_v2 = store.download_file("user_a:report.pdf", version=-1)
    print(f"[Check] report.pdf v2 size: {len(edited_data):,} == {len(downloaded_v2):,}? "
          f"{'✓' if downloaded_v2 == edited_data else '✗'}")

    # Storage efficiency stats
    print("\n=== Storage Efficiency ===")
    stats = store.storage_efficiency()
    for k, v in stats.items():
        if "bytes" in k:
            print(f"  {k}: {v:,} bytes ({v/ONE_MB:.1f} MB)")
        else:
            print(f"  {k}: {v}")


if __name__ == "__main__":
    demo()
```

---

## Exercise 2: File Sync State Machine

Simulates client-side sync: tracking local vs remote states, resolving conflicts.

```python
from dataclasses import dataclass
from enum import Enum


class SyncStatus(str, Enum):
    SYNCED     = "synced"
    LOCAL_ONLY = "local_only"
    REMOTE_NEW = "remote_new"
    CONFLICT   = "conflict"
    DELETED    = "deleted"


@dataclass
class FileSyncState:
    path: str
    local_hash: str | None      # Hash of local file content
    remote_hash: str | None     # Hash of remote file content
    base_hash: str | None       # Hash at last known sync point

    @property
    def status(self) -> SyncStatus:
        local_changed  = self.local_hash  != self.base_hash
        remote_changed = self.remote_hash != self.base_hash

        if self.local_hash is None and self.remote_hash is None:
            return SyncStatus.DELETED

        if self.remote_hash is None:
            return SyncStatus.LOCAL_ONLY

        if self.local_hash is None:
            return SyncStatus.REMOTE_NEW

        if self.local_hash == self.remote_hash:
            return SyncStatus.SYNCED

        if local_changed and remote_changed:
            return SyncStatus.CONFLICT

        if remote_changed:
            return SyncStatus.REMOTE_NEW

        if local_changed:
            return SyncStatus.LOCAL_ONLY

        return SyncStatus.SYNCED


def resolve_sync_state(state: FileSyncState) -> str:
    """Returns the action to take for this file."""
    match state.status:
        case SyncStatus.SYNCED:
            return "no action needed"
        case SyncStatus.LOCAL_ONLY:
            return f"→ upload to remote, update base_hash to {state.local_hash[:8]}"
        case SyncStatus.REMOTE_NEW:
            return f"→ download from remote, update base_hash to {state.remote_hash[:8]}"
        case SyncStatus.CONFLICT:
            return (f"→ CONFLICT: keep both copies\n"
                    f"     local  saved as: {state.path}.local_conflict\n"
                    f"     remote saved as: {state.path}.remote_conflict\n"
                    f"     user must resolve manually")
        case SyncStatus.DELETED:
            return "→ delete local copy from UI (cleaned up)"
        case _:
            return "unknown state"


def demo():
    print("=== File Sync State Demo ===\n")

    scenarios = [
        FileSyncState("report.pdf",      local_hash="abc", remote_hash="abc", base_hash="abc"),
        FileSyncState("notes.txt",       local_hash="xyz", remote_hash="abc", base_hash="abc"),
        FileSyncState("presentation.pptx", local_hash="abc", remote_hash="new", base_hash="abc"),
        FileSyncState("shared_doc.docx", local_hash="v2",  remote_hash="v3",  base_hash="v1"),
        FileSyncState("log.txt",         local_hash=None,  remote_hash="def", base_hash="def"),
    ]

    for state in scenarios:
        action = resolve_sync_state(state)
        print(f"  {state.path}")
        print(f"    status: {state.status.value}")
        print(f"    action: {action}\n")


if __name__ == "__main__":
    demo()
```

**Expected output:**
```
  report.pdf
    status: synced
    action: no action needed

  notes.txt
    status: local_only
    action: → upload to remote, update base_hash to xyz

  presentation.pptx
    status: remote_new
    action: → download from remote, update base_hash to new

  shared_doc.docx
    status: conflict
    action: → CONFLICT: keep both copies
                 local  saved as: shared_doc.docx.local_conflict
                 remote saved as: shared_doc.docx.remote_conflict
                 user must resolve manually

  log.txt
    status: deleted
    action: → delete local copy from UI
```
