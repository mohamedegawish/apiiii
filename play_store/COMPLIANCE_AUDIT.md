# Google Play compliance audit — Android

Audit date: 2026-09-15. Toolchain: Flutter 3.44.9 / Dart 3.12.2, AGP 9.0.1, Gradle
9.1.0, Kotlin 2.3.20, Android SDK platforms 34–37, build-tools 36.0.0.
Evidence = source files plus the **merged manifest dumped from the built AABs**
with bundletool 1.18.1 (`bundletool dump manifest`).

Play target-API rule: new apps and updates must target **Android 16 / API 36** from
2026-08-31.

## 1. Build configuration

| Check | admin_app | student_app | Status |
|---|---|---|---|
| applicationId | `com.itqan.itqan` | `com.altabari.altabari` | ✔ unique. **ACTION REQUIRED:** confirm final ids before first upload (permanent) |
| namespace | same as id | same as id | ✔ |
| compileSdk | 36 (`flutter.compileSdkVersion`) | 36 | ✔ |
| targetSdk (AAB) | **36** | **36** | ✔ meets API 36 rule |
| minSdk (AAB) | 24 | 24 | ✔ |
| versionCode / versionName (AAB) | 1 / 1.0.0 (`pubspec.yaml` `1.0.0+1`) | 1 / 1.0.0 | ✔ valid. No release history exists in the repo — **ACTION REQUIRED:** confirm nothing was uploaded to Play before; every later upload must increase the `+N` build number |
| Release signing | Upload keystore from `android/key.properties` only; **no debug-key fallback**; build fails without it | same | ✔ config correct; ✘ **no `key.properties`/keystore present → no signed AAB** (ACTION REQUIRED) |
| Play App Signing | — | — | **ACTION REQUIRED:** enrol (recommended) when creating the apps |
| R8 / shrinking | enabled (AAB contains `proguard.map`) | enabled | ✔ |
| Native debug symbols | included in `BUNDLE-METADATA/…/debugsymbols` | included | ✔ (Flutter's post-build *strip* check failed because Android SDK **cmdline-tools** are not installed — install them; the bundle itself is valid) |
| `android:debuggable` | absent (release) | absent | ✔ |
| Flutter build mode | AOT `libapp.so`, no `kernel_blob.bin` / JIT snapshots | same | ✔ |
| Debug banner | `debugShowCheckedModeBanner: false` | same | ✔ |

## 2. Manifest (merged, from AAB)

| Item | admin_app | student_app | Status |
|---|---|---|---|
| INTERNET | ✔ | ✔ | required |
| ACCESS_NETWORK_STATE | ✔ (connectivity_plus — offline sync) | — | justified |
| CAMERA | ✔ (mobile_scanner — QR only), `uses-feature camera required=false` | — | justified; runtime prompt only when a scanner opens; Arabic explanation on denial added |
| POST_NOTIFICATIONS | — | — | ✔ not needed: no push SDK, only in-app notification lists |
| Storage / media / location / contacts / AD_ID / exact alarm / foreground service | none | none | ✔ |
| `<app>.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` | present | present | ✔ AndroidX-generated signature permission, harmless |
| Exported components | `MainActivity` (launcher); `androidx.profileinstaller.ProfileInstallReceiver` (guarded by `android.permission.DUMP`) | same | ✔ |
| Other components | ML Kit init provider/discovery service, CameraX metadata service, `GoogleApiActivity`, Google `datatransport` (ML Kit telemetry) — all `exported=false` | androidx-startup provider (`exported=false`) | ✔ |
| Package visibility `<queries>` | `PROCESS_TEXT` only (Flutter template) | same | ✔ no `QUERY_ALL_PACKAGES` |
| Deep links / App links | none | none | ✔ |
| WebView | none | none | ✔ |
| `usesCleartextTraffic` | **false** (added; debug manifest re-enables for local dev only) | **false** | ✔ |
| Network security config | none — platform default (API 36 blocks cleartext) | none | ✔ |
| `allowBackup` / `fullBackupContent` | false / false | false / false | ✔ |
| `dataExtractionRules` (API 31+) | excludes every domain from cloud backup **and** device transfer | same | ✔ |
| App label | «الطبري» | «الطبري» | ✔ (store titles differ — see STORE_LISTING) |

## 3. Third-party SDKs

| SDK | App | Network / data | Declared |
|---|---|---|---|
| mobile_scanner 7.4.0 → Google ML Kit barcode-scanning 17.3.0 (bundled model) + CameraX | admin | ML Kit sends diagnostics/usage analytics to Google (HTTPS, not transferred to third parties) | Privacy policy + DATA_SAFETY row 18 |
| connectivity_plus | admin | none | — |
| flutter_secure_storage (9.x admin, 10.x student) | both | none (Keystore-backed token storage) | policy |
| shared_preferences | both | none (offline cache) | policy |
| http / dio | both | our API only | policy |
| google_fonts | student | downloads Tajawal from fonts.gstatic.com at runtime | policy; **recommended: bundle font** |
| qr_flutter | both | none (renders QR locally) | — |
| Firebase / FCM / Analytics / Crashlytics / Maps / Ads | — | **not present** in either app | — |

Backend: uses `kreait/firebase` for FCM, but no app registers device tokens — no
push reaches these builds.

## 4. Network & security

| Check | Result |
|---|---|
| Production API | `https://test.talentooo.com/api/v1` compiled into both AABs (verified in `libapp.so`). ⚠ **ACTION REQUIRED:** the host is named `test.` — confirm it is the production server, has a valid TLS certificate and serves Laravel (an earlier deployment note recorded a Hostinger 404). If a different production host is intended, build with `--dart-define=API_BASE_URL=https://…/api/v1` and update `admin_app/test/api_client_test.dart` |
| Localhost / emulator / staging URLs in release | none (`10.0.2.2`, `localhost:8000` absent from `libapp.so`) |
| Test credentials | none (`@itqan.test` absent from AABs; demo seeders refuse production) |
| Hard-coded secrets | none found in `lib/` (API keys, tokens). `api/.env` is local dev config and not part of the apps |
| Debug OTP | `debug_code` handling compiled out (`kDebugMode` const) — string absent from AAB |
| Token storage | OS secure storage (admin: EncryptedSharedPreferences; student: flutter_secure_storage 10) |
| Logging | student Dio log interceptor debug-only; `main.dart` error `debugPrint` now debug-only; admin app has no print/debugPrint |
| QR content | `student_code` only (no token/password) — test enforced |
| Preview / role bypass | removed earlier; test guards against `?role=` / `debugAuthenticate` in production code |
| Misleading security claim | «جميع بياناتك محمية ومشفّرة بالكامل» (admin welcome) replaced with an accurate HTTPS statement |

## 5. Icon / splash / branding

| Check | Result |
|---|---|
| Launcher icon | Generated from the official «الطبري — رواية ودراية» logo (`assets/brand/icon.png`), not the Flutter template icon (hash differs) ✔ |
| Adaptive icon | `mipmap-anydpi-v26/ic_launcher.xml`: white background + logo foreground with 16 % inset ✔ |
| Splash | admin: brand green `#0C2E1B` launch background; student: platform background colour, then the in-app animated splash with logo ✔ (optional polish: match student launch colour) |
| Debug branding / placeholder icons | none ✔ |
| Play 512×512 icon, feature graphic, screenshots | **ACTION REQUIRED** |

## 6. Broken or misleading UI found and fixed

- student_app «سياسة الخصوصية», «مركز المساعدة» and «تفضيلات الإشعارات» rows had empty
  `onTap` handlers (dead buttons). Privacy/help now open real pages; the notification
  preferences row was removed (no such feature exists).
- Welcome/login text referenced terms and privacy policy that did not exist; they are
  now links to the real pages.
- Remaining: student achievements / certificates / student payments screens show
  "feature not available" (no backend endpoint) — **ACTION REQUIRED (decide to hide)**.
