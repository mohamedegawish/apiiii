# INTEGRATION_TEST_REPORT.md

> Phase 11. Verification of the student/parent app ↔ Laravel API integration.
> Tooling present: Flutter 3.44.4, Dart 3.12.2, PHP 8.2. **No live backend was running during this pass**, so verification is: (a) static analysis, (b) automated unit tests of the mapping + auth layer against the backend's exact response contracts, (c) contract review of every endpoint against its controller/resource.

## 1. Automated results
| Check | Command | Result |
|---|---|---|
| Static analysis | `flutter analyze` | **No issues found** (0 errors, 0 warnings, 0 infos) |
| Unit / mapping tests | `flutter test` | **15 passed** |

**Test files added:**
- `test/api_mappers_test.dart` — 9 tests: user/role mapping (+ unsupported-role rejection), student profile/statistics, memorization (surah name + verses + type), homework status, attendance status, notification title/body-from-`data` + read state, evaluation score→rating.
- `test/api_auth_repository_test.dart` — 5 tests (stubbed Dio): login persists token, session restore via `/auth/me`, null when no token, verify-otp reset-token, logout clears token.
- `test/widget_test.dart` — bootstrap smoke.

## 2. Endpoint-by-endpoint status
Legend: ✅ connected & mapping unit-tested · 🟡 connected, contract-reviewed (needs live smoke) · ⚠ partial (backend/UI gap) · ❌ no backend.

### Auth
| Endpoint | Read/Write | Status |
|---|---|---|
| POST `/auth/login` | W | ✅ (token persisted — tested) |
| GET `/auth/me` | R | ✅ (session restore — tested) |
| POST `/auth/logout` | W | ✅ (token cleared — tested) |
| POST `/auth/forgot-password` | W | 🟡 |
| POST `/auth/verify-otp` | W | ✅ (reset token — tested) |
| POST `/auth/reset-password` | W | 🟡 |

### Student
| Endpoint | Status |
|---|---|
| GET `/student/profile` | ✅ (mapping tested) |
| GET `/student/dashboard` | 🟡 (counts available; home uses profile) |
| GET `/student/memorization-sessions` | ✅ |
| GET `/student/revision-sessions` | ✅ |
| GET `/student/attendance` | ✅ |
| GET `/student/homework` | ✅ |
| GET `/student/assessments` | ✅ |
| GET `/student/sessions` | ⚠ endpoint exists; no repo/screen |
| GET `/student/learning-records` | ⚠ endpoint exists; no repo/screen |

### Notifications
| Endpoint | Status |
|---|---|
| GET `/notifications` | ✅ (mapping tested) |
| POST `/notifications/{id}/read` | 🟡 |
| POST `/notifications/read-all` | ⚠ no button |

### Parent
| Endpoint | Status |
|---|---|
| GET `/parent/my-children` | 🟡 (mapped; fields limited by backend) |
| GET `/parent/dashboard` | 🟡 |
| GET `/parent/evaluations` | ✅ (mapping tested) |
| POST `/parent/evaluations` | ⚠ needs session-picker UI (`circle_session_id`) |
| PUT `/parent/evaluations/{id}` | ⚠ no edit UI |
| GET `/parent/my-children/{id}/sessions` | ⚠ no screen |

### No backend (rendered as "not available")
`certificates, payments, badges, behavior, prayer, per-child parent records` — ❌.

## 3. CRUD / pagination / permissions review
- **Pagination:** `PaginatedResult`/`PaginationMeta` parse `meta.pagination` exactly as `ApiResponseTrait::paginate` emits it; repos request `per_page=100`. ✅
- **Read (R):** all student lists + notifications + parent children/evaluations mapped. ✅
- **Create/Update/Delete:** the student app is read-only except parent-evaluation create/update (UI-gated) — matches backend role gates (`role:student` is read-only on `/student/*`). ✅
- **Permissions:** app rejects non-student/parent roles at mapping time (`unsupported_role`) and honors 401→logout. Backend `role:*` gates remain authoritative. ✅
- **Validation errors (422):** surfaced via `ApiException.fieldErrors` + `authErrorMessage` (used by all auth screens). ✅

## 4. Required to reach full "green" (live smoke)
Run against a seeded instance:
```
cd api && php artisan serve            # http://127.0.0.1:8000
cd student_app && flutter run --dart-define=API_BASE_URL=http://10.0.2.2:8000/api/v1
```
Then log in as a seeded student and a seeded parent and walk each screen. The mapping/auth contracts are unit-verified; live smoke confirms seed-data shapes and the 🟡 rows.
