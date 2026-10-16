# SECURITY_AUDIT.md

> Verified against actual backend code. The backend has a strong security posture; findings are limited.

## Posture summary
| Vector | Control (evidence) | Verdict |
|---|---|---|
| **Teacher/guardian/student injection** | `teacher_id/circle_id/attendance_date` `prohibited` in all 13 field requests; identity derived from session/circle/auth | ✅ blocked |
| **Mass assignment** | No model uses `$guarded = []`; **30 models** use `$fillable` allowlists; `BaseRequest::validatedData()` passes only validated keys | ✅ |
| **Broken authorization** | Per-feature Policies (`view/create/update/delete`) + `role:*` middleware; requests `authorize()` delegate to policies | ✅ |
| **Tenant / cross-center isolation** | center_owner scoped to `center->owner_id === user->id`; teacher to `teacher->user_id === user->id`; per-feature `*Query` scoping; `AcademicSessionGuard` re-checks tenant | ✅ |
| **IDOR** | Ownership re-checked in services/policies, not just route binding (e.g., `AttendanceService::assertSessionAccess`, `ParentService::assertGuardianChild`) | ✅ |
| **Privilege escalation** | Salary gated `role:super_admin,center_owner`; admin-dashboard `role:super_admin`; lifecycle/enrollment/users `role:super_admin,center_owner` | ✅ |
| **Concurrency / race conditions** | `lockForUpdate()` on session/student/attendance/enrollment; duplicate checks with `withTrashed()`; capacity check under lock | ✅ |
| **Replay / double-submit** | `idempotency` middleware (Idempotency-Key) on sensitive writes (sessions, enrollments, evaluations, lifecycle, devices) | ✅ |
| **Token security** | Sanctum bearer; both apps store token in `flutter_secure_storage`; 401 → delete token + logout | ✅ |
| **Rate limiting** | `throttle:login/otp/password-reset/sensitive-write/...` named limiters | ✅ |
| **Sensitive data exposure** | `UserResource` omits password/hashes; student PII gated (`viewStudentPii` admin-only) | ✅ |

## Findings
### 🟠 S1 — Cross-model integrity gap in memorization/revision (was Rule-14 bypass) — FIXED
Legacy create actions accepted **suspended** students. Fixed with an active-student guard; **37 backend tests pass**. Residual: these endpoints validate `student_id` only as `exists` and (for admins) don't re-check that the student's **center** matches the circle's center the way `EnrollStudentAction` does — low risk because teacher-owns-circle is enforced and the student-active/enrollment invariants hold elsewhere; recommend adding the same-center assertion for defense in depth.

### 🟢 S2 — Enrollment dual-model (Finding D) — FIXED
`enrollments` vs `circle_student` divergence could misrepresent circle membership to any path trusting the pivot. **Fixed:** enroll/close now keep the pivot synced with the canonical `enrollments` table, so membership reads are consistent. Verified by `EnrollmentSyncTest`.

### 🟡 S3 — Offline queue exploit surface (admin) — low
Queued ops replay whatever body was captured offline; the server re-validates every op (prohibited fields, policies, guards), so a tampered local queue cannot bypass server rules — it just gets 422/403 and is dropped. ✅ server-authoritative. No fix required; the risk is data-loss (Finding A), not escalation.

## No issues found for
SQL injection (Eloquent/param binding), open mass-assignment, missing auth on protected routes, secret leakage in `UserResource`, or cross-tenant list leakage (query scoping verified).
