# Data Safety audit — الطبري (admin_app + student_app)

Internal working document for the Google Play **Data safety** form. Every row was
verified against the code on 2026-09-15 (apps: `admin_app/lib`, `student_app/lib`;
backend: `api/`). Where a fact depends on production server configuration that is
not visible in the repository, the row says **ACTION REQUIRED**.

Google's definitions used here:

- **Collected** = transmitted off the device (to our backend or to an SDK vendor).
  Data only processed on the device (camera frames, local cache) is *not* collected.
- **Shared** = transferred to a third party for *its own* purposes. Transfers to
  service providers processing on our behalf (hosting, e-mail delivery) are not
  "sharing".
- Data an app user enters **about other people** (staff entering student data)
  still counts as collected by that app.

---

## 1. Master audit table (platform = both apps + backend)

| # | Data type | Collected (off device)? | Stored where | Transmitted to | Shared with 3rd parties? | Purpose | Retention | Deletion behaviour |
|---|---|---|---|---|---|---|---|---|
| 1 | Account name | Yes | Backend `users.name` | Our API (HTTPS) | No | Account, app functionality | While account active | Student/guardian: replaced by «حساب محذوف». Staff: **kept** (payroll/session history) — disclosed |
| 2 | E-mail | Yes (staff login; optional for student/guardian) | `users.email`, `students.email`, `guardians.email` | Our API; e-mail provider for password-reset OTP | No | Login, password reset | While account active | Replaced by `deleted-<uuid>@deleted.invalid`; cleared on student/guardian record; erased from account audit trail |
| 3 | Phone number | Yes | `users.phone`, `students.phone`, `guardians.phone` | Our API | No | Mobile login, center contact | While account active | Cleared on the account. **Center record phone retained** (center data) — disclosed |
| 4 | User IDs (account UUID, student code) | Yes | Backend | Our API | No | Login (student code), QR lookup | While record exists | Account UUID row soft-deleted (student/guardian); student code stays on the center's student record |
| 5 | Password | Yes (at login) | Backend, bcrypt hash only | Our API | No | Authentication | While account active | Replaced by random hash |
| 6 | Other personal info — date of birth, gender, national ID (optional), notes, guardian relationship, occupation | Yes (admin_app, entered by staff) | `students`, `guardians` | Our API | No | Student administration | No automatic deletion (verified: no pruning job) — disclosed as such; **ACTION REQUIRED** if the operator wants a defined period (needs a purge feature) | Retained on center record after account deletion. The admin "delete student/guardian" action is a **soft delete** (`SoftDeletes`): hidden, not erased |
| 7 | Staff employment data — qualification, specialization, experience, hire date, salary type/amounts, payroll | Yes (admin_app) | `teachers`, `payroll_*` | Our API | No | Payroll, administration | **ACTION REQUIRED** — legal/accounting retention | Retained (financial/regulatory) |
| 8 | Educational records — attendance, memorization/revision/matn records, assessments, homework, monthly progress, circle enrollment | Yes (admin_app writes; student_app reads) | Backend tables | Our API | No | Core app functionality, parent reporting | **ACTION REQUIRED** | Retained by the center after account deletion — disclosed |
| 9 | Guardian evaluations (behaviour / prayer / homework-follow-up scores + notes) | Yes (student_app, guardian submits) | `parent_evaluations` | Our API | No | Parent–center reporting | **ACTION REQUIRED** | Retained on the guardian/student record |
| 10 | Financial records — invoices, fees, payments, expenses | Yes (admin_app) ; student_app only displays | `invoices`, `payments`, `transactions`, `expenses` | Our API | No | Center accounting | **ACTION REQUIRED** (tax/accounting law) | Retained — disclosed. **No card / bank data; no in-app payments** |
| 11 | In-app notifications | Server-generated | `notifications` | Our API | No | App functionality | **Auto-deleted after 90 days** (`automation:cleanup-notifications --days=90`) — requires production scheduler cron | Deleted on account deletion |
| 12 | Auth tokens (Sanctum) + fixed device label (`itqan-admin-app` / `altabari-student-app`) + last-used time | Yes | `personal_access_tokens`; on device: OS secure storage | Our API | No | Session security | Expire per `SANCTUM_TOKEN_EXPIRATION_MINUTES` (example env: 43200 = 30 days) — **ACTION REQUIRED: confirm set in production** | All revoked on logout-all / account deletion |
| 13 | IP address + User-Agent | Yes (server side, audit logs) | `audit_logs.ip_address`, `user_agent` | — | No | Security, audit | Not deleted automatically (no pruning job) — disclosed in the privacy policy | Retained for security (allowed by Play policy); e-mail/phone/name stripped from the deleted account's own entries |
| 14 | Push device tokens (FCM) | **Not by these apps** — no FCM SDK in either app; backend supports `/devices` but nothing registers | `device_tokens` (empty for these apps) | — | — | — | 90 days inactivity | Deleted on account deletion |
| 15 | Camera frames | **No** — processed on device by ML Kit; not saved, not uploaded | — | — | — | QR scanning (admin_app only) | — | — |
| 16 | QR code content | Student QR = `student_code` only (no token, no password) | — | — | — | Attendance / payment lookup | — | — |
| 17 | Local offline cache (lists, profile, sync queue) | No (on device only) | SharedPreferences (app-private, **not encrypted**) | — | — | Offline use | Cleared on logout (admin_app keeps *unsynced* ops of that user until they sync) | Cleared on account deletion |
| 18 | ML Kit diagnostics (admin_app) | **Yes, by the SDK** — "device information, app information, performance metrics, API configuration… error codes", purpose "diagnostics and usage analytics", "does not transfer this data to third-parties", encrypted in transit (source: developers.google.com/ml-kit/android-data-disclosure) | Google | Google | No | Diagnostics/analytics (SDK) | Google's policy | n/a |
| 19 | Google Fonts request (student_app) | IP / HTTP request to fonts.gstatic.com when the Tajawal font is first downloaded (`google_fonts` runtime fetching) | Google | Google | No | Rendering | Google's policy | n/a — **Recommended: bundle the font files and disable runtime fetching** |
| 20 | Location, contacts, photos/media, audio, files, calendar, SMS, advertising ID, installed apps, web history, health | **No** | — | — | — | — | — | — |
| 21 | Crash reporting / analytics SDKs (Firebase Analytics, Crashlytics, Sentry…) | **None present** in either app | — | — | — | — | — | — |
| 22 | Database backups | Copies of all backend data | Backup disk (spatie/laravel-backup) | — | No | Disaster recovery | `config/backup.php` defaults (not overridden by `.env.example`): all 7 d, daily 30 d, weekly 12 w, monthly 12 m, **yearly 5 years**, oldest removed above 10 GB — disclosed. **ACTION REQUIRED: confirm production `.env` does not override `BACKUP_KEEP_*`; do not claim backup encryption unless `BACKUP_ARCHIVE_PASSWORD` is set** | Deleted data persists in backups until rotated (up to 5 years) — disclosed |
| 23 | Server error reports (Sentry) | Server side, only if `SENTRY_LARAVEL_DSN` is set (empty in `.env.example`) | Sentry | Sentry (service provider) | No | Error monitoring | Sentry account settings | `SENTRY_SEND_DEFAULT_PII=false` — user data and IP not attached automatically. Disclosed. **ACTION REQUIRED: confirm whether a DSN is configured in production** |
| 24 | Server application logs | Server side | `storage/logs` (daily channel) | — | No | Operations | Daily rotation, 14 days (`LOG_DAILY_DAYS` default; `.env.example` uses `LOG_CHANNEL=daily`) — disclosed | Rotated out |

