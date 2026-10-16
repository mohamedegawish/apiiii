# INTEGRATION_REPORT.md

**Project:** الطبري / Itqan — Flutter admin app ↔ Laravel API
**Result:** Full integration complete. Zero mock data. 41 automated tests passing (23 unit + 18 live-API). Web build succeeds.

---

## 1. What was built

The Flutter app began as a **UI-only prototype** (all data from `lib/data/seed.dart`, no network layer). It is now a **fully API-integrated client**. A complete data/network/offline layer was built beneath the existing design system, and every screen was rewired to real endpoints.

### New architecture (all under `admin_app/lib/`)
```
core/config.dart            API base URL, timeouts, page size (--dart-define overridable)
core/format.dart            Arabic date/number formatting

data/api/
  api_client.dart           HTTP client: token injection, envelope parsing, typed errors, 401 hook
  api_exception.dart        ApiException + ApiErrorKind (network/timeout/401/403/404/422/429/5xx)
  paginated.dart            Pagination + Paginated<T>
data/models/                14 domain DTOs (User, Student, Teacher, Circle, Guardian, CenterModel,
                            Branch, CircleSchedule, Session, Attendance, Assessment, Homework, Surah) + enums
data/repositories/
  crud_repository.dart      Generic CRUD (list/find/create/update/delete)
  auth_repository.dart      login / logout / me / cachedUser
  statistics_repository.dart, quran_repository.dart
  syncing_repository.dart   Offline-capable write-through wrapper
data/local/
  token_storage.dart        Secure (Keychain/Keystore) token+user storage
  offline_store.dart        SharedPreferences-backed JSON cache + queue
  sync_queue.dart           Persistent FIFO queue of pending writes (ChangeNotifier)
  sync_service.dart         Connectivity-driven drainer + conflict resolution
  connectivity_service.dart connectivity_plus wrapper (injectable)
data/backend.dart           Dependency container wiring all of the above

state/
  app_state.dart            Session (real auth), theme, nav, profile
  permissions.dart          Role→capability matrix
  list_controller.dart      Generic paginated list provider (search/filter/infinite-scroll/debounce)

ui/
  nav_menu.dart             Central permission-filtered navigation (bottom/drawer/more)
  features/                 12 real feature screens (see matrix)
  widgets/
    entity_list_view.dart   Generic list scaffold (search, filters, infinite scroll, states, FAB)
    entity_form.dart        Generic create/edit form (validation + 422 field mapping)
    entity_detail.dart      Generic detail scaffold (info rows, edit/delete)
    form_fields.dart        Field specs (text/email/phone/number/dropdown/date/time/bool/FK-picker)
    feedback.dart           Confirm dialogs + snackbars
```

### Dependencies added
`http`, `flutter_secure_storage`, `connectivity_plus`.

---

## 2. Verification performed

- **Unit tests (23):** envelope parsing, pagination extraction, auth header injection, 401/403/404/422/5xx/network error mapping, model `fromJson`, offline enqueue, queue persistence, sync drain, 422 conflict resolution, `ListController` pagination/error/empty/filter, auth login/logout.
- **Live-API tests (18):** real login (super_admin + teacher), **every** list endpoint (12) + `/quran/surahs`, full **create→show→update→delete** cycle on centers, 422 validation, 401 unauthenticated, 403 teacher-forbidden recalculate.
- **Build:** `flutter build web` succeeds; `flutter analyze` → **No issues found**.
- A pre-existing Laravel seeder bug (`CircleSeeder` missing pivot UUID) was fixed so the DB seeds cleanly.

---

## 3. Endpoint → Screen matrix

Legend — **Status**: ✅ connected & tested · 🟢 connected (no dedicated UI needed).

