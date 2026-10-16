# MISSING_FEATURES.md

> Gaps found by cross-referencing backend routes ↔ both apps ↔ expected business rules. Prioritized.

## ✅ Critical items — FIXED this pass
1. ~~**Admin attendance session-centric (Finding A)**~~ — **done**: session picker + `{circle_session_id, records}` + offline enqueue with `circle_session_id`. Data-loss path closed.
2. ~~**Admin unenroll (B) + enrollment dual-model (D)**~~ — **done**: `enrollments` is canonical; enroll/close keep the pivot synced; roster returns `enrollment_id`; unenroll via `/enrollments/{id}/close`. Verified by `EnrollmentSyncTest`. *(The orphan `CircleController@enrollStudent/unenrollStudent` methods can now be safely deleted — left in place to avoid unrelated churn.)*

## 🟠 Admin UI missing for existing endpoints (Rule 18 violations)
3. **Session lifecycle actions** — no UI for `POST /sessions/{id}/start|complete|cancel|lock`. Field staff can't progress or lock sessions from the app.
4. **Student lifecycle transitions** — no UI for `suspend|reactivate|graduate|withdraw|transfer` (`/students/{student}/*`). Add actions on `student_profile` gated to super_admin/center_owner, honoring `allowedTransitions()`.
5. **Audit logs** — `GET /audit-logs` has no screen. Add a read-only audit viewer for super_admin/center_owner.
6. **Users management** — `GET/POST /users`, `POST /users/{user}/deactivate` unused. Add a users admin screen.
7. **Statistics recalculation** — `POST /statistics/{students|teachers|circles}/{id}/recalculate` has no trigger in UI.
8. **Bulk delete** — `/students/bulk-delete`, `/teachers/bulk-delete` unused (multi-select + confirm).
9. **Device/push registration** — `/devices` unused in both apps; no FCM/APNs SDK. Notifications are pull-only.

## 🟡 student_app / parent gaps (backend-bounded)
10. **Parent evaluation submit** — needs a session picker to supply `circle_session_id` + a homework-follow-up score (`POST /parent/evaluations`).
11. **Parent child drill-down** — no backend endpoints for a parent to read a child's memorization/attendance/assessments; either add `/parent/my-children/{id}/{records}` endpoints or keep "not available".
12. **student `/student/sessions` & `/student/learning-records`** — endpoints exist, no screen.
13. **Notifications read-all** — button missing (endpoint exists).

## 🟢 Backend feature gaps vs. product (need product decision)
14. **Payments / certificates / gamification (badges, points, rank)** — surfaced in student_app design but **no API**. Either build the backend domain or drop from the design (currently "not available").
15. **Same-center assertion** in memorization/revision create (defense in depth — see `SECURITY_AUDIT.md` S1).

## Not gaps (intentional / ops-only)
`/health` (ops), admin-only cross-center dashboard, teacher salary hidden from teachers.
