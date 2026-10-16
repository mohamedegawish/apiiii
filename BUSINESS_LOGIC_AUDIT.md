# BUSINESS_LOGIC_AUDIT.md

> Cross-project audit of `api/` (Laravel 12), `admin_app/` (Flutter, provider+http), `student_app/` (Flutter, Riverpod+dio). Source code is the source of truth; every claim below cites the file that proves it. Verified 2026-07-24.

## Verdict
The **backend is enterprise-grade** and enforces the expected rules rigorously through a shared `AcademicSessionGuard`, `prohibited` request fields, policies, tenant scoping, and `lockForUpdate` concurrency control. The audit found **3 real defects** and **1 architectural inconsistency** — **all four are now fixed** (backend feature-test verified; admin analyzer-clean against test-verified contracts). See `IMPLEMENTED_FIXES.md`.

---

## 1. Expected business rules — verification (Phase 4)
Legend: ✅ enforced (evidence) · ⚠ partial · ❌ violated/gap.

| # | Rule | Status | Evidence |
|---|---|---|---|
| 1 | Student belongs to one center | ✅ | `students.center_id`; `EnrollStudentAction` rejects cross-center enroll (`student.center_id !== circle.center_id`). |
| 2 | Student → many guardians | ✅ | `guardian_student` pivot; `Student::guardians()`. |
| 3 | Guardian → many students | ✅ | `ParentService::myChildren` via guardian. |
| 4 | Teacher → many circles | ✅ | `circles.teacher_id`. |
| 5 | Circle → many students | ✅ | `Circle::students()` / `enrollments`. |
| 6 | Attendance only via Circle Session | ✅ | `StoreAttendanceRequest` requires `circle_session_id`; teacher/circle/date derived from session in `AttendanceService::attendanceData`. |
| 7 | Teacher can't record outside his circles | ✅ | `AttendanceService::assertSessionAccess` + `AcademicSessionGuard` (`session->teacher->user_id === user->id`). |
| 8 | Teacher identity from auth | ✅ | derived from session/circle, never from input. |
| 9 | Teacher cannot inject `teacher_id` | ✅ | `'teacher_id' => ['prohibited']` in **all 13** field requests (Attendance, Assessments, Homework, LearningRecords, Memorization, Revision × store/update). |
| 10 | Parent evaluates behavior/prayer/commitment per completed session | ✅ | `StoreParentEvaluationRequest`: `behavior_score`, `prayer_score`, `homework_followup_score` (1–5) + `circle_session_id`. |
| 11 | Parent cannot evaluate future sessions | ✅ | `ParentService::assertWritableSession` / `AcademicSessionGuard` (`session_date->isFuture()`). |
| 12 | Locked sessions immutable | ✅ | `AcademicSessionGuard` (`locked_at !== null`) + `AttendanceService::lockWritableSession`. |
| 13 | Completed sessions immutable | ✅ | `session->isImmutable()` + status not in [Scheduled, InProgress]. |
| 14 | Suspended students can't receive Attendance/Assessments/Homework/Learning Records | ✅ (4/4) **but** ❌ for Memorization/Revision **(FIXED)** | `AcademicSessionGuard` guards the 4; Memorization/Revision "legacy" actions bypassed it → **Finding C**, now fixed. |
| 15 | Student app shows attendance/memorization/revision/homework/assessments/reports/notifications/session history/progress/payments(if API) | ⚠ | Connected for supported features; payments/certificates/badges have **no API** (see `API_STUDENT_MAPPING.md`). |
| 16 | Parent app shows child performance/attendance/prayer/behavior/notifications/reports | ⚠ | Backend only exposes children list, sessions, evaluations, dashboard, notifications; per-child drill-down has **no parent endpoint**. |
| 17 | Admin app exposes every admin backend feature | ⚠ | Broad coverage; gaps in `API_ADMIN_MAPPING.md` (audit-logs, users, lifecycle, session lifecycle actions, statistics recalculation UI). |
| 18 | No API endpoint without UI access | ⚠ | Several admin endpoints lack UI (audit-logs, users mgmt, student lifecycle transitions, session start/complete/lock). |
| 19 | No UI feature without API support | ⚠ | student_app gamification (payments/certificates/badges) has no API — handled as "not available". |
| 20 | Every workflow executable from UI | ⚠ | Attendance (A) + enroll/unenroll (B) **now fixed**; remaining UI gaps are session-lifecycle & student-lifecycle actions (`MISSING_FEATURES.md`). |

