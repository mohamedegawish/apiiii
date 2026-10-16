# DATABASE_CHANGES.md

Scope: the two migrations added for this phase (Finance + Payroll). Student Tracking and Analytics are pure read-aggregations over existing tables (`attendances`, `student_learning_records`, `parent_evaluations`, `circle_sessions`, `homework`, `teachers`, `students`) and required no schema changes.

## `2026_07_30_000006_create_finance_tables.php`

| Table | Purpose | Notable columns / constraints |
|---|---|---|
| `financial_periods` | Month/year boundaries for period-based reporting | unique `[start_date, end_date]`; `status: open\|closed` |
| `revenue_categories` | Reference data (subscription, registration fee, material fee, donation, other) | unique `code`; soft-deletes |
| `expense_categories` | Reference data (teacher/staff salaries, rent, utilities, marketing, technology, misc) | unique `code`; soft-deletes |
| `financial_accounts` | Cash box / bank account | unique `code`; `opening_balance` |
| `invoices` | Charge to a student | unique `invoice_number`; FK `student_id`/`revenue_category_id` (restrict on delete); `amount_paid`, `status` |
| `invoice_items` | Line items | FK `invoice_id` (cascade on delete) |
| `payments` | Money received | FK `invoice_id`/`student_id` nullable (nullOnDelete — a payment record must survive even if its invoice/student is later removed); FK `revenue_category_id` (restrict) |
| `expenses` | Money owed/paid | `status: pending\|approved\|paid`; nullable free-text `reference_type`/`reference_id` (deliberately no FK — a future expense source could come from more than one table) |
| `transactions` | Immutable cash-movement ledger | `type: income\|expense`; `source_type`/`source_id` (payment or expense); FK `financial_period_id` nullable (nullOnDelete) |

The migration also seeds realistic Arabic-labeled reference data (5 revenue categories, 7 expense categories, 2 financial accounts) via a private `seedCategories()` method, so the Finance module is usable immediately after migrating — no separate seeder command required.

## `2026_07_30_000007_create_payroll_tables.php`

| Table | Purpose | Notable columns / constraints |
|---|---|---|
| `payroll_runs` | One pay period (typically a month) | unique `[period_start, period_end]`; `status: draft\|generated\|approved\|paid` |
| `payroll_items` | One teacher's computed pay within a run | unique `[payroll_run_id, teacher_id]`; FK `expense_id` nullable (nullOnDelete — set once the run is marked paid, see PAYROLL_ARCHITECTURE.md); snapshots the performance metrics and pay components computed at generation time |

`payroll_items` intentionally stores **computed snapshots** (`sessions_count`, `attendance_rate`, `base_amount`, `bonus_amount`, etc.) rather than being a view over live data — a payroll run's numbers must stay stable once generated, even as new sessions are recorded afterward for the same historical period.

## Code-level fix (not a migration, but database-relevant)

`Invoice::$fillable` was missing `amount_paid` — a real bug caught by this phase's own test suite (`FinanceTest.php`): `RecordPaymentAction`'s mass-assignment update silently dropped the paid amount while `status` still flipped to `'paid'`, leaving `amount_paid` frozen at `0.00` on a fully-paid invoice. This is a fillable-array fix, not a schema change, but is called out here because the symptom (a column silently not persisting) looks database-related. See FINANCE_ARCHITECTURE.md for the full account of how the test caught it.

## Migration safety

Both migrations were run against the test suite's SQLite database (`php artisan test`, 330 passing) and are additive-only: no existing table is altered, no existing column is dropped or renamed, and no existing foreign key is changed. They are safe to run against a populated production database with zero downtime risk to existing functionality.
