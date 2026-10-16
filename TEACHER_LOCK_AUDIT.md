# Teacher Lock Permission — Audit

> Product decision implemented: a teacher (المحفّظ) may now lock their own completed sessions. Previously this was restricted to `super_admin`/`center_owner`. This document records the review of the existing implementation, the change made, and the tests added to prove it.

## 1. What was reviewed

| Item | File | Finding before this change |
|---|---|---|
| `CircleSessionPolicy::lock()` | `app/Features/CircleSessions/Policies/CircleSessionPolicy.php:54-57` | `return $this->isSuperAdmin($user) \|\| $this->hasRole($user, Role::CenterOwner);` — teacher branch entirely absent, unlike every other action on this policy (`view`, `start`, `complete`, `cancel` all delegate to `canManage()`, which already has a correct teacher-ownership branch). |
| Session-transition guard | `app/Features/CircleSessions/Actions/TransitionCircleSessionAction.php` + `app/Enums/CircleSessionStatus::allowedTransitions()` | Already correct and role-independent: `Completed → [Locked]` is the only edge into `Locked`; `Scheduled`/`InProgress`/`Cancelled` have no path to `Locked`. Enforced atomically under `lockForUpdate()`, inside a DB transaction, for **every** caller regardless of role. Nothing here needed to change. |
| Route + middleware | `routes/api/v1/platform.php:27` — `POST sessions/{circleSession}/lock` under `auth:sanctum`, `active`, `idempotency`, `throttle:sensitive-write` | Role-neutral at the route level; all role/ownership decisions are delegated to the policy. No middleware change needed. |
| Controller | `app/Features/CircleSessions/Controllers/CircleSessionController.php:90-105` | `lock()` → shared `transition()` helper → `$this->authorize('lock', $session)` before calling the service. Correctly wired; no change needed. |
| `admin_app` teacher UI | `lib/ui/teacher/today_screen.dart`, `lib/ui/teacher/widgets/session_card.dart` | **Already built** a "قفل الجلسة" (lock) button on `completed` session cards in the teacher's Today dashboard, calling `Backend.lockSession()` → `POST /sessions/{id}/lock`, with a confirmation dialog and error-snackbar handling. This button was reachable but non-functional for teachers until this fix (every tap would have received a 403). No client-side role gate exists that needed removing — the client was already correctly built for this permission; only the server was behind. |

## 2. The change

One line, in `app/Features/CircleSessions/Policies/CircleSessionPolicy.php`:

```diff
 public function lock(?User $user, CircleSession $session): bool
 {
-    return $this->isSuperAdmin($user) || $this->hasRole($user, Role::CenterOwner);
+    return $this->canManage($user, $session);
 }
```

`canManage()` (unchanged, already existed) is:

```php
private function canManage(?User $user, CircleSession $session): bool
{
    if ($this->isSuperAdmin($user)) {
        return true;
    }

    if ($this->hasRole($user, Role::CenterOwner)) {
        return true;
    }

    return $this->hasRole($user, Role::Teacher)
        && $user?->teacher?->id === $session->teacher_id
        && $session->circle?->teacher_id === $session->teacher_id;
}
```

This was the minimal correct fix rather than a bespoke lock-only check, because it makes `lock` consistent with `view`/`start`/`complete`/`cancel` (all five actions on this model now share one authorization rule), and it double-checks ownership two ways: the session's own `teacher_id` **and** the circle's currently-assigned `teacher_id` must both match the caller. A teacher reassigned off a circle after the session was scheduled cannot lock a session that's now attributed to someone else's circle.

The `Completed`-only status requirement was **not** re-implemented in the policy — it was already enforced independently, before authorization is even consulted for the *value* being transitioned to, by `CircleSessionStatus::canTransitionTo()` inside `TransitionCircleSessionAction`, which throws a `ConflictException` (HTTP 409) for any other source status. This guard is role-blind by design: it protects `super_admin`/`center_owner` from the same mistake, not just teachers.

## 3. Security implications

- **No privilege widening beyond the stated decision.** A teacher gains exactly one new capability — locking a session that is (a) their own and (b) already `completed`. They gain no ability to lock another teacher's session, no ability to skip the `completed` state, and no change to any other policy method.
- **No IDOR introduced.** The double ownership check (`session.teacher_id` and `circle.teacher_id` both matching) closes the gap where a session's `teacher_id` could theoretically drift from its circle's current teacher (e.g. after a circle reassignment) — in that edge case neither the old nor the new teacher can lock it, which is the conservative direction to fail in.
- **No new attack surface.** No new route, no new middleware, no new request/validation surface — the existing `POST /sessions/{id}/lock` endpoint, already idempotency-protected and rate-limited (`throttle:sensitive-write`), is unchanged.
- **Consistent with the rest of the lifecycle.** `start`/`complete`/`cancel` were already teacher-ownership-gated via the same `canManage()` helper; `lock` was the one outlier. This fix removes an inconsistency rather than introducing a new authorization pattern.
- **Admin permissions unchanged**, verified by an explicit regression test (below): `super_admin`/`center_owner` can still lock any session regardless of teacher.

## 4. Tests added

`tests/Feature/CircleSessionModuleTest.php` (policy-level, matching the file's existing convention):
- `allows a teacher to lock their own completed session`
- `prevents a teacher from locking another teacher's completed session`

`tests/Feature/CircleSessionLockTest.php` (new file, HTTP-level — exercises the real route, auth, policy, and transition action together, not just the policy object in isolation):
- `lets a teacher lock their own completed session` — `POST /sessions/{id}/lock` as the owning teacher → `200`, response status `locked`, `locked_at` set.
- `forbids a teacher from locking another teacher's completed session` → `403`, session status unchanged.
- `rejects a teacher locking their own session in an invalid status` — parameterized over `scheduled` / `in_progress` / `cancelled` → `409` in all three cases, session status unchanged.
- `still lets a center owner lock any teacher's completed session` — explicit regression check that the admin path is untouched.

## 5. Final verification

```
$ php artisan test --filter=CircleSessionLockTest
Tests:  6 passed (14 assertions)

$ php artisan test --filter=CircleSessionModuleTest
Tests:  7 passed (19 assertions)

$ php artisan test
Tests:  1 failed, 283 passed (1094 assertions)
```

The one failure (`PushNotificationTest > it keeps database notification safe when local FCM credentials are unavailable`) is unrelated to this change — it asserts a notification count of `1` inside the full suite run but passes cleanly in isolation (`php artisan test --filter=PushNotificationTest` → 8/8 passed). This is a pre-existing test-order/state-leak flake in the push-notification suite, not a regression from the policy change; it touches no code this change modified. Flagged separately in [FINAL_PRODUCTION_AUDIT.md](FINAL_PRODUCTION_AUDIT.md)'s testing section rather than fixed here, per the instruction not to modify anything unrelated to teacher lock permission.

No changes were required in `admin_app` or `student_app` — the teacher-facing lock button already existed and called the correct endpoint; it was simply blocked server-side until now.
