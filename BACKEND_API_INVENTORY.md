# BACKEND_API_INVENTORY.md

> Laravel 12 · Sanctum · feature-sliced (`app/Features/*`). Generated from a full read of `routes/api/v1/*`, controllers, resources, requests, and the shared response trait.

## 1. Conventions

**Base URL:** `/api/v1` (see `routes/api.php` → `routes/api/v1.php`).

**Auth:** Laravel Sanctum bearer tokens. Protected routes use `auth:sanctum` + `active` (account-active check). Role gates via `role:...` middleware. `throttle:*` named limiters on writes/sensitive ops. Some writes require an `idempotency` middleware (send `Idempotency-Key` header).

**Success envelope** (`ApiResponseTrait::success`):
```json
{ "success": true, "message": "…", "data": { … }, "meta": { … } }
```
`meta` present only when non-empty. Resource-collection lists put pagination under `meta.pagination`.

**Paginated envelope** (`ApiResponseTrait::paginate`):
```json
{ "success": true, "message": "…", "data": [ … ],
  "meta": { "pagination": { "total", "per_page", "current_page", "last_page", "from", "to", "has_more_pages" } } }
```
List size controlled by `?per_page=` (student endpoints clamp 1–100, default 20).

**Error envelope:**
```json
{ "success": false, "message": "…", "error_code": "…", "errors": { "field": ["…"] } }
```
Status codes: `401` unauthorized, `403` forbidden, `404` not found, `422` validation (with `errors`), `429` rate-limited, `5xx` server.

**Roles:** `super_admin`, `center_owner`, `teacher`, `parent`, `student`.

---

## 2. Endpoints relevant to the Student/Parent app

### 2.1 Authentication — `/auth` (`AuthController`)
| Method | Path | Auth | Body / Notes | Response `data` |
|---|---|---|---|---|
| POST | `/auth/login` | public (`throttle:login`) | `email`, `password`, `device_name?` | `{ user: UserResource, token: { token, type:"Bearer", name, expires_at } }` |
| POST | `/auth/forgot-password` | public | `email` | `null` (always 200, no account disclosure) |
| POST | `/auth/verify-otp` | public | `email`, `code` (4 digits) | `{ reset_token }` |
| POST | `/auth/reset-password` | public | `email`, `token`, `password`, `password_confirmation` | `null` |
| POST | `/auth/logout` | sanctum | — | `null` |
| POST | `/auth/logout-all` | sanctum | — | `null` |
| GET | `/auth/me` | sanctum | — | `UserResource` |
| GET | `/auth/devices` | sanctum | — | `[TokenDeviceResource]` |
| DELETE | `/auth/devices/{token}` | sanctum | — | `null` |
| PUT | `/auth/device-name` | sanctum | `device_name` | `null` |

**UserResource:** `id, name, email, role:{value,label}, teacher_id, student_id, center_ids[], locale, timezone, is_active, email_verified_at, created_at`. ⚠️ No `phone`, no `avatar`.

### 2.2 Student self-service — `/student` (`role:student`) (`Student\StudentController`)
All operate on the authenticated student (path ignores any id).
| Method | Path | Response `data` |
|---|---|---|
| GET | `/student/profile` | `{ id, name, phone, email, guardian_id, status, center:{id,name}, circle:{id,name,level,teacher:{id,name}}, statistics:{ memorized_pages, memorized_juz, completed_surahs, attendance_rate, average_grade, last_calculated_at } }` |
| GET | `/student/dashboard` | `{ student:<profile>, counts:{ pending_homework, memorization_sessions, revision_sessions, assessments }, next_session:{ id, circle_id, session_date, start_time, end_time, status }? }` |
| GET | `/student/memorization-sessions` | paginated `[MemorizationSessionResource]` |
| GET | `/student/revision-sessions` | paginated `[RevisionSessionResource]` |
| GET | `/student/attendance` | paginated `[AttendanceResource]` |
| GET | `/student/homework` | paginated `[HomeworkResource]` |
| GET | `/student/assessments` | paginated `[AssessmentResource]` |
| GET | `/student/sessions` | paginated `[CircleSessionResource]` |
| GET | `/student/learning-records` | paginated `[LearningRecordResource]` |

