# IMPLEMENTATION_PLAN.md

## Overview
Complete implementation of all three business changes across API, Admin App, and Student App.

---

## CHANGE 1: Remove Unused Fields from Circle Creation ✅ DONE

### Fields Removed
- `center_id` (Center)
- `level` (Level)
- `ends_at` (End Date)

### Backend Changes (api)
- **Migration**: `2026_07_24_233340_make_circle_removed_fields_nullable.php` - Made columns nullable for backward compatibility
- **Circle Model**: Removed `center_id`, `level`, `ends_at` from `$fillable` and `$casts`; removed `center()` relationship
- **StoreCircleRequest / UpdateCircleRequest**: Removed validation rules for removed fields
- **CircleResource**: Removed removed fields from API response
- **CircleQuery**: Removed `center_id` and `level` filters; updated tenant scope to use teacher's center and `created_by`
- **CirclePolicy**: Updated ownership checks to use teacher's center or `created_by`
- **OwnedCircle / OwnedTeacher rules**: Added validation to prevent cross-tenant circle/teacher assignment
- **Tests**: Updated CircleManagementTest, EnhancedSecurityTest, ValidationDepthTest

### Admin App Changes
- **Circle Model**: Removed `centerId`, `level`, `endsAt` properties
- **CirclesScreen**: Removed center picker, level text field, end date picker from form and detail view
- **Circle list**: Updated to show "حلقة" instead of level

### Student App Changes
- **api_mappers.dart**: Removed dependency on `circle['level']` (now uses constant fallback)

---

## CHANGE 2: Fix Guardian Relationship Bug ✅ DONE

### Root Cause
- `GuardianRelationship` enum only had 3 values: `father`, `mother`, `guardian`
- `GuardianFactory` generated `uncle` which wasn't in enum
- `UpdateGuardianRequest` used string validation instead of enum
- `CreateGuardianAction` hardcoded 'guardian' fallback

### Fix
- **Enum**: Expanded to 7 canonical values: `father`, `mother`, `brother`, `sister`, `grandfather`, `grandmother`, `guardian`
- **Arabic labels**: Added `label()` method with Arabic translations
- **Factory**: Now uses `GuardianRelationship::values()`
- **CreateGuardianAction**: Uses `GuardianRelationship::Guardian->value` as default
- **UpdateGuardianRequest**: Now uses `Rule::enum(GuardianRelationship::class)`
- **Tests**: GuardianManagementTest validates all 7 values accepted, rejects `أب` (Arabic) and `uncle`

---

## CHANGE 3: Remove Separate Guardian Creation ✅ DONE

### Removed
- `POST /api/v1/guardians` endpoint
- `StoreGuardianRequest` class
- `CreateGuardianAction` class
- `GuardianService::create()` method
- `GuardiansScreen` (Admin App)
- Guardians navigation item, drawer item, bottom nav mapping
- `Pickers.guardians` (Admin App)
- `guardian_id` from student model

### New Flow: Automatic Guardian Creation on Student Creation
**Admin enters:**
- Student Name
- Mobile Number
- Password
- Guardian Name
- Guardian Relationship

**System automatically creates:**
1. Student User (role: `student`)
2. Guardian User (role: `guardian`) 
3. Both share same mobile + password
4. Student profile linked to student user
5. Guardian profile linked to guardian user
6. Pivot link in `guardian_student` with relationship + `is_primary=true`

### Implementation
- **StoreStudentRequest**: Added `mobile`, `password`, `guardian_name`, `guardian_relationship` (required)
- **CreateStudentAction**: Transaction creates both users + profiles + pivot link
- **Guardian endpoints**: Removed `POST`; kept GET/PUT/DELETE for existing records
- **Admin App**: Student form uses new fields, no guardian picker

---

## NEW LOGIN FLOW ✅ DONE

### Step 1: POST `/api/v1/auth/login`
**Request:**
```json
{
  "mobile": "01000000000",  // or "email" for staff
  "password": "secret123",
  "device_name": "app-name"
}
```

**Scenario A - Single account:**
```json
{
  "user": { ... },
  "token": "sanctum_token",
  "permissions": ["*"],
  "requires_role_selection": false
}
```

**Scenario B - Multiple accounts (student + guardian):**
```json
{
  "requires_role_selection": true,
  "available_roles": ["student", "guardian"]
}
```

### Step 2: POST `/api/v1/auth/select-role`
**Request:**
```json
{
  "mobile": "01000000000",
  "password": "secret123",
  "role": "student"
}
```

**Response:**
```json
{
  "user": { ... },
  "token": "sanctum_token",
  "permissions": ["*"],
  "requires_role_selection": false
}
```

### Implementation
- **LoginRequest**: Accepts `email` OR `mobile`
- **LoginAction**: 
  - Email → single account lookup (staff)
  - Mobile → finds all active `student`/`guardian` accounts
  - 1 match → direct login
  - >1 match → role selection response
- **SelectRoleAction**: Validates role, finds user by mobile+role, issues token
- **AuthController**: Returns `permissions` array in both login and select-role responses
- **Tests**: AuthLoginTest covers both scenarios + isolation test

---

## ROLE VALUE MIGRATION: `parent` → `guardian` ✅ DONE

### Changes
- **Role enum**: `Role::Parent = 'guardian'` (internal name `Parent`, stored value `guardian`)
- **Migration**: `2026_07_25_012249_migrate_parent_role_to_guardian.php` - updates existing `parent` → `guardian`
- **Routes**: `role:guardian` middleware for `/parent` prefix
- **SelectRoleRequest**: Validates `role` as `student` or `guardian`
- **Student App**: Maps `guardian` → `UserRole.parent` (UI unchanged)
- **Tests**: Updated to use `guardian` value

---

## SECURITY & TENANT ISOLATION ✅ DONE

### Verified
- Password hashing: `Hash::make()` for both accounts
- Role isolation: Student token → 403 on guardian routes, Guardian token → 403 on student routes
- Tenant isolation: Cross-tenant operations denied for centers, teachers, students, circles
- Token permissions: Returns `["*"]` (Sanctum default; policies enforce authorization)
- RequireRole middleware: Unchanged, works with new `guardian` value
- All policies: Use `Role::Parent` enum (now `guardian`)

---

## TESTING ✅ ALL PASS

### Backend (283 tests, 1074 assertions)
- CircleManagementTest: 8 passed
- StudentManagementTest: 18 passed (auto-guardian creation, all 7 relationships, invalid rejection)
- GuardianManagementTest: 15 passed (7 relationships, enum validation)
- AuthLoginTest: 11 passed (email/mobile login, role selection, isolation)
- EnhancedSecurityTest: 22 passed
- All other feature tests: passed

### Admin App (24 tests)
- flutter analyze: No issues
- flutter test: 24 passed (auth, list controller, models, student creation flow, sync, live API)

### Student App (17 tests)
- flutter analyze: No issues
- flutter test: 17 passed (login, role selection, mappers, widget test)

---

## DOCUMENTATION FILES UPDATED

1. **IMPLEMENTATION_PLAN.md** - This file
2. **DATABASE_IMPACT.md** - Zero destructive migrations
3. **API_CHANGES.md** - All endpoint changes documented
4. **ADMIN_APP_CHANGES.md** - Form, navigation, model changes
5. **STUDENT_APP_CHANGES.md** - Login, role selection, routing
6. **MIGRATION_STRATEGY.md** - Safe rollout, no data loss
7. **TEST_PLAN.md** - Complete verification checklist