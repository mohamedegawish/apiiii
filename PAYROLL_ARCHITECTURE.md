# PAYROLL_ARCHITECTURE.md

## Salary models

The `Teacher` model already carried the four salary fields the schema needed (`salary_type`, `salary_amount`, `hourly_rate`, `per_session_rate`, `per_student_rate`) via the existing `SalaryType` enum (`fixed | per_session | per_student | hourly`). The spec's request for a "hybrid" model is satisfied not by a fifth `SalaryType` case, but by layering **bonus/penalty/overtime/deduction on top of whichever base type applies** — every teacher's pay is base-by-type *plus* performance adjustments, which is exactly what "hybrid" means in practice. No schema change to `teachers` was needed.

## Schema (pre-existing, now wired up)

`payroll_runs` (one per pay period: `period_start`/`period_end`, unique together, `status: draft→generated→approved→paid`) and `payroll_items` (one per teacher per run, unique on `[payroll_run_id, teacher_id]`, FK to `expenses.id` nullable) already existed in the database from schema work earlier in this phase. This document describes the feature code built on top of that schema.

## `PayrollCalculator` — the only place pay is computed

Given a `Teacher` and a date range, it computes, from **real data only**:

- `sessions_count` / `completed_sessions_count` — from `circle_sessions` in range.
- `attendance_rate` — present+late / total `attendances` rows for that teacher in range.
- `evaluation_completion_rate` — reuses `CircleSessionCompletionQuery` (built earlier in this session for the session-completion gate) to check, per completed session, whether every present student has a score or explicit N/A for every `LearningRecordType`. This is the same completion check that gates whether a *teacher* can mark a session complete, now reused to score how consistently they actually did it.
- `students_count` — distinct students with an attendance row in range.
- `homework_reviews_count` — `Homework` rows this teacher graded (`status=graded`) in range.
- **Base amount**, by `salary_type`:
  - `fixed` → `teacher.salary_amount` flat.
  - `per_session` → `completed_sessions_count × per_session_rate`.
  - `per_student` → `students_count × per_student_rate`.
  - `hourly` → real hours computed from each completed session's actual `start_time`/`end_time` (not fabricated — `circle_sessions` already stores both), summed and multiplied by `hourly_rate`.
- **Bonus**: 5% of base if both `attendance_rate ≥ 90` and `evaluation_completion_rate ≥ 90`.
- **Penalty**: automatically proportional to the evaluation-completion gap (`base × (1 − completion/100) × 0.1`).
- **Deduction / overtime**: deliberately left at `0` at generation time. There is no "unpaid leave" concept anywhere else in the schema to derive a deduction from, and no contracted-hours baseline to compare actual hours against for overtime — fabricating either would be a fake number. Both are exposed for **manual admin entry** via `AdjustPayrollItemAction` (`PUT /payroll-items/{id}/adjust`), which recomputes `total_amount` and only works while the item is still `pending`.

## Lifecycle

```
generate  → PayrollRun(status=generated) + one PayrollItem(status=pending) per active teacher
approve   → PayrollRun/items → status=approved, approved_by/approved_at stamped
mark-paid → for each item: create Expense(category=teacher_salaries, status=pending)
              → ApproveExpenseAction → MarkExpensePaidAction (creates the Transaction)
              → item.expense_id set, item.status=paid
            → PayrollRun.status=paid
```

The "mark paid" step deliberately **reuses `ApproveExpenseAction`/`MarkExpensePaidAction` from the Finance module** rather than writing a parallel expense-creation path. This means every payroll payout automatically becomes a real `Expense` → real `Transaction`, and therefore automatically shows up in Accounts Payable (while pending), Cash Flow, and P&L (once paid) — payroll and finance reporting can never disagree about a payout, because they are the same underlying row.

Generating a duplicate run for an identical `[period_start, period_end]` is rejected with a 409 before insert (checked explicitly, not left to the DB unique-constraint exception).

## Teacher-facing read path

`GET /payroll/mine` (role:teacher) resolves the caller's own `Teacher` record and returns:
- `current` — their pending or approved item (whichever exists), if any.
- `summary` — `pending_amount`, `last_paid_amount`, `lifetime_paid_total`, `runs_paid_count`.
- `history` — their last 12 payroll items across all runs, newest first.

A teacher can only ever see their **own** `PayrollItem` rows — `PayrollPolicy::viewOwn()` checks `$item->teacher->user_id === $user->id` (center_owner+ bypasses this for admin views).

## Policy

`PayrollPolicy` — `viewAny`/`manage` are center_owner+ only (generate/approve/pay/adjust are all gated this way); `viewOwn` additionally allows a teacher to read their own item.

## Routes

`routes/api/v1/payroll.php`: `payroll-runs` (index/store/show/approve/mark-paid, `role:super_admin,center_owner`), `payroll-items/{id}/adjust` (same gate), `payroll/mine` (`role:teacher`).
