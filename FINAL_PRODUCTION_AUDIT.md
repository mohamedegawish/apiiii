# Final Production Audit

> Full-system audit of `api` (Laravel), `admin_app` (Flutter — admin + teacher), and `student_app` (Flutter — student + parent), performed by four independent research passes reading actual source code (not prior markdown reports). Each finding below cites the file/line evidence the reviewing pass actually read. Read-only audit — no fixes were applied here except the teacher-lock permission change, which is scoped and documented separately in [TEACHER_LOCK_AUDIT.md](TEACHER_LOCK_AUDIT.md).

## Severity counts

| Severity | Count |
|---|---|
| CRITICAL | 6 |
| HIGH | 8 |
| MEDIUM | 9 |
| LOW | 9 |

## Top priority — fix before anything else ships

1. **`GET /guardians` leaks every family's name/phone/email to any authenticated user** — no role restriction, no tenant scope. (Security, CRITICAL)
2. **Home screens in `student_app` show hardcoded fake financial/personal data as if real** — a fabricated "500 ﷼ payment due," a fabricated evaluation notice for a named fictional child, a nonexistent achievement badge, on every parent's/student's real account. (Parent/Student Workflow, CRITICAL)
3. **Offline-queued attendance is silently lost** when a 409 (session locked/completed mid-flight) is misclassified as retryable, retried to exhaustion, then dropped with no durable trace. (Offline Sync, CRITICAL)
4. **Core memorization statistics are computed from tables the live teacher workflow no longer writes to** — `StatisticsCalculatorService` reads legacy `memorization_sessions`/`revision_sessions`; the shipped teacher workflow writes `student_learning_records`. Progress stats silently go stale. (Database/Teacher Workflow, CRITICAL)
5. **Deleting and re-adding a learning record of the same type crashes with an unhandled 500** (soft-delete + unique constraint, no restore-on-conflict, unlike the identical pattern already correctly handled for attendance/parent-evaluations). (Database, CRITICAL)
6. **Save failures inside the teacher's guided workflow (Attendance/Evaluation steps) are silently swallowed** — no error is ever shown to the teacher; the UI just stops advancing. (Teacher Workflow, CRITICAL)

---

## 1. Security

### CRITICAL — `GET /guardians` is an unrestricted IDOR / PII leak
- `app/Features/Guardians/Policies/GuardianPolicy.php:12-15` — `viewAny()` returns `$user instanceof User`, true for every role including student and guardian.
- `app/Features/Guardians/Queries/GuardianQuery.php:17-22` — `applyTenantScope()` is a no-op for any non-super-admin/non-center-owner role (no `whereIn`/`1=0` fallback, unlike sibling Query classes).
- `routes/api/v1/guardians.php:9-10` — `index` has no role middleware beyond `auth:sanctum, active`.
- `GuardianResource` returns `name`/`phone`/`email` per row.
- **Exploit**: any logged-in parent or student calls `GET /api/v1/guardians` and receives a full, unfiltered, paginated dump of every other family's contact info. Not mitigated by "no client UI calls it" — it's a live, directly-callable API endpoint.
- **Fix**: make `GuardianQuery::applyTenantScope` fail-closed for non-staff roles; restrict `GuardianPolicy::viewAny` to staff.

### HIGH — `GET /circles` and `GET /teachers` are unscoped for guardian/student roles
- `app/Features/Circles/Queries/CircleQuery.php:18-29`, `app/Features/Teachers/Queries/TeacherQuery.php:19-30` — only add a scoping `where` when the caller is a teacher; no `else` branch, so guardian/student requests fall through unscoped (contrast `HomeworkQuery`/`AssessmentQuery`/`StudentQuery`, which use `accessibleCircleIds()`/`accessibleStudentIds()` and fail closed with `whereRaw('1 = 0')`).
- `CirclePolicy::viewAny`/`TeacherPolicy::viewAny` both just `return $user instanceof User;`.
- **Exploit**: a student or guardian can list every circle and every teacher in the system, not just their own.
- **Fix**: route both through the same fail-closed `accessibleCircleIds()`/`accessibleStudentIds()` helpers already used elsewhere.

