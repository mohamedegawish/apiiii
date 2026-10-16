# PERMISSION_AUDIT.md

Role-based access is derived from the authenticated user's role (from `/auth/me`) via `state/permissions.dart`, and mirrors the server-side policy rules. It governs navigation visibility, screen access, and per-action buttons. The demo role-switcher was removed — the role now comes from the real account.

## Roles (task ↔ API)
| Task role | API value | Capability tier |
|-----------|-----------|-----------------|
| المدير العام | `super_admin` | full |
| — | `center_owner` | admin (+ salaries) |
| — | `branch_manager` | admin |
| المحفّظ | `teacher` | field operations only |

## Capability matrix (`Permissions`)

| Capability | super_admin | center_owner | branch_manager | teacher |
|------------|:-:|:-:|:-:|:-:|
| Manage centers | ✅ | ✖ | ✖ | ✖ |
| Manage branches | ✅ | ✅ | ✅ | ✖ |
| Manage guardians | ✅ | ✅ | ✅ | ✖ |
| Manage teachers | ✅ | ✅ | ✅ | ✖ |
| Manage circles / schedules | ✅ | ✅ | ✅ | ✖ |
| Create/edit/delete students | ✅ | ✅ | ✅ | ✖ (view only) |
| View student PII | ✅ | ✅ | ✅ | ✖ |
| View teacher salaries | ✅ | ✅ | ✖ | ✖ |
| Record attendance / sessions / assessments / homework | ✅ | ✅ | ✅ | ✅ |
| Delete field records | ✅ | ✅ | ✅ | ✖ |
| Finance / Reports | ✅ | ✅ | ✅ | ✖ |
| Recalculate statistics | ✅ | ✅ | ✅ | ✖ |

## Where it is enforced (client)

- **Navigation visibility** — `ui/nav_menu.dart` filters bottom nav, drawer, and the "More" grid by `_canSee(screen, permissions)`. A teacher never sees Teachers/Centers/Branches/Guardians/Schedules/Finance/Reports entries.
- **Screen access** — `shell.dart` `_canView` gates the screen host; a disallowed screen renders `PermissionDeniedView` instead of the module.
- **Button/action permissions** — create FABs, edit, and delete actions are only rendered when the corresponding capability is true (e.g. `onCreate: perms.manageCircles ? _create : null`, delete gated by `perms.deleteFieldRecords`, salary fields by `perms.viewSalaries`).

## Server is the source of truth
Client gating is UX only; the API independently enforces every rule (policies + `role`/`active` middleware). This was verified live: a teacher token calling `POST /statistics/students/{id}/recalculate` receives **403** (`live_api_test.dart`).

## Note
Client-side gating hides unauthorized actions but is not a security boundary — all authorization is ultimately enforced by the API, as intended.
