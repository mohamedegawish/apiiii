# IMPLEMENTED_FIXES.md

> All four findings are now **fixed**. Backend fixes are feature-test verified; admin fixes are analyzer-clean and target test-verified backend contracts (live UI smoke is the remaining sign-off — see the scope note at the end).

## ✅ Implemented & verified

### Fix C — Suspended-student guard on memorization & revision (backend)
**Problem:** `CreateMemorizationSessionAction` / `CreateRevisionSessionAction` (legacy, circle-based) recorded academic sessions for **any** student, bypassing Rule 14 (suspended students must not receive records). Active admin path.
**Change:**
- `api/app/Features/MemorizationSessions/Actions/CreateMemorizationSessionAction.php`
- `api/app/Features/RevisionSessions/Actions/CreateRevisionSessionAction.php`
Added after the teacher-owns-circle check:
```php
$student = Student::findOrFail($data['student_id']);
if (! $student->status->allowsAcademicActivity()) {
    throw new ConflictException('Only active students may receive memorization/revision records.');
}
```
**Verification:** `php artisan test --filter="Memorization|Revision|Attendance|StudentLifecycle"` → **37 passed (117 assertions)**. No regressions.

### Fix B (enroll) — Correct enrollment endpoint (admin_app)
**Problem:** `Backend.enrollStudent` posted to `POST /circles/{id}/students` — a route that does not exist (`route:list` proof) → 404.
**Change:** `admin_app/lib/data/backend.dart` → `POST /circles/{circle}/enrollments {student_id}` (the real `EnrollmentController@enroll`, source-of-truth table used by academic guards).
**Verification:** `flutter analyze admin_app` → **No issues found**. (Runtime smoke pending a live server; the target route/payload is confirmed against `EnrollStudentRequest`.)

### Fix D + B — Enrollment unified on the canonical `enrollments` table (backend + admin)
**Problem (Finding D):** `enrollments` (source of truth for academic guards/capacity/lifecycle) and the `circle_student` pivot (read by `GET /circles/{id}/students` + statistics) were **never synced** — `EnrollStudentAction` wrote only `enrollments`. So a properly-enrolled student never appeared on the roster; unenroll (Finding B) had no working route.
**Change (backend):**
- `EnrollStudentAction` now `syncWithoutDetaching` the pivot after creating the enrollment.
- `CloseEnrollmentAction` now `detach`es the pivot after closing.
- `CircleController@students` returns each student **with `enrollment_id`** (active enrollment) so clients can close it.
**Change (admin):**
- `Student` model parses `enrollment_id`; `Backend.unenrollStudent(enrollmentId)` → `POST /enrollments/{id}/close {status:'withdrawn', reason}`; `circle_students_screen` passes `s.enrollmentId` (guards null).
- `Backend.enrollStudent` → `POST /circles/{circle}/enrollments`.
**Verification:** new `tests/Feature/EnrollmentSyncTest.php` drives enroll → roster (asserts `enrollment_id` present) → close → roster empty → enrollment `withdrawn`. **Passes.** Full backend suite **265 passed**. `admin_app` analyzer clean.

### Fix A — Session-centric admin attendance (admin)
**Problem:** `bulkAttendance` sent prohibited `teacher_id/circle_id/attendance_date` and no `circle_session_id` → 422 online; offline it enqueued the same → dropped as terminal 422 → **data loss**.
**Change (admin):**
- New `CircleSession` model + `Backend.circleSessions({circleId,status})` (`GET /sessions` filtered).
- `Backend.bulkAttendance({circleSessionId, records})` → `POST /attendance/bulk {circle_session_id, records}`; offline enqueues each `/attendance` record **with `circle_session_id`** (matches `StoreAttendanceRequest`).
- `attendance_screen` reworked: pick **circle → session** (auto-selects an open session), load the **enrolled roster** via `circleStudents`, mark, submit. Teacher/circle/date are derived from the session server-side (teacher picker removed).
**Verification:** `admin_app` `flutter analyze` → **No issues found**. Backend `/attendance/bulk` contract is covered by the existing green `AttendanceTest`. The offline enqueue now carries `circle_session_id`, so sync no longer 422s → **the data-loss path is closed**.

## Regression safety
- Backend: **full suite 265 passed, 0 failed** (includes the new `EnrollmentSyncTest`). Note: `PushNotificationTest` is **order-dependent/flaky** (it failed in a prior run, passed here); it is unrelated to any change in this work and should be de-flaked separately by the owner.
- admin_app: `flutter analyze` **clean** after all three fixes.
- student_app: unchanged this pass — still `flutter analyze` clean + 15 tests passing.

## Honest scope note (runtime verification)
The **backend** fixes are unit/feature-test verified. The **admin_app** fixes compile clean and target test-verified backend contracts, but the admin UI flows were **not exercised against a live server** in this environment (no running instance). A seeded live smoke of: (1) enroll → roster shows student → unenroll, and (2) pick circle+session → mark roster → save (online and offline→reconnect) is the remaining sign-off step. A prerequisite of the new attendance flow is that a `CircleSession` exists for the circle (created via the sessions screen / schedule automation) — attendance is now correctly session-bound.
