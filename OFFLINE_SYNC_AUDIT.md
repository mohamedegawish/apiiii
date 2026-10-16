# OFFLINE_SYNC_AUDIT.md

> Offline behavior across `admin_app` (write-queue, offline-first) and `student_app` (read-cache).

## admin_app — offline-first write queue (mature)
Stack: `data/local/{connectivity_service, offline_store, sync_queue, sync_service}.dart` + `SyncingRepository`.

| Property | Implementation | Verdict |
|---|---|---|
| Queue durability | `SyncQueue` persists ops via `OfflineStore` (SharedPreferences); reloaded on boot | ✅ survives restart |
| Ordering | FIFO over `queue.operations`; stops at first network/server error | ✅ |
| Trigger | `SyncService` listens to `connectivity.onStatusChange`; drains on reconnect; manual `sync()` | ✅ background + manual |
| Retry | `retryCount` is retained for diagnostics; network/server/timeout operations remain queued until delivery | ✅ no silent data loss |
| Conflict resolution | success → remove; DELETE 404 → success; validation/permission/conflict → terminal with visible error; network/server → keep | ✅ |
| Optimistic local ids | `generateLocalId()` `local-…`; PUT/DELETE use `remoteId` | ✅ |
| Re-entrancy | `_draining` guard prevents concurrent drains | ✅ |

### 🟢 Data-loss scenario (Finding A, offline facet) — FIXED
Previously, offline attendance ops enqueued `teacher_id/circle_id/attendance_date` with **no `circle_session_id`** → 422 on reconnect → classified terminal → **dropped (data loss)**. Fixed: the attendance flow is now session-centric, so each offline op enqueues `{circle_session_id, student_id, status,…}` which satisfies `StoreAttendanceRequest` on sync. The terminal-drop path is no longer reachable for a validly-marked roster. **Residual:** a still-recommended enhancement is a user-visible "dropped op" (dead-letter) view for the genuinely-terminal cases (e.g., a session locked after the mark was queued).

### Other notes
- Memorization/revision/assessments/homework offline writes still respect server validation (for example suspended students). Such operations are blocked with a visible reason and can be retried after correction; they are not silently deleted.
- Learning-record writes from the teacher session workspace now use the same durable queue as attendance/homework.
- Offline create followed by edit is coalesced into one POST, so the latest local draft is uploaded.
- Statistics responses and searchable-picker results are cached after a successful online load and can be reused offline.
- Circle and circle-schedule creation now queue offline; server-generated circle IDs are remapped into dependent queued schedule operations on reconnect.
- Other administrative master-data writes (for example finance and user provisioning) remain online-only because their server-generated relationships and accounting rules require an online transaction.

## student_app — read-through cache (adequate for a read-heavy app)
`OfflineSyncService.cacheResponse/getCachedResponse` wired into `ApiStudentRepository`/`ApiParentRepository`; every successful GET is cached and re-served **only** on `network`/`timeout` errors (4xx/5xx still surface). Covers profile, memorization, revision, homework, assessments, attendance, notifications, parent children/evaluations.
- Gaps: no TTL/expiry; reactive (not proactive) connectivity; no encryption of cached payloads. Documented in the prior `OFFLINE_SYNC_REPORT.md`.

## Verification matrix
| Scenario | admin | student |
|---|---|---|
| Offline write then reconnect | ✅ field operations and teacher learning records queue and drain | N/A (read-only) |
| Offline read (previously loaded) | cached lists in-app | ✅ cache served |
| App killed with pending ops | ✅ queue persisted | cache persisted |
| Server rejects (422/403) | op dropped with error | error surfaced |
| No data loss guarantee | ✅ network/server failures remain queued; logical rejects are visible | ✅ |