### MEDIUM — Idempotency-Key protection is inconsistently applied
- `platform.php` applies `idempotency` middleware to session transitions, parent evaluations, student lifecycle, enrollments, users. `attendance.php`, `homework.php`, `circles.php`, `assessments.php`, `memorization-sessions.php`, `revision-sessions.php` use `throttle:sensitive-write` only.
- `CreateLearningRecordAction` and `CreateHomeworkAction`/`BulkCreateHomeworkAction` have no duplicate-request guard.
- **Failure scenario**: a retried timed-out `POST /learning-records` or `POST /homework` after a genuine server-side success creates a second record.
- **Fix**: add `idempotency` middleware to these routes for parity. (Separately: neither Flutter app currently sends an `Idempotency-Key` header at all — see Infrastructure §11.)

### LOW — Architecture note: single-tenant, not multi-tenant
- `2026_07_25_060326_remove_center_id_from_tables.php` drops `center_id` from `students`/`teachers`/`guardians`/`circles` and drops the `centers` table entirely; `Center` model relations are now dead code. Every policy checking `hasRole($user, Role::CenterOwner)` without an ownership comparison is therefore correct for the *current* single-tenant-per-deployment schema, not a cross-tenant IDOR. **If multi-center-per-deployment is still an intended future state, this migration is itself a CRITICAL regression and needs revisiting; if the single-tenant pivot is deliberate, only the dead `Center` relations need cleanup.** This changes the interpretation of "tenant isolation" for this codebase — there is currently one tenant per install by design.

### LOW — Minor validation gap
- `StoreLearningRecordRequest::rules()` validates `surah_number`/`to_surah_number` as `integer, min:1` with no `max:114` (present on `BulkStoreHomeworkRequest`). Not exploitable — `QuranRangeQuery::resolve()` rejects any range that doesn't resolve to a real `Ayah` row — just an inconsistency.

### LOW — `UserPolicy` restricts co-owners from managing each other's created users
- `UserPolicy::canManage` scopes a center_owner to only `created_by === self`. Not permissive, so not a security issue — worth flagging only if multiple center_owner accounts sharing one deployment is an intended scenario.

**What's solid**: `CircleSessionPolicy`/`TransitionCircleSessionAction` enforce the exact transition graph atomically under row locks; every teacher-facing write (attendance, learning-records, homework) independently re-validates ownership below the policy layer via `AcademicSessionGuard`; guardian-to-student ownership on all `Parent/*` endpoints (`accessibleStudent()`, `assertGuardianChild()`, `ParentEvaluationQuery::scopeEvaluations`) is correct and was specifically verified to reject cross-family access; mass-assignment protection (`teacher_id`/`circle_id`/`student_id`/`guardian_id` all `prohibited` on client input, server-derived) is consistent everywhere checked; the one known QueryBuilder-cross-contamination bug class (documented in [TEACHER_EXPERIENCE_REPORT.md](TEACHER_EXPERIENCE_REPORT.md)) was not found to recur elsewhere.

---

## 2. Session Lifecycle (start / complete / lock / cancel)

- **Transition graph is atomic and correctly enforced for every caller**: `CircleSessionStatus::allowedTransitions()` (`Scheduled→InProgress|Cancelled`, `InProgress→Completed|Cancelled`, `Completed→Locked`) is checked inside `DB::transaction()` with `CircleSession::lockForUpdate()`, independent of role — this protects even `super_admin` from an illegal transition, not just teachers.
- **Teacher self-lock**: was `super_admin`/`center_owner`-only prior to this audit cycle; **fixed during this session** (see [TEACHER_LOCK_AUDIT.md](TEACHER_LOCK_AUDIT.md)) so `CircleSessionPolicy::lock()` now shares the same ownership check as `start`/`complete`/`cancel`. 6 new tests confirm: teacher locks own completed session (200), teacher blocked from another teacher's session (403), invalid-status lock attempts rejected (409×3), admin permissions unchanged.
- **HIGH — `admin_app`'s `session_workflow.dart` swallows ALL `startSession()` errors, not just the anticipated race.**
  - `session_workflow.dart:38-46` — blanket `catch (_) {}` around the auto-start call, commented as handling "session may already be in progress (race)," but catches everything indiscriminately.
  - `CircleSessionStatus` has no `Scheduled → Completed` shortcut, and the scheduled-start-time check is only advisory client-side.
  - **Failure scenario**: if `startSession()` fails for any *other* reason (clock/timezone skew, transient network error), the workflow silently continues on the stale `Scheduled` session. Attendance and evaluations both accept `Scheduled` or `InProgress`, so the teacher completes the **entire** attendance + per-student evaluation flow, only to have `completeSession()` reject `Scheduled → Completed` with a raw 409 at the very last step — after all the data entry is done.
  - **Fix**: only swallow a confirmed "already started" condition (check status after the failure), surface anything else immediately.
