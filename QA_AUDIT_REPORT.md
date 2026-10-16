# QA Audit Report — Al-Tabari Mobile Apps

**Date:** 2026-09-14
**Scope:** `student_app` (package `altabari`, student + guardian, Riverpod + go_router + Dio) and `admin_app` (package `itqan`, super_admin / center_owner / teacher, Provider + state-based navigation + http). Backend `api/` (Laravel) was read only to verify contracts.
**Method:** Static review of all startup, auth, routing, repository, and core-flow code; cross-check of every API call against `php artisan route:list` (196 routes); `flutter analyze` + `flutter test` on both apps; temporary reproduction tests for the top findings (deleted afterwards — no project files were changed).

**Legend:** **CONFIRMED** = proven by a failing reproduction test, a build artifact, or a direct code path with no alternative. **PLAUSIBLE** = real code risk that needs a specific runtime condition to trigger.

---

## A. Executive Summary

**Overall health: fair for the code, not ready for release.** Both apps are well structured. The analyzer is clean and all existing tests pass. Error handling is mostly deliberate. The API contracts are almost entirely correct: every endpoint the student app calls exists, and so does every admin endpoint except `/centers`, which the app never reaches. However, four P0 problems block a real release:
- The student app's release build has no network access.
- Unticking "Remember me" makes student login unusable.
- A 401 silently deletes teachers' unsynced offline data.
- The admin app will crash on iOS when a QR scanner opens.

| Metric | Count |
|---|---|
| Critical (P0) | 4 |
| High (P1) | 8 |
| Medium (P2) | 14 |
| Low (P3) | 10 |
| Missing screens/features: confirmed | 4 |
| Missing screens/features: likely | 2 |
| Broken navigation issues | 7 |
| Crash risks | 6 |

**Automated checks run**

| Check | admin_app | student_app |
|---|---|---|
| `flutter analyze` | No issues | No issues |
| `flutter test` | 85 passed, 21 skipped (live API) | 60 passed |
| Reproduction tests written for this audit | 2 / 2 failed (bug confirmed) | 3 / 3 failed (bug confirmed) |

---

## B. Critical Issues (P0)

### P0-1 — Student app release build cannot access the internet — CONFIRMED
- **File:** [student_app/android/app/src/main/AndroidManifest.xml](student_app/android/app/src/main/AndroidManifest.xml)
- **Why it fails:** `android.permission.INTERNET` is declared only in `src/debug` and `src/profile` (Flutter template default), not in `src/main`. None of the plugins add it back. The existing merged release manifest `student_app/build/app/intermediates/merged_manifests/release/processReleaseManifest/AndroidManifest.xml` contains **0** `INTERNET` entries. The already-built `build/app/outputs/flutter-apk/app-release.apk` has this problem.
- **Reproduction:** `flutter build apk --release` → install → log in → "تعذر الاتصال بالخادم" on every request. It works in `flutter run` (debug), which is why it went unnoticed.
- **Fix:** Add `<uses-permission android:name="android.permission.INTERNET" />` to the main manifest (admin_app already has it).

