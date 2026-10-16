# REPORTING_ARCHITECTURE.md

## One export mechanism, shared by every report in the system

Before this phase, `ReportController` (student progress, attendance, teacher performance, parent engagement, circle performance, admin dashboard) already had a working CSV-export pattern: any report request accepting `?format=csv` streams a download instead of the JSON envelope. Rather than re-implementing this for the eight new Finance reports, it was extracted into a shared trait — `App\Support\Http\Controllers\Concerns\RespondsWithReport` — and both `ReportController` and the new `FinanceReportController` now use it. This is a pure refactor (verified against `ReportServiceTest.php` before and after — no behavior change) and means **every report in the system, operational and financial, is filterable, paginated, and exportable the same way**, satisfying Part 6's "downloadable, filterable, exportable, printable" requirement uniformly rather than building a second, divergent mechanism for Finance.

## Coverage

| Domain | Controller | Reports |
|---|---|---|
| Student/operational | `ReportController` | student-progress, attendance, teacher-performance, parent-engagement, circle-performance, admin-dashboard |
| Finance | `FinanceReportController` | accounts-receivable, accounts-payable, cash-flow, profit-and-loss (= income statement), revenue, expenses, monthly-summary, yearly-summary |

Every endpoint above accepts `format=csv` and streams a `.csv` download with the exact same rows as the JSON `data` array — there is no separate "export" code path that could drift from what the JSON view shows.

## What "printable" means here

No native PDF/print renderer was added (see FINANCE_ARCHITECTURE.md's note on this) — a CSV opens directly in Excel/Sheets/Numbers, which covers the same practical need (a formatted, printable, shareable report) without introducing a new rendering dependency this codebase has no prior use of. This is an explicit, documented scope decision, not an oversight: PDF generation is flagged as a candidate follow-up if a literal PDF artifact (rather than a spreadsheet) turns out to be required.

## Filtering conventions

All reports share the same filter vocabulary where applicable: `date_from`/`date_to` (date range), `page`/`per_page` (capped at 100, matching the platform-wide pagination cap fixed in Part 9's audit pass), and report-specific filters (`circle_id`/`student_id` for operational reports; `year`/`month` for Finance's monthly/yearly summaries). This consistency means a single admin_app export helper could, in principle, drive every report screen — though building that generic UI layer was out of scope for this pass (the admin_app screens built in this phase call the new Finance/Payroll/Analytics endpoints directly for dashboard display, not through a generic report-browser UI).

---

## Reporting contract (authoritative — supersedes the coverage table above)

Every operational report lives under `/api/v1/reports/*`, is built from `ReportQueries` (one SQL expression per figure), is scoped by `ReportScope`, and returns `data` + `meta.pagination` + `meta.totals`. `format=csv|tsv` streams the **whole** filtered set from the same query (UTF-8 with BOM). The full parameter/response contract is in `api/openapi.yaml`.

### Role matrix

A = super_admin / center_owner, T = teacher, S = student, G = guardian.

| Report | A | T | S | G | Canonical source |
|---|---|---|---|---|---|
| overview | center | own circles (finance = null) | – | – | all below |
| student-performance | ✓ | own circles | self | linked child | all below |
| attendance, attendance/records | ✓ | own circles | self | linked child | attendances |
| sessions | ✓ | own circles | self | linked child | circle_sessions |
| learning, learning/records (memorization / revision / matn / listening) | ✓ | own circles | self | linked child | student_learning_records |
| homework | ✓ | own circles | self | linked child | homework |
| monthly-progress (read) | ✓ | own circles | self | linked child | monthly_progress_records |
| monthly-progress/generate | ✓ | – | – | – | attendances + student_learning_records |
| guardian-evaluations | ✓ | own circles | about self | only evaluations they wrote | parent_evaluations |
| students, circles, missing-evaluations | ✓ | own circles | – | – | multiple |
| teachers | ✓ | own row (hidden in app) | – | – | multiple |
| activity | ✓ | – | – | – | audit_logs, notifications |
| **finance reports** (`/finance/reports/*`) | ✓ | – | – | – | invoices, payments, transactions, payroll |
| own ledger (`/student/payments`, `/parent/my-children/{id}/payments`) | – | – | self | linked child | invoices, payments |

Rules enforced server-side (never trusted from the client):

- **Teacher** = circles they *currently* own (`circles.teacher_id`). After a reassignment the circle's whole history moves to the new teacher; the previous teacher loses access. A foreign `circle_id`, `teacher_id` or `student_id` returns 403.
- **Guardian** must send one `student_id` per request (422 without it, 403 if not linked), so two children are never blended.
- **Student** is pinned to themselves; another `student_id` returns 403.

### Finance

Finance reports are **super_admin and center_owner only**. This is enforced by route middleware (`role:super_admin,center_owner`) and `FinancePolicy::viewAny`, and mirrored in the admin app (`Permissions.viewFinance`, `ReportDef.finance`). Teachers, students and guardians get 403. Students and guardians see only their own ledger through the existing ledger endpoints, which is an existing business rule. No wider finance access exists without an explicit new business rule.

### Decision: students may read guardian evaluations about themselves

Kept as-is. The data is about the student, every request is pinned to their own id, they never see another student's evaluations and cannot write any. Guardians see only the evaluations they wrote themselves, never a co-guardian's. Covered by `ReportingHardeningTest`.

### Unknown vs zero

A figure with no underlying data is `null` and renders as "—" / "غير متاح"; a real zero stays `0`. This includes the student home card. Its attendance rate and average now come from the same SQL as the reports (`ReportQueries::summaryByStudent`), not the cached `student_statistics` row, which stores `0.0` for "no data".

### Endpoint classification

| Endpoint | Class | Notes |
|---|---|---|
| `/reports/*` (new set above), `/finance/reports/student-balances|payments|payroll-summary` | ACTIVE | Used by admin_app and student_app |
| `/finance/reports/{accounts-receivable, accounts-payable, cash-flow, profit-and-loss, revenue, expenses, monthly-summary, yearly-summary}` | ACTIVE | cash-flow and yearly-summary in the hub; the rest API-only; their csv exports the current page (aggregated in PHP) |
| `/analytics/executive-dashboard` | ACTIVE | ExecutiveDashboardScreen |
| `/students/{id}/tracking/profile` | ACTIVE | guardian "التقرير الشامل" |
| `/reports/monthly-progress/generate`, `PUT /reports/monthly-progress/{id}` | ACTIVE | the explicit recalculation and curation flows |
| `/statistics/students|teachers|circles/{id}` | LEGACY — deprecated header | cached aggregates; only the admin profile's memorized-pages figure; recalculate actions kept |
| `/reports/student-progress, monthly-attendance, teacher-performance, parent-engagement, circle-performance, admin-dashboard` | DEPRECATED — `Deprecation` + `Link: successor-version` | no app screen calls them |
| `/student/memorization-sessions`, `/student/revision-sessions` | DEPRECATED | read legacy tables; the student app now uses `reports/learning/records` |
| `/analytics/teacher-dashboard`, `/students/{id}/tracking/history`, `/parent/dashboard` | DEAD (no client) | kept; not removed without a product decision |
| `/memorization-sessions`, `/revision-sessions` (CRUD) | LEGACY write path | admin "الحفظ والمراجعة" screen still writes here; **no report reads these tables** |