- **Idempotency-Key infrastructure is inert**: `EnsureIdempotency` middleware no-ops without the header, and neither Flutter app currently sends one on session transitions — see Infrastructure §11.

---

## 3. Teacher Workflow

- **CRITICAL — save failures during Attendance and Evaluation steps are never shown to the teacher.**
  - `workflow_controller.dart` sets `controller.error` on failure in both `saveAttendanceAndContinue()` and `saveCurrentEvaluation()`, but neither `step_attendance.dart` nor `step_evaluation.dart` ever reads it (confirmed via full-file grep, zero matches). Only `step_summary.dart` renders it.
  - **Failure scenario**: any `ApiException` while saving attendance (network drop, the 409-misclassification bug above) or while saving a memorization/revision score (a bad Quran range, an inactive student) is caught, stored, and dropped from view. The "التالي"/"إنهاء التقييم" button just stops spinning — no indication anything failed, no indication the record wasn't saved.
  - **Fix**: render `controller.error!.arabicMessage` in both steps exactly as `StepSummary` already does.
- **RESOLVED — teacher's "قفل الجلسة" (lock) button previously always failed with 403.** The teacher's Today dashboard already had a working lock-button UI (`session_card.dart`, gated only on `session.canLock`, no role check) wired to `POST /sessions/{id}/lock`, but the server rejected every teacher call until the policy fix in [TEACHER_LOCK_AUDIT.md](TEACHER_LOCK_AUDIT.md) landed earlier in this same review cycle. As of that fix, this button now works end-to-end for a teacher's own completed sessions. No further client change is needed; flagged here only so the "was this ever broken" question has a paper trail.
- **HIGH** — `session_workflow.dart`'s blanket start-error swallowing — see Session Lifecycle §2 above (same finding, cross-referenced since its consequence is entirely inside the teacher workflow).
- **What's solid**: every payload shape sent by the workflow (`bulkAttendance`, `learningRecords.create/update`, `assignHomeworkBulk`) matches its server-side `FormRequest` exactly; `flutter analyze` is clean; outside the guided workflow, error handling elsewhere in `admin_app` (`circle_sessions_screen.dart`, `homework_screen.dart`, `assessments_screen.dart`, `today_screen.dart`) consistently catches and displays `ApiException`s.

---

## 4. Parent Workflow

- **CRITICAL — `parent_home_screen.dart` shows fabricated data as if real**, see §Top Priority #2 and the Student Workflow section below (shared finding, both home screens affected).
- **MEDIUM — a parent can submit an evaluation without ever rating "متابعة الواجب" (homework follow-up), and the client silently substitutes their behavior score to satisfy the server's required field.**
  - `evaluation_screen.dart`'s `canSubmit` checks child/session/prayer/behavior but not `_homeworkFollowupRating`.
  - `api_parent_repository.dart:108` — `'homework_followup_score': (evaluation.homeworkFollowup ?? evaluation.behavior).value`.
  - Backend requires it (`StoreParentEvaluationRequest.php:26`, `required|integer|between:1,5`).
  - **Impact**: teachers/reporting see a homework-followup rating the parent never actually gave, with no marker that it's a fallback.
  - **Fix**: add `_homeworkFollowupRating != null` to `canSubmit`.
