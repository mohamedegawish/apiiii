# API_STUDENT_MAPPING.md

> `student_app` (student + parent). Detailed per-endpoint mapping lives in [`API_MAPPING.md`](API_MAPPING.md); this is the cross-audit summary. The student app was fully integrated in the prior pass (mock→REST); `flutter analyze` clean, 15 unit tests passing.

## Student role — all connected ✅
| API | Screen → provider → repo | Status |
|---|---|---|
| `GET /student/profile` (+`/dashboard`) | Home/Profile → `currentStudentProvider` → `ApiStudentRepository` | ✅ |
| `GET /student/memorization-sessions` | Memorization → `memorizationRecordsProvider` | ✅ |
| `GET /student/revision-sessions` | Revisions | ✅ |
| `GET /student/attendance` | Attendance | ✅ |
| `GET /student/homework` | Homework | ✅ |
| `GET /student/assessments` | Assessments | ✅ |
| `GET /student/sessions` | — | ⚠ endpoint available, no screen |
| `GET /student/learning-records` | — | ⚠ endpoint available, no screen |
| `GET /notifications` (+read) | Notifications | ✅ |

## Auth — connected ✅
`login`, `me` (session restore), `logout`, `forgot-password`, `verify-otp`, `reset-password`, global 401→logout. Token in secure storage. Non-student/parent roles rejected (`unsupported_role`).

## Parent role — partial (backend-bounded) ⚠
| API | Status |
|---|---|
| `GET /parent/my-children` | ⚠ mapped; backend returns only id/name/status/center/relationship |
| `GET /parent/dashboard` | ⚠ mapped |
| `GET /parent/evaluations` | ✅ read |
| `POST /parent/evaluations` | ⚠ needs session-picker UI (`circle_session_id`) — gated |
| `GET /parent/my-children/{id}/sessions` | ⚠ repo-ready, no screen |
| per-child memorization/attendance/assessments/behavior/prayer | ❌ no parent endpoint → "not available" |

## No backend (rendered "not available", no fake data) ❌
`certificates`, `payments`, `badges/achievements`, `behavior analytics`, `prayer tracking`.

## Cross-app consistency notes
- The student app **reads** memorization/revision (`/student/*`, `role:student`); the admin app **writes** them (`/memorization-sessions`, `/revision-sessions`). Fix C now guards the write path against suspended students, so a suspended student cannot accumulate new records that the student app would later display. ✅ consistent.
- Notifications endpoint is shared and role-agnostic — used by both student and parent screens. ✅
