# DATABASE_IMPACT.md

## Overview
This document describes the database impact of the changes. **No destructive migrations** - all existing data is preserved.

---

## CHANGE 1: Circle Fields Removal

### Current `circles` Table
```sql
- id (uuid, PK)
- center_id (uuid, FK)        -- KEEP (existing data)
- teacher_id (uuid, FK, nullable)
- name (string)
- description (text, nullable)
- level (string, nullable)    -- KEEP (existing data)
- capacity (unsignedInteger)
- is_active (boolean)
- starts_at (date, nullable)
- ends_at (date, nullable)    -- KEEP (existing data)
- created_by (uuid, nullable)
- updated_by (uuid, nullable)
- timestamps
- softDeletes
```

### Action Required
- **NO MIGRATION NEEDED** - Keep all columns for existing data
- Backend will stop writing to `center_id`, `level`, `ends_at`
- Frontend will stop sending these fields
- Existing records retain their values

### Future Consideration
A separate cleanup migration can be created later (after verification) to drop these columns if desired:
```php
// Future migration (NOT part of this change)
Schema::table('circles', function (Blueprint $table) {
    $table->dropForeign(['center_id']);
    $table->dropColumn(['center_id', 'level', 'ends_at']);
});
```

---

## CHANGE 2: Guardian Relationship Enum

### Current `guardians` Table
```sql
- relationship (string, nullable)  -- Stores: 'father', 'mother', 'guardian'
```

### Change
- Enum values expanded from 3 to 6 values
- **No schema change needed** - column is string, accepts any value
- New writes validate the canonical Arabic values: `أب`, `أم`, `أخ`, `أخت`, `جد`, `جدة`.

### Existing Data
- Existing values remain untouched in storage to avoid data loss.
- New and updated values use the canonical Arabic set going forward.

---

## CHANGE 3: Student Creation Creates Guardian

### Current Tables

#### `users` Table
```sql
- id (uuid, PK)
- name (string)
- email (string, unique)
- phone (string, nullable)       -- NEW: will store mobile
- password (string)
- role (enum: super_admin, center_owner, teacher, student, guardian)
- is_active (boolean)
- email_verified_at (timestamp)
- timestamps
- softDeletes
```

#### `students` Table
```sql
- id (uuid, PK)
- user_id (uuid, FK, nullable)   -- Links to users table
- center_id (uuid, FK, nullable)
- guardian_id (uuid, FK, nullable)  -- Links to guardians table
- name (string)
- national_id (string, nullable)
- birth_date (date, nullable)
- gender (enum: male, female)
- phone (string, nullable)
- email (string, nullable)
- status (string)
- enrolled_at (date, nullable)
- graduated_at (date, nullable)
- notes (text, nullable)
- timestamps
- softDeletes
```

#### `guardians` Table
```sql
- id (uuid, PK)
- user_id (uuid, FK, nullable)   -- Links to users table
- center_id (uuid, FK, nullable)
- name (string)
- phone (string, nullable)
- email (string, nullable)
- relationship (string, nullable)
- gender (enum: male, female)
- occupation (string, nullable)
- timestamps
- softDeletes
```

#### `guardian_student` Pivot Table
```sql
- id (uuid, PK)
- guardian_id (uuid, FK)
- student_id (uuid, FK)
- relationship (string)
- is_primary (boolean)
- timestamps
```

### New Flow - What Happens on Student Creation

1. **Create Student User**
   ```sql
   INSERT INTO users (id, name, phone, password, role, is_active)
   VALUES (uuid(), 'Student Name', '01000000000', Hash::make('password'), 'student', true);
   ```

2. **Create Guardian User**
   ```sql
   INSERT INTO users (id, name, phone, password, role, is_active)
   VALUES (uuid(), 'Guardian Name', '01000000000', Hash::make('password'), 'guardian', true);
   ```

3. **Create Student Record**
   ```sql
   INSERT INTO students (id, user_id, center_id, name, phone, status)
   VALUES (uuid(), student_user_id, center_id, 'Student Name', '01000000000', 'active');
   ```

4. **Create Guardian Record**
   ```sql
   INSERT INTO guardians (id, user_id, center_id, name, phone, relationship)
   VALUES (uuid(), guardian_user_id, center_id, 'Guardian Name', '01000000000', 'father');
   ```

5. **Link in Pivot Table**
   ```sql
   INSERT INTO guardian_student (id, guardian_id, student_id, relationship, is_primary)
   VALUES (uuid(), guardian_id, student_id, 'father', true);
   ```

### Migration Required
**No schema migration needed** - all tables and columns already exist.

### Data Migration for Existing Records
**NO DATA MIGRATION NEEDED** - Requirements state:
1. Existing students remain untouched ✓
2. Existing guardians remain untouched ✓
3. Existing relationships remain untouched ✓
4. No data loss ✓

The new flow only applies to **newly created students**.

---

## Login Flow Database Impact

### Current Login
- Uses `email` + `password`
- Finds user by email

### New Login
- Uses `mobile` (phone) + `password`
- Finds user(s) by phone
- If multiple users with same phone → requires role selection

### Query for Role Selection
```sql
SELECT id, name, role, phone 
FROM users 
WHERE phone = '01000000000' 
AND is_active = true;
```

If count > 1, return available roles for selection.

---

## Summary of Database Changes

| Table | Change | Migration Needed |
|-------|--------|------------------|
| circles | No schema change (keep columns) | No |
| guardians | No schema change (enum in app only) | No |
| students | No schema change | No |
| users | No schema change | No |
| guardian_student | No schema change | No |

**Total Migrations: 0** (for this implementation)

All changes are application-level logic changes only.
