# FINANCE_ARCHITECTURE.md

## Model: lightweight, not double-entry

The system does **not** implement full double-entry bookkeeping (no debit/credit ledger, no chart of accounts in the GAAP sense). That was a deliberate scope decision: every report the spec requires (Accounts Receivable, Accounts Payable, Cash Flow, P&L/Income Statement, Revenue/Expense reports, Monthly/Yearly summaries) can be produced correctly from three simple, real-world concepts:

1. **Invoices** — money owed *to* the center (Accounts Receivable when unpaid/partial).
2. **Expenses** — money owed *by* the center (Accounts Payable when pending/approved-but-unpaid).
3. **Transactions** — an immutable ledger of money that has *actually moved*, created automatically the instant a `Payment` is recorded or an `Expense` is marked paid.

This separation is the core invariant of the whole module: **"money owed" and "money that has moved" are different tables**, so A/R and Cash Flow can never accidentally double-count or contradict each other. A `FinancialPeriod` (typically a calendar month) is auto-resolved by date-range containment and attached to every transaction, for period-based reporting/closing.

## Entities

| Table | Purpose | Key fields |
|---|---|---|
| `financial_accounts` | Where money sits (cash box, bank account) | `code`, `type`, `opening_balance` |
| `revenue_categories` | Subscription, registration fee, material fee, donation, other | `code` (unique), `is_active` |
| `expense_categories` | Teacher/staff salaries, rent, utilities, marketing, technology, misc | `code` (unique), `is_active` |
| `financial_periods` | Month/year boundary + open/closed status | `start_date`, `end_date`, `status` |
| `invoices` + `invoice_items` | A charge to a student, with line items | `invoice_number` (generated), `subtotal/tax/total`, `amount_paid`, `status` |
| `payments` | Money received (against an invoice, or unattached e.g. a donation) | `invoice_id?`, `student_id?`, `amount`, `method` |
| `expenses` | Money owed/paid out | `status: pending→approved→paid`, `amount`, `expense_category_id` |
| `transactions` | Immutable ledger of real cash movement | `type: income|expense`, `source_type/source_id`, `financial_period_id` |

Reference data (`financial_accounts`, `revenue_categories`, `expense_categories`) is seeded by the schema migration with realistic Arabic-labeled defaults, so the module is usable out of the box.

## Business rules enforced in code, not just the UI

- **Invoice totals/`invoice_number` are computed server-side** at creation (`CreateInvoiceAction`) — never trusted from client input. The invoice number format is `INV-{Ym}-{6 random chars}`, collision-checked against `withTrashed()`.
- **`amount_paid`/`status` on an invoice are maintained exclusively by `RecordPaymentAction`** — never directly editable via the update endpoint. A payment cannot push `amount_paid` past `total + 0.01` (small float tolerance); doing so throws a 409 Conflict.
- **An invoice with any payment recorded can never be voided** (`UpdateInvoiceAction` throws `ConflictException`) — voiding is only for invoices nobody has paid against yet.
- **An expense can only be edited while `pending`** (`UpdateExpenseAction`) — once `approved`/`paid`, its category/amount/date are locked, because a `Transaction` may already reference it.
- **A `Transaction` is created in exactly two places** — `RecordPaymentAction` (income) and `MarkExpensePaidAction` (expense) — both delegate to the shared `TransactionRecorder` service, so there is exactly one code path that can ever write to the ledger.
- **Real bug caught by this module's own tests**: `Invoice::$fillable` was initially missing `amount_paid`, so `RecordPaymentAction`'s mass-assignment update silently dropped the paid amount while `status` still flipped to `paid` — a full-payment test caught the mismatch (status "paid" but `amount_paid` "0.00") before it reached any real user. Fixed by adding the column to `$fillable`.

## Reports (`FinanceReportService`, all center_owner+ only)

All eight reports are paginated (`LengthAwarePaginator`) and — via the shared `RespondsWithReport` trait — support `?format=csv` streamed download identically to every other report in the system.

| Report | Method | Notes |
|---|---|---|
| Accounts Receivable | `accountsReceivable()` | Unpaid/partial/overdue invoices, aged into `current / 1-30 / 31-60 / 61-90 / 90+` buckets computed in PHP (portable across DB drivers) |
| Accounts Payable | `accountsPayable()` | Pending/approved expenses, same aging buckets |
| Cash Flow | `cashFlow()` | Transactions grouped by month, income/expense/net |
| Profit & Loss **(= Income Statement)** | `profitAndLoss()` | One computation satisfies both names in the spec — they are the same report under two accounting conventions. Revenue rows + expense rows + one `summary` row (net profit) |
| Revenue Report | `revenueReport()` | Income transactions grouped by revenue category |
| Expense Report | `expenseReport()` | Expense transactions grouped by expense category |
| Monthly Financial Summary | `monthlySummary()` | Revenue, expenses, net profit, invoices issued/paid, outstanding receivables for one month |
| Yearly Financial Summary | `yearlySummary()` | Same shape, one row per month, 12 rows |

Non-tabular aggregates (cash flow, P&L, summaries) are computed as PHP collections and wrapped in a manually-constructed `LengthAwarePaginator` (`paginateCollection()`) rather than raw SQL date-grouping functions — this keeps the queries portable between SQLite (tests) and MySQL/Postgres (production), since `DATE_FORMAT`/`strftime` are not portable SQL.

## Payment-reminder automation (pre-existing job, now activated)

A `SendPaymentReminders` job already existed in the codebase as a graceful no-op ("skipped because the payments table does not exist"), clearly written in anticipation of this exact module. It is now real: it finds invoices due within 3 days or already overdue, flips `sent`/`partial` → `overdue` automatically, and notifies each guardian once per day (Cache-keyed dedup) via the existing `StudentAlertNotification` pipeline — reusing `Student::guardians()` (the many-to-many relation), matching the pattern in `SendStudentReports`.

## Routes

All under `/api/v1/finance/*`, gated `role:super_admin,center_owner` (see `routes/api/v1/finance.php`): `accounts`, `revenue-categories`, `expense-categories`, `periods` (reference-data CRUD), `invoices` (index/show/store/update), `payments` (index/show/store — create-only, no update/void semantics), `expenses` (index/show/store/update + `/approve` + `/mark-paid`), and `reports/*` (the eight reports above).

## Why some things were deliberately *not* built

- **No PDF export** — CSV was chosen as the pragmatic, already-proven export mechanism (reusing the pre-existing `ReportController` CSV pattern). PDF generation would require a new rendering dependency with no existing precedent in the codebase; flagged as a follow-up, not silently dropped.
- **No manual double-entry ledger UI** — since the system is not double-entry, there is nothing to reconcile manually; the `transactions` table is derived, not hand-edited.
