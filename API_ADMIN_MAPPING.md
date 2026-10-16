# API_ADMIN_MAPPING.md

> Every backend route vs. `admin_app` usage. Wiring hub: `admin_app/lib/data/backend.dart`. Legend: ✅ used & correct · ⚠ used but defective · ❌ no UI · N/A student-only.

## CRUD resources (via `CrudRepository`)
| API route | Admin screen | Repo | Status |
|---|---|---|---|
| `/centers` (index/show/store/update/delete) | `centers_screen` | `centers` crud | ✅ |
| `/guardians` … | `guardians_screen` | `guardians` crud | ✅ |
| `/teachers` … (+ `/teachers/{id}/salary`) | `teachers_screen` | `teachers` crud + `updateTeacher` | ✅ (salary payload `salary_type,salary_amount` matches `UpdateTeacherSalaryRequest`) |
| `/students` … (+ bulk-delete) | `students_screen`, `student_profile` | `students` crud | ✅ (bulk-delete: ⚠ no UI button) |
| `/circles` … | `circles_screen` | `circles` crud | ✅ |
| `/circle-schedules` … | `schedules_screen` | `circleSchedules` crud | ✅ |

## Field ops (via `SyncingRepository`, offline-capable)
| API route | Admin screen | Status |
|---|---|---|
| `/memorization-sessions` (CRUD) | `sessions`-related | ✅ (now guarded for suspended students — Fix C) |
| `/revision-sessions` (CRUD) | `sessions`-related | ✅ (Fix C) |
| `/attendance` (offline sync) | `attendance_screen` | ✅ **fixed** — offline ops now carry `circle_session_id` (no more terminal 422) |
| `/attendance/bulk` | `attendance_screen` | ✅ **fixed (Finding A)** — session-centric: sends `{circle_session_id, records}` |
| `/sessions` (filtered) | `attendance_screen` (session picker) | ✅ new `circleSessions()` fetch |
| `/assessments` (CRUD) | `assessments_screen` | ✅ |
| `/homework` (CRUD) | `homework_screen` | ✅ |

## Enrollment
| API route | Admin call | Status |
|---|---|---|
| `GET /circles/{id}/students` | `circleStudents` | ✅ now returns `enrollment_id`; pivot synced from `enrollments` (Finding D fixed) |
| `POST /circles/{circle}/enrollments` | `enrollStudent` | ✅ **fixed** (was `POST /circles/{id}/students` → 404) |
| `POST /enrollments/{enrollment}/close` | `unenrollStudent(enrollmentId)` | ✅ **fixed (Finding B)** — uses `enrollment_id` from roster |

## Sessions & lifecycle
| API route | Admin | Status |
|---|---|---|
| `GET/POST/PUT/DELETE /sessions` (CircleSession) | `sessions_screen` | ⚠ verify create/update payload vs `CircleSessionRequest` |
| `POST /sessions/{id}/start\|complete\|cancel\|lock` | — | ❌ **no UI** for session lifecycle actions |
| `POST /students/{student}/suspend\|reactivate\|graduate\|withdraw\|transfer` | — | ❌ **no UI** for student lifecycle |

## Reports / statistics / platform
| API route | Admin | Status |
|---|---|---|
| `/reports/student-progress\|attendance\|teacher-performance\|parent-engagement\|circle-performance\|center-dashboard` | `reports_screen` | ⚠ partial (verify each report wired) |
| `/reports/admin-dashboard` (super_admin) | `dashboard` | ⚠ |
| `/statistics/{students\|teachers\|circles}/{id}` (+ recalculate) | `StatisticsRepository` | ⚠ read used; recalculate: ❌ no UI |
| `/audit-logs` | — | ❌ **no UI** |
| `/users` (index/show/store/deactivate) | — | ❌ **no UI** |
| `/health` | — | ❌ (ops-only, acceptable) |
| `/quran/surahs` | `QuranRepository` | ✅ |
| `/notifications` (+ read/read-all) | `notifications_screen` | ✅ |
| `/devices` | — | ❌ no UI (push not wired) |
| `/circles/{id}` etc. | — | — |

## Summary
- **Broken:** bulk/offline attendance (A), unenroll (B).
- **Fixed:** enroll route (B-enroll), memorization/revision suspended guard (C).
- **No UI (endpoint exists):** session lifecycle actions, student lifecycle transitions, audit-logs, users management, statistics recalculation, teacher/student bulk-delete, devices. → `MISSING_FEATURES.md`.
