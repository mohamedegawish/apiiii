# STUDENT_TRACKING_ARCHITECTURE.md

## Source of truth

`StudentTrackingService` reads exclusively from `attendances`, `student_learning_records` (typed `memorization|revision|matn|listening` via the `LearningRecordType` enum), and `parent_evaluations` — the same tables every other analytics/reporting/payroll service in this phase reads from. There is no separate cached "tracking" table; a profile is always computed live from current data.

## Granularity & periods

Four granularities are supported: `weekly`, `monthly` (default), `quarterly`, `yearly`. For any reference date, the service computes the **current period's range** and the **immediately preceding period of the same granularity** (e.g. this month vs. last month), so every metric can be compared against its own recent history — this comparison is what risk-flag detection is built on.

## `profile(Student, granularity, referenceDate?)`

Returns, for the current period:

- **`attendance`** — `total_records`, `present_count`, `rate` (null if no records, not `0`, so "no data" is distinguishable from "0% attendance").
- **`learning`** — one block per type (`memorization`/`revision`/`matn`/`listening`): `records_count`, `average_score` (of scored records only), `na_count`.
- **`parent_engagement`** — `expected_evaluations` (distinct sessions the student was *present* at), `submitted_evaluations` (evaluations actually recorded for those sessions), `engagement_rate`. Expected is derived from real attendance, not assumed to be "one per session".
- **`risk_flags`** — see below.

## `history(Student, granularity, periods=6)`

A trend of up to 24 consecutive periods (oldest → newest) with `attendance_rate`, `memorization_average_score`, `revision_average_score`, `parent_engagement_rate` — the data behind a weekly/monthly/quarterly/yearly performance chart.

## Risk-flag detection

Exactly the four flags the spec names, each requiring **both periods to have real data** (never flags on an absence of data, only on a real decline):

| Flag | Condition |
|---|---|
| `attendance_decline` | previous attendance rate − current ≥ 15 percentage points |
| `memorization_decline` | previous memorization average score − current ≥ 10 points |
| `revision_decline` | previous revision average score − current ≥ 10 points |
| `low_parent_engagement` | current period had ≥1 expected evaluation and `engagement_rate < 50` |

These thresholds (15pp / 10pts / 50%) are reasonable, documented defaults, not values derived from any external spec — they are the first tunable knob if real usage data suggests otherwise.

## Access control

Reuses the **existing** `StudentPolicy::view()` rather than introducing a new policy — a teacher can only view tracking for a student enrolled in one of their own active circles; center_owner+ can view any student; this is the exact same IDOR-safe check already used elsewhere for student records, so tracking data can't leak across teachers.

## Routes

`GET /students/{student}/tracking/profile` and `GET /students/{student}/tracking/history`, both under `routes/api/v1/student-tracking.php`, `auth:sanctum` + the reused policy check performed in the controller (not the FormRequest, since the FormRequest doesn't have the resolved `Student` model available at `authorize()` time in this route shape).
