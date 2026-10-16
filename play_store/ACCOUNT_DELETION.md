# Account deletion — implementation record

Google Play: *"If your app allows users to create an account from within your app…
it must also allow users to request for their account to be deleted"*, with a
prominent in-app path **and** a working web link in the Data safety form. Limited
retention is allowed for *"security, fraud prevention or regulatory compliance"* if
disclosed. (support.google.com/googleplay/android-developer/answer/13327111)

The backend had **no** self-service deletion (only admin-side deactivation). A real,
minimal endpoint was added — the feature is not simulated.

## Backend (`api/`)

`POST /api/v1/auth/delete-account` — `auth:sanctum` + `active` middleware,
rate-limited to 5/min per user (`throttle:account-deletion`).

Request: `{"password": "<current password>", "confirm": true}`

| Response | When |
|---|---|
| 200 `{data:{deleted:true}}` | Deleted |
| 401 | No / invalid token |
| 422 `errors.password` «كلمة المرور غير صحيحة.» | Wrong password |
| 422 `errors.confirm` | Missing explicit confirmation |
| 409 `error_code: last_super_admin` | Last active super admin |
| 429 | Rate limited |

Authorization: the action only ever acts on `$request->user()`; any `user_id` in the
body is ignored (tested).

`DeleteOwnAccountAction` (one DB transaction, row lock):

| Removed for every role | |
|---|---|
| All Sanctum tokens (every device) | ✔ |
| Push device tokens, push delivery logs | ✔ |
| In-app notifications | ✔ |
| Password-reset codes/tokens for the old e-mail, web sessions, stored idempotent responses | ✔ |
| E-mail → `deleted-<uuid>@deleted.invalid`; phone → null; password → random hash; `is_active=false` | ✔ |
| name/e-mail/phone stripped from the account's own `audit_logs` old/new values; a PII-free `account_deleted` entry is written | ✔ (saved quietly so the observer does not re-log the erased values) |

| Role | Additionally | Retained (disclosed in app + policy) |
|---|---|---|
| Student / guardian | Name → «حساب محذوف»; account soft-deleted; unlinked from the student/guardian record (`user_id=null`); e-mail cleared on that record | The center's record: name, student code, phone, enrollment, attendance, learning records, assessments, homework, guardian evaluations, invoices/payments |
| Teacher / center owner / super admin | Account deactivated (not soft-deleted) | Name — payroll, expenses and session history resolve staff through the user row; all center records |
| Last active super admin | **Refused (409)** | — |

Why records are retained: they belong to the center (educational follow-up,
accounting). The code has **no automatic deletion** of these records, and the admin
"delete student / guardian" action is a **soft delete** (record hidden, not erased).
`LegalInfo.recordsRetention` now states exactly that. There is no hard-delete path
for center records. **ACTION REQUIRED (optional):** if the operator's legal advisor
requires a fixed retention period or permanent erasure on request, a purge feature
must be built first — and only then may the policy promise it.

Backups: deleted data remains in database backups until rotated — per
`config/backup.php` defaults up to **5 years** (yearly copies). `LegalInfo.backupRetention`
states the full schedule. **ACTION REQUIRED:** confirm production `.env` does not
override `BACKUP_KEEP_*`.

Tests: `api/tests/Feature/AccountDeletionTest.php` (8 tests, 66 assertions) —
unauthenticated rejected, password + confirm required, student/guardian/teacher
behaviour, last super admin, caller-only, operator console command.

## Apps

| | admin_app | student_app |
|---|---|---|
| Entry points | «المزيد» → «حذف الحساب»; teacher «حسابي» → «حذف الحساب»; «الخصوصية والمعلومات» hub | «حسابي» (student & guardian) → «حذف الحساب»; also shown when the profile fails to load; `/legal` hub; route `/account/delete` (requires session) |
| Explains consequences | Removed / retained lists (staff) | Removed / retained lists (student/guardian) |
| Accidental-deletion guards | Acknowledgement checkbox + current password + final confirm dialog | Same |
| Extra guard | Blocked while the user has unsynced offline changes (offers «مزامنة الآن») | — |
| On success | Clears token + caches, returns to login | Clears token + caches, returns to login |
| On failure | Session kept; server reason shown (wrong password, last super admin, network) | Session kept; reason shown |
| Tests | `test/play_compliance_test.dart` | `test/play_compliance_test.dart` |

## Web deletion page — ACTION REQUIRED

Play Console requires a public URL. Content is generated from the in-app source:

```
cd admin_app && dart run tool/export_legal_markdown.dart ../play_store/legal
```

Publish `legal/account_deletion_ar.md` (and `privacy_policy_ar.md`) on a domain you
control, add a contact method for users who cannot sign in (e-mail placeholder in
`LegalInfo.supportEmail`), then put the URL into `LegalInfo.accountDeletionUrl` and
the Play Console Data safety form. There is deliberately no unauthenticated deletion
API. A request received through the web page (from a user who cannot sign in) is
verified by the operator (e.g. reply from the account's e-mail / phone) and then
executed on the server with the **same** erasure logic:

```
php artisan account:delete <user-id | e-mail | phone> [--role=guardian|student|teacher]
```

(asks for confirmation; `--force` skips it; audit entry `initiated_by=operator_request`;
refuses the last active super admin). **ACTION REQUIRED:** define and document the
identity-verification step and a response time (e.g. within 30 days).
