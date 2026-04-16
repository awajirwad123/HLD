# File Storage System — Interview Simulator

## Scenario 1: "Design Dropbox (File Storage and Sync)"

**Prompt:** "Design a cloud file storage system that allows users to store files, sync across devices, and share with others. Support 500M users, up to 5GB files."

---

**Clarifying questions to ask:**

1. Is this for personal files (Dropbox-style) or collaborative editing (Google Docs)?
2. Do we need real-time collaboration, or just sync?
3. Versioning: how many versions to keep?
4. What sharing model? (Link sharing, per-user permissions, team folders?)
5. Mobile clients too, or desktop only?

---

**Model answer outline:**

**Capacity estimates:**
- 500M users × 10GB average = 5 exabytes total storage
- 1% active/day = 5M users uploading
- 5M users × 1 file/day × 1MB avg = 5TB uploads/day
- Downloads: 10× uploads = 50TB/day
- S3 cost: ~$0.023/GB = $115M/yr storage alone (use tiered storage to reduce)

**Component walkthrough:**

```
Desktop/Mobile Client
  ├── Sync Agent (monitors filesystem, computing diffs)
  └── Chunker (4MB fixed chunks, SHA-256)

APIs:
  ├── Metadata API   → PostgreSQL
  ├── Upload API     → Block Store (S3)
  └── Sync API       → Redis Pub/Sub → WebSocket/long-poll

Storage:
  ├── Metadata DB (PostgreSQL): files, folders, versions, shares
  ├── Block Store (S3): chunk data, keyed by SHA-256
  └── Chunk Index (DB): content_hash → S3 key, ref_count
```

**Upload flow:**
1. Client splits file into 4MB chunks, computes SHA-256 per chunk
2. `POST /upload/init` with all hashes → server returns missing hashes
3. `PUT /upload/chunk/{hash}` for each missing chunk (parallel, 5 at once)
4. `POST /upload/finalize` → creates `file_version` record

**Sync flow:**
1. Upload completes → Metadata DB updated → event published
2. Sync server fans out to connected devices via WebSocket/long-poll
3. Devices download only changed chunks

**Sharing:**
- Per-user: DB `shares` record with `permission = view|edit`
- Link sharing: `share_token` (UUID) → share page
- Folder sharing: permission inheritance via ancestor traversal + ACL cache

**Versioning:**
- Keep 30 versions per file
- Background GC decrements `ref_count` for expired versions; deletes S3 objects when count=0

**Trade-offs to mention proactively:**
- Cross-user dedup saves storage but creates privacy implications (convergent encryption attack)
- Delta sync is great for text files, less so for binary files (video, images rarely have shared chunks across versions)
- Conflict resolution UX: Dropbox "keeps both" is simple but leaves users with duplicate confusing files

---

## Scenario 2: "Design Google Drive Team Folder with Permissions"

**Prompt:** "Engineers at a company use Google Drive. Design the permission system so team folders work: when someone shares a folder with a 10-person team, all members see the files, and new files added to the folder are immediately accessible to all team members."

---

**Requirements:**
- Folder shares inherit to all contents
- Adding a new file to a shared folder = immediately accessible to all grantees
- New team member added = immediately sees all files in shared folders
- Check if user X can read file Y must be < 10ms

---

**Data model:**

```sql
CREATE TABLE shares (
    resource_id   UUID,      -- file or folder
    principal     TEXT,      -- user:42, team:backend, public
    permission    TEXT,
    PRIMARY KEY (resource_id, principal)
);

CREATE TABLE team_members (
    team_id    BIGINT,
    user_id    BIGINT,
    PRIMARY KEY (team_id, user_id)
);
```

**Access check algorithm:**

```python
async def can_access(user_id: int, file_id: uuid, required: str) -> bool:
    # 1. Owner always has access
    if await is_owner(user_id, file_id): return True

    # 2. Direct share check
    if await has_direct_share(user_id, file_id, required): return True

    # 3. Team share check
    user_teams = await get_user_teams(user_id)
    for team_id in user_teams:
        if await has_direct_share(f"team:{team_id}", file_id, required): return True

    # 4. Ancestor folder share (walk up tree)
    parent = await get_parent(file_id)
    while parent:
        if await can_access(user_id, parent.id, "view"): return True
        parent = await get_parent(parent.id)

    return False
```

