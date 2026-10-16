# Release checklist & Google Play pre-submission check

Status legend: **READY** = done and verified in this repository. **ACTION REQUIRED** =
needs information, a decision, a credential or a console step only the developer /
operator can provide.

## A. One-time setup (per app)

1. **Upload keystore** — ACTION REQUIRED
   ```
   keytool -genkey -v -keystore upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
   ```
   Place it at `android/upload-keystore.jks` (git-ignored), copy
   `android/key.properties.example` → `android/key.properties`, fill the passwords.
   Back up keystore + passwords outside the repo. Use a separate keystore per app or
   one shared upload key — both are valid.
2. Install **Android SDK Command-line Tools** (SDK Manager) so Flutter can strip/verify
   native symbols (`flutter doctor` must be clean).
3. Fill the remaining 10 `[مطلوب: …]` values (operator identity, contact, URLs, governing law, effective date, hosting and e-mail providers) in `lib/core/legal/legal_content.dart`
   (identical file in both apps — edit one, copy to the other; a test checks they match):
   ```
   grep -rn "\[مطلوب:" admin_app/lib/core/legal
   ```
4. Regenerate and **publish** the web pages (privacy policy + account deletion) on a
   domain you control, then put their URLs into `LegalInfo`:
   ```
   cd admin_app && dart run tool/export_legal_markdown.dart ../play_store/legal
   ```
5. Backend deploy (no new migrations): deploy the new route/action/command
   (`api/`), confirm `SANCTUM_TOKEN_EXPIRATION_MINUTES`, the scheduler cron
   (`php artisan schedule:run` every minute — needed for notification/token cleanup),
   backup retention env, `APP_DEBUG=false`, valid TLS on the API host.
6. Create reviewer accounts on demo data (`APP_ACCESS_REVIEWER.md`).

## B. Build (each app)

```
flutter clean
flutter pub get
flutter analyze
flutter test
flutter build appbundle --release          # add --dart-define=API_BASE_URL=https://<prod>/api/v1 if the host changes
```

Verify the output (`build/app/outputs/bundle/release/app-release.aab`):

```
java -jar bundletool.jar dump manifest --bundle=build/app/outputs/bundle/release/app-release.aab
jarsigner -verify -verbose -certs build/app/outputs/bundle/release/app-release.aab   # must say "jar verified"
```

Expected manifest: target 36, min 24, correct package, versionCode > last upload,
`usesCleartextTraffic="false"`, `allowBackup="false"`, no `debuggable`; permissions
admin = INTERNET, ACCESS_NETWORK_STATE, CAMERA; student = INTERNET.

Compile-only check without a keystore (output is UNSIGNED, cannot be uploaded):
`flutter build appbundle --release -P allowUnsignedRelease=true`.

## C. Results of this preparation run (2026-09-15)

| Step | admin_app | student_app |
|---|---|---|
| flutter clean / pub get | ✔ | ✔ |
| flutter analyze | ✔ No issues | ✔ No issues |
| flutter test | ✔ 169 passed, 21 skipped (live-API tests, opt-in) | ✔ 149 passed |
| New compliance tests | ✔ 15 (`test/play_compliance_test.dart`) | ✔ 21 (`test/play_compliance_test.dart`) |
| Signed `flutter build appbundle --release` | ✘ stopped by the signing guard: no `key.properties` (expected, correct) | ✘ same |
| Unsigned compile check (`-P allowUnsignedRelease=true`) | ✔ AAB produced, 69.5 MB (includes debug-symbol metadata); Flutter reported "failed to strip debug symbols" (cmdline-tools missing) | ✔ AAB produced, 57.3 MB; same warning |
| AAB manifest | target 36, min 24, 1/1.0.0, cleartext off, backup off, not debuggable | same |
| AAB signature | unsigned (`jarsigner`: no manifest) | unsigned |
| Backend (`php artisan test`) | ✔ 544 passed, 1 skipped, 1 risky (pre-existing) — includes `AccountDeletionTest` 8/8 | |

