# ANALYTICS_ARCHITECTURE.md

## Two audiences, two services, one shared metric core

- **`TeacherAnalyticsService`** — a single teacher's own dashboard.
- **`AdminAnalyticsService`** — the school-wide executive dashboard, which **reuses `TeacherAnalyticsService::performanceMetrics()`** to rank teachers rather than recomputing a separate scoring formula. A teacher's own dashboard and the admin's "top performers" widget can never disagree about what a given teacher's performance score is, because it's the same method call either way.

Both are read-only aggregations over `circle_sessions`, `attendances`, `student_learning_records`, `homework`, `parent_evaluations`, `students`, `teachers` — plus, for the admin dashboard's finance widget, a delegated call into `FinanceReportService::monthlySummary()` (see FINANCE_ARCHITECTURE.md) rather than a second finance computation.

## Teacher dashboard (`GET /analytics/teacher-dashboard`, `role:teacher`)

```
sessions:      today[], upcoming[] (next 10 scheduled), completed_count_this_period
performance:   attendance_rate, evaluation_completion_rate, student_completion_rate,
               homework_completion_rate, parent_engagement_rate, performance_score
learning_growth: per type (memorization/revision/matn/listening) — current vs previous
               period average score, and the delta ("growth")
ranking:       { position, out_of, performance_score }
```

Two metrics are worth explaining because they are easy to conflate:

- **`attendance_rate`** — percentage of individual attendance *records* marked present/late.
- **`student_completion_rate`** — percentage of the teacher's *currently-enrolled roster* who attended at least once this period. A teacher could have 100% attendance_rate (every recorded attendance was "present") while student_completion_rate is low, if most of the roster simply never showed up to be recorded at all. They measure different failure modes and are both real, not redundant.

`performance_score` is a weighted composite: `0.3×attendance + 0.3×evaluation_completion + 0.2×homework_completion + 0.2×parent_engagement`. This weighting is a documented default, not a spec-mandated formula — it privileges the two metrics the platform already treats as gating a session's completion (attendance, evaluation) over the two secondary ones.

`ranking` recomputes `performanceMetrics()` for every other active teacher over the same period to find the caller's position. This is O(teacher count) queries — acceptable at the scale of a single Quran center (tens, not thousands, of teachers); flagged here as the first thing to optimize (e.g. with a cached leaderboard) if the teacher count ever grows large.

## Executive dashboard (`GET /analytics/executive-dashboard`, `role:super_admin,center_owner`)

```
students:        total, active, inactive, new_this_period (by enrolled_at)
teachers:        total, active, top_performers[5] (via TeacherAnalyticsService)
sessions:        scheduled / in_progress / completed / locked / cancelled — this month
learning_growth: school-wide average score per type, current vs previous month
parent_engagement: expected vs submitted evaluations, school-wide, this month
finance:         revenue / expenses / profit / outstanding_receivables — delegated to
                  FinanceReportService::monthlySummary()
```

All widgets are scoped to the current calendar month (the same period `FinanceReportService`'s monthly summary uses), so every number on the dashboard describes the same window.

## Why no separate "Analytics" database tables

Every number here is derivable, correctly and cheaply at this data scale, from tables that already exist and are already the source of truth elsewhere (sessions, attendance, learning records, evaluations, finance). Introducing a materialized/cached analytics table would add a second source of truth that could drift from reality — the explicit design principle carried through this entire phase (see also FINANCE_ARCHITECTURE.md's ledger separation and PAYROLL_ARCHITECTURE.md's expense-reuse). If dashboard load time becomes a real problem at scale, the fix is a cache layer in front of these services, not a new schema.