---

## 2. Findings (evidence-based)

### 🟢 Finding A — Admin bulk attendance broken + offline data loss (FIXED)
- **Was:** `bulkAttendance` POSTed `/attendance/bulk` with `teacher_id/circle_id/attendance_date` (all `prohibited`) and **no `circle_session_id`** → 422 online; offline it enqueued the same → dropped as terminal 422 → **silent data loss**.
- **Fix:** admin attendance is now **session-centric** — new `CircleSession` model + `circleSessions()` fetch; the screen picks circle→session and loads the enrolled roster; `bulkAttendance({circleSessionId, records})` posts `{circle_session_id, records}`; offline enqueues each record **with `circle_session_id`**. Teacher/circle/date derive from the session server-side. `admin_app` analyzer-clean; the closed data-loss path no longer 422s.

### 🟢 Finding B — Admin enroll/unenroll hit non-existent routes (FIXED)
- **Was:** `enrollStudent` → `POST /circles/{id}/students`; `unenrollStudent` → `DELETE /circles/{id}/students/{student}`. `route:list` proved **neither exists** (orphan `CircleController` methods).
- **Fix:** enroll → `POST /circles/{circle}/enrollments`; unenroll → `POST /enrollments/{enrollment}/close` using the `enrollment_id` now returned by the roster endpoint. Verified by `EnrollmentSyncTest`.

### 🟢 Finding D — Enrollment split across two unsynced tables (FIXED)
- **Was:** `enrollments` (source of truth for guards/capacity/lifecycle) and the `circle_student` pivot (read by the roster + statistics) were **never synced** — `EnrollStudentAction` wrote only `enrollments`, so enrolled students never appeared on the roster and pivot-only students couldn't receive attendance.
- **Fix:** the canonical enroll/close actions now **drive the pivot** (`syncWithoutDetaching` on enroll, `detach` on close), and the roster returns the canonical `enrollment_id`. End-to-end verified by `tests/Feature/EnrollmentSyncTest.php`. Canonical table is `enrollments`; the pivot is now a synchronized read-model.

### 🟢 Finding C — Memorization/Revision bypassed the suspended-student guard (FIXED + TESTED)
- `CreateMemorizationSessionAction` / `CreateRevisionSessionAction` (labelled "legacy") validated teacher-owns-circle but **not** student status — violating Rule 14 on an **active** admin path (`admin_app` records via `/memorization-sessions` & `/revision-sessions`).
- **Fix:** added `Student::findOrFail(...)->status->allowsAcademicActivity()` guard to both create actions (mirrors `AcademicSessionGuard`).
- **Verified:** `php artisan test --filter="Memorization|Revision|Attendance|StudentLifecycle"` → **37 passed (117 assertions)**.

---

## 3. Where the three systems agree (confirmed consistent)
- **Response envelope** `{success,message,data,meta.pagination}` — identical parsing in api ↔ admin (`ApiClient._handle`) ↔ student (`ApiEnvelope`).
- **401 handling** — both apps auto-logout on 401 (`api_client.onUnauthorized`).
- **Role gating** — admin `Permissions` class mirrors backend `role:*` middleware & policies (e.g., `viewSalaries` = superAdmin|centerOwner ↔ `role:super_admin,center_owner` on `/teachers/{id}/salary`).
- **Teacher injection prevention** — enforced backend-side; neither app can override it.

See `SECURITY_AUDIT.md`, `ROLE_MATRIX.md`, `WORKFLOW_AUDIT.md`, `OFFLINE_SYNC_AUDIT.md` for depth.
