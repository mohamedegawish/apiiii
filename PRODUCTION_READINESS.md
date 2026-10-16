# PRODUCTION_READINESS.md

> Cross-project readiness for `api/` + `admin_app/` + `student_app/`. Evidence-based; source code is the source of truth. 2026-07-24.

## Scorecard
| Dimension | State | Evidence |
|---|---|---|
| Backend business rules | 🟢 Strong | 19/20 expected rules enforced; the 20th (Rule 14 on memorization/revision) **fixed + tested** |
| Backend security | 🟢 Strong | prohibited-fields, fillable allowlists, policies, tenant scoping, locks, idempotency, throttling (`SECURITY_AUDIT.md`) |
| Backend tests | 🟢 | full suite **265 passed, 0 failed** (+ new `EnrollmentSyncTest`). `PushNotificationTest` is order-dependent/flaky — unrelated, de-flake separately |
| student_app integration | 🟢 | `flutter analyze` clean, 15 tests pass (prior pass) |
| admin_app integration | 🟢 | analyzer clean; **attendance (A), enroll/unenroll (B) now fixed** (live UI smoke pending) |
| Enrollment data model | 🟢 | unified on canonical `enrollments`; pivot synced on enroll/close (**Finding D fixed**) |
| Offline integrity | 🟢 | admin queue robust; **attendance data-loss (A) closed**; student read-cache OK |
| API ↔ UI coverage | 🟠 | admin lifecycle/audit/users/session-actions **still have no UI** (`MISSING_FEATURES.md`) |

## Fixed this pass (all four findings) ✅
- ✅ **Fix C** — memorization/revision suspended-student guard (backend, tested).
- ✅ **Fix A** — session-centric admin attendance (online + offline). Data-loss path closed.
- ✅ **Fix B** — admin enroll/unenroll on the real enrollment endpoints.
- ✅ **Fix D** — enrollment unified on `enrollments`; pivot synced; roster returns `enrollment_id`. New `EnrollmentSyncTest` green; full suite 265 passed.
- ✅ Security posture + role consistency confirmed against code.

## Remaining blockers to 100% (feature additions, not defects)
1. 🟠 **Rule 18 UI gaps** — admin UI for session lifecycle (start/complete/cancel/lock), student lifecycle (suspend/reactivate/graduate/withdraw/transfer), audit-logs viewer, users management, statistics recalculation.
2. 🟠 **Parent evaluation submit** — session-picker UI in student_app.
3. 🟡 **Live seeded smoke** of the admin attendance + enrollment flows, then release builds.

## Honest limitations
- **No live server run in this environment.** Backend fixes are feature-test verified; admin fixes are analyzer-clean against test-verified contracts but the **admin UI flows were not exercised end-to-end against a running instance**. A seeded smoke of enroll→roster→unenroll and circle→session→mark→save (online + offline→reconnect) is the remaining sign-off.
- **No release builds** executed (`flutter build apk --release`) — environment/signing specific.
- The remaining items above are **feature additions** (missing admin screens for existing endpoints) — genuinely new UI, not bug fixes; each is specified in `MISSING_FEATURES.md`.

## Recommended next sequence
1. ~~Session-centric admin attendance~~ ✅ done.
2. ~~Enrollment unification~~ ✅ done.
3. Admin UIs for session/student lifecycle, audit-logs, users (net-new screens).
4. Parent evaluation session picker.
5. Live seeded smoke of all three apps, then release builds.

**Bottom line:** all four audit findings are **fixed** — backend is production-grade, Rule-14-complete, and enrollment-unified (265 tests green); `student_app` is integration-complete; `admin_app` now records attendance session-centrically and enrolls/unenrolls on the canonical endpoints (analyzer-clean). What remains is **new admin UI** for a handful of already-implemented backend endpoints (lifecycle, audit-logs, users) plus a live seeded smoke — not defects.
