# FRONTEND_INVENTORY.md

> Flutter app `altabari` (student + parent). Riverpod · Dio · go_router · flutter_secure_storage. Arabic-only, RTL, `maxWidth: 600` shell. Generated from a full read of `student_app/lib`.

## 1. App shell & entry
- `main.dart` — `ProviderScope` → `MaterialApp.router`; global `FlutterError`/`PlatformDispatcher` guards; RTL + light/dark themes.
- `app/router/app_router.dart` — `GoRouter`; redirect based on `currentUserProvider` (null → `/welcome`; logged-in on auth route → `/student` or `/parent` by role).
- `app/localization/`, `app/theme/` — Arabic locale, theme + `themeModeProvider`.

## 2. Network layer (`core/network`) — already API-ready
| File | Role |
|---|---|
| `api_config.dart` | `API_BASE_URL` via `--dart-define`, default `http://10.0.2.2:8000/api/v1`. |
| `api_client.dart` | `ApiClient` (Dio) + `apiClientProvider`; `get/post/put/delete`; maps `DioException`→`ApiException`. |
| `auth_interceptor.dart` | Injects `Authorization: Bearer <token>`; deletes token on 401. |
| `api_response.dart` | `ApiEnvelope<T>`, `PaginatedResult<T>`, `PaginationMeta` — **matches backend envelope exactly** (`success/message/data/meta.pagination`). |
| `api_exception.dart` | Typed `ApiException` (network/timeout/unauthorized/forbidden/validation/rateLimited/server…), parses `errors` + `error_code`, Arabic default messages. |
| `safe_log_interceptor.dart` | Debug-only logging. |

## 3. Services (`core/services`)
- `secure_storage_service.dart` — `TokenStorage` + `SecureStorageService` (secure token), `InMemoryTokenStorage` (tests).
- `offline_sync_service.dart` — `OfflineSyncService`: `cacheResponse/getCachedResponse` (SharedPreferences) + `enqueueRequest/clearQueue` (write queue). **Scaffold only — not wired to repos.**
- `toast_service.dart` — snackbars.

## 4. Data layer (`data/`)
**Models (`data/models`)** — domain-shaped, camelCase JSON (built for the mock source, not backend snake_case):
`AppUser, Halaqa, Student, Parent, ChildSummary` (user_models) · `MemorizationRecord, AttendanceLog, Homework, Assessment, Certificate, Payment, AppNotification, Badge, IconDataRef, Evaluation` (domain_models) · enums `UserRole, SessionType, RatingLevel, PrayerRating, HomeworkStatus, AttendanceStatus, NotificationType`.

**Repository interfaces (`data/repositories/repositories.dart`)** — `AuthRepository`, `StudentRepository`, `ParentRepository` (+ `BehaviorData/PrayerData` DTOs).

**Implementations (BEFORE this integration): 100% mock**
- `mock_auth_repository.dart`, `mock_student_repository.dart`, `mock_parent_repository.dart`, `datasources/mock_data_source.dart` (650 lines of seeded fake data).

**Providers (`data/providers/app_providers.dart`)** — Riverpod providers wired to the mock repos; `currentUserProvider`, `currentStudentProvider`, `homeworkProvider`, `memorizationRecordsProvider`, `revisionsProvider`, `assessmentsProvider`, `certificatesProvider`, `paymentsProvider`, `badgesProvider`, `attendanceProvider`, `notificationsProvider`, `currentParentProvider`, `childrenProvider`, `childDetailsProvider`, `evaluationsProvider`, `evaluationStateProvider`, `familyPaymentsProvider`, `parentNotificationsProvider`, `behaviorProvider`, `prayerProvider`, `childMemorization/Attendance/AssessmentsProvider`.

## 5. Screens (`features/*`)
**Auth (`features/auth/screens`)** — `splash, welcome, login, forgot, otp, reset, reset_success`. `auth_reset_flow.dart` holds email+reset-token between OTP/reset. `auth_error_message.dart` maps errors→Arabic.
> ⚠️ **Auth screens already call a REST-shaped repository** (`login(email:,password:,deviceName:)`, `forgotPassword(email)`, `verifyOtp(email:,code:)→String resetToken`, `resetPassword(email:,token:,password:,passwordConfirmation:)`) via `dynamic`. These signatures did **not** match the mock `AuthRepository` → auth was broken until this integration.

**Student (`features/student/screens`)** — `home, memorization, student_attendance, revisions, homework, assessments, achievements, certificates, payments, notifications, profile`. Shell: 5 tabs (home / memorization / attendance / homework / profile).

**Parent (`features/parent/screens`)** — `parent_home, children, child_profile, attendance, evaluation, behavior, prayer, parent_memorization, parent_assessments, parent_payments, parent_notifications, parent_profile`. Parent shell.

## 6. Local storage / offline
- Secure token: `flutter_secure_storage` (`itqan.auth.token`).
- Cache/queue scaffold: `OfflineSyncService` (SharedPreferences) — present, previously unused.

## 7. State of play at audit time
- Network + envelope + exception layer: **production-shaped and correct.**
- Data: **entirely mock.** Auth flow: **broken** (screens migrated to REST, repos not).
- Several screens depend on data the backend does not expose (see `API_MAPPING.md`).