### 2.3 Parent — `/parent` (`role:parent`) (`Parent\ParentController`)
| Method | Path | Body | Response `data` |
|---|---|---|---|
| GET | `/parent/my-children` | — | `[{ id, name, status, center:{id,name}, relationship, is_primary }]` |
| GET | `/parent/my-children/{student}/sessions` | — | `[{ id, circle_id, circle_name, teacher_id, teacher_name, session_date, start_time, end_time, status, completed_at, locked_at }]` |
| GET | `/parent/dashboard` | — | `{ …engagement counts (int/float) }` |
| GET | `/parent/evaluations` | — | paginated `[ParentEvaluationResource]` |
| POST | `/parent/evaluations` | `circle_session_id`, `student_id`, `behavior_score` 1–5, `prayer_score` 1–5, `homework_followup_score` 1–5, `notes?` | `ParentEvaluationResource` (201) |
| PUT | `/parent/evaluations/{evaluation}` | scores + `notes?` | `ParentEvaluationResource` |

**ParentEvaluationResource:** `id, circle_session_id, student_id, guardian_id, behavior_score, prayer_score, homework_followup_score, notes, student{id,name}?, session{id,circle_id,session_date,status,locked_at}?, created_at, updated_at`.

### 2.4 Notifications — `/notifications` (`NotificationController`)
| Method | Path | Response |
|---|---|---|
| GET | `/notifications` | paginated `[{ id, type, data{}, read_at, created_at, updated_at }]` |
| POST | `/notifications/read-all` | `null` |
| POST | `/notifications/{notification}/read` | `null` |

⚠️ Title/body live inside `data{}`; there is no dedicated title/body column.

### 2.5 Devices (push) — `/devices` (`DeviceController`)
| Method | Path | Body |
|---|---|---|
| GET | `/devices` | — |
| POST | `/devices` | device registration (`idempotency`) |
| DELETE | `/devices/{device}` | — |

### 2.6 Quran reference — `/quran`
| Method | Path | Response |
|---|---|---|
| GET | `/quran/surahs` | `[Surah]` (not enveloped as list-meta; plain `data`) |
| GET | `/quran/surahs/{surahNumber}` | `Surah` |

---

## 3. Endpoints NOT for the student app (admin/teacher surface — belong to `admin_app`)

`/centers`, `/guardians`, `/teachers` (+ `/salary`, `/bulk-delete`), `/students` CRUD (+ lifecycle: suspend/reactivate/graduate/withdraw/transfer), `/circles` (+ `/students`, `/enrollments`), `/circle-schedules`, `/memorization-sessions` CRUD, `/revision-sessions` CRUD, `/attendance` CRUD (+ `/bulk`), `/assessments` CRUD, `/homework` CRUD, `/statistics/*` (+ recalculate), `/sessions` (circle-session lifecycle: start/complete/cancel/lock), `/learning-records` CRUD (`role: teacher+`), `/reports/*`, `/users` (+ deactivate), `/audit-logs`, `/health`.

These are **out of scope** for the student/parent app and are correctly gated behind `role:super_admin,center_owner,teacher`.

---

## 4. Feature modules discovered
`Assessments, Attendance, AuditLogs, Auth, Centers, CircleSchedules, CircleSessions, Circles, Devices, Enrollments, Guardians, Health, Homework, LearningRecords, MemorizationSessions, Notifications, Parent, Push, Reports, RevisionSessions, Statistics, Student, StudentLifecycle, Students, Teachers, Users.`

Each slice: `Controllers/ Requests/ Resources/ Services/ Actions/ Queries/ Policies/`. Consistent `BaseController` + `ApiResponseTrait`; resources extend `BaseResource`.
