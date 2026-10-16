# App access — Google Play reviewer strategy

Both apps are **fully behind login**. There is **no self sign-up, no invitation link,
no guest mode and no demo/preview login** in the release builds (the old web
`?role=admin` preview path was removed; `test/play_compliance_test.dart` guards it).
Play Console → *App content → App access* must therefore be set to
**"All or some functionality is restricted"** with working credentials.

## What each app requires

| App | Who signs in | Identifier | Requires |
|---|---|---|---|
| admin_app `com.itqan.itqan` | Super admin, center owner, teacher (staff only — student/guardian accounts are rejected with «هذا التطبيق مخصّص للإدارة والمحفّظين») | E-mail + password | An active staff account created by a super admin / center owner. Teachers only see circles assigned to them. |
| student_app `com.altabari.altabari` | Student or guardian (staff are rejected) | Mobile number **or** student code, + password. If one mobile belongs to both a student and a guardian, a role-selection screen appears. | An active student account linked to a student record, or a guardian account linked to at least one child. |

## Production-safe reviewer accounts (ACTION REQUIRED — create before submission)

The release builds talk to the production API (`https://test.talentooo.com/api/v1`,
see `RELEASE_CHECKLIST.md`). Reviewer accounts must **never expose real students'
data**. Do **not** hand Google a super-admin or center-owner login on a database that
contains real children — those roles can read every student, guardian, payment and
salary.

Recommended setup (all through the admin app, no code changes, no hard-coded
passwords):

1. **Demo circle** — create a circle named e.g. «حلقة مراجعة Google Play» with a
   schedule and 2–3 **fictitious** students (invented names, no national ID, no real
   phone numbers — use numbers you own or leave optional fields empty).
2. **Reviewer teacher** (admin app) — create a teacher account
   `[ACTION REQUIRED: reviewer e-mail you control]` with a strong unique password and
   assign it **only** to the demo circle. A teacher is scoped server-side to its own
   circles (`StudentQuery`/`StudentPolicy`), so the reviewer sees only demo data.
   Open one session so the reviewer can try attendance and QR scanning.
3. **Reviewer guardian + student** (student app) — create one fictitious guardian
   linked to one fictitious student in the demo circle, with a student code and
   password. Record a few attendance/memorization entries so the screens are not empty.
4. **Admin-only screens** (finance, payroll, users, audit logs) are role-gated. If you
   want them reviewed, only provide a center-owner login on an environment whose data
   is entirely demo data — otherwise state in the instructions that these screens are
   restricted to the center's management and are not accessible with the review login.
5. Keep the accounts **active** for the whole review and every later update review;
   do not change passwords without updating Play Console. Deactivate them if the
   listing is removed. Never delete them through «حذف الحساب» (that is irreversible).
6. Store the credentials only in Play Console (and your password manager). They are
   **not** in the app, the repository, or these documents.

## Text for Play Console → App access → Instructions

> Accounts are created by the Quran center's administration; there is no public
> sign-up.
>
> **Admin/teacher app (الطبري — admin_app):** open the app → «تسجيل الدخول» → enter
> the e-mail and password below. The account is a *teacher* assigned to a demo circle.
> Go to «اليوم» to open a session, record attendance, or tap the QR icon to scan a
> student QR code (camera permission is requested only then). Finance, payroll and
> user management are limited to center management and are not available to this
> account.
> E-mail: `[ACTION REQUIRED]` — Password: `[ACTION REQUIRED]`
>
> **Student/guardian app (الطبري — student_app):** open the app → «ابدأ الآن» → enter
> the student code (or mobile) and password below.
> Student code: `[ACTION REQUIRED]` — Password: `[ACTION REQUIRED]`
> Guardian mobile: `[ACTION REQUIRED]` — Password: `[ACTION REQUIRED]`
>
> Privacy policy: «حسابي» → «سياسة الخصوصية» (also on the welcome/login screens).
> Account deletion: «حسابي» → «حذف الحساب» (admin app: «المزيد» → «حذف الحساب»).
> The app interface is Arabic only.

## Things a reviewer might hit — check before submitting

- student_app shows a "feature not available" state on **achievements, certificates
  and student payments** (the backend has no endpoint; see
  `api_student_repository.dart`). Consider hiding those menu entries before release,
  or mention it in the instructions — Play may treat dead screens as broken
  functionality. **ACTION REQUIRED (developer decision).**
- Offline mode: admin app works offline for attendance; review devices are online.
- The API host must be reachable and serving the Laravel app (the deployment note
  `PRODUCTION_DEPLOYMENT_SHARED_HOSTING.md` recorded a Hostinger 404 at one point).
