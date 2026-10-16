# OFFLINE_SYNC_REPORT.md

> Phase 9. The student app is **read-heavy** (students/parents consume records; writes are performed by teachers/admins in `admin_app`). Offline support is therefore centered on **read-through caching**, with a write-queue scaffold retained for the one parent write path.

## 1. What is implemented

### Read-through cache (active)
`OfflineSyncService.cacheResponse` / `getCachedResponse` (SharedPreferences) is now wired into the real repositories. Every successful GET stores the raw envelope; on a **network/timeout** error the repository transparently serves the last cached envelope.

| Data | Cache key | Repo |
|---|---|---|
| Student profile | `student.profile` | `ApiStudentRepository.getStudent` |
| Memorization sessions | `student.memorization` | `getMemorizationRecords` |
| Revision sessions | `student.revisions` | `getRevisions` |
| Homework | `student.homework` | `getHomework` |
| Assessments | `student.assessments` | `getAssessments` |
| Attendance | `student.attendance` | `getAttendance` |
| Notifications | `notifications` | `getNotifications` (student & parent) |
| Parent children | `parent.children` | `ApiParentRepository` |
| Parent evaluations | `parent.evaluations` | `getEvaluations` |

Fallback is **scoped to connectivity errors only** (`ApiExceptionType.network` / `timeout`); real 4xx/5xx still surface so the UI shows the correct error, not stale data.

### Write queue (scaffold, retained)
`OfflineSyncService.enqueueRequest` / `clearQueue` remain available for deferred writes. The only student-app write that mutates server state is **parent evaluation submission**, which is currently gated behind missing UI (session picker — see `MISSING_UI_REPORT.md`). When that UI lands, the submit path should enqueue on network failure and flush on reconnect.

## 2. Verification matrix (behavior by design)
| Scenario | Behavior |
|---|---|
| **Internet OFF**, screen previously loaded | Cached envelope is served → data still visible. |
| **Internet OFF**, never loaded before | Connectivity `ErrorState` with retry (no fake data). |
| **Internet ON** | Fresh fetch; cache refreshed. |
| **App closed & reopened**, valid token | Splash calls `/auth/me`; on success restores session, else routes to welcome. Cached lists remain until refreshed. |
| **Token expired (401)** | Interceptor deletes token + clears `currentUserProvider` → GoRouter redirects to login. No data loss (reads are idempotent). |

## 3. Honest gaps / follow-ups
- **Connectivity awareness is reactive, not proactive** — there is no `connectivity_plus` listener; the app discovers offline state on the failed request. Adding a connectivity banner is a recommended enhancement.
- **No cache expiry / TTL** — cached reads have no staleness bound. Acceptable for a fallback, but a timestamp-based TTL would prevent showing very old data offline.
- **Write queue is not yet exercised** because the only write path (parent evaluations) is UI-gated. No offline conflict-resolution is needed for the read paths.
- **Cache is unencrypted** (SharedPreferences). Tokens are in `flutter_secure_storage`; cached record data is not sensitive credentials but consider encrypting if policy requires.

**Net:** no data loss for the read flows across OFF/ON/close/reopen; writes are limited and UI-gated.
