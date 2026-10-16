# WORKFLOW_AUDIT.md

> End-to-end workflows traced through actual code across the three projects.

## 1. Attendance capture (teacher/admin)
**Intended:** open session → mark roster → record.
**Backend (correct):** `POST /attendance` or `/attendance/bulk` with `circle_session_id` → `AttendanceService` locks the session (`lockWritableSession`), asserts tenant/teacher access, rejects future/locked/completed sessions, asserts each student is **Active** and **enrolled** for the session date, derives teacher/circle/date from the session, blocks duplicates (`withTrashed` + `lockForUpdate`).
**Admin (FIXED — Finding A):** the screen now picks **circle → session**, loads the enrolled roster, and submits `{circle_session_id, records}` to `/attendance/bulk`; offline enqueues each record with `circle_session_id`. Prerequisite: a `CircleSession` must exist for the circle.
**Student:** read-only attendance list ✅.
**Status:** ✅ backend + admin aligned (live UI smoke pending).

## 2. Enrollment lifecycle
**Backend:** `POST /circles/{circle}/enrollments` → `EnrollStudentAction` (active student, active circle, same center, no existing active enrollment, capacity check) writes `enrollments`. Close via `POST /enrollments/{enrollment}/close` (status Completed/Transferred/Withdrawn + reason).
**Admin (FIXED — Findings B & D):** enroll → `/circles/{circle}/enrollments`; unenroll → `/enrollments/{id}/close` via the `enrollment_id` now returned by the roster. Backend enroll/close now keep the `circle_student` pivot in sync with the canonical `enrollments` table, so the roster reflects enrollments. Verified by `EnrollmentSyncTest`.
**Status:** ✅ backend + admin aligned; canonical table = `enrollments`.

## 3. Academic records (memorization/revision/homework/assessments/learning-records)
**Backend:** Homework/Assessments/LearningRecords go through `AcademicSessionGuard` (session writable + tenant + **student active** + enrolled). Memorization/Revision were **legacy** (circle-based, no session) and skipped the active-student check → **Finding C, now fixed**.
**Status:** ✅ (after Fix C).

## 4. Session lifecycle (scheduled → in-progress → completed/cancelled → locked)
**Backend:** `POST /sessions/{id}/start|complete|cancel|lock` with idempotency + immutability guards; locked/completed sessions reject writes everywhere via `isImmutable()`/`locked_at`.
**Admin:** ❌ **no UI** for start/complete/cancel/lock. Sessions can be listed/created but not transitioned. → `MISSING_FEATURES.md`.
**Status:** ⚠ backend complete, admin UI missing.

## 5. Student lifecycle (suspend/reactivate/graduate/withdraw/transfer)
**Backend:** `StudentLifecycleService` enforces `StudentStatus::allowedTransitions()` (e.g., Graduated is terminal; Suspended→Active/Withdrawn only) + `role:super_admin,center_owner`.
**Admin:** ❌ **no UI** for transitions.
**Status:** ⚠ backend complete, admin UI missing.

## 6. Parent evaluation
**Backend:** `POST /parent/evaluations` requires `circle_session_id` + 1–5 scores; rejects future/locked sessions, non-guardian, non-child, unenrolled; one evaluation per (guardian, session, student).
**Student app:** read works; submit **gated** (UI lacks a session picker to supply `circle_session_id`).
**Status:** ⚠ read ✅, write UI-gated.

## 7. Auth & session
Login → token (secure storage) → `me` restore → 401 auto-logout. Consistent across both apps. ✅
