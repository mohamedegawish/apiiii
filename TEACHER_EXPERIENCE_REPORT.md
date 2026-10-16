# Teacher Experience Redesign — Consolidated Report

> Consolidated report for the Teacher (المحفّظ) experience in `admin_app`, with the resulting reads/writes wired into `student_app` for students and parents. This report has gone through two rounds: **Round 1** built a guided step-by-step wizard (Attendance → Evaluation → Summary); **Round 2** (this revision) replaced that wizard with a free-navigation **Session Workspace**, added two new evaluation types (Matn/المتن, Listening/السماع), moved student behavior rating to the parent exclusively, and closed several security/data-integrity findings from a full production audit. Section 1 is kept as the historical "before" record; sections 2 onward describe the **current, shipped** design.

---

## 1. The original problem (Round 1 audit)

`admin_app` gave a teacher the same admin information architecture with a few relabelled tiles: a 19-item drawer (مراكز، تقارير، سجلّ تدقيق، مستخدمون…), a "الشهادات" tile that always rendered `قريباً`, and no "today" concept. The actual job — attendance, evaluation, homework, close session — was spread across five disconnected screens:

- **Attendance implemented twice** (inside a session workspace tab, and again as a standalone screen) with two different status sets for the same fact.
- **`session_detail_screen.dart` (941 lines)** — no lifecycle guidance, a raw `api.post` bypassing the offline sync queue, an in-memory "recorded" set that reset on every re-entry, no permission gating.
- **Memorization/revision entry** was a raw `0–100` text field — no surah/ayah range picker.
- **Backend endpoints already existed and were never called**: `GET /student/sessions`, `/student/learning-records`, `/parent/my-children/{id}/sessions`.
- **Dead code**: a 217-line legacy mock `models.dart`.

Round 1 replaced this with a linear wizard. In real classroom use, that wizard turned out to be the wrong shape: teachers needed to jump back to attendance mid-evaluation, revisit a student already graded, or add a skipped grade — all without walking a forced sequence backwards. Round 2 (below) replaces the wizard with a free-navigation workspace.

---

## 2. The current design: Session Workspace

Server-enforced session state machine (unchanged across both rounds):

```
scheduled ──start──▶ in_progress ──complete──▶ completed ──lock──▶ locked
    │                     │
    └──────cancel─────────┘
```

**One screen, six free-navigation tabs** — no forced order, no "Next/Previous" gate between tabs, no auto-advance to another student on save:

| Tab | Purpose |
|---|---|
| **الحضور (Attendance)** | Present/Absent per student, bulk-present/bulk-absent, search, live attendance %. Stays editable for the entire session — no freeze after the first save; explicit "حفظ الحضور" button with an unsaved-changes indicator. |
| **الحفظ (Memorization)** | Quran range (cross-surah capable) + score, per present student. |
| **المراجعة (Revision)** | Same shape as Memorization, independent record. |
| **المتن (Matn)** | Matn type + from/to section range (admin-configurable catalog — see §3) + score. |
| **السماع (Listening)** | Free-text topic + score. |
| **الملخّص (Summary)** | Live present/absent counts + an evaluated count per type, warnings sourced from the server's completion check, and the "إنهاء الجلسة" action. |

Each of the four evaluation tabs (Memorization/Revision/Matn/Listening) hosts the same **student switcher**: a list of present students with a completion dot, next/previous controls, and a quick-pick jump — the currently-selected student is shared across all four tabs, so moving from Memorization to Matn keeps the teacher on the same student rather than resetting to the first one. Ranges auto-prefill from the student's last record of that type. Every type supports an explicit **N/A** toggle — a student who doesn't study Matn, for instance, is marked N/A rather than forced to have a fabricated range.

The session header (circle name, date, time, status badge) carries all four lifecycle actions — **Start / Complete / Lock / Cancel** — each shown only when the current status allows it, so the same actions available from the Today dashboard are also reachable from inside the workspace.

**Session completion rule (new, server-enforced):** a session can only complete once (a) attendance is recorded for every roster student, and (b) every **present** student has, for each of the four types, either a real score or an explicit N/A. This is enforced in the API itself (`TransitionCircleSessionAction` + a dedicated `CircleSessionCompletionQuery`), not just client-side — the Summary tab's warnings and the Complete button's enabled state both read the same `GET /sessions/{id}/completion-status` the transition itself checks.

**What moved out of the teacher's hands:** teachers no longer rate student behavior/conduct or prayer commitment at all — `attendances.conduct_level` (and its enum, UI selector, and every reference) was removed outright. Those two observations belong exclusively to the **parent**, who already rates Behavior + Prayer + Homework Follow-up (1–5 each) via the existing `parent_evaluations` flow in `student_app`, submitted only after the teacher completes a session and before it's locked.

**Non-goals kept out of teacher reach:** locking another teacher's session (still ownership-checked server-side), the whole admin taxonomy, a teacher picker on any form (`teacher_id` is server-derived everywhere), a notifications tab (no backend event ever targets a teacher).

**Shell:** `TeacherShell` — 4 bottom tabs, no drawer: اليوم (Today) · طلابي (reused `StudentsScreen`) · الواجبات (`HomeworkScreen`, grade-only) · حسابي (profile). The workspace route is `PopScope`-guarded: backing out of an `in_progress` session with drafted attendance prompts a confirmation instead of silently discarding it.

**Cross-app:** after a teacher completes a session, `student_app`'s session history and the parent's child-records screens show all four record types (memorization/revision/matn/listening), and `/parent/evaluations` becomes submittable for that session.

