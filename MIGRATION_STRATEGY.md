# MIGRATION_STRATEGY.md

## Overview
Safe migration strategy for existing data. **Zero data loss. Zero breaking changes for existing records.**

---

## Principles

1. **No Destructive Migrations** - No `DROP COLUMN`, `DROP TABLE`, or data deletion
2. **Backward Compatibility** - Existing records continue to work
3. **Forward Compatibility** - New code handles both old and new data
4. **No Data Migration Scripts Needed** - Application logic handles the transition

---

## CHANGE 1: Circle Fields (Center, Level, End Date)

### Current State
- Columns exist in `circles` table: `center_id`, `level`, `ends_at`
- Existing records have values in these columns
- API returns these fields
- Admin App displays/edits these fields

### Migration Approach
**NO DATABASE MIGRATION REQUIRED**

### Application-Level Changes
1. **Backend**: Stop writing to these columns, stop returning in API
2. **Admin App**: Remove form fields, remove display columns
3. **Existing Data**: Remains in database, ignored by new code

### Verification
- Existing circles still display (level, end date hidden in new UI)
- No errors when reading existing circles
- New circles created without these fields

### Future Cleanup (Optional, Post-Verification)
```php
// Future migration - ONLY after confirming no issues
// php artisan make:migration remove_unused_circle_fields --table=circles

Schema::table('circles', function (Blueprint $table) {
    $table->dropForeign(['center_id']);
    $table->dropColumn(['center_id', 'level', 'ends_at']);
});
```

---

## CHANGE 2: Guardian Relationship Enum

### Current State
- `guardians.relationship` column: string, nullable
- Existing values: `'father'`, `'mother'`, `'guardian'`
- Legacy rows may contain older English relationship values.

### Migration Approach
**NO DATABASE MIGRATION REQUIRED**

### Application-Level Changes
1. **Backend**: Canonicalize `GuardianRelationship` to six Arabic values
2. **Validation**: Use `Rule::enum(GuardianRelationship::class)` 
3. **Existing Data**: Current values remain valid (subset of new enum)

### Verification
- Existing guardian relationship data is not rewritten or deleted.
- New writes and updates accept the six canonical values.
- Update operations on existing guardians work

---

## CHANGE 3: Student Creation Creates Guardian

### Current State
- Separate student and guardian creation
- `students.guardian_id` links to existing guardian
- `guardians` table has independent records
- `guardian_student` pivot table links them

### New Flow
- Single student creation request
- Backend creates BOTH student user + guardian user
- Backend creates BOTH student record + guardian record
- Links them in pivot table

### Migration Approach
**NO DATABASE MIGRATION REQUIRED**

### Application-Level Changes
1. **Backend**: 
   - New `StoreStudentRequest` with guardian fields + mobile + password
   - `CreateStudentAction` creates both users + records + pivot link
   - Remove `POST /guardians` endpoint
2. **Admin App**: 
   - New student form with guardian fields
   - Remove guardian dropdown
   - Remove Guardians screen

### Existing Data Handling
| Entity | Handling |
|--------|----------|
| Existing Students | Unchanged - keep `guardian_id` link |
| Existing Guardians | Unchanged - keep all data |
| Existing Pivot Links | Unchanged |
| Existing Users | Unchanged |

### New Student Creation
```php
// Only for NEW students created after deployment
// Creates:
1. User (role: student) - mobile, password
2. User (role: guardian) - SAME mobile, SAME password
3. Student record - linked to student user
4. Guardian record - linked to guardian user
5. guardian_student pivot - links student + guardian
```

### Edge Cases
- **Existing student with existing guardian**: Continue working normally
- **Admin edits existing student**: Use `UpdateStudentRequest` (unchanged)
- **Admin views existing student**: Shows linked guardian (unchanged)

---

## Login Flow Migration

### Current State
- Login uses `email` + `password`
- Single user per email
- Direct token issuance

### New Flow
- Login uses `mobile` (phone) + `password`
- Multiple users can share same mobile
- Role selection when multiple accounts exist

### Migration Approach
**NO DATABASE MIGRATION REQUIRED**

### Data Preparation (One-time, Before Deployment)
```sql
-- Ensure all users have phone numbers populated
-- For existing users without phone, they cannot use new login
-- Options:
-- 1. Backfill phone from student/guardian records
-- 2. Require password reset flow to set phone
-- 3. Support both email and mobile login temporarily
```

### Recommended: Backfill Phone Numbers
```php
// One-time seeder/command before deployment
User::whereNull('phone')->each(function ($user) {
    if ($user->student && $user->student->phone) {
        $user->update(['phone' => $user->student->phone]);
    } elseif ($user->guardian && $user->guardian->phone) {
        $user->update(['phone' => $user->guardian->phone]);
    }
});
```

### Backward Compatibility Option
If needed, support BOTH email and mobile login:
```php
// LoginAction - temporary dual support
$field = filter_var($data['identifier'], FILTER_VALIDATE_EMAIL) ? 'email' : 'phone';
$user = User::where($field, $data['identifier'])->first();
```

---

## Deployment Checklist

### Pre-Deployment
- [ ] Run phone backfill for existing users
- [ ] Deploy backend changes
- [ ] Deploy admin app changes
- [ ] Deploy student app changes
- [ ] Test with staging data

### Post-Deployment Verification
- [ ] Existing circles readable (level/ends_at hidden)
- [ ] Existing students/guardians readable
- [ ] New student creation works (creates guardian)
- [ ] Login with mobile works
- [ ] Role selection works for shared mobile
- [ ] Admin cannot create guardians separately
- [ ] Guardian relationship dropdown has 7 values

### Rollback Plan
- Revert code deployments (no DB rollback needed)
- All data preserved in original state

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Existing users can't login (no phone) | Medium | High | Pre-deployment backfill |
| Guardian relationship validation fails | Low | Medium | Enum includes old values |
| Circle data loss | None | N/A | Columns not dropped |
| Student-guardian link broken | None | N/A | No changes to existing links |

---

## Timeline
- **Week 1**: Backend implementation + tests
- **Week 2**: Admin App implementation + tests
- **Week 3**: Student App implementation + tests
- **Week 4**: Integration testing + phone backfill + deployment
