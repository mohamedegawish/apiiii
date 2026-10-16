# Test Plan

## Scope

This plan verifies the circle contract cleanup, automatic guardian creation,
shared-mobile authentication, role selection, authorization isolation, and the
Flutter client integrations.

## Backend

### Circle Contract

- Create and update requests accept `name`, `teacher_id`, `capacity`,
  `starts_at`, `description`, and `is_active` only.
- `center_id`, `level`, and `ends_at` are not validated, persisted from input,
  filtered, or returned by `CircleResource`.
- A center owner can access a new circle through its assigned teacher or
  creator scope; a teacher from another tenant cannot be injected.
- Legacy circles with persisted `center_id`, `level`, or `ends_at` remain
  readable and no existing data is deleted.

### Guardian Relationships

- The canonical API values are `أب`, `أم`, `أخ`, `أخت`, `جد`, and `جدة`.
- Student creation and guardian update accept each canonical value.
- `أب` validates successfully; unsupported English or Arabic values are
  rejected.
- Factories generate values exclusively from `GuardianRelationship`.

### Student And Guardian Creation

- One student request creates two distinct `users` rows with roles `student`
  and `guardian`.
- Both users have the same phone and independently hashed copies of the same
  password.
- Student and guardian profiles point to the correct user records.
- The legacy `students.guardian_id` and `guardian_student` pivot are both
  populated for compatibility.
- The complete operation rolls back on any failure because it runs in one
  database transaction.
- `POST /guardians` returns Method Not Allowed.

### Authentication

- Email login continues to work for staff accounts.
- A single mobile account receives a token immediately.
- A mobile shared by student and guardian receives
  `requires_role_selection: true` and both roles, with no token.
- `POST /auth/select-role` validates the mobile, password, and requested role,
  then issues a token for exactly that user.
- Login responses include `user`, `token`, and the token permissions.
- Invalid credentials and unavailable roles do not reveal account details.

### Authorization And Tenant Security

- A student token receives `403` from guardian-only routes.
- A guardian token receives `403` from student-only routes.
- Existing `RequireRole` middleware remains the enforcement mechanism.
- Cross-tenant student, guardian, teacher, and circle operations remain denied.

### Commands

```bash
php artisan test --compact
vendor/bin/pint --dirty --format agent
```

## Admin App

- Circle forms and details contain no center, level, or end-date controls.
- Student creation includes mobile, password, guardian name, and canonical
  guardian relationship values.
- Student creation contains no guardian picker or `guardian_id` payload field.
- Guardian creation screen and navigation item do not exist.
- Static analysis and Flutter tests pass.

### Commands

```bash
flutter analyze
flutter test
```

## Student App

- Login submits mobile and password.
- A direct login stores the token and opens the shell matching the user role.
- A shared mobile stores no token before role selection.
- The role-selection screen appears only for a role-selection response.
- Selecting `student` opens Student UI; selecting `guardian` maps to and opens
  the existing Parent UI.
- Session restoration and existing routing continue to work.
- Static analysis and Flutter tests pass.

### Commands

```bash
flutter analyze
flutter test
```