---

## 3. Matn catalog (new admin-configurable feature)

Matn (متن) evaluation needed a "from section / to section" range the same way Quran evaluation has "from ayah / to ayah" — but unlike the Quran, Matn texts (الأربعين النووية، تحفة الأطفال، الجزرية، …) are not fixed reference data. A new admin-only catalog was added:

- **`matn_types`** — name, description, active flag. Managed from a new admin screen (كتالوج المتون), mirroring the existing `Circles` CRUD screen exactly.
- **`matn_sections`** — an ordered, named child list per matn type (e.g. "الحديث الأول".."الحديث العاشر" under الأربعين النووية). Managed from a nested screen off each matn type's detail page.
- Teachers only **read** this catalog (to populate the range picker in the Matn evaluation tab); only center_owner+ can create/edit/delete matn types or sections.

---

## 4. API mapping (screen → endpoint)

| Area | Endpoint | Payload / notes |
|---|---|---|
| Today | `GET /sessions?filter[session_date]={today}&sort=start_time` | Server-scoped to the teacher. |
| Lifecycle actions | `POST /sessions/{id}/start\|complete\|cancel\|lock` | All four reachable from the workspace header. |
| Attendance | `GET /sessions/{id}/roster`, `GET /attendance?filter[circle_session_id]`, `POST /attendance/bulk` | Roster resolved at the session's own date; save is an upsert, safe to resend. |
| Memorization / Revision | `POST`/`PUT /learning-records` | `{circle_session_id, student_id, type: 'memorization'\|'revision', surah_number, from_ayah, to_surah_number, to_ayah, score}` |
| Matn | `POST`/`PUT /learning-records` | `{..., type: 'matn', matn_type_id, from_matn_section_id, to_matn_section_id, score}` |
| Listening | `POST`/`PUT /learning-records` | `{..., type: 'listening', topic, score}` |
| Mark a type N/A | `POST`/`PUT /learning-records` | `{..., is_na: true}` — no range/topic required. |
| Matn catalog | `GET/POST/PUT/DELETE /matn-types`, `GET/POST /matn-types/{id}/sections`, `PUT/DELETE /matn-sections/{id}` | Read = staff+teacher; write = center_owner+. |
| Completion status | `GET /sessions/{id}/completion-status` | Drives the Summary tab and the Complete button; same check the server enforces. |
| Students / Homework tabs | `GET /students`, `/statistics/students/{id}`, `GET/PUT /homework` | Unchanged. |
| Parent evaluation | `POST /parent/evaluations` | `{circle_session_id, student_id, behavior_score, prayer_score, homework_followup_score, notes?}` — unchanged; already exclusively parent-owned. |
| Student/parent record visibility | `GET /student/learning-records`, `/parent/my-children/{id}/learning-records` | Now return all four types; both `student_app` screens render matn range / listening topic alongside the existing Quran range display. |

**Offline policy (unchanged):** attendance queues offline (idempotent, safe to replay — and a 409 conflict, e.g. a session locked mid-flight, is now correctly classified as terminal instead of endlessly retried). Evaluations, homework-bulk, matn catalog writes, and all session transitions are online-only.

---

## 5. Security & data-integrity fixes landed alongside this round

A full production audit (`FINAL_PRODUCTION_AUDIT.md`) surfaced several issues fixed in this same pass:

- **`GET /guardians` IDOR** — any authenticated user (including students) could list every family's contact info; now staff-only with a fail-closed tenant scope.
- **`GET /circles` / `GET /teachers` IDOR** — guardian/student roles fell through unscoped instead of failing closed; now routed through the same `accessibleCircleIds()`/`accessibleTeacherIds()` helpers `HomeworkQuery` already used.
- **Learning-record soft-delete + unique-constraint 500** — deleting then re-adding an evaluation of the same type crashed instead of restoring the trashed row; fixed to mirror the existing `AttendanceService` restore-or-conflict pattern.
- **Statistics silently going stale** — `StatisticsCalculatorService` read only the legacy `memorization_sessions`/`revision_sessions` tables; it now merges those with the new `student_learning_records` rows the workspace actually writes, and gained dedicated Matn/Listening count + average-grade columns.
- **409 offline-sync misclassification** — a locked/completed-session conflict during a queued attendance replay was falling into the generic "retry" bucket instead of being dropped as terminal; `admin_app` now has a proper `conflict` error kind.

---

## 6. Verification

| Layer | Command | Result |
|---|---|---|
| `api` | `php artisan test` | 303 passing (1 pre-existing, unrelated flaky push-notification test) |
| `admin_app` | `flutter analyze` / `flutter test` | Clean / all passing |
| `student_app` | `flutter analyze` / `flutter test` | Clean / all passing |

New backend test coverage: matn catalog CRUD + policy gating, matn/listening record creation and validation, N/A bypass, the soft-delete-restore regression, the session-completion gate (attendance-missing / evaluation-missing / present-only / happy-path), and IDOR regression tests for guardians/circles/teachers. New Flutter coverage: `SessionWorkspaceController` (free student navigation, N/A completeness bypass, completion-status wiring) and `student_app` mapper tests for matn/listening/N/A record parsing.

Manual end-to-end device walkthroughs (teacher login, full workspace pass across all four evaluation tabs, cross-app visibility as the enrolled student/guardian) still require a running `php artisan serve` + emulator session and are recommended before a device build ships.
