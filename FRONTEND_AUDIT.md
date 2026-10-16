# FRONTEND_AUDIT.md

**App:** `admin_app` (Flutter 3.44, Dart 3.12) — الطبري / Itqan admin
**State:** `provider` (single `AppState` ChangeNotifier) · **Persistence:** `shared_preferences` (theme/accent only)
**Generated:** Phase 2 — Frontend Audit (pre-integration baseline)

---

## 1. Executive summary

The app is a **high-fidelity UI prototype with zero backend integration**. Every screen renders **static mock data** from `lib/data/seed.dart` (506 lines). There is **no HTTP client dependency at all** (`pubspec.yaml` has only `provider` + `shared_preferences`), no repositories, no services, no models mapped to the API, and no real authentication.

| Concern | Current state |
|---------|---------------|
| Networking layer | ❌ None (no `http`/`dio`) |
| API models / DTOs | ❌ None — only presentation models in `models.dart` |
| Repositories / services | ❌ None |
| Authentication | ❌ Fake — `state.login(UserRole)` with no credentials |
| Token storage | ❌ None |
| Offline / sync | ❌ None (no Isar) |
| Error/loading states | ⚠️ Widgets exist (`ErrorState`, `EmptyState`, `DelayedContent`) but wired to a fake `Future.delayed` |
| Role gating | ⚠️ Demo-only toggle in drawer; two hardcoded gates (finance/settings) |
| Pagination / search | ⚠️ Client-side filtering over the static list only |

---

## 2. Screens inventory (16 `AppScreen` values)

| Screen | Widget | Data source today | Backing API | Status |
|--------|--------|-------------------|-------------|--------|
| home (admin) | `AdminDashboard` | `Seed.adminKpis/charts/tasks/activity` | `/statistics/*` + counts | 🔴 mock |
| home (teacher) | `TeacherDashboard` | `Seed.teacherKpis/todaySessions/progress` | `/statistics/teachers/{id}` + lists | 🔴 mock |
| students | `StudentsScreen` | `Seed.students` | `/students` | 🔴 mock |
| profile | `StudentProfileScreen` | `Seed.studentById` + logs | `/students/{id}`, `/statistics/students/{id}`, sessions/attendance/assessments | 🔴 mock |
| circles | `ModuleListScreen(Seed.circlesList)` | mock rows | `/circles` | 🔴 mock |
| teachers | `ModuleListScreen(Seed.teachersList)` | mock rows | `/teachers` | 🔴 mock |
| memo | `MemoWizard` | `Seed.students`, `sessionTypes`, grades | `/memorization-sessions` + `/revision-sessions` | 🔴 mock |
| attendance | `AttendanceScreen` | `Seed.students.take(6)` | `/attendance` | 🔴 mock |
| assess | `ModuleListScreen(Seed.assessList)` | mock rows | `/assessments` | 🔴 mock |
| homework | `ModuleListScreen(Seed.homeworkList)` | mock rows | `/homework` | 🔴 mock |
| reports | `ModuleListScreen(Seed.reportsList)` | mock rows | `/statistics/*` (loose) | 🟡 no direct API |
| notifications | `NotificationsScreen` | `Seed.notifications` | — none | ⚪ no API |
| cert | `ModuleListScreen(Seed.certList)` | mock rows | — none | ⚪ no API |
| achieve | `ModuleListScreen(Seed.achieveList)` | mock rows | — none | ⚪ no API |
| finance | `ModuleListScreen(Seed.financeList)` | mock rows | — none (teacher salary fields only) | ⚪ no API |
| more / settings | `MoreScreen` / `SettingsScreen` | local nav / prefs | — | ⚪ local |

---

## 3. Endpoints with **no** UI (Phase 4 build targets)

| Endpoint group | UI today | Action |
|----------------|----------|--------|
| `/centers` (CRUD) | none | **Build** list + detail + create/edit + delete |
| `/branches` (CRUD) | none | **Build** list + detail + create/edit + delete |
| `/guardians` (CRUD) | none | **Build** list + detail + create/edit + delete |
| `/circle-schedules` (CRUD) | none | **Build** (nested under circle detail) |
| `/revision-sessions` (CRUD) | folded into memo wizard | **Build** dedicated list + form |
| `/quran/surahs` | free-text surah name in wizard | **Wire** as surah picker source |
| `/statistics/*` recalculate | none | **Wire** admin "recalculate" action |

---

## 4. Missing pieces

**Fake data:** all of `lib/data/seed.dart` — to be replaced by repository calls.

**Missing API calls:** every screen. No GET/POST/PUT/DELETE is issued anywhere in the codebase.

**Missing forms (no create/edit form exists for any entity):**
- Student create/edit · Teacher create/edit · Circle create/edit · Guardian, Center, Branch create/edit
- Attendance record form (per student/day) · Memorization & Revision session forms · Assessment form · Homework form + grading · Circle schedule form
- The "إضافة طالب" (add student) empty-state CTA is a dead button.

**Missing workflows:**
- Real login → token → session; logout → revoke; auto-login on launch; auto-logout on 401.
- Create → refresh list; edit → update row; delete → confirm + remove.
- Server-driven search / filter / pagination / infinite scroll.
- Offline capture + sync for attendance / sessions / assessments / homework.

**Auth screens with no backend:** `forgot` / `otp` / `reset` — the API exposes **only** `login`, `logout`, `me`. These flows have no endpoint and will be marked as backend-unsupported (kept as UI, disabled action) rather than faked.

---

## 5. Reusable assets to keep

Design system is solid and worth preserving: `core/tokens.dart` (theming), `core/icons.dart`, `widgets/primitives.dart` (`ItqanField`, `PrimaryButton`, `FilterPillsRow`, `ModuleListRow`, `TagChip`…), `widgets/states.dart` (`EmptyState`, `ErrorState`, `PermissionDeniedView`, `ListSkeleton`, `DelayedContent`), `widgets/charts.dart`.

**Integration strategy:** keep the UI/design system; insert a real data layer beneath it (`data/api`, `data/models`, `data/repositories`, `data/local`), add per-feature providers, and rewire each screen from `Seed.*` to its provider. `DelayedContent`'s fake delay is replaced by real async load/error/empty states.