**Performance concern:** Recursive ancestor walk = N DB queries per check.

**Optimization: Permission cache**

```redis
Key: perm:{user_id}:{file_id}
Value: view|edit|admin|none
TTL: 5 minutes

# Invalidate on: share change, team membership change, file move (parent changes)
DEL perm:{user_id}:{file_id}
```

**When new team member is added:**
- Fan-out: add their user_id to team_members
- Invalidate all permission cache keys for that user? Too many.
- Better: the permission check naturally re-evaluates the team membership on next access when cache expires. Team membership changes take up to 5 minutes (cache TTL) to reflect. For immediate effect: no change — cache miss on next request will compute correctly.

**When file is moved to a new folder:**
- Permissions change (inherits from new parent)
- Invalidate cache for all users who had this file in their cache: impractical to enumerate
- Better approach: include parent_id in the cache key: `perm:{user_id}:{file_id}:{parent_id}`
  - When file moves, parent_id changes → old cache keys become irrelevant (not hit again) and expire naturally

---

## Scenario 3: "Debugging — Users Report Their Files Are Missing After Sync"

**Prompt:** "Users are reporting that after syncing, some files have disappeared from their devices. How do you investigate and fix this?"

---

**Investigation framework:**

**Step 1: Scope the problem**
- How many users? (1 report = probably a user error; 100 reports = system bug)
- Which platform? (Windows, Mac, iOS, Android — could be a client-side bug)
- What time did it start? (Correlates with deploys)
- What type of files? (All files? Only files in certain folders? Only shared files?)

**Step 2: Check the audit log**

```sql
-- Audit: was this file deleted?
SELECT event_type, user_id, metadata, created_at
FROM audit_log
WHERE file_id = $1
ORDER BY created_at DESC;
```

If `event_type = 'file_deleted'` with a `user_id` different from the owner: someone with edit access deleted it (intentional if legitimate, bug if not).

If no delete event: the file metadata still exists. The sync client may have a bug.

**Step 3: Check sync events**

```sql
SELECT device_id, event_type, payload, created_at
FROM sync_events
WHERE user_id = $1
  AND created_at > NOW() - INTERVAL '24 hours'
ORDER BY created_at;
```

Look for: `file_deleted` events being incorrectly sent to devices.

**Step 4: Check for sync client bug — "phantom delete"**

Common sync bug: Client A renames a file → generates local `delete(old_name)` + `create(new_name)` events → if the sequence is applied on Device B as just `delete(old_name)` without `create(new_name)` (network partition mid-sequence): file appears deleted on Device B.

Fix: on the server, rename = one atomic transaction; propagate as a single `file_renamed` event, not as separate delete+create.

**Step 5: Check for conflict resolution bug**

If two devices both modify a file and the server applies conflict resolution as "last-write-wins" deleting one version — check whether deleted version was the one the user expected to keep.

Check: `file_versions` table. Is there a v2 record that was created then immediately superseded? Restore the missing version.

**Step 6: Check deletion propagation to shared folders**

If user B has a shared folder from user A, and user A deleted the folder (moved to trash) — all of user B's shared files would appear to disappear from their connected device. This is correct behavior, but confusing to user B.

Check: `shares` table for the affected files. Were the shares revoked recently?

**Root causes (priority):**
1. Sync client bug that interprets `rename` as `delete+create` with a gap
2. Incorrect conflict resolution deleting a version the user expected to keep
3. Legitimate deletion by a co-editor (user wasn't aware of the share)
4. Trashing of a shared parent folder by owner

**Recovery:** Re-implement "undo" as a first-class operation. Every destructive sync action (delete, overwrite) creates a recovery snapshot. Expose "undo last 10 changes" in the client and web UI.
