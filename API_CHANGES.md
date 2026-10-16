# API_CHANGES.md

## Overview
Documentation of all API changes for the three business changes.

---

## CHANGE 1: Circle API - Removed Fields

### POST /api/v1/circles (Store)
**Removed Request Fields:**
- `center_id` (was required)
- `level` (was nullable)
- `ends_at` (was nullable)

**Request Body (New):**
```json
{
  "teacher_id": "uuid|null",
  "name": "string",
  "description": "string|null",
  "capacity": "integer|null",
  "is_active": "boolean|null",
  "starts_at": "date|null"
}
```

### PUT /api/v1/circles/{circle} (Update)
**Removed Request Fields:**
- `center_id`
- `level`
- `ends_at`

### GET /api/v1/circles (Index) & GET /api/v1/circles/{circle} (Show)
**Removed Response Fields:**
- `center_id`
- `level`
- `ends_at`

**Response (New):**
```json
{
  "id": "uuid",
  "teacher_id": "uuid|null",
  "name": "string",
  "description": "string|null",
  "capacity": "integer",
  "is_active": "boolean",
  "starts_at": "date|null",
  "created_at": "datetime",
  "updated_at": "datetime"
}
```

### Filtering Removed
- `center_id` filter
- `level` filter
- `ends_at` filter

---

## CHANGE 2: Guardian Relationship Enum

### Enum Values (GuardianRelationship)
| Case | API Value | Arabic Label |
|------|-----------|--------------|
| Father | `أب` | أب |
| Mother | `أم` | أم |
| Brother | `أخ` | أخ |
| Sister | `أخت` | أخت |
| Grandfather | `جد` | جد |
| Grandmother | `جدة` | جدة |

### Validation
- All endpoints now use `Rule::enum(GuardianRelationship::class)`
- Only the 6 canonical Arabic values above are accepted.
- The UI submits the same values it displays; `أب` now validates successfully.

### Affected Endpoints
- `POST /api/v1/guardians` - Store (relationship validated)
- `PUT /api/v1/guardians/{guardian}` - Update (relationship validated)

---

## CHANGE 3: Student Creation & Guardian Auto-Creation

### POST /api/v1/students (Store) - **MAJOR CHANGE**

**Removed Request Fields:**
- `guardian_id` (was nullable uuid)

**New Required Request Fields:**
- `mobile` (string, required, unique per center)
- `password` (string, required, min 8)
- `guardian_name` (string, required)
- `guardian_relationship` (string, required, enum: GuardianRelationship)

**New Request Body:**
```json
{
  "center_id": "uuid|null",
  "name": "string",
  "national_id": "string|null",
  "birth_date": "date|null",
  "gender": "male|female|null",
  "mobile": "string",
  "email": "email|null",
  "password": "string",
  "guardian_name": "string",
  "guardian_relationship": "أب|أم|أخ|أخت|جد|جدة",
  "status": "string|null",
  "enrolled_at": "date|null",
  "graduated_at": "date|null",
  "notes": "string|null"
}
```

**Response (201 Created):**
```json
{
  "message": "Student created successfully.",
  "data": {
    "student": {
      "id": "uuid",
      "center_id": "uuid|null",
      "guardian_id": "uuid",
      "name": "string",
      "gender": "male|female|null",
      "status": { "value": "active", "label": "Active" },
      "enrolled_at": "date|null",
      "graduated_at": "date|null",
      "created_at": "datetime",
      "updated_at": "datetime"
    },
    "guardian": {
      "id": "uuid",
      "center_id": "uuid|null",
      "name": "string",
      "phone": "string",
      "email": "string|null",
      "relationship": "father",
      "gender": "male|female|null",
      "occupation": "string|null",
      "created_at": "datetime",
      "updated_at": "datetime"
    }
  }
}
```

### What Happens Internally
1. Creates **Student User** (role: student) with mobile + password
2. Creates **Guardian User** (role: guardian) with SAME mobile + password
3. Creates **Student** record linked to student user
4. Creates **Guardian** record linked to guardian user
5. Links Student ↔ Guardian in `guardian_student` pivot table

---

## CHANGE 4: Login Flow Redesign

### POST /api/v1/auth/login - **CHANGED BEHAVIOR**

**Request (Changed):**
```json
{
  "mobile": "01000000000",  // was "email"
  "password": "secret123",
  "device_name": "string|null"
}
```

**Response - Scenario A: Single Account (200 OK)**
```json
{
  "message": "Login successful.",
  "data": {
    "user": { ... },
    "token": "sanctum_token",
    "requires_role_selection": false
  }
}
```

**Response - Scenario B: Multiple Accounts (200 OK)**
```json
{
  "message": "Multiple accounts found. Please select a role.",
  "data": {
    "requires_role_selection": true,
    "available_roles": ["student", "guardian"],
    "mobile": "01000000000"
  }
}
```

### POST /api/v1/auth/select-role - **NEW ENDPOINT**

**Request:**
```json
{
  "mobile": "01000000000",
  "password": "secret123",
  "role": "student"  // or "guardian"
}
```

**Response (200 OK):**
```json
{
  "message": "Login successful.",
  "data": {
    "user": { ... },
    "token": "sanctum_token",
    "requires_role_selection": false
  }
}
```

**Error Responses:**
- `422` - Invalid credentials / role not found for this mobile
- `400` - Role selection not required (single account)

---

## REMOVED ENDPOINTS

### Guardian Creation
- `POST /api/v1/guardians` - **REMOVED**

### Guardian Endpoints Kept
- `GET /api/v1/guardians` - List
- `GET /api/v1/guardians/{guardian}` - Show
- `PUT /api/v1/guardians/{guardian}` - Update
- `DELETE /api/v1/guardians/{guardian}` - Delete

---

## ERROR RESPONSES

### Validation Errors (422)
```json
{
  "message": "Validation failed.",
  "errors": {
    "mobile": ["The mobile field is required."],
    "guardian_relationship": ["The selected guardian relationship is invalid."]
  }
}
```

### Authentication Errors (401)
```json
{
  "message": "The provided credentials are incorrect."
}
```

### Role Selection Errors (422)
```json
{
  "message": "Invalid role selected for this mobile number.",
  "errors": {
    "role": ["No account found with role 'student' for this mobile."]
  }
}
```

---

## RATE LIMITING
- `POST /auth/login` - `throttle:login` (5/minute)
- `POST /auth/select-role` - `throttle:login` (5/minute)
- `POST /students` - `throttle:sensitive-write` (30/minute)

---

## PERMISSIONS
- `students.create` - Required for student creation (auto-creates guardian)
- `guardians.create` - **REMOVED** (no longer used)
- `circles.create` - Required for circle creation (unchanged)
- `auth.login` - Public (rate limited)
