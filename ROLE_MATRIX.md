# ROLE_MATRIX.md

> Roles from `api/app/Enums/Role.php` (`super_admin`, `center_owner`, `admin`†, `teacher`, `parent`, `student`). Derived from route middleware, policies, and `admin_app/lib/state/permissions.dart`.
> † `admin` = branch admin; treated `isAdminLike` in the admin app. Backend gates use `super_admin,center_owner` for tenant-admin ops.

## Capability matrix (backend-enforced)
| Capability | super_admin | center_owner | teacher | parent | student | Enforced by |
|---|:--:|:--:|:--:|:--:|:--:|---|
| Manage centers | ✅ | ❌ | ❌ | ❌ | ❌ | `role` + `CenterPolicy` |
| Manage guardians/teachers/circles/schedules | ✅ | ✅(own center) | ❌ | ❌ | ❌ | policies + tenant scope |
| Create/edit/delete students | ✅ | ✅ | ❌ | ❌ | ❌ | `StudentPolicy` |
| Student lifecycle (suspend/reactivate/graduate/withdraw/transfer) | ✅ | ✅ | ❌ | ❌ | ❌ | `role:super_admin,center_owner` (platform.php) |
| Enroll / close enrollment | ✅ | ✅ | ❌ | ❌ | ❌ | `role:super_admin,center_owner` |
| Record attendance | ✅ | ✅(own) | ✅(own circles) | ❌ | ❌ | `AttendancePolicy::create` + `assertSessionAccess` |
| Record assessments/homework/memorization/revision/learning-records | ✅ | ✅(own) | ✅(own circles) | ❌ | ❌ | `AcademicSessionGuard` / policies |
| Delete field records | ✅ | ✅(own) | ❌ | ❌ | ❌ | policies (`delete`) |
| Session lifecycle (start/complete/cancel/lock) | ✅ | ✅(own) | ✅(own) | ❌ | ❌ | `CircleSessionPolicy` |
| Update teacher salary | ✅ | ✅ | ❌ | ❌ | ❌ | `role:super_admin,center_owner` on `/teachers/{id}/salary` |
| Reports & statistics | ✅ | ✅ | ✅(scoped) | ❌ | ❌ | `role:super_admin,center_owner,teacher` |
| Admin dashboard (cross-center) | ✅ | ❌ | ❌ | ❌ | ❌ | `role:super_admin` on `reports/admin-dashboard` |
| Audit logs | ✅ | ✅ | ❌ | ❌ | ❌ | `role:super_admin,center_owner` |
| Users management | ✅ | ✅ | ❌ | ❌ | ❌ | `role:super_admin,center_owner` |
| View own student data | ❌ | ❌ | ❌ | ❌ | ✅ | `role:student` on `/student/*` |
| Parent: children/evaluations/dashboard | ❌ | ❌ | ❌ | ✅ | ❌ | `role:parent` on `/parent/*` |
| Notifications / devices | ✅ | ✅ | ✅ | ✅ | ✅ | authenticated |

## App ↔ backend role consistency
| Concern | admin_app (`Permissions`) | Backend | Consistent? |
|---|---|---|---|
| Salaries | `viewSalaries = superAdmin \|\| centerOwner` | `role:super_admin,center_owner` | ✅ |
| Field ops (teacher) | `recordAttendance/Sessions/Assessments/Homework = adminLike \|\| teacher` | teacher allowed on own circles | ✅ |
| Delete field records | `deleteFieldRecords = adminLike` (teacher ❌) | policy `delete` excludes teacher for attendance | ✅ |
| Manage centers | `manageCenters = superAdmin` | `CenterPolicy`/`role` | ✅ |
| student_app | student & parent only; rejects other roles at login (`unsupported_role`) | `role:student` / `role:parent` | ✅ |

**Tenant isolation:** center_owner is consistently scoped to `center->owner_id === user->id` in every policy and in `AcademicSessionGuard`/`AttendanceService`. Teachers scoped to `teacher->user_id === user->id`. Query classes (`*/Queries/*Query.php`) apply per-feature scoping. ✅