## D. Final pre-submission checklist

| Area | admin_app | student_app |
|---|---|---|
| **Target API 36+** | READY (36) | READY (36) |
| **Release signing** | ACTION REQUIRED — create upload keystore + `key.properties`; config already refuses debug signing | ACTION REQUIRED — same |
| Play App Signing enrolment | ACTION REQUIRED | ACTION REQUIRED |
| Version code/name | READY (1 / 1.0.0) — ACTION REQUIRED confirm no prior upload | READY — same |
| Production API configured | ACTION REQUIRED — confirm `https://test.talentooo.com` is production and live with valid TLS | ACTION REQUIRED — same |
| Debug / test access removed | READY (no preview login, no test credentials, debug OTP compiled out, tests guard it) | READY |
| Permissions minimal & explained | READY (INTERNET, NETWORK_STATE, CAMERA for QR with Arabic denial message) | READY (INTERNET only) |
| Cleartext / network security | READY | READY |
| Backup of sensitive local data | READY (disabled + extraction rules) | READY |
| In-app privacy policy | READY (real content) — ACTION REQUIRED fill placeholders | READY — same |
| Privacy policy URL (public) | ACTION REQUIRED publish | ACTION REQUIRED publish |
| In-app terms / about / contact | READY — ACTION REQUIRED contact details | READY — same |
| Account deletion in app | READY (backend endpoint + UI + tests) | READY |
| Account deletion web URL | ACTION REQUIRED publish + verification procedure (`php artisan account:delete`) | ACTION REQUIRED |
| Data safety form | ACTION REQUIRED fill from `DATA_SAFETY.md` (2 mapping decisions marked) | ACTION REQUIRED fill from `DATA_SAFETY.md` |
| Data retention disclosure | READY — policy states implemented behaviour (records & audit logs not auto-deleted, soft delete, backups up to 5 years, logs 14 days, notifications 90 days). ACTION REQUIRED: confirm production `.env` matches defaults; decide whether a defined retention period (requires a purge feature) is legally needed | same |
| Content rating | ACTION REQUIRED complete IARC questionnaire (`CONTENT_RATING_AND_DECLARATIONS.md`) | ACTION REQUIRED |
| Target audience | ACTION REQUIRED confirm 18+ | **ACTION REQUIRED — decide (possible under-13 users → Families Policy)** |
| Ads declaration | READY — "No ads" | READY — "No ads" |
| Financial features declaration | ACTION REQUIRED confirm "none" | ACTION REQUIRED confirm "none" |
| UGC | ACTION REQUIRED confirm "not UGC" | ACTION REQUIRED confirm |
| App access / reviewer instructions | ACTION REQUIRED create demo-data reviewer account + paste credentials | ACTION REQUIRED |
| Store listing text | READY (drafts in `STORE_LISTING.md`) — ACTION REQUIRED confirm titles | READY — same |
| App icon 512×512 | ACTION REQUIRED export from official logo | ACTION REQUIRED |
| Feature graphic 1024×500 | ACTION REQUIRED | ACTION REQUIRED |
| Screenshots | ACTION REQUIRED (demo data only) | ACTION REQUIRED |
| Support e-mail / website | ACTION REQUIRED | ACTION REQUIRED |
| Dead / unavailable screens | READY (dead buttons removed) | ACTION REQUIRED decide: certificates, achievements, student payments show "not available" and are linked from home/menu |
| Operator legal identity | ACTION REQUIRED (`LegalInfo.operatorName`, governing law) | ACTION REQUIRED |

## E. Every later release

- Bump `version:` in `pubspec.yaml` (`x.y.z+N`, N strictly increasing per app).
- Re-run section B and the manifest/signature checks.
- If data practices change, update `legal_content.dart`, re-export the web pages,
  update `DATA_SAFETY.md` and the Play Console form **before** the release.