### P0-2 — Unticking "تذكرني" (Remember me) logs the student out immediately — CONFIRMED (code path)
- **File:** [student_app/lib/features/auth/screens/login_screen.dart:74-77](student_app/lib/features/auth/screens/login_screen.dart#L74-L77)
- **Function:** `_LoginScreenState._login`
- **Why it fails:** After a successful login, `deleteToken()` runs when `_rememberMe == false`. `SecureStorageService` has no in-memory copy and `AuthInterceptor.onRequest` reads the token from storage on each request. The first dashboard call therefore goes out without `Authorization`, gets a 401, and `onUnauthorized` clears `currentUserProvider`. The router then redirects to `/welcome`.
- **Reproduction:** Login screen → untick "تذكرني" → valid credentials → home flashes → redirected to Welcome.
- **Fix:** Keep the token in memory for the session: add an in-memory cache to `SecureStorageService`, or add a `persist: false` mode that stores only in memory. Don't delete the only copy.

### P0-3 — Any 401 wipes the teacher's unsynced offline queue (data loss) — CONFIRMED (repro test)
- **Files:** [admin_app/lib/data/api/api_client.dart:187-189](admin_app/lib/data/api/api_client.dart#L187-L189), [admin_app/lib/state/app_state.dart:345-360](admin_app/lib/state/app_state.dart#L345-L360)
- **Functions:** `ApiClient._handle` → `AppState._handleUnauthorized` → `_forceSignOut` → `_clearLocalUserData` → `syncQueue.clear()`
- **Why it fails:** Tokens expire after 30 days (`SANCTUM_TOKEN_EXPIRATION_MINUTES=43200` in `api/.env`, counted from creation) and can also be revoked via `auth/devices`/`logout-all`. If a teacher recorded attendance or evaluations offline, the first sync after reconnecting receives a 401. Every pending operation is then deleted without a warning.
- **Reproduction (test):** Queue one `/attendance` POST → mock server returns 401 → `syncService.sync()` → `pendingCount` goes from 1 to 0.
- **Fix:** On a *forced* sign-out, keep the queue, tagged with the owner's user id. After the same user logs back in, resume syncing. Only discard it when a *different* user logs in, and warn first. At minimum, show a blocking "N unsynced records will be lost" dialog.

### P0-4 — Admin app on iOS is killed when any QR scanner opens — CONFIRMED (config)
- **File:** [admin_app/ios/Runner/Info.plist](admin_app/ios/Runner/Info.plist) (no `NSCameraUsageDescription`)
- **Why it fails:** `mobile_scanner` needs the camera. iOS terminates any app that accesses the camera without this key. Entry points: the QR icon in the admin top bar, the payment QR scanner in Accounts, and the offline QR scanner.
- **Reproduction:** iOS build → admin top bar → QR icon → app crashes.
- **Severity note:** P0 only if iOS is shipped (`flutter_launcher_icons` has `ios: true` for admin_app).
- **Fix:** Add `<key>NSCameraUsageDescription</key><string>لمسح رمز QR الخاص بالطالب</string>`.

---

## C. Crash Risks

| # | Status | File | Risk | Trigger |
|---|---|---|---|---|
| C1 | CONFIRMED | `admin_app/ios/Runner/Info.plist` | iOS terminates the app (P0-4) | Open a QR scanner on iOS |
| C2 | CONFIRMED | [qr_scanner_screen.dart:53-58](admin_app/lib/ui/screens/qr_scanner_screen.dart#L53-L58), [qr_payment_scanner_screen.dart:54-58](admin_app/lib/ui/screens/qr_payment_scanner_screen.dart#L54-L58), [offline_qr_scanner_screen.dart:92-97](admin_app/lib/ui/screens/offline_qr_scanner_screen.dart#L92-L97) | `setState()` called after `dispose()`: `mounted` is checked *before* `await Future.delayed(...)` but not after | Scan an invalid code, then press back within 2 s. Logged FlutterError (red screen in debug) |
| C3 | PLAUSIBLE | [admin_app/lib/main.dart:78](admin_app/lib/main.dart#L78), [app_state.dart:160-200](admin_app/lib/state/app_state.dart#L160-L200) | `bootstrap()` is not awaited and catches only `ApiException`. A `PlatformException` from secure storage, or a `TypeError` from malformed `/auth/me` JSON, leaves `phase == loading` forever (infinite spinner), with no global error handler | Known `flutter_secure_storage` decryption failure after Android backup restore (`allowBackup` defaults to true) |
| C4 | PLAUSIBLE | [session_workspace_screen.dart:39-55](admin_app/lib/ui/teacher/session_workspace/session_workspace_screen.dart#L39-L55), [backend.dart:178,374](admin_app/lib/data/backend.dart#L374) | `load()` catches only `ApiException`. `(res.data as List)` casts in `sessionRoster`/`circleStudents` throw `TypeError`, so `_starting` never becomes false and the workspace spins forever | Backend returns a non-list `data` (e.g. contract change or proxy error page with 200) |
| C5 | CONFIRMED | [student_app/lib/app/router/app_router.dart:222-229](student_app/lib/app/router/app_router.dart#L222-L229) | `state.extra as Map` with a non-null cast. `extra` is lost on web refresh or deep link | Web: open `/parent/evaluation-detail` and refresh. TypeError in route builder |
| C6 | PLAUSIBLE | [student_app/lib/main.dart:27-30](student_app/lib/main.dart#L27-L30) | `PlatformDispatcher.onError` returns `true` for **all** async errors. Crashes are hidden instead of reported, and there is no crash reporting at all | Any unexpected async exception is silently dropped (masks bugs in production) |

---

## D. Broken Navigation

| # | Status | App | File | Problem | Reproduction |
|---|---|---|---|---|---|
| D1 | CONFIRMED (repro) | student | [student_shell.dart:38-42](student_app/lib/features/student/shells/student_shell.dart#L38-L42), [parent_shell.dart:36-40](student_app/lib/features/parent/shells/parent_shell.dart#L36-L40) | Bottom nav always highlights the **first** tab. `indexWhere(location.startsWith(path))` matches `/student` (or `/parent`) before any more specific path | Tap "الواجبات": content changes, but "الرئيسية" stays highlighted |
| D2 | CONFIRMED | admin | [admin_app/lib/ui/screens/qr_scanner_screen.dart:43-48](admin_app/lib/ui/screens/qr_scanner_screen.dart#L43-L48) | Successful scan calls `state.openProfile()` but never pops the scanner route. The student profile opens *underneath* a stopped camera, so the scan looks like it did nothing | Top bar QR → scan valid student code → camera freezes, nothing visible happens → press back → profile is there |
| D3 | CONFIRMED | student | `sessions_screen.dart`, `my_qr_screen.dart`, `revisions_screen.dart`, `assessments_screen.dart`, `certificates_screen.dart`, `achievements_screen.dart` | Opened with `context.go()` from Home quick actions, so there is no back stack and no back arrow (`AppBar` implies leading only when `canPop`). Android system back **exits the app** | Home → "سجلّ الجلسات" → Android back → app closes |
| D4 | CONFIRMED | admin | [admin_app/lib/ui/shell.dart](admin_app/lib/ui/shell.dart), [teacher_shell.dart](admin_app/lib/ui/teacher/teacher_shell.dart) | Navigation is an `AppState.screen` enum inside a single route with no `PopScope`, so Android back **exits the app** from any screen (student profile, circles, settings…) | Admin → Students → open a student → Android back → app closes |
| D5 | CONFIRMED | student | [child_profile_screen.dart:215](student_app/lib/features/parent/screens/child_profile_screen.dart#L215) | "المدفوعات" from a child's profile opens `/parent/payments` without the child id, so it always shows the **first** child | Parent with 2 children → open 2nd child → المدفوعات → 1st child's payments shown |
| D6 | CONFIRMED | student | [app_router.dart:80-93](student_app/lib/app/router/app_router.dart#L80-L93) | `/select-role` receives mobile + **password** via `extra`. On web refresh both become `''`, so select-role always fails with no way back except the text button | Web: login with shared mobile → refresh on role screen → "Continue" errors |
| D7 | CONFIRMED | both | see §E | Implemented routes/screens with **no entry point**: student `/student/payments`, `/parent/child/:id/assessments`, `MoreMenuSheet`, `ParentMoreSheet`. Admin `AttendanceScreen`, `SessionsScreen` (memo), `CircleSessionsScreen`, `MatnTypesScreen`, `PayrollScreen`, `ExecutiveDashboardScreen`, `CentersScreen` | — |

No infinite redirect loops were found. The earlier splash loop is fixed and covered by `router_provider_test.dart`. Auth guards are correct in both apps.

---

## E. Missing Screens / Features

### Confirmed missing (backend supports it, no reachable UI)
1. **Edit profile (student & guardian).** `PUT /auth/me` exists (name/email/phone). The app has a "تعديل الملف الشخصي" button that does nothing ([profile_screen.dart:120](student_app/lib/features/student/screens/profile_screen.dart#L120), [parent_profile_screen.dart:107](student_app/lib/features/parent/screens/parent_profile_screen.dart#L107)). The repository deliberately throws `featureUnavailable` ([api_student_repository.dart:134-141](student_app/lib/data/repositories/api_student_repository.dart#L134-L141)). The admin app already implements this flow.
2. **Mark notifications as read (student & guardian).** `POST /notifications/{id}/read` and `/read-all` exist, and `markNotificationRead()` exists in both repositories but is **never called**. The Home unread badge therefore never clears.
3. **Notifications in admin/teacher app.** The top-bar bell opens a static "not enabled" placeholder ([admin_app/lib/ui/screens/notifications_screen.dart](admin_app/lib/ui/screens/notifications_screen.dart)), although `GET /notifications` exists.
4. **Matn catalog management (admin).** `MatnTypesScreen` is complete, the `manageCatalog` permission exists, and `NavMenu._canSee` has a `matnCatalog` case. The screen is **missing from `NavMenu._all`**, so it is unreachable. Teachers' matn evaluations depend on this catalog. Unlike payroll and the executive dashboard, no code comment says it was removed on purpose.

### Likely missing
1. **Password reset for students without email.** Students log in by code or mobile, but Forgot Password only accepts email (backend `forgot-password` validates `email`). Students whose accounts have no email have no self-service recovery.
2. **Offline access for teachers from the Today screen.** The app has a full offline queue, but [`Backend.todaySessions()`](admin_app/lib/data/backend.dart#L275-L289) is online-only with no cache. Offline, the Today screen shows an error and the teacher cannot open a session workspace. The offline attendance/evaluation features are unreachable from the main teacher path (see P1-6).

### Not enough evidence (intentional or no backend)
- Certificates, achievements, student payments: no backend endpoints. Placeholder screens are acceptable, but Home quick actions link to two of them (P3).
- Admin payroll runs / executive dashboard: complete screens, but a comment in [nav_menu.dart:30-32](admin_app/lib/ui/nav_menu.dart#L30-L32) says they were removed from navigation on purpose.
- Centers: backend has no `/centers` routes (single-center install). `CentersScreen` would 404, but it is hidden.
- `GET /parent/dashboard`, `auth/devices`, `logout-all`, `students/{id}/tracking/history`: unused by the apps, and there's no evidence a screen was planned.

### Placeholder screens
`CertificatesScreen`, `AchievementsScreen`, `student PaymentsScreen`, `ParentAssessmentsScreen` (all `FeatureUnavailableScreen`), and admin `NotificationsScreen`.

### Buttons that do nothing
- Student profile: "تعديل الملف الشخصي", "تفضيلات الإشعارات", "مركز المساعدة", "سياسة الخصوصية".
- Parent profile: two tiles ([parent_profile_screen.dart:107,147](student_app/lib/features/parent/screens/parent_profile_screen.dart#L107)).

---

## F. API / Integration Issues

| # | Pri | Status | File | Problem | Impact | Fix |
|---|---|---|---|---|---|---|
| F1 | P1 | CONFIRMED (repro) | [student_app/lib/features/auth/auth_error_message.dart](student_app/lib/features/auth/auth_error_message.dart) | Handles only `DioException` and a non-existent `.arabicMessage`, but `ApiClient._request` always converts errors to `ApiException` (`.message`) | **All** login/OTP/reset errors show "تعذر إتمام العملية. حاول مرة أخرى.": wrong password, 429 rate-limit, validation, offline | Add an `if (error is ApiException)` branch: field errors → `message` → type-based text |
| F2 | P1 | CONFIRMED | [splash_screen.dart:94-110](student_app/lib/features/auth/screens/splash_screen.dart#L94-L110), [api_auth_repository.dart:32-39](student_app/lib/data/repositories/api_auth_repository.dart#L32-L39) | `getCurrentUser()` returns `null` for network/timeout/5xx, not only 401/403 | Server down or offline at launch → user sent to Welcome as if logged out. Cached offline data is unreachable. The token is kept, so the user is "logged out" but not really | Distinguish "unauthenticated" from "unreachable". On network errors, restore a cached user (as admin_app does) or show a retry screen |
| F3 | P1 | CONFIRMED (repro) | [admin_app/lib/state/app_state.dart:147-148,274-286](admin_app/lib/state/app_state.dart#L147-L148) | `UserRoleType` has no guardian/student values → `role == null` → `role` getter returns **admin**. `signIn` accepts any role | A guardian or student with an email signs in to the admin app, sees "المدير العام" and the admin shell, and every screen 403s | In `signIn` and `bootstrap`, reject users whose `role == null` with "هذا التطبيق مخصّص للإدارة والمحفّظين" and revoke the token |
| F4 | P2 | CONFIRMED (clamp) / PLAUSIBLE (effect) | [session_workspace_controller.dart:236-262](admin_app/lib/ui/teacher/session_workspace/session_workspace_controller.dart#L236-L262) | Loads **all** circle homework (no status/date filter) with `perPage: 200`, but the server clamps to 100 (`BaseQuery::paginate`, `min($perPage, 100)`). With the default `-created_at` sort, the loop also overwrites `_activeHomework` with **older** entries | In circles with >100 non-cancelled/non-graded homework, "active homework" is missing or stale. Memorization/revision tabs mark students "not required" and learning records link the wrong `homework_id`. Also affects `attendanceForSession` / learning records (200) | Filter by `status` and `student_id`/recent date, request ≤100, and keep the **newest** per student+type |
| F5 | P2 | CONFIRMED | [api_student_repository.dart:20](student_app/lib/data/repositories/api_student_repository.dart#L20), parent repository | Every list requests `per_page=100` and reads page 1 only; there is no pagination | Long-term students/guardians silently see only the latest 100 attendance/homework/notifications/sessions | Add "load more", or at least show a truncation note |
| F6 | P2 | PLAUSIBLE | [backend.dart:293-305](admin_app/lib/data/backend.dart#L293-L305) | `pastSessions()` fetches 50 sessions sorted `-session_date` (future ones first), then filters to before today | If schedules pre-generate ≥50 future sessions, "الجلسات السابقة" is empty | Add a `session_date` upper-bound filter server-side |
| F7 | P3 | CONFIRMED | [evaluation_screen.dart:34](student_app/lib/features/parent/screens/evaluation_screen.dart#L34), [api_parent_repository.dart:107](student_app/lib/data/repositories/api_parent_repository.dart#L107) | `homework_followup_score` silently falls back to the behavior score when the guardian doesn't pick one | Fabricated data in reports | Make it required in the UI, or send `null` |
| F8 | P3 | PLAUSIBLE | [sync_service.dart:132-145](admin_app/lib/data/local/sync_service.dart#L132-L145) | Bulk attendance reuses `ops.first.id` as the idempotency key. That op's own individual retry uses the same key on a different path → backend 409 `idempotency_key_reused` → op marked blocked | A single attendance op stuck as "blocked" after a partial bulk failure | Derive the bulk key, e.g. `bulk-${sessionId}-${hash(opIds)}` |

**Verified correct:**
- Auth headers are sent by both apps.
- Error envelopes are parsed correctly.
- 401 handling works in both apps; only its side effect (P0-3) is a problem.
- Idempotency keys are sent on writes.
- Numeric fields in parent endpoints are cast to float server-side, so the `as num?` casts are safe.
- The guardian → `students/{id}/tracking/profile` route is scoped by `StudentPolicy::view`.
- OTP length (4) matches `VerifyOtpRequest`.
- `filter[student_id]` on `parent/evaluations` is allowed.

---

## G. UI / UX Issues

| # | Pri | App | File | Problem |
|---|---|---|---|---|
| G1 | P1 | student | [profile_screen.dart:28-31](student_app/lib/features/student/screens/profile_screen.dart#L28-L31), [parent_profile_screen.dart:32-34](student_app/lib/features/parent/screens/parent_profile_screen.dart#L32-L34) | **No way to log out when the profile fails to load.** The logout button is inside the `data:` branch, the error branch is plain text with no retry, and the "More" sheets containing logout are never used. Offline without cache, 403, or 500 all trap the user |
| G2 | P2 | student | many screens (see list) | Error states without retry: student homework, assessments, revisions, attendance, memorization, notifications, profile, my-qr; parent home, children, child profile, profile, evaluation. Only 2 of 27 screens have pull-to-refresh (`parent_payments_screen`, `student_report_screen`) |
| G3 | P2 | admin | [session_workspace_controller.dart:691-700](admin_app/lib/ui/teacher/session_workspace/session_workspace_controller.dart#L691-L700) | A touched but unscored draft stays in `_dirtyDrafts`, so "إنهاء الجلسة" fails with a generic "لم يتم حفظ كل التغييرات قبل الإنهاء". The only escape is leaving the session and choosing "خروج وتجاهل", then re-entering |
| G4 | P2 | student | [otp_screen.dart:159-303](student_app/lib/features/auth/screens/otp_screen.dart#L159-L303) | `Column` + `Spacer` without a scroll view. On small screens with the keyboard open, expect a RenderFlex overflow that hides the "تحقّق" button (PLAUSIBLE) |
| G5 | P2 | student | [splash_screen.dart:94](student_app/lib/features/auth/screens/splash_screen.dart#L94) | Fixed 2.8 s delay, then `auth/me` with 15 s connect + 20 s receive timeouts and no progress/timeout message. On slow networks the splash can sit ~38 s |
| G6 | P3 | student | [home_screen.dart:700-706](student_app/lib/features/student/screens/home_screen.dart#L700-L706) | Home quick actions "الشهادات" and "الإنجازات" lead to "not available" placeholders |
| G7 | P3 | student | [my_qr_screen.dart:19](student_app/lib/features/student/screens/my_qr_screen.dart#L19) | Mixed English title "My QR (رمز الاستجابة السريعة)" in an Arabic-only app |
| G8 | P3 | admin | [shell.dart:233-236](admin_app/lib/ui/shell.dart#L233-L236) | Top bar subtitle hard-codes "المدير العام" for center owners too (drawer footer uses the real role label) |

---

## H. Performance Issues

No serious problems. Items worth noting:

| # | Pri | File | Issue |
|---|---|---|---|
| H1 | P2 | [session_workspace_controller.dart:236](admin_app/lib/ui/teacher/session_workspace/session_workspace_controller.dart#L236) | Session load pulls the whole circle's homework history. This grows without bound and is clamped to 100 anyway (see F4) |
| H2 | P3 | [app_providers.dart:106-165](student_app/lib/data/providers/app_providers.dart#L106-L165) | `currentParentProvider` issues **two** requests (`parent/my-children` + `auth/me`) and is `autoDispose`, so every parent screen switch refetches both. Tab switching is noticeably chatty |
| H3 | P3 | [session_workspace_controller.dart:161-194](admin_app/lib/ui/teacher/session_workspace/session_workspace_controller.dart#L161-L194) | `presentStudents` rebuilds a filtered list on every getter call and is called repeatedly per build (`countFor`, `evaluatedCount`, `incompleteStudents`). Fine for circle-sized rosters |

The admin token read is already cached in memory, the sync queue batches disk writes, and the student router is not rebuilt on session change.

---

## I. Security Issues

| # | Pri | File | Issue | Fix |
|---|---|---|---|---|
| S1 | P2 | [admin_app/android/app/build.gradle.kts:29-32](admin_app/android/app/build.gradle.kts#L29-L32), [student_app/android/app/build.gradle.kts:29-32](student_app/android/app/build.gradle.kts#L29-L32) | Release builds signed with the **debug** key (template TODO). Not publishable to Play; anyone with the SDK debug key can produce an "update" | Configure an upload keystore via `key.properties` (not committed) |
| S2 | P3 | both `AndroidManifest.xml` | `android:allowBackup` not set (defaults to true). Backed-up encrypted secure-storage blobs often fail to decrypt on restore (C3) | `android:allowBackup="false"` or exclude secure-storage prefs in `dataExtractionRules` |
| S3 | P3 | [login_screen.dart:61-68](student_app/lib/features/auth/screens/login_screen.dart#L61-L68) | Plain-text password carried in go_router `extra` to the role-selection screen (kept in router state; on web it is kept in history state) | Hold it in a short-lived provider cleared after `selectRole` |
| S4 | P3 | [admin_app/lib/ui/auth/auth_flow.dart:629-631](admin_app/lib/ui/auth/auth_flow.dart#L629-L631) | Shows an OTP "رمز الاختبار" snackbar when the server returns `debug_code`. The backend doesn't currently return it | Guard with `kDebugMode` so a misconfigured server can't leak OTPs to screens |

**Clean:**
- No hard-coded secrets, API keys, or passwords in either app.
- Tokens are in `flutter_secure_storage`.
- The student HTTP logger is debug-only and logs method + path only (no headers/bodies); the admin app has no request logging.
- HTTPS default base URL.
- Logout clears caches and the queue.
- The old `?role=admin` preview bypass is removed and covered by tests.

---

## J. Test Coverage Gaps

Existing coverage is good for: mappers, login identifier dispatch, router stability, the admin API client, sync queue mechanics, the session workspace controller, and the auth hardening regressions.

High-value tests to add, ordered by bugs they would have caught:

1. **Student `authErrorMessage` with `ApiException`** (401/422/429/network) → F1.
2. **Student login with "remember me" off** → subsequent request carries a token (P0-2).
3. **Admin: 401 during `sync()` preserves the queue** → P0-3 (repro test in this audit can be reused).
4. **Admin: `signIn`/`bootstrap` rejects non-staff roles** → F3.
5. **Student splash: network error at startup** does not route to `/welcome` (F2).
6. **Shell active-tab index** for every tab path (D1).
7. **Widget test: profile screen error state still offers logout** (G1).
8. **QR scanner success pops the route** (D2), with a fake `Backend`.
9. **Session workspace `load()` with >100 homework / paginated responses** (F4).
10. **Release manifest check in CI**: grep the merged release manifest for `INTERNET` (P0-1). Cheap and prevents regression.

---

## K. Prioritized Fix Plan

| Priority | Category | File | Problem | Impact | Recommended Fix |
|---|---|---|---|---|---|
| P0 | Platform | student_app/android/app/src/main/AndroidManifest.xml | No INTERNET permission in release | Release app fully offline; login impossible | Add the permission |
| P0 | Auth | student_app/lib/features/auth/screens/login_screen.dart:75 | "Remember me" off deletes the live token | User kicked out right after login | In-memory token for non-persistent sessions |
| P0 | Data loss | admin_app/lib/state/app_state.dart:353 | 401 → `syncQueue.clear()` | Unsynced attendance/evaluations lost | Preserve queue per user across forced sign-out; warn before discarding |
| P0 | Crash (iOS) | admin_app/ios/Runner/Info.plist | Missing `NSCameraUsageDescription` | App terminated opening QR scanner | Add the key |
| P1 | API/UX | student_app/lib/features/auth/auth_error_message.dart | `ApiException` not handled | Every auth error shows a generic message | Handle `ApiException` |
| P1 | Startup | student_app/lib/features/auth/screens/splash_screen.dart:98 | Network error treated as logged-out | Offline/slow server → Welcome screen; cache unusable | Retry screen or cached-user restore |
| P1 | Auth/Nav | admin_app/lib/state/app_state.dart:274 | Guardian/student accounts accepted → admin shell | Confusing admin UI full of 403s | Reject `role == null` at sign-in/bootstrap |
| P1 | UX | student_app/lib/features/student/screens/profile_screen.dart:30 (+ parent) | Logout only in success state | User can't log out when profile fails | Move logout outside `when()`/add retry |
| P1 | Navigation | admin_app/lib/ui/screens/qr_scanner_screen.dart:47 | Scanner not popped after success | QR lookup appears broken | `Navigator.pop(context)` before `openProfile` |
| P1 | Offline | admin_app/lib/data/backend.dart:275 | `todaySessions()` online-only | Teachers can't reach offline workflow | Cache Today list like `circleSessions()` |
| P1 | Missing UI | admin_app/lib/ui/nav_menu.dart:33 | Matn catalog screen unreachable | Admins can't manage matn types/sections | Add `NavItem(... AppScreen.matnCatalog)` |
| P1 | Crash | admin_app/lib/main.dart:78 / app_state.dart:160 | Unhandled non-API bootstrap errors | Infinite loading spinner (plausible) | `catch (_)` → `_forceSignOut()`; add `runZonedGuarded`/`PlatformDispatcher.onError` |
| P2 | Navigation | student_shell.dart:39 / parent_shell.dart:37 | Wrong active tab | Nav bar always shows Home | Match exact path or longest prefix |
| P2 | Navigation | student_app sub-screens (D3) | No back button; Android back exits | Users lose place / exit app | `context.push` or `SafeRouteBackButton` |
| P2 | Navigation | admin_app/lib/ui/shell.dart, teacher_shell.dart | Android back exits app | Accidental app exits | `PopScope` → `state.nav(home/students)` |
| P2 | API | session_workspace_controller.dart:236 | Homework query clamped to 100, oldest wins | Wrong "required" evaluations; wrong homework link | Filtered query, ≤100, newest wins |
| P2 | Missing UI | student notifications | Never marked read | Badge never clears | Call `markNotificationRead` on tap / read-all |
| P2 | Missing UI | student/parent profile | Dead buttons; no profile edit | Broken-feeling settings | Implement `PUT auth/me` edit; hide others |
| P2 | UX | session_workspace_controller.dart:693 | Incomplete draft blocks completion | Teacher stuck at end of session | Show which drafts block; allow discard in place |
| P2 | Crash | 3 QR scanner screens | `setState` after dispose | Error logs / red screen in debug | Re-check `mounted` after the delay |
| P2 | UX | student_app error states (G2) | No retry / refresh | Dead-end on transient errors | Pass `onRetry: ref.invalidate(...)`; add `RefreshIndicator` |
| P2 | API | student repositories | Single page of 100 | Old records silently missing | Pagination |
| P2 | UX | otp_screen.dart | Overflow with keyboard | Verify button hidden on small phones | Wrap in `SingleChildScrollView` |
| P2 | UX | splash_screen.dart | ~38 s worst-case splash | Looks frozen | Shorter timeout + retry UI |
| P2 | Security | both build.gradle.kts | Debug-signed release | Not publishable | Release keystore |
| P2 | API | backend.dart:293 | Past sessions may be empty | History screen empty | Server-side date filter |
| P3 | Navigation | child_profile_screen.dart:215 | Payments ignores child | Wrong child shown | Pass child id |
| P3 | Crash (web) | app_router.dart:224 | `extra as Map` | Web refresh crash | Null-check → redirect |
| P3 | Data quality | api_parent_repository.dart:107 | Follow-up score defaulted | Misleading reports | Require or send null |
| P3 | Dead code | student More sheets, unreachable routes, admin unused screens, `CentersScreen` (`/centers` 404) | Maintenance noise | Remove or wire up |
| P3 | UX | home_screen.dart:700 | Quick actions to placeholders | Disappointing taps | Hide until backend exists |
| P3 | Sync | sync_service.dart:138 | Idempotency key reuse | One op may stick as blocked | Distinct bulk key |
| P3 | Security | AndroidManifest (both) | `allowBackup` default | Restore decryption failures | Disable/exclude |
| P3 | Security | login_screen.dart:63 | Password in router `extra` | Lingers in router state | Short-lived provider |
| P3 | Security | auth_flow.dart:629 | OTP debug snackbar not gated | Potential OTP exposure | `kDebugMode` guard |
| P3 | Resilience | student_app/lib/main.dart:27 | All async errors swallowed, no reporting | Production bugs invisible | Add crash reporting; don't blanket-return true |

---

## Top 10 fixes to do first

1. **Add `INTERNET` permission to `student_app` main manifest.** The release APK cannot connect today.
2. **Fix "Remember me" in the student login.** Unticking it logs the user out instantly.
3. **Stop `_forceSignOut` from deleting the teacher's offline sync queue.** This is silent data loss after token expiry (30 days).
4. **Add `NSCameraUsageDescription` to admin iOS Info.plist** (if iOS ships).
5. **Handle `ApiException` in `authErrorMessage`.** Users currently never learn their password was wrong, or that they are offline or rate-limited.
6. **Don't treat "server unreachable" as "logged out" on student splash.** Show a retry screen or restore the cached user.
7. **Reject non-staff roles at admin sign-in.** Guardians and students currently land in the admin shell.
8. **Make logout reachable when profile loading fails** (student + parent profile screens).
9. **Pop the QR scanner after a successful lookup** so the student profile is actually visible.
10. **Fix bottom-nav active-tab matching and add back handling** (student sub-screens via `push`/back button, `PopScope` in admin shells).
