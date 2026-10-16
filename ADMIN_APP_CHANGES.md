# ADMIN_APP_CHANGES.md

## Overview
Changes required in the Admin App (Flutter) to support the three business changes.

---

## CHANGE 1: Circles Screen - Remove Fields

### File: `lib/ui/features/circles_screen.dart`

**Removed Form Fields:**
- `center_id` (FK dropdown)
- `level` (text input)
- `ends_at` (date picker)

**Updated `_fields()` Method:**
```dart
List<FieldSpec> _fields(Circle? c) => [
  FieldSpec.text('name', 'اسم الحلقة', icon: 'circle', required: true, initial: c?.name),
  fkField('teacher_id', 'المحفّظ', Pickers.teachers(_backend), icon: 'teachers', initial: c?.teacherId),
  FieldSpec.number('capacity', 'السعة', initial: c?.capacity?.toString()),
  FieldSpec.date('starts_at', 'تاريخ البدء', initial: c?.startsAt?.toIso8601String()),
  FieldSpec.multiline('description', 'الوصف', initial: c?.description),
  FieldSpec.boolean('is_active', 'نشط', initial: (c?.isActive ?? true).toString()),
];
```

**Removed Detail View Rows:**
- `InfoRow('المستوى', c.level, icon: 'reports')`
- `InfoRow('تنتهي', formatDate(c.endsAt), icon: 'calendar')`

**Updated Circle Model** (`lib/data/models/circle.dart`):
- Remove `centerId`, `level`, `endsAt` properties
- Remove from `fromJson` parsing

---

## CHANGE 2: Guardian Relationship - Updated Enum

### File: `lib/data/models/enums.dart`

**Add GuardianRelationship enum:**
```dart
enum GuardianRelationship implements _ApiEnum {
  father('father', 'أب'),
  mother('mother', 'أم'),
  brother('brother', 'أخ'),
  sister('sister', 'أخت'),
  grandfather('grandfather', 'جد'),
  grandmother('grandmother', 'جدة'),
  guardian('guardian', 'ولي أمر');

  const GuardianRelationship(this.api, this.ar);
  @override final String api;
  final String ar;

  static GuardianRelationship? fromApi(Object? raw) => _byApi(values, raw);
}
```

### File: `lib/ui/features/guardians_screen.dart`

**Update Form Field:**
```dart
FieldSpec.dropdown('relationship', 'صلة القرابة',
    GuardianRelationship.values.map((e) => FieldOption(e.api, e.ar)).toList(),
    initial: g?.relationship),
```

---

## CHANGE 3: Student Creation - New Flow

### File: `lib/ui/screens/students_screen.dart`

**Updated `_fields()` Method:**
```dart
List<FieldSpec> _fields(Student? s) => [
  FieldSpec.text('name', 'اسم الطالب', icon: 'students', required: true, initial: s?.name),
  
  // NEW: Guardian Information Section
  FieldSpec.text('guardian_name', 'اسم ولي الأمر', icon: 'user', required: true, initial: s?.guardianName),
  FieldSpec.dropdown('guardian_relationship', 'صلة القرابة',
      GuardianRelationship.values.map((e) => FieldOption(e.api, e.ar)).toList(),
      initial: s?.guardianRelationship),
  
  // NEW: Credentials
  FieldSpec.phone('mobile', 'رقم الجوال', required: true, initial: s?.mobile),
  FieldSpec.password('password', 'كلمة المرور', required: true),
  
  // Existing fields
  FieldSpec.dropdown('gender', 'الجنس',
      GenderType.values.map((e) => FieldOption(e.api, e.ar)).toList(), initial: s?.gender?.api),
  FieldSpec.dropdown('status', 'الحالة',
      StudentStatus.values.map((e) => FieldOption(e.api, e.ar)).toList(), initial: s?.status?.api),
  FieldSpec.text('national_id', 'الهوية الوطنية', icon: 'user', initial: s?.nationalId),
  FieldSpec.date('birth_date', 'تاريخ الميلاد', initial: s?.birthDate?.toIso8601String()),
  FieldSpec.email('email', 'البريد الإلكتروني', initial: s?.email),
  FieldSpec.date('enrolled_at', 'تاريخ الالتحاق', initial: s?.enrolledAt?.toIso8601String()),
  FieldSpec.multiline('notes', 'ملاحظات', initial: s?.notes),
];
```

**Removed Field:**
- `fkField('guardian_id', 'ولي الأمر', Pickers.guardians(_backend), ...)` - REMOVED

### File: `lib/data/models/student.dart`

**Add new fields:**
```dart
class Student {
  // ... existing fields ...
  
  // NEW fields for creation
  final String? guardianName;
  final String? guardianRelationship;
  final String? mobile;
  final String? password; // Not stored, only for creation
  
  const Student({
    // ... existing ...
    this.guardianName,
    this.guardianRelationship,
    this.mobile,
    this.password,
  });
  
  factory Student.fromJson(Map<String, dynamic> j) => Student(
    // ... existing ...
    guardianName: asString(j['guardian_name']),
    guardianRelationship: asString(j['guardian_relationship']),
    mobile: asString(j['mobile']),
  });
}
```

---

## CHANGE 4: Remove Guardians Screen Entirely

### Files to Delete:
- `lib/ui/features/guardians_screen.dart`

### Files to Update:

**`lib/ui/shell.dart`** / Navigation Menu:
- Remove Guardians menu item
- Remove Guardians route

**`lib/ui/features/module_lists.dart`** (if referenced):
- Remove Guardian from module lists

**`lib/data/backend.dart`**:
- Remove `guardians` repository/calls
- Remove `create` method for guardians

**`lib/data/repositories/crud_repository.dart`**:
- Remove guardian-specific CRUD if any

---

## CHANGE 5: Circle Model Updates

### File: `lib/data/models/circle.dart`

```dart
class Circle {
  final String id;
  final String? teacherId;  // centerId REMOVED
  final String name;
  final String? description;
  // level REMOVED
  final int capacity;
  final bool isActive;
  final DateTime? startsAt;  // endsAt REMOVED
  final DateTime? createdAt;
  final DateTime? updatedAt;

  const Circle({
    required this.id,
    this.teacherId,
    required this.name,
    this.description,
    required this.capacity,
    required this.isActive,
    this.startsAt,
    this.createdAt,
    this.updatedAt,
  });

  factory Circle.fromJson(Map<String, dynamic> j) => Circle(
    id: j['id'].toString(),
    teacherId: asString(j['teacher_id']),
    name: asString(j['name']) ?? '',
    description: asString(j['description']),
    capacity: (j['capacity'] as num?)?.toInt() ?? 20,
    isActive: j['is_active'] as bool? ?? true,
    startsAt: parseDate(j['starts_at']),
    createdAt: parseDate(j['created_at']),
    updatedAt: parseDate(j['updated_at']),
  );
}
```

---

## Summary of Files to Modify

| File | Change Type |
|------|-------------|
| `lib/ui/features/circles_screen.dart` | Modify - Remove fields |
| `lib/data/models/circle.dart` | Modify - Remove properties |
| `lib/data/models/enums.dart` | Add - GuardianRelationship enum |
| `lib/ui/features/guardians_screen.dart` | **DELETE** |
| `lib/ui/screens/students_screen.dart` | Modify - New student form |
| `lib/data/models/student.dart` | Add - New fields |
| `lib/ui/shell.dart` | Remove - Guardians navigation |
| `lib/data/backend.dart` | Remove - Guardian create API |