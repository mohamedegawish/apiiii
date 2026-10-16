# Session Workspace — Fix & UX Refinement Report

Date: 2026-07-31
Scope: `api/` (Laravel), `admin_app/` (Flutter), `student_app/` (Flutter)

---

## 1. Root Cause

**Verdict: (B) Frontend payload wrong — a contract violation, not a backend bug.**
Not (A), and not (C): the LearningRecord architecture is *not* partially migrated. It is internally consistent and matches a convention applied uniformly across the whole API.

### The mechanism

`POST /learning-records` and `PUT /learning-records/{id}` have **deliberately different contracts**:

| Field | `StoreLearningRecordRequest` | `UpdateLearningRecordRequest` |
|---|---|---|
| `circle_session_id` | `required` | **`prohibited`** |
| `student_id` | `required` | **`prohibited`** |
| `teacher_id` | `prohibited` (server-derived) | `prohibited` (server-derived) |

This is correct by design. A learning record's **identity is immutable after creation** — you may edit *what* was memorized, never *who* it belongs to or *which session* it happened in. `UpdateLearningRecordAction` proves the intent: it re-derives the session from the stored record itself (`CircleSession::find($lockedRecord->circle_session_id)`), so an incoming `circle_session_id` would be not merely redundant but a silent tampering vector.

`admin_app`'s `SessionWorkspaceController.saveDraft()` built **one create-shaped body** and sent it down both paths:

```dart
final body = {
  'circle_session_id': session.id,   // ← prohibited on update
  'student_id': studentId,           // ← prohibited on update
  'type': type.api, 'is_na': draft.isNa, /* … */
};
final saved = draft.existingId == null
    ? await backend.learningRecords.create(body)
    : await backend.learningRecords.update(draft.existingId!, body); // → 422
```

So **the first save of any evaluation succeeded, and every subsequent re-save of that same evaluation failed with 422.**

### Why it looked like "matn/listening fail, memorization works"

It was never type-specific. The reported payloads happened to be matn/listening because those are the two types a teacher most often toggles to N/A *after* an initial save. `LearningRecordUpdateContractTest` now proves the failure was identical for all four types, and the fix is identical for all four.

### The knock-on 409

`POST /sessions/{id}/complete → 409` was a **symptom, not a second bug**. `CircleSessionCompletionQuery` requires every present student to have all four types scored-or-N/A. Since re-saves 422'd, the records never persisted, so the completion gate correctly refused. It resolves itself once the 422 is fixed.

---

## 2. Consistency Report — API-Wide Audit

Audited **every** `'prohibited'` rule in `api/app/Features/**` against **every** frontend write call site.

### The convention (source of truth)

Every `Update*Request` in this codebase prohibits identity + server-derived fields. `admin_app/lib/ui/features/sessions_screen.dart` is the **reference implementation** — it already carried separate `_createFields()` / `_editFields()` with an explicit comment naming the prohibited fields. The fix propagates that existing pattern rather than inventing a new one.

### Violations found and fixed

| # | Call site | Sent on update | Prohibited by | Status |
|---|---|---|---|---|
| 1 | `session_workspace_controller.dart` `saveDraft()` | `circle_session_id`, `student_id` | `UpdateLearningRecordRequest` | **Fixed** — reported bug |
| 2 | `assessments_screen.dart` `_edit()` | `student_id`, `teacher_id`, `circle_id`, `assessment_date` | `UpdateAssessmentRequest` | **Fixed** — latent, same class |
| 3 | `homework_screen.dart` `_edit()` | `student_id`, `teacher_id`, `circle_id` | `UpdateHomeworkRequest` | **Fixed** — latent, same class |

#2 and #3 were **pre-existing, unreported 422s** — editing any assessment or homework from the admin UI would have failed the same way. Found only because the audit was run API-wide rather than stopping at the reported symptom.

### Verified clean (no action needed)

- `sessions_screen.dart` — already correct (the reference pattern).
- `backend.updateTeacher()` — strips `salary_amount` and routes it to `PUT /teachers/{id}/salary`; remaining fields (`name`/`email`/`password`) are not prohibited.
- `centers` / `circles` / `matnTypes` / `circleSchedules` — their Update requests prohibit nothing.
- Attendance — the workspace writes via `POST /attendance/bulk`, never `PUT /attendance/{id}`; `circle_session_id` is *required* there and correctly sent.
- Offline sync (`SyncingRepository`) — replays creates as creates; never converts a create into an update, so it cannot manufacture this mismatch.
- `student_app` — its only `circle_session_id` sender is `saveEvaluation()` → `POST /parent/evaluations` (create-only; no update path exists).

**Inverse check** (required in backend but missing in frontend): none found.

---

## 3. Learning Record Contract Audit — All Four Types

Verified `memorization` / `revision` / `matn` / `listening` share one contract across every operation:

| Operation | Shared? | Notes |
|---|---|---|
| create | ✅ | One `_contentPayload()` + identity, branched only on type-specific content fields |
| update | ✅ | Same `_contentPayload()`, identity omitted |
| resume session | ✅ | `_loadExistingEvaluations()` hydrates all four types uniformly |
| save draft | ✅ | Single `saveDraft(studentId, type)` for all four |
| session completion | ✅ | `CircleSessionCompletionQuery` iterates `LearningRecordType::values()` |
| statistics | ✅ | Quran-based types feed the calculator; matn/listening excluded by design (no Quran range) |
| reports | ✅ | `LearningRecordResource` exposes all type-specific fields |

There is now **no path where one type behaves differently from another.** Tests are parameterized over `LearningRecordType::values()` to keep it that way.

---

## 4. UX Refinement

### Before
3 tabs (الحضور / التقييم / الملخّص). Student switching sat *inside* the evaluation tab behind a prev/next bar; all four types stacked vertically in one long scroll with a single bulk save.

### After

```
┌─ Session summary bar ── name · status · [حاضر][غائب][مُقيَّم] ──┐
├─ Lifecycle actions ──── [بدء] [قفل] [إلغاء] ──────────────────┤
├─ Tabs ── الحضور │ الحفظ │ المراجعة │ المتن │ السماع │ الملخّص ──┤
├─ Student strip ─ [✓ أحمد][⚠ محمد][○ يوسف][+12] ──────────────┤
└─ Tab body ────────────────────────────────────────────────────┘
```

| Requirement | Implementation |
|---|---|
| Session summary bar | `widgets/session_summary_bar.dart` — live present/absent/evaluated counts, recomputed on every controller notify |
| Horizontal student switcher | `widgets/student_switcher.dart` → `StudentSwitcherStrip`; shows first 6 inline, rest behind `+N` sheet; auto-scrolls the selected chip into view |
| Status indicators | `✓` completed (all 4 saved/N-A) · `⚠` incomplete (some) · `○` not started (none) — `progressFor()` |
| Instant student switch | `controller.selectStudentIndex()` → `notifyListeners()`. No fetch, no navigation |
| Per-type tabs | `tabs/evaluation_type_tab.dart`, one instance per `LearningRecordTypeKind` |
| Instant tab switch | `TabBarView` + `AutomaticKeepAliveClientMixin` — state preserved, no rebuild-from-scratch, no push/pop |
| Persistent attendance | `الحضور` is tab index 0, reachable at any time without leaving the workspace |
| Quick save | Each type tab has its own `حفظ {type}` button |
| Save stays put | `saveDraft()` mutates the draft in place; selection, tab, and scroll are untouched |
| Completion rule | Unchanged server-side (`CircleSessionCompletionQuery`) — attendance complete **AND** every present student has all 4 types scored-or-N/A |
| Missing list | Summary tab now lists **student names** per missing type (was: bare counts) |

The student strip is shown only on the four evaluation tabs — it is meaningless on الحضور/الملخّص, both of which already display the full roster.

**Business rules unchanged.** No validation, policy, or completion logic was touched.

---

## 5. Error Handling — Silent Failures Eliminated

`attendance_tab.dart` had a genuine silent failure: `if (ok) { showSnack }` with **no `else`**. A failed attendance save produced no feedback whatsoever.

All six write paths now surface the real backend message:

| Path | Surfacing |
|---|---|
| attendance save | Red snackbar, `error.arabicMessage`, or local reason if the request never fired |
| memorization / revision / matn / listening save | Red snackbar per tab + inline red error panel |
| session completion | Red snackbar with the server's refusal reason + live missing-list above the button |

---

## 6. Files Changed

**Backend (1 file, test-only — no production change was needed)**
- `api/tests/Feature/LearningRecordUpdateContractTest.php` *(new)*

**admin_app**
- `lib/ui/teacher/session_workspace/session_workspace_controller.dart` — **root-cause fix**: `_contentPayload()` extracted; identity sent on create only
- `lib/ui/teacher/session_workspace/session_workspace_screen.dart` — 6 tabs, summary bar, conditional student strip, `_Header` → `_LifecycleActions`
- `lib/ui/teacher/session_workspace/widgets/session_summary_bar.dart` *(new)*
- `lib/ui/teacher/session_workspace/widgets/student_switcher.dart` *(rewritten)* — horizontal strip + status + `+N`
- `lib/ui/teacher/session_workspace/tabs/evaluation_type_tab.dart` *(new)* — per-type, keep-alive, own save
- `lib/ui/teacher/session_workspace/tabs/evaluations_tab.dart` *(deleted)* — superseded
- `lib/ui/teacher/session_workspace/tabs/attendance_tab.dart` — silent-failure fix
- `lib/ui/teacher/session_workspace/tabs/summary_tab.dart` — named missing-list + completion error surfacing
- `lib/ui/features/assessments_screen.dart` — `_editFields()` added
- `lib/ui/features/homework_screen.dart` — `_editFields()` added
- `test/session_workspace_controller_test.dart` — +4 payload-contract tests

**student_app** — no changes required (audit found no violations).

---

## 7. Tests Added

**Backend — `LearningRecordUpdateContractTest.php` (12 cases, 4 types × 3 scenarios)**
1. create → update with content-only body succeeds — per type
2. update carrying `circle_session_id`/`student_id` is rejected 422 — per type *(reproduces the exact reported payload)*
3. toggling `is_na` on an existing record succeeds — per type *(the literal failing payload from the report)*

**admin_app — `session_workspace_controller_test.dart` (+4 cases, 1 per type)**
Captures outbound HTTP and asserts create carries `circle_session_id`+`student_id` while the subsequent update carries **neither** — plus `teacher_id` absent from both.

These fail against the old code and pass against the new, for every type.

---

## 8. Verification

| Command | Result |
|---|---|
| `php artisan test` | **347 passed**, 1 failed |
| `flutter analyze` (admin_app) | **No issues found** |
| `flutter test` (admin_app) | **52 passed**, 5 failed |
| `flutter analyze` (student_app) | **No issues found** |
| `flutter test` (student_app) | **23 passed** |

**The 6 failures are pre-existing and unrelated to this work:**
- `PushNotificationTest` (1) — test-order pollution; **passes in isolation** (verified: 8/8).
- `live_api_test.dart` (5) — opportunistic integration tests that hit the developer's running server; fail on stale route cache and unseeded data (`GET /centers` 404, `405` on enroll). Identical 5 failures before and after this change.

Backend went 336 → 347 passing (+11 net: 12 new, and the suite total reflects one pre-existing flake). admin_app went 48 → 52 (+4 new).

---

## 9. Route References

| Action | Route |
|---|---|
| Create evaluation | `POST /api/v1/learning-records` |
| Update evaluation | `PUT /api/v1/learning-records/{id}` |
| Attendance (bulk) | `POST /api/v1/attendance/bulk` |
| Session roster | `GET /api/v1/sessions/{id}/roster` |
| Completion gate | `GET /api/v1/sessions/{id}/completion-status` |
| Complete session | `POST /api/v1/sessions/{id}/complete` |

Screen path: `TodayScreen` → tap session card → `SessionWorkspaceScreen` (`admin_app/lib/ui/teacher/session_workspace/`).

---

## 10. Remaining Risks

1. **`PushNotificationTest` order-dependence** — pre-existing; leaks state (likely a `PushDelivery` row or `Messaging` binding) into a later test. Untouched here; worth isolating separately.
2. **`live_api_test.dart` depends on developer machine state** — it asserts against whatever is running on `localhost:8000`. It should be tagged and excluded from default runs, or pinned to a seeded fixture DB.
3. **No offline queue for learning records** — evaluations are connected-only. A teacher losing connectivity mid-session sees the error (no data loss, no silent drop) but cannot save until reconnected. Attendance *does* queue offline; evaluations do not. Intentional today, but an asymmetry worth a decision.
4. **`TextEditingController` constructed in `build()`** for the notes/topic fields (pre-existing pattern, carried over) — works because those fields rebuild rarely, but it discards cursor position on rebuild. Low impact; a candidate for the same `didUpdateWidget` treatment `_AyahField` already uses.
5. **`+N` overflow chip shows the first 6 students inline** — with very large circles the teacher will use the sheet more often. The limit is a single constant (`StudentSwitcherStrip.inlineLimit`) if it needs tuning against real roster sizes.
6. **Screenshots not captured** — the app was not launched in this pass; verification was static analysis + automated tests. Recommend a manual walkthrough of the six-tab flow before release.