Encryption **in transit**: yes — both release manifests set `usesCleartextTraffic="false"`
(and targetSdk 36 blocks cleartext by default); the API base URL is `https://`.
**ACTION REQUIRED:** confirm the production host serves a valid TLS certificate.

Encryption **at rest**: **do not claim it** on the form. Passwords are hashed; tokens
on device are in OS secure storage; but the database, backups and the on-device
offline cache are not verified as encrypted.

---

## 2. Suggested Data safety answers — admin_app (`com.itqan.itqan`)

| Question | Answer | Notes |
|---|---|---|
| Does your app collect or share any of the required user data types? | **Yes** | |
| Is all user data encrypted in transit? | **Yes** | After confirming production TLS |
| Do you provide a way for users to request that their data is deleted? | **Yes** | In-app «حذف الحساب» + web URL (**ACTION REQUIRED**: publish page) |
| Account creation methods | Username/e-mail + password; accounts are created by the center admin **inside this app** | Therefore the account-deletion requirement applies |
| Delete account URL | `[ACTION REQUIRED]` | Must be a working public page (see `legal/account_deletion_ar.md`) |

Data types to declare (all: **Collected = Yes, Shared = No, Processed ephemerally = No,
Required** unless noted):

| Category → type | Purposes |
|---|---|
| Personal info → Name | App functionality, Account management |
| Personal info → Email address | App functionality, Account management |
| Personal info → User IDs | App functionality, Account management |
| Personal info → Phone number | App functionality, Account management |
| Personal info → Other info (DOB, gender, national ID, relationship, qualifications, educational records) | App functionality |
| Financial info → Other financial info (fees, payments, expenses, salaries) | App functionality |
| App activity → Other user-generated content (attendance, assessments, homework, notes entered by staff) | App functionality — **ACTION REQUIRED: developer decision** whether to map educational records here or only under "Other info" |
| App info and performance → Diagnostics (ML Kit SDK) | Analytics — collected by SDK, not shared |
| Device or other IDs | **ACTION REQUIRED** — ML Kit lists "device information"; decide after reading the ML Kit disclosure whether this includes an identifier. Conservative answer: declare, purpose Analytics |

