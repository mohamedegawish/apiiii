# STUDENT_CIRCLE_DISPLAY_REPORT.md

Report for the "Students list + profile show current circle" task.

## Summary

The admin Flutter app now shows each student's current circle (الحلقة الحالية) both under their name in the Students list and in a dedicated card in the student profile. The data is served by the Laravel API via a new `current_circle` block on the student resource, derived from the canonical **active enrollment** (`status = active` AND `left_at = null`) — the same definition already used by `StudentPolicy`, `CircleRosterService`, `BaseQuery::accessibleStudentIds`, `EnrollStudentAction`, and `TransferStudentAction`. No circle name is hardcoded in the app; teacher authorization is enforced server-side.

## API changes (`api/`)

### `app/Features/Students/Resources/StudentResource.php`

- Added `current_circle` to the resource shape:
  ```json
  {
    "id": "c1",
    "name": "حلقة النور",
    "teacher_id": "t1",
    "teacher_name": "الشيخ أحمد",
    "schedules": [{ "id", "day_of_week": 1, "start_time": "17:00", "end_time": "18:00", "room" }]
  }
  ```
  `schedules` is only included when `circle->relationLoaded('schedules')` — `null` in the list (avoids N+1), populated in the profile.
- `current_enrollment` was slimmed to `{enrollment_id, circle_id, joined_at, status}` (the old circle display fields were moved into `current_circle` to avoid duplication).
- Added `formatCurrentCircle()` helper.

### `app/Features/Students/Queries/StudentQuery.php`

- Overrode `paginate()` to eager-load **only active** enrollments plus `enrollments.circle.teacher.user` (`withWhereHas` with `status = active` + `left_at = null`), so the list stays cheap and the circle/teacher names render without per-row queries.
- Fixed a closure type hint in the eager-load (`Relation` vs `EloquentBuilder`).

### Profile (`app/Features/Students/Services/StudentService.php`)

- Unchanged: `findOrFail` already loads `enrollments.circle.teacher.user` and `enrollments.circle.schedules`.

## Flutter changes (`admin_app/`)

- `lib/data/models/student.dart` — new `CurrentCircle` model (id, name, teacherId, teacherName, schedules via `CircleSchedule.fromJson`); `Student` gained `currentCircle`, `currentEnrollmentJoinedAt`, `currentEnrollmentStatus`.
- `lib/ui/list_models.dart` + `lib/ui/screens/module_lists.dart` — `ModuleListItem` gained an optional `sub2` line rendered under `sub` in `ModuleListRow`.
- `lib/ui/screens/students_screen.dart` — removed `include: 'circles'` from the list fetch; added `_circleLine(Student)` helper; both the selection-mode row and the normal `ModuleListItem` show `الحلقة: <name>` (or `الحلقة: غير مسجل في حلقة`); the status chip now shows `currentCircle?.name` with status fallback.
- `lib/ui/screens/student_profile.dart` — removed `include: 'circles'` from `find`; added `_CurrentCircleCard` (الحلقة الحالية) rendered after the header, with `_formatTime12h()` and `_enrollmentStatusAr()` helpers; the header subtitle now uses `currentCircle?.name`.

## Performance / N+1

- Backend test `StudentCircleDisplayTest` asserts the students-list endpoint runs **< 10 queries** for a page of 10 students (eager-loads active enrollments + circle + teacher + user in 4 queries regardless of page size).
- Flutter never renders per-row circle data that requires extra round-trips; the list uses only the `current_circle` already embedded in the student payload.

## Tests executed

### Backend (`api/`) — `php artisan test --compact`

New: `api/tests/Feature/StudentCircleDisplayTest.php` — 6 Pest tests, all passing:
1. students list returns `current_circle` with name/teacher/schedules
2. `current_circle` is `null` when the student has no enrollment
3. after a transfer, the list reflects the new circle (and not the old one)
4. after closing the enrollment, `current_circle` becomes `null`
5. a teacher cannot see another teacher's student/circle (403)
6. no N+1 (query count < 10)

Full suite: **371 passed, 1 risky, 1 skipped** (both pre-existing, unrelated).

### Flutter (`admin_app/`) — `flutter test`

New: `test/student_circle_display_test.dart` — 5 tests, all passing:
1. `Student.fromJson` parses `current_circle` + schedules (int `day_of_week`) + `current_enrollment`
2. `Student.fromJson` handles missing circle (`null`)
3. Students list renders the circle line under each student (`الحلقة: حلقة النور` / `الحلقة: غير مسجل في حلقة`)
4. Student profile shows the الحلقة الحالية card with name, teacher, and joined date
5. Student profile shows `غير مسجل في حلقة` when the student has no circle