- **MEDIUM — dead/unused parent surfaces**: `ParentMoreSheet` is fully built but never shown anywhere, and three of its four items route to non-existent/wrong paths (`/parent/assessments` doesn't exist in the router — only `/parent/child/:childId/assessments` does; since `go_router` has no `errorBuilder`, wiring this sheet up as-is would crash on tap). `evaluationsProvider`/`childrenProvider` are fetched but never displayed — there is no "past evaluations" screen despite the full stack (endpoint, repository, model, mapper) already supporting one.
- **LOW — client never validates `childId` against the parent's own children before calling child-scoped endpoints** (unlike `getChild()`, which does). Not exploitable — server-side `ParentService::accessibleStudent`/`assertGuardianChild` fail closed — worst case is a generic error screen instead of a friendly message on a stale/manipulated deep link.
- **LOW — several settings rows are inert `onTap: () {}`** (`parent_profile_screen.dart`: edit profile, notification prefs, help center, privacy policy) — reads as broken rather than intentionally unavailable, unlike the honest `FeatureUnavailableScreen` pattern used elsewhere in the same app.

---

## 5. Student Workflow

- **CRITICAL — `student/home_screen.dart` also carries hardcoded fake content**: `hasNotifications: true` is a fixed literal (the notification-bell red dot is always on regardless of real unread count), and a fixed fake "next class: today at 4pm" is shown combined with the student's real teacher name — indistinguishable from real schedule data.
- **HIGH — a homework item's teacher-note box will fail to lay out and error-render whenever `description` is non-empty** (a normal case): a nested `Expanded` inside a `Column` that receives unbounded height from its `Row`/`Container`/`ListView.separated` ancestry — the classic "RenderFlex... unbounded" Flutter crash. Fix: drop the inner `Expanded`; plain wrapping `Text` is sufficient.
- **HIGH — two parallel, inconsistent data sources for memorization/revision.** `memorization_screen.dart`/`revisions_screen.dart` read the legacy `student/memorization-sessions`/`revision-sessions` endpoints; `sessions_screen.dart` reads the newer unified `student/learning-records`. On the write side, the shipped teacher workflow (`workflow_controller.dart`) writes exclusively to `learning-records`, while a separate, still-routed admin_app screen (`sessions_screen.dart` under `ui/features/`) still writes to the legacy tables. **A student's "سجل الحفظ" tab and "سجلّ الجلسات" tab can show different numbers for the same session**, with no indication which is authoritative — this is the same root cause as the CRITICAL statistics finding in §7 (Database).
- **LOW — several settings rows are inert `onTap: () {}`** (`profile_screen.dart`, same pattern as the parent side).
- **What's solid**: router coverage is complete (every screen has exactly one matching route, both `extra:`-based routes null-handle a missing `extra`); mapper code (`api_mappers.dart`) is consistently null-safe; the honest `FeatureUnavailableScreen` stubs (`behavior_screen.dart`, `prayer_screen.dart`, `certificates_screen.dart`, etc.) correctly match verified backend gaps rather than faking data — which makes the fabricated-content finding above more clearly an anomaly than the app's general pattern.

---

## 6. Offline Sync (`admin_app` — the only client with a write queue)

- **CRITICAL — a 409 Conflict response is misclassified as retryable, causing silent, permanent data loss for a legitimately-queued attendance write.**
  - `api_client.dart`'s `_kindFor` has no case for HTTP 409 → falls into `ApiErrorKind.unknown`. `sync_service.dart`'s `_isTerminal()` only treats `validation`/`forbidden`/`notFound` as terminal — `unknown` is retried.
  - Server-side, exactly this status is what a locked/completed session's attendance guard throws (`ConflictException`, `AttendanceService::lockWritableSession`).
  - **Failure scenario**: a teacher records attendance offline for a session that gets completed+locked by someone else before the device reconnects. On reconnect, the queued write replays, gets 409, is treated as retryable — and because `sync()` breaks its FIFO loop on any non-terminal error, **every other queued operation behind it is also stuck** until this one exhausts its retry count. It is then dropped purely because `retryCount >= maxRetries`, not because the conflict was understood, with only a transient in-memory `_lastError` string as any trace. The attendance record is gone with no durable record anywhere.
  - This confirms the documented "evaluations/homework must stay online-only because the sync service silently drops conflicts" design rationale is correct in spirit — but the *implementation gap* means attendance, which was deliberately kept offline-capable, is not actually protected from the same class of loss.
  - **Fix**: add a `conflict` `ApiErrorKind` for 409, treat it as terminal, and persist dropped-op history somewhere user-visible instead of only an overwritten in-memory string.
- **MEDIUM — the standalone "الواجبات" (homework) tab uses the general `SyncingRepository`, undermining the online-only rationale for homework.**
  - `assignHomeworkBulk` (bulk-assign from Step 3) is correctly a plain online-only `api.post`. But `homework_screen.dart`'s edit/delete actions go through `backend.homework`, which **is** a `SyncingRepository`, and teachers have edit rights there (`Permissions.manageHomework`, `HomeworkScreen(allowCreate:false)` still permits `_edit`/`_openDetail`).
  - **Impact**: an offline homework edit (grade/feedback/due date) queues and later replays with no conflict detection, subject to the same 409-misclassification bug above — this only partially delivers on the stated "homework must be online because of the immediate parent notification" rationale, which really only applies to the bulk-assign creation path, not general edits.
  - **Fix**: either make homework edit/delete online-only too, or explicitly scope the "online-only" design intent to session-time bulk assignment only (and document that distinction, since it currently reads as a blanket rule that isn't actually applied blanket-wide).
- **LOW — `offline_store.dart`'s list-cache methods (`cacheList`/`readList`/`hasCache`) are dead code**, called nowhere — `SyncingRepository.list()`'s offline fallback therefore only ever surfaces optimistic pending-create items, never a real cache of previously-fetched server data, despite the storage layer having a mechanism built for exactly that.

---

## 7. Database

- **CRITICAL — `StatisticsCalculatorService` computes progress metrics from a data source the live teacher workflow no longer writes to.**
  - `recalculateStudent()`/`recalculateTeacher()`/`recalculateCircle()` query `MemorizationSession`/`RevisionSession` (legacy tables). The shipped teacher session workflow writes exclusively to `student_learning_records` via `POST /learning-records`.
  - **Impact**: `memorized_pages`, `memorized_juz`, `completed_surahs`, `average_memorization_grade`, shown on the student dashboard and `/statistics/*`, silently go stale as normal day-to-day grading no longer lands in the tables these stats read from — the product's core "track memorization progress" feature degrades over time with no error, alarm, or symptom other than numbers quietly not moving.
  - **Fix**: point `StatisticsCalculatorService` at `student_learning_records` grouped by `type`, or dual-write, and retire one path. This is the same root inconsistency flagged from the client side in §5.
- **CRITICAL — deleting then re-adding a learning record of the same type crashes with an unhandled 500.**
  - `student_learning_records` has `unique(circle_session_id, student_id, type)` + `softDeletes()`. `DeleteLearningRecordAction` does a plain soft delete. `CreateLearningRecordAction` does a plain `save()` with **no `withTrashed()` check or restore** — unlike the identical situation in `AttendanceService::create()` and `ParentService::storeEvaluation()`, both of which correctly check-then-restore-or-conflict.
  - **Impact**: teacher deletes a mistaken grade, re-enters a corrected one of the same type for the same student/session → raw `QueryException` unique-violation surfaces as a 500 instead of succeeding.
  - **Fix**: mirror the existing `withTrashed()`-then-restore-or-conflict pattern.
- **MEDIUM — `GenerateCircleSessionsAction` conflates "soft-deleted/cancelled session" with "concurrent duplicate," permanently blocking regeneration of that date slot.**
  - The unique-constraint-violation catch treats any collision as "a concurrent run already created this," but if the existing row is actually a soft-deleted/cancelled session rather than a true race, the slot can never regenerate via automation again.
  - **Fix**: query `withTrashed()` first and `restore()` if the existing row is soft-deleted, rather than assuming every collision is benign.
- **MEDIUM — FK cascade behavior is inconsistent between legacy and new academic tables.**
  - `students.id` cascade-deletes `memorization_sessions`/`revision_sessions`/`attendances`/`assessments`/`homework`/`circle_student` (original migrations) but **restricts** deletion via `student_learning_records`/`parent_evaluations`/`enrollments` (newer migration). Currently dormant since students are only ever soft-deleted in application code, but a real inconsistency if a hard delete path is ever exercised.
  - **Fix**: standardize on `restrictOnDelete()` for all student-history tables.

**What's solid**: soft-delete + unique-index handling is correctly done for `attendances` and `parent_evaluations` (the pattern just wasn't copied to `student_learning_records`); `enrollments`/legacy `circle_student` pivot dual-write is correct and tested; migrations are otherwise clean, chronologically coherent, and every core table has an appropriate business-grain unique constraint.

---

## 8. API Coverage

- **Orphan/dead endpoints (built server-side, called by neither client)**:
  - `GET/PUT/DELETE /guardians` — also a live security hole, see §1.
  - The entire Reports feature (6 endpoints: student-progress, attendance, teacher-performance, parent-engagement, circle-performance, admin-dashboard) — `admin_app`'s "Reports" screen calls `/statistics/*` only, never `/reports/*`, despite substantial backend investment (joins, date filters, tenant scoping) sitting fully implemented and dead.
  - `POST auth/logout-all`, `GET auth/devices`, `DELETE auth/devices/{token}`, `PUT auth/device-name` — server-ready session management, no client UI.
  - `GET /parent/dashboard` — implemented, never called (parent home screen composes its view from `my-children`/`evaluations`/`notifications` instead).
  - `CircleController::enrollStudent`/`unenrollStudent` — dead, **unrouted** methods that only touch the legacy `circle_student` pivot directly (the real, tested path is `EnrollmentController@enroll/@close`, which correctly dual-writes). Currently harmless because unreachable, but a latent trap if ever re-routed.
- **Client calling a nonexistent endpoint**: none found — every repository call site in both Flutter apps matched an actual route.
- **Fix direction**: for each orphan, either wire up the missing client screen or remove the endpoint; delete the unrouted `enrollStudent`/`unenrollStudent` methods outright since re-exposing them would silently reintroduce the enrollment/pivot desync bug that was already fixed once.

---

## 9. Flutter Coverage

**`admin_app`** — clean. Full orphan-file sweep (every `.dart` basename cross-referenced against every other file's imports) found **zero orphaned screens** — every screen is reachable from the `AppScreen` enum switch or a direct `Navigator.push`. `session_detail_screen.dart` and the old duplicate attendance screen are confirmed fully deleted with zero remaining references. Every `Backend` field has a real call site. Two small dead-code items: `offline_store.dart`'s `cacheList`/`readList`/`hasCache` (see §6) and `quran_repository.dart`'s unused `surah(int)` single-lookup method.

**`student_app`** — mostly clean but with real dead surfaces:
- `MoreMenuSheet` and `ParentMoreSheet` are fully built, never shown anywhere, and `ParentMoreSheet` additionally has a broken route (`/parent/assessments` doesn't exist — only `/parent/child/:childId/assessments` does; `go_router` has no `errorBuilder`, so wiring this up as-is would crash on tap rather than degrade gracefully).
- `childrenProvider` and `evaluationsProvider` are fetched but never rendered anywhere — a real functional gap (no "past evaluations" history screen), not just unused code.
- `ParentRepository.getChildAssessments`/`getFamilyPayments` and domain models `Certificate`/`Payment`/`Badge`/`IconDataRef` are fully modeled with `fromJson`/`toJson` but never constructed.
- `OfflineSyncService.enqueueRequest`/`clearQueue` are never invoked by any repository — consistent with the app's "read-through cache, no write queue" design, but the dead surface is misleading to future maintainers.

---

## 10. Performance

- **HIGH — parent-facing history endpoints are unpaginated.** `ParentEvaluationQuery::childSessions()`/`childLearningRecords()`/`childAttendance()` all end in `->get()` with no `paginate()`, while the equivalent student self-service endpoints are correctly paginated and capped. A student with years of history returns their entire dataset in one response every time a parent opens the child's page.
- **HIGH — `ReportService::paginate()` has no upper bound on `per_page`**, unlike every other list endpoint (`BaseQuery::paginate()` caps at 100). Currently low-risk only because the whole feature is unused (§8), but would allow an unbounded aggregation dump the moment it's wired up or called directly.
- **N+1 avoidance is otherwise deliberate and consistent**: every Resource checked uses `whenLoaded()` rather than unconditionally dereferencing relations, and `BaseQuery` drives every list endpoint through declared `allowedIncludes` with proper eager-loading. No N+1 instances found outside the two pagination gaps above.
- **Dashboard load paths**: today's-sessions and roster queries are appropriately scoped and indexed per the Phase 0 hardening work; no O(n²) shapes found in `ReportService` beyond the missing pagination cap.

---

## 11. Production Infrastructure

- **HIGH — the push notification pipeline is fully built server-side and completely unreachable — push notifications cannot be delivered as shipped.** `PushNotificationService`, the delivery job, `device_tokens`/`push_deliveries` tables, and `/devices` endpoints all exist and are tested, but grepping both Flutter apps for FCM/device-registration code returns nothing. No device ever registers a token, so every push send is a no-op in production.
- **MEDIUM — Idempotency-Key infrastructure is inert end-to-end.** ~15 mutating endpoints use the `idempotency` middleware (a graceful no-op without the header), but neither app ever sends an `Idempotency-Key` header. No functional breakage today (status-machine/unique-constraint guards provide a partial safety net), but the replay-safety this was built for doesn't exist yet.
- **What's materially solid, and above typical bar for this project stage**: a real queue driver (`QUEUE_CONNECTION=database`) with Supervisor configs for both worker and scheduler; `spatie/laravel-backup` configured for daily DB + weekly full backups plus `backup:monitor`; Sentry SDK integrated and conditionally activated (verify `SENTRY_LARAVEL_DSN` is actually set in the deployed `.env` — it's blank in `.env.example`, which is expected for a template but worth an explicit pre-launch checklist item); 13 scheduled automation commands actually registered in `routes/console.php` (not just defined) and covered by a dedicated test (`EnterpriseHardeningIntegrationTest.php`) that directly asserts they're on the schedule; endpoint-differentiated rate limiting (`throttle:sensitive-write`, `throttle:login`, `throttle:otp`, `throttle:device-registration`).
- **LOW** — `.env.example` ships `LOG_STACK=single` (no rotation) rather than `daily`; worth confirming the deployed config overrides this.

---

## 12. Testing

- **`api`**: `php artisan test` → 284 passing (one independent run showed 283 passed / 1 flaky order-dependent failure in `PushNotificationTest` that passes cleanly in isolation — confirmed pre-existing and unrelated to any change made in this review, see [TEACHER_LOCK_AUDIT.md](TEACHER_LOCK_AUDIT.md) §5).
- **`admin_app`**: `flutter analyze` → 0 issues. `flutter test` → all passing (live-API tests skip gracefully when no backend is running, as designed).
- **`student_app`**: `flutter analyze` → 0 issues. `flutter test` → 17/17 passing.
- **Coverage gaps**:
  - `StudentLifecycle` (suspend/reactivate/graduate/withdraw/transfer) has no dedicated test file — only appears incidentally as a precondition inside other tests.
  - No single HTTP-level test chains the **entire** session lifecycle end to end (`POST /sessions` → `start` → `attendance/bulk` → `learning-records` → `parent/evaluations` → `complete` → `lock`) in one flow — coverage is thorough but siloed per domain/action.
  - Several files are thin relative to their surface area: `EnrollmentSyncTest.php` (1 test), `ProductionSmokeTestCommandTest.php` (2), `AuthTokenSecurityTest.php`/`EnterpriseRateLimitingTest.php`/`GenerateCircleSessionsTest.php`/`PlatformStorageTest.php`/`ReportServiceTest.php` (3 each).
- **What's notably good**: `EnterpriseHardeningIntegrationTest.php` directly verifies scheduled commands and backup/queue command registration — exactly the "is it just defined or actually running" check this kind of audit usually has to ask for by hand.

---

## Final Verdict

**NOT_READY**

Reasoning: a live, directly-exploitable IDOR exposes every family's contact PII to any authenticated account (§1), independent of and in addition to a second unscoped-listing IDOR on circles/teachers. Separately, both consumer-facing home screens currently display fabricated payment/evaluation/achievement content to real users on every load (§4/§5) — a product-trust and potentially a financial-miscommunication issue serious enough to block launch on its own. A core progress-tracking feature (memorization statistics) is silently drifting from the data teachers actually enter (§7), a data-integrity failure with no user-visible symptom until someone notices the numbers stopped moving. And the one offline-safety mechanism this system relies on for attendance data has a real, demonstrated silent-data-loss path (§6). None of these are edge cases requiring adversarial effort to trigger — the guardians IDOR is one authenticated `GET`, the fake home-screen data is on by default for every user, and the statistics drift and offline-loss bugs will surface during completely normal day-one usage.

None of the above require a large redesign — each has a narrow, mechanical fix already named in its section — but they need to land, be verified, and be re-tested before this is safe to expose to real families' data or a real payment-adjacent UI. Once the CRITICAL and HIGH items in this report are resolved and re-verified (particularly §1, §4/§5's fabricated content, §6's 409 handling, and §7's statistics source), this system would be a reasonable candidate for `READY_FOR_STAGING_ONLY` pending a staging soak; production readiness should be reassessed after that, not assumed from this pass alone.
