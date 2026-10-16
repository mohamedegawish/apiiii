# Parent Access Audit

## Verified Capabilities

| Capability | API | App | Authorization |
|---|---|---|---|
| View children | `/parent/my-children` | Parent home/children | Guardian-owned children only |
| View sessions | `/parent/my-children/{student}/sessions` | Session evaluation status | Guardian-owned child only |
| Evaluate prayer and behavior per session | `POST /parent/evaluations` | Home evaluation screen | Completed and locked sessions; locked sessions are still reviewable |
| View parent prayer evaluations | `/parent/evaluations?filter[student_id]=...` | Prayer screen | Guardian scope enforced |
| View parent behavior evaluations | `/parent/evaluations?filter[student_id]=...` | Behavior screen | Guardian scope enforced |
| View teacher learning evaluations | `/parent/my-children/{student}/learning-records` | Parent memorization screen | Child ownership enforced |
| View teacher and session context | Learning-record resource fields | Memorization cards | Teacher/session relation loaded server-side |
| View memorized ayah/surah totals | `/students/{student}/tracking/profile` | Child profile/report | Guardian ownership policy |
| View attendance | `/parent/my-children/{student}/attendance` | Attendance screen/report | Guardian-owned child only |
| View invoices and payments | `/parent/my-children/{student}/payments` | Parent payments screen | Guardian-owned child only |

## Payment Behavior

The parent ledger shows invoices and payment entries together, plus total invoiced, total paid, and balance due. It is read-only for the parent.

## Evaluation Behavior

Prayer and behavior evaluations are stored against a specific child and circle session. A parent can submit one evaluation per child/session and can review it afterward. The API validates the child relationship, enrollment, session completion, and duplicate submissions.

## Memorization Behavior

The parent sees the teacher evaluation type, score/result, session date, teacher name, range, and notes when available. The tracking profile also exposes total memorized ayahs and surahs.

## Security Checks

- A guardian cannot read another guardian's child records, attendance, sessions, evaluations, or payments.
- Teacher/admin roles cannot access guardian child endpoints.
- Parent payment data is scoped by the authenticated guardian-child relationship, not by a client-supplied unrestricted student filter.

## Verification

- Student app `flutter analyze`: passed.
- Student app `flutter test`: passed, 24 tests.
- Parent/API feature tests: passed.
- Full Laravel suite: passed, 352 tests and 1433 assertions.