Full suite: **47 passed, 21 skipped (live-api, no server), 1 failing** — the failing test is the pre-existing `student_creation_flow_test.dart` (asserts a form field named `mobile`; the form uses `student_phone`), unrelated to this feature and failing before these changes.

`flutter analyze` — no new issues (9 pre-existing infos/warnings in unrelated files).

## Files changed

**Backend:** `app/Features/Students/Resources/StudentResource.php`, `app/Features/Students/Queries/StudentQuery.php`, `tests/Feature/StudentCircleDisplayTest.php` (new).

**Flutter:** `lib/data/models/student.dart`, `lib/ui/list_models.dart`, `lib/ui/screens/module_lists.dart`, `lib/ui/screens/students_screen.dart`, `lib/ui/screens/student_profile.dart`, `test/student_circle_display_test.dart` (new).

## Remaining issues

- **Manual E2E verification pending** (no running MySQL/UI in this environment): enroll a student → confirm list/profile show the circle → transfer → confirm the new circle → close enrollment → confirm `غير مسجل في حلقة`. The backend tests cover this flow with a real test DB.
- Pre-existing unrelated failures: `student_creation_flow_test.dart` (mobile vs student_phone) and the 1 risky/1 skipped backend tests.

---

# Follow-up: transfer crash (red screen) + lifecycle buttons verification

## Summary

Reported: transferring a student from circle to circle crashed the app with a red screen, plus a request to verify every lifecycle button on the student page (transfer / graduate / suspend / reactivate / withdraw) and confirm each has both a UI button and a working API.

## Root cause of the crash (fixed)

The crash was in the app, in `promptText` (`admin_app/lib/ui/widgets/feedback.dart`). It called `controller.dispose()` immediately after `showDialog(...)` returned, but the dialog's exit animation was still running and rebuilding its `TextField` with the now-disposed controller → `TextEditingController was used after being disposed` → Flutter error/red screen.

Because every lifecycle action prompts for a reason via `promptText` (transfer, graduate, suspend, reactivate, withdraw), **all of them crashed at the moment of confirming the dialog** — not just transfer.

**Fix:** the prompt dialog is now a dedicated stateful widget (`_PromptTextDialog`) that owns its `TextEditingController` and disposes it in `State.dispose()` (i.e., after the route's exit animation completes). Public `promptText` signature unchanged, so all callers are unaffected.

## Verification: all lifecycle buttons have UI + API and work

Backend endpoints (all exist and return 200; confirmed by new test file `api/tests/Feature/StudentLifecycleCrashTest.php`):
- `POST /students/{id}/suspend` (reason), `POST /students/{id}/reactivate` (reason), `POST /students/{id}/graduate` (reason), `POST /students/{id}/withdraw` (reason), `POST /students/{id}/transfer` (target_circle_id + reason). All gated by the student-status transition rules server-side; `StudentLifecycleResource` handles the no-active-enrollment case safely (`EnrollmentResource::make(null)` serializes to `null`, no 500).

App UI (`admin_app/lib/ui/screens/student_profile.dart`, `_actionsFor`):
- active → إيقاف / تخريج / نقل / سحب
- suspended & transferred → تفعيل / سحب
- withdrawn → تفعيل
- graduated → none (terminal)
- rendered only when `permissions.editStudents`.

New Flutter test file `admin_app/test/student_lifecycle_actions_test.dart` drives the real UI flow against a mocked backend — opens the profile, taps the lifecycle button, picks a circle (transfer), types a reason, confirms — and asserts the correct HTTP request is sent with no exception:
1. transfer: circle picker → prompt → `POST /students/s1/transfer` with `target_circle_id` — no exception
2. graduate → `POST /students/s1/graduate` — no exception
3. suspend → `POST /students/s1/suspend` — no exception

## Results

- `flutter test`: 50 passed, 21 skipped (live-api, no server), 1 pre-existing failure (`student_creation_flow_test.dart`, unrelated).
- `flutter analyze`: no new issues (9 pre-existing infos in unrelated files).
- Backend `php artisan test`: 375 passed, 1 risky + 1 skipped (pre-existing) — plus an intermittent pre-existing flake in `PushNotificationTest` (FCM notification-count timing) that predates this work.

## Files changed (follow-up)

- `admin_app/lib/ui/widgets/feedback.dart` — `_PromptTextDialog` stateful dialog; crash fix.
- `admin_app/test/student_lifecycle_actions_test.dart` — new UI flow tests.
- `api/tests/Feature/StudentLifecycleCrashTest.php` — new backend lifecycle-endpoint tests.