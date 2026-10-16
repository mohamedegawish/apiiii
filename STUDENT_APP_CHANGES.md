# STUDENT_APP_CHANGES.md

## Overview
Changes required in the Student App (Flutter) to support the new login flow with mobile/password and role selection.

---

## NEW LOGIN FLOW

### Current Flow (INVALID)
1. User enters email + password
2. Direct login → redirects to student or parent shell based on role

### New Flow (REQUIRED)
1. **Login Screen**: User enters **mobile** + **password**
2. **Backend Check**: How many accounts exist with this mobile?
   - **1 account**: Login directly, issue token for that role
   - **Multiple accounts**: Return `requires_role_selection: true` + `available_roles: ['student', 'guardian']`
3. **Role Selection Screen** (only shown when backend requires it):
   - User selects: Student OR Guardian
4. **Select Role Endpoint**: POST `/auth/select-role` with mobile, password, role
5. Backend issues token for selected role
6. Navigate to appropriate shell

---

## FILES TO MODIFY

### 1. Login Screen - `lib/features/auth/screens/login_screen.dart`

**Changes:**
- Replace email field with **mobile** field
- Update validation and API call
- Handle role selection response
- Navigate to RoleSelectionScreen when needed

**New UI Fields:**
```dart
// Mobile field (replaces email)
TextField(
  controller: _mobile,
  keyboardType: TextInputType.phone,
  textAlign: TextAlign.right,
  textDirection: TextDirection.ltr,
  decoration: InputDecoration(
    hintText: '01000000000',
    prefixIcon: Icon(LucideIcons.phone),
  ),
)
```

**Updated Login Logic:**
```dart
Future<void> _login() async {
  final mobile = _mobile.text.trim();
  if (mobile.isEmpty || _password.text.isEmpty) {
    setState(() => _error = 'أدخل رقم الجوال وكلمة المرور.');
    return;
  }

  setState(() { _loading = true; _error = null; });

  try {
    final repository = ref.read(authRepositoryProvider);
    final result = await repository.login(
      mobile: mobile,
      password: _password.text,
      deviceName: 'altabari-student-app',
    );

    if (result.requiresRoleSelection) {
      // Navigate to role selection screen
      if (mounted) {
        context.go('/select-role', extra: {
          'mobile': mobile,
          'password': _password.text,
          'availableRoles': result.availableRoles,
        });
      }
    } else {
      // Direct login - existing behavior
      final user = result.user;
      ref.read(currentUserProvider.notifier).state = user;
      if (!_rememberMe) {
        await ref.read(secureStorageProvider).deleteToken();
      }
      if (mounted) {
        context.go(user.role == UserRole.student ? '/student' : '/parent');
      }
    }
  } catch (error) {
    if (mounted) setState(() => _error = authErrorMessage(error));
  } finally {
    if (mounted) setState(() => _loading = false);
  }
}
```

---

### 2. New: Role Selection Screen - `lib/features/auth/screens/role_selection_screen.dart`

**New File:**
```dart
/// شاشة اختيار الدور عند وجود حسابات متعددة لنفس الجوال
class RoleSelectionScreen extends ConsumerStatefulWidget {
  final String mobile;
  final String password;
  final List<String> availableRoles;
  
  const RoleSelectionScreen({
    super.key,
    required this.mobile,
    required this.password,
    required this.availableRoles,
  });

  @override
  ConsumerState<RoleSelectionScreen> createState() => _RoleSelectionScreenState();
}

class _RoleSelectionScreenState extends ConsumerState<RoleSelectionScreen> {
  String? _selectedRole;
  bool _loading = false;
  String? _error;

  Future<void> _selectRole() async {
    if (_selectedRole == null) {
      setState(() => _error = 'الرجاء اختيار دور.');
      return;
    }

    setState(() { _loading = true; _error = null; });

    try {
      final repository = ref.read(authRepositoryProvider);
      final user = await repository.selectRole(
        mobile: widget.mobile,
        password: widget.password,
        role: _selectedRole!,
        deviceName: 'altabari-student-app',
      );

      ref.read(currentUserProvider.notifier).state = user;
      if (mounted) {
        context.go(user.role == UserRole.student ? '/student' : '/parent');
      }
    } catch (error) {
      if (mounted) setState(() => _error = authErrorMessage(error));
    } finally {
      if (mounted) setState(() => _loading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final isDark = context.isDark;
    
    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(kSpace24),
          child: Column(
            children: [
              const SizedBox(height: kSpace16),
              // Logo
              Container(
                width: 80, height: 80,
                decoration: BoxDecoration(
                  shape: BoxShape.circle,
                  color: isDark ? kGreen.withValues(alpha: 0.08) : kGreen50,
                  border: Border.all(color: kGold.withValues(alpha: 0.2), width: 1.5),
                ),
                child: Icon(Icons.account_circle, size: 40, color: isDark ? kGreen300 : kGreen),
              ),
              const SizedBox(height: kSpace16),
              
              Text(
                'اختيار الحساب',
                style: TextStyle(fontSize: 24, fontWeight: FontWeight.w800, 
                    color: isDark ? kDarkInk : kInk),
              ),
              const SizedBox(height: kSpace4),
              Text(
                'لديك حسابات متعددة مرتبطة بهذا الرقم',
                style: TextStyle(fontSize: 13, color: isDark ? kDarkMuted : kMuted),
              ),
              const SizedBox(height: kSpace8),
              Text(
                widget.mobile,
                style: TextStyle(fontSize: 16, fontWeight: FontWeight.w600, color: kGreen),
              ),
              
              const SizedBox(height: kSpace28),
              
              // Role options
              ...widget.availableRoles.map((role) {
                final isStudent = role == 'student';
                return Padding(
                  padding: const EdgeInsets.only(bottom: kSpace12),
                  child: GestureDetector(
                    onTap: () => setState(() => _selectedRole = role),
                    child: AnimatedContainer(
                      duration: kDurationFast,
                      padding: const EdgeInsets.all(kSpace16),
                      decoration: BoxDecoration(
                        color: _selectedRole == role 
                            ? kGreen.withValues(alpha: 0.1) 
                            : (isDark ? kDarkSurface : kSurface),
                        borderRadius: kBorderRadius12,
                        border: Border.all(
                          color: _selectedRole == role ? kGreen : (isDark ? kDarkLine : kLine),
                          width: _selectedRole == role ? 2 : 1.5,
                        ),
                      ),
                      child: Row(
                        children: [
                          Container(
                            width: 24, height: 24,
                            decoration: BoxDecoration(
                              shape: BoxShape.circle,
                              border: Border.all(
                                color: _selectedRole == role ? kGreen : (isDark ? kDarkMuted : kMuted),
                                width: 2,
                              ),
                              color: _selectedRole == role ? kGreen : Colors.transparent,
                            ),
                            child: _selectedRole == role
                                ? const Icon(Icons.check, size: 16, color: Colors.white)
                                : null,
                          ),
                          const SizedBox(width: kSpace16),
                          Icon(
                            isStudent ? LucideIcons.graduationCap : LucideIcons.users,
                            size: 24,
                            color: _selectedRole == role ? kGreen : (isDark ? kDarkInk : kInk),
                          ),
                          const SizedBox(width: kSpace12),
                          Expanded(
                            child: Column(
                              crossAxisAlignment: CrossAxisAlignment.start,
                              children: [
                                Text(
                                  isStudent ? 'طالب' : 'ولي أمر',
                                  style: TextStyle(
                                    fontSize: 16,
                                    fontWeight: FontWeight.w700,
                                    color: isDark ? kDarkInk : kInk,
                                  ),
                                ),
                                Text(
                                  isStudent 
                                      ? 'الوصول لحلقة التحفيظ والواجبات والتقييمات'
                                      : 'متابعة الأبناء والحضور والتقييمات',
                                  style: TextStyle(fontSize: 12, color: isDark ? kDarkMuted : kMuted),
                                ),
                              ],
                            ),
                          ),
                        ],
                      ),
                    ),
                  ),
                );
              }),
              
              if (_error != null) ...[
                const SizedBox(height: kSpace12),
                Container(
                  width: double.infinity,
                  padding: const EdgeInsets.all(kSpace12),
                  decoration: BoxDecoration(
                    color: kDangerBg,
                    borderRadius: kBorderRadius12,
                    border: Border.all(color: kDanger.withValues(alpha: 0.25)),
                  ),
                  child: Text(_error!, textAlign: TextAlign.center,
                      style: const TextStyle(color: kDanger, fontSize: 12.5, fontWeight: FontWeight.w600)),
                ),
              ],
              
              const Spacer(),
              
              PrimaryButton(
                label: 'متابعة',
                icon: LucideIcons.arrowRight,
                loading: _loading,
                onPressed: _selectRole,
              ),
              
              const SizedBox(height: kSpace20),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

### 3. Auth Repository - `lib/data/repositories/api_auth_repository.dart`

**Add selectRole method:**
```dart
@override
Future<AppUser> selectRole({
  required String mobile,
  required String password,
  required String role,
  String? deviceName,
}) async {
  final res = await _client.post('auth/select-role', data: {
    'mobile': mobile,
    'password': password,
    'role': role,
    if (deviceName != null && deviceName.isNotEmpty) 'device_name': deviceName,
  });

  final envelope = ApiEnvelope.parse(res.data, (data) {
    final json = asJsonMap(data, context: 'select-role');
    final token = asJsonMap(json['token'], context: 'token')['token']?.toString();
    if (token == null || token.isEmpty) {
      throw const ApiException(
        type: ApiExceptionType.response,
        message: 'لم يتضمّن الرد رمز الدخول.',
      );
    }
    return (user: appUserFromApi(json['user']), token: token);
  });

  await _tokenStorage.saveToken(envelope.data.token);
  return envelope.data.user;
}
```

**Update login method to handle new response:**
```dart
@override
Future<LoginResult> login({
  required String mobile,  // Changed from email
  required String password,
  String? deviceName,
}) async {
  final res = await _client.post('auth/login', data: {
    'mobile': mobile,  // Changed from email
    'password': password,
    if (deviceName != null && deviceName.isNotEmpty) 'device_name': deviceName,
  });

  final envelope = ApiEnvelope.parse(res.data, (data) {
    final json = asJsonMap(data, context: 'login');
    
    // Check if role selection is required
    if (json['requires_role_selection'] == true) {
      final roles = (json['available_roles'] as List?)?.map((e) => e.toString()).toList() ?? [];
      return LoginResult(
        requiresRoleSelection: true,
        availableRoles: roles,
        user: null,
        token: null,
      );
    }
    
    // Normal login response
    final token = asJsonMap(json['token'], context: 'token')['token']?.toString();
    if (token == null || token.isEmpty) {
      throw const ApiException(
        type: ApiExceptionType.response,
        message: 'لم يتضمّن الرد رمز الدخول.',
      );
    }
    return LoginResult(
      requiresRoleSelection: false,
      availableRoles: [],
      user: appUserFromApi(json['user']),
      token: token,
    );
  });

  if (envelope.data.requiresRoleSelection) {
    return envelope.data;
  }

  await _tokenStorage.saveToken(envelope.data.token!);
  return envelope.data;
}
```

---

### 4. New: LoginResult Model - `lib/data/models/models.dart`

**Add to models.dart:**
```dart
/// نتيجة تسجيل الدخول - قد تتطلب اختيار دور
class LoginResult {
  final bool requiresRoleSelection;
  final List<String> availableRoles;
  final AppUser? user;
  final String? token;

  const LoginResult({
    required this.requiresRoleSelection,
    required this.availableRoles,
    this.user,
    this.token,
  });
}
```

---

### 5. Auth Repository Interface - `lib/data/repositories/repositories.dart`

**Update AuthRepository:**
```dart
abstract interface class AuthRepository {
  Future<AppUser?> getCurrentUser();
  
  // Changed: email -> mobile, returns LoginResult
  Future<LoginResult> login({
    required String mobile,
    required String password,
    String? deviceName,
  });
  
  // NEW: Select role after login
  Future<AppUser> selectRole({
    required String mobile,
    required String password,
    required String role,
    String? deviceName,
  });
  
  Future<void> logout();
  Future<void> forgotPassword(String email);
  Future<String> verifyOtp({required String email, required String code});
  Future<void> resetPassword({
    required String email,
    required String token,
    required String password,
    required String passwordConfirmation,
  });
}
```

---

### 6. Router Configuration - `lib/app/router/app_router.dart`

**Add route:**
```dart
GoRoute(
  path: '/select-role',
  name: 'role-selection',
  builder: (context, state) {
    final extra = state.extra as Map<String, dynamic>?;
    return RoleSelectionScreen(
      mobile: extra?['mobile'] as String? ?? '',
      password: extra?['password'] as String? ?? '',
      availableRoles: (extra?['availableRoles'] as List?)?.cast<String>() ?? [],
    );
  },
),
```

---

### 7. Update API Config - `lib/core/network/api_client.dart`

**No changes needed** - uses relative paths.

---

### 8. Models - `lib/data/models/user_models.dart`

**Ensure UserRole enum matches:**
```dart
enum UserRole { student, parent }  // parent = guardian
```

---

## SECURITY CONSIDERATIONS

1. **Password Hashing**: Backend handles - no change needed in app
2. **Token Storage**: Existing secure storage works
3. **Role Isolation**: 
   - Student token only accesses student endpoints
   - Guardian token only accesses guardian endpoints
   - Enforced by backend middleware
4. **Mobile Number**: Used as identifier, not secret

---

## TESTING SCENARIOS

| Scenario | Expected Behavior |
|----------|-------------------|
| Single student account | Direct login → Student shell |
| Single guardian account | Direct login → Parent shell |
| Student + Guardian same mobile | Login → Role Selection → Select → Appropriate shell |
| Wrong password | Error message |
| Invalid mobile format | Validation error |
| Role selection timeout | Return to login |

---

## SUMMARY OF FILES

| File | Change Type |
|------|-------------|
| `lib/features/auth/screens/login_screen.dart` | Major - Mobile field, role selection handling |
| `lib/features/auth/screens/role_selection_screen.dart` | **NEW FILE** |
| `lib/data/repositories/api_auth_repository.dart` | Major - New login response, selectRole |
| `lib/data/models/models.dart` | Add - LoginResult class |
| `lib/data/repositories/repositories.dart` | Update - AuthRepository interface |
| `lib/app/router/app_router.dart` | Add - Role selection route |