Not collected: Location, Photos/videos (camera frames stay on device), Audio, Files,
Calendar, Contacts, Messages, Health, Web browsing, Installed apps.

## 3. Suggested Data safety answers — student_app (`com.altabari.altabari`)

| Question | Answer | Notes |
|---|---|---|
| Collects or shares required data types? | **Yes** | |
| Encrypted in transit? | **Yes** | |
| Deletion request mechanism? | **Yes** | In-app + web URL |
| Account creation | Accounts are **created outside this app** by the center | Play's deletion rule targets in-app account creation; the feature is provided anyway because the same platform account is used |

| Category → type | Purposes |
|---|---|
| Personal info → Name | App functionality, Account management (editable in app) |
| Personal info → Email address | Account management (optional, editable; password reset) |
| Personal info → Phone number | Account management (mobile login) |
| Personal info → User IDs | Account management (student code login) |
| App activity → Other user-generated content (guardian evaluations: behaviour/prayer/homework scores + notes) | App functionality |
| App activity → App interactions (notification read state) | App functionality — optional to declare; conservative: declare |

Not collected by this app: Financial info (displayed only), Location, Photos, Camera
(no camera permission), Contacts, Device IDs, Diagnostics (no analytics/crash SDK).
Google Fonts runtime download: not a Play data type, but disclosed in the privacy
policy; bundling the font removes it.

---

## 4. Consistency checks performed

- Privacy policy text (`*/lib/core/legal/legal_content.dart`) mentions every "Yes"
  row above; a test (`admin_app/test/play_compliance_test.dart`) fails if key topics
  are removed and if the two apps' copies diverge.
- No "no data collected" claim anywhere.
- The admin welcome screen claimed «جميع بياناتك محمية ومشفّرة بالكامل» — unverifiable;
  replaced with «يتم الاتصال بالخادم عبر اتصال مشفّر (HTTPS)».
- No claim of encryption at rest, certifications, or third-party sharing.