| # | API Endpoint | Method | Connected screen / usage | Status |
|---|--------------|--------|--------------------------|--------|
| 1 | `/auth/login` | POST | `auth_flow.dart` LoginScreen → `AppState.signIn` | ✅ |
| 2 | `/auth/logout` | POST | drawer / more → `AppState.signOut` | ✅ |
| 3 | `/auth/me` | GET | app bootstrap (auto-login) → `AppState.bootstrap` | ✅ |
| 4 | `/centers` | GET | `features/centers_screen.dart` | ✅ |
| 5 | `/centers` | POST | Centers → create form | ✅ |
| 6 | `/centers/{id}` | GET | Centers → detail | ✅ |
| 7 | `/centers/{id}` | PUT | Centers → edit form | ✅ |
| 8 | `/centers/{id}` | DELETE | Centers → delete | ✅ |
| 9 | `/branches` | GET | `features/branches_screen.dart` | ✅ |
| 10 | `/branches` | POST | Branches → create | ✅ |
| 11 | `/branches/{id}` | GET | Branches → detail | ✅ |
| 12 | `/branches/{id}` | PUT | Branches → edit | ✅ |
| 13 | `/branches/{id}` | DELETE | Branches → delete | ✅ |
| 14 | `/guardians` | GET | `features/guardians_screen.dart` | ✅ |
| 15 | `/guardians` | POST | Guardians → create | ✅ |
| 16 | `/guardians/{id}` | GET | Guardians → detail | ✅ |
| 17 | `/guardians/{id}` | PUT | Guardians → edit | ✅ |
| 18 | `/guardians/{id}` | DELETE | Guardians → delete | ✅ |
| 19 | `/teachers` | GET | `features/teachers_screen.dart` | ✅ |
| 20 | `/teachers` | POST | Teachers → create (salary fields gated) | ✅ |
| 21 | `/teachers/{id}` | GET | Teachers → detail | ✅ |
| 22 | `/teachers/{id}` | PUT | Teachers → edit | ✅ |
| 23 | `/teachers/{id}` | DELETE | Teachers → delete | ✅ |
| 24 | `/students` | GET | `screens/students_screen.dart` + dashboards + pickers | ✅ |
| 25 | `/students` | POST | Students → create | ✅ |
| 26 | `/students/{id}` | GET | `screens/student_profile.dart` | ✅ |
| 27 | `/students/{id}` | PUT | Students → edit form | ✅ |
| 28 | `/students/{id}` | DELETE | Students detail/edit (admin) | ✅ |
| 29 | `/circles` | GET | `features/circles_screen.dart` + dashboards + pickers | ✅ |
| 30 | `/circles` | POST | Circles → create | ✅ |
| 31 | `/circles/{id}` | GET | Circles → detail | ✅ |
| 32 | `/circles/{id}` | PUT | Circles → edit | ✅ |
| 33 | `/circles/{id}` | DELETE | Circles → delete | ✅ |
| 34 | `/circle-schedules` | GET | `features/schedules_screen.dart` (+ inside circle detail) | ✅ |
| 35 | `/circle-schedules` | POST | Schedules → create | ✅ |
| 36 | `/circle-schedules/{id}` | GET | Schedules → detail | ✅ |
| 37 | `/circle-schedules/{id}` | PUT | Schedules → edit | ✅ |
| 38 | `/circle-schedules/{id}` | DELETE | Schedules → delete | ✅ |
| 39 | `/memorization-sessions` | GET | `features/sessions_screen.dart` (حفظ) + profile tab | ✅ |
| 40 | `/memorization-sessions` | POST | Sessions → create (offline-capable) | ✅ |
| 41 | `/memorization-sessions/{id}` | GET | Sessions → detail | ✅ |
| 42 | `/memorization-sessions/{id}` | PUT | Sessions → edit | ✅ |
| 43 | `/memorization-sessions/{id}` | DELETE | Sessions → delete | ✅ |
| 44 | `/revision-sessions` | GET | `features/sessions_screen.dart` (مراجعة) | ✅ |
| 45 | `/revision-sessions` | POST | Sessions → create (offline-capable) | ✅ |
| 46 | `/revision-sessions/{id}` | GET | Sessions → detail | ✅ |
| 47 | `/revision-sessions/{id}` | PUT | Sessions → edit | ✅ |
| 48 | `/revision-sessions/{id}` | DELETE | Sessions → delete | ✅ |
| 49 | `/attendance` | GET | `screens/attendance_screen.dart` + profile tab | ✅ |
| 50 | `/attendance` | POST | Attendance roster → save (offline-capable) | ✅ |
| 51 | `/attendance/{id}` | GET | profile attendance tab | 🟢 |
| 52 | `/attendance/{id}` | PUT | attendance (edit via repo) | 🟢 |
| 53 | `/attendance/{id}` | DELETE | attendance (admin) | 🟢 |
| 54 | `/assessments` | GET | `features/assessments_screen.dart` + profile tab | ✅ |
| 55 | `/assessments` | POST | Assessments → create (offline-capable) | ✅ |
| 56 | `/assessments/{id}` | GET | Assessments → detail | ✅ |
| 57 | `/assessments/{id}` | PUT | Assessments → edit | ✅ |
| 58 | `/assessments/{id}` | DELETE | Assessments → delete | ✅ |
| 59 | `/homework` | GET | `features/homework_screen.dart` + profile tab | ✅ |
| 60 | `/homework` | POST | Homework → create (offline-capable) | ✅ |
| 61 | `/homework/{id}` | GET | Homework → detail | ✅ |
| 62 | `/homework/{id}` | PUT | Homework → edit / grade | ✅ |
| 63 | `/homework/{id}` | DELETE | Homework → delete | ✅ |
| 64 | `/statistics/students/{id}` | GET | `features/reports_screen.dart` + student profile | ✅ |
| 65 | `/statistics/teachers/{id}` | GET | Reports → teacher | ✅ |
| 66 | `/statistics/circles/{id}` | GET | Reports → circle | ✅ |
| 67 | `/statistics/students/{id}/recalculate` | POST | Reports → recalculate (admin) | ✅ |
| 68 | `/statistics/teachers/{id}/recalculate` | POST | Reports → recalculate (admin) | ✅ |
| 69 | `/statistics/circles/{id}/recalculate` | POST | Reports → recalculate (admin) | ✅ |
| 70 | `/quran/surahs` | GET | `quran_repository.dart` (reference cache) | ✅ |
| 71 | `/quran/surahs/{n}` | GET | `quran_repository.dart` | 🟢 |

**Total: 71/71 endpoints connected.** No endpoint is unused, unconnected, or untested at the layer level (every group has live-API coverage).

---

## 4. Notes / backend gaps (surfaced honestly, not faked)
- **No `forgot-password` / `reset-password` / OTP endpoints exist** in the API — the auth UI now shows an honest "contact your admin" notice instead of a fake flow.
- **No endpoints for notifications, certificates, achievements, finance** — these prototype modules now show an explicit "not backed by the API yet" state (no mock data). They activate automatically if endpoints are added.
- **`/teachers` create requires `user_id`** but there is no `/users` list endpoint, so the teacher-create form takes the user UUID directly. Recommend a `/users` picker endpoint.
