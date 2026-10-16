# API_MAPPING.md

> Backend endpoint ↔ Flutter screen / provider / repository / model. Status after this integration pass.
> ✅ Connected · ⚠ Partially connected (backend can't supply all UI fields, or write needs extra UI) · ❌ No backend support.

## Legend of Flutter wiring
`Screen → Provider → Repository method → Model`

---

## Authentication
| Endpoint | Flutter wiring | Status |
|---|---|---|
| POST `/auth/login` | `LoginScreen` → `authRepositoryProvider.login(email,password,deviceName)` → `ApiAuthRepository` → `AppUser` (+token stored) | ✅ |
| POST `/auth/forgot-password` | `ForgotScreen` → `.forgotPassword(email)` | ✅ |
| POST `/auth/verify-otp` | `OtpScreen` → `.verifyOtp(email,code)→resetToken` | ✅ |
| POST `/auth/reset-password` | `ResetScreen` → `.resetPassword(email,token,password,confirm)` | ✅ |
| GET `/auth/me` | `SplashScreen` session restore → `.getCurrentUser()` | ✅ |
| POST `/auth/logout` | `ProfileScreen`/`ParentProfileScreen` logout → `.logout()` | ✅ |
| GET `/auth/devices`, DELETE `/auth/devices/{t}`, PUT `/auth/device-name`, POST `/auth/logout-all` | no dedicated UI (device manager not built) | ❌ UI |

## Student role (`role:student`)
| Endpoint | Flutter wiring | Status |
|---|---|---|
| GET `/student/profile` (+`/dashboard`) | `HomeScreen`/`ProfileScreen` → `currentStudentProvider` → `ApiStudentRepository.getStudent` → `Student` | ✅ (gamification fields — points/rank/level/badges — default; not in backend) |
| GET `/student/memorization-sessions` | `MemorizationScreen`,`HomeScreen` → `memorizationRecordsProvider` → `.getMemorizationRecords` → `MemorizationRecord` | ✅ |
| GET `/student/revision-sessions` | `RevisionsScreen` → `revisionsProvider` → `.getRevisions` | ✅ |
| GET `/student/attendance` | `StudentAttendanceScreen` → `attendanceProvider` → `.getAttendance` → `AttendanceLog` | ✅ |
| GET `/student/homework` | `HomeworkScreen`,`HomeScreen` → `homeworkProvider` → `.getHomework` → `Homework` | ✅ |
| GET `/student/assessments` | `AssessmentsScreen` → `assessmentsProvider` → `.getAssessments` → `Assessment` | ✅ |
| GET `/student/sessions` | available in `ApiStudentRepository.getSessions`; no dedicated screen | ⚠ repo-ready, no screen |
| GET `/student/learning-records` | available in `ApiStudentRepository.getLearningRecords`; no dedicated screen | ⚠ repo-ready, no screen |

## Notifications
| Endpoint | Flutter wiring | Status |
|---|---|---|
| GET `/notifications` | `NotificationsScreen` → `notificationsProvider`/`parentNotificationsProvider` → `.getNotifications` → `AppNotification` | ✅ (title/body read from `data{}`) |
| POST `/notifications/{id}/read` | list item tap → `.markNotificationRead` | ✅ |
| POST `/notifications/read-all` | available in repo; no bulk button yet | ⚠ repo-ready, no button |

## Parent role (`role:parent`)
| Endpoint | Flutter wiring | Status |
|---|---|---|
| GET `/parent/my-children` | `ParentHomeScreen`,`ChildrenScreen` → `childrenProvider`/`currentParentProvider` → `ApiParentRepository` → `ChildSummary`/`Parent` | ⚠ backend gives only `id,name,status,center,relationship`; halaqa/teacher/attendance/memorization fields default |
| GET `/parent/dashboard` | `ParentHomeScreen` engagement counts | ⚠ mapped, minimal UI use |
| GET `/parent/evaluations` | `EvaluationScreen` → `evaluationsProvider` → `.getEvaluations` → `Evaluation` | ⚠ read works; UI fields (halaqa/teacher) default |
| POST `/parent/evaluations` | `EvaluationScreen` save → `.saveEvaluation` | ⚠ backend requires `circle_session_id` the current UI doesn't capture — see MISSING_UI_REPORT |
| GET `/parent/my-children/{student}/sessions` | `.getChildSessions` (repo-ready) | ⚠ repo-ready, no screen |
| `ChildProfileScreen`, parent per-child memorization/attendance/assessments | no parent endpoint for arbitrary child records | ❌ backend |

## Backend-unsupported UI (no endpoint exists) → "Feature not available yet"
| Screen | Provider | Status |
|---|---|---|
| `CertificatesScreen` | `certificatesProvider` | ❌ no `/certificates` |
| `PaymentsScreen` (student) | `paymentsProvider` | ❌ no payments API |
| `AchievementsScreen` | `badgesProvider` | ❌ no badges/gamification API |
| `ParentPaymentsScreen` | `familyPaymentsProvider` | ❌ no payments API |
| `BehaviorScreen` | `behaviorProvider` | ❌ no behavior-analytics API |
| `PrayerScreen` | `prayerProvider` | ❌ no prayer-tracking API |

## Quran reference
| Endpoint | Flutter wiring | Status |
|---|---|---|
| GET `/quran/surahs` | surah-number→name shown via bundled `quran_surahs.dart` reference (no live call needed) | ✅ reference bundled |

## Admin/teacher endpoints (`admin_app`, out of scope here)
`/centers /guardians /teachers /students(CRUD) /circles /circle-schedules /statistics /sessions(lifecycle) /reports /users /audit-logs /health /learning-records(CRUD) /devices(register)` — **N/A for student app**.

---

## Finance / Payroll / Student Tracking / Analytics (added this phase — `admin_app` only)

> Everything below is center_owner+ only unless noted. Legend unchanged: `Screen → Provider/Controller → Repository/Backend method → Model`.

### Finance (`role:super_admin,center_owner`)
| Endpoint | Flutter wiring | Status |
|---|---|---|
| `/finance/accounts`, `/finance/revenue-categories`, `/finance/expense-categories`, `/finance/periods` | `Backend.revenueCategories/expenseCategories/financialAccounts` (`CrudRepository`) → used as picker sources (`Pickers.revenueCategories/expenseCategories/financialAccounts`) in Invoice/Expense/Payment forms | ✅ read/create via pickers; no dedicated catalog-management screen (scope decision, see IMPLEMENTATION_REPORT.md) |
| GET/POST `/finance/invoices`, PUT `/finance/invoices/{id}` | `InvoicesScreen` (bespoke create form with line items — doesn't fit the generic `FieldSpec` system) → `Backend.invoices` (`CrudRepository<Invoice>`) → `Invoice`/`InvoiceItem` | ✅ |
| GET/POST `/finance/payments` | `PaymentsScreen` → `Backend.payments` → `Payment` | ✅ create-only, matching the API (no update/void route) |
| GET/POST/PUT `/finance/expenses`, POST `/finance/expenses/{id}/approve`, `/mark-paid` | `ExpensesScreen` → `Backend.expenses` + `Backend.approveExpense`/`markExpensePaid` → `Expense` | ✅ |
| GET `/finance/reports/monthly-summary` | `FinanceScreen`'s Overview tab → `Backend.financeReportFirstRow('monthly-summary', ...)` | ✅ (the other 7 Finance reports have no dedicated admin_app screen yet — CSV/JSON accessible via the API directly) |

### Payroll
| Endpoint | Flutter wiring | Status |
|---|---|---|
| GET/POST `/payroll-runs`, GET `/payroll-runs/{id}`, POST `/approve`, `/mark-paid` (`role:super_admin,center_owner`) | `PayrollScreen` (list + generate + detail) → `Backend.payrollRuns/generatePayrollRun/payrollRun/approvePayrollRun/markPayrollRunPaid` → `PayrollRun`/`PayrollItem` | ✅ |
| PUT `/payroll-items/{id}/adjust` (`role:super_admin,center_owner`) | `PayrollScreen`'s item-adjust form → `Backend.adjustPayrollItem` | ✅ |
| GET `/payroll/mine` (`role:teacher`) | `TeacherDashboardScreen`'s earnings section → `Backend.myPayroll` | ✅ |

### Student Tracking
| Endpoint | Flutter wiring | Status |
|---|---|---|
| GET `/students/{id}/tracking/profile`, `/tracking/history` | no admin_app screen built this phase | ❌ UI — backend complete and tested, exposed for a future student-profile "tracking" tab |

### Analytics
| Endpoint | Flutter wiring | Status |
|---|---|---|
| GET `/analytics/teacher-dashboard` (`role:teacher`) | `TeacherDashboardScreen` (new 5th tab in `TeacherShell`, "أدائي") → `Backend.teacherAnalyticsDashboard` | ✅ |
| GET `/analytics/executive-dashboard` (`role:super_admin,center_owner`) | `ExecutiveDashboardScreen` (new nav item, gated by `Permissions.viewFinance`) → `Backend.executiveDashboard` | ✅ |

### New models
`Invoice`/`InvoiceItem`, `Expense`, `Payment`, `RevenueCategory`/`ExpenseCategory`/`FinancialAccount`, `PayrollRun`/`PayrollItem` — all plain classes with `fromJson`, following the existing `Circle`/`Teacher` model pattern (no freezed/json_serializable in this codebase).

### Nav/permission wiring
`AppScreen.finance/payroll/executiveDashboard` (admin, gated by the pre-existing-but-previously-unused `Permissions.viewFinance`/`viewSalaries` getters — both already correctly scoped to `super_admin\|center_owner` before this phase) and `AppScreen.teacherDashboard` (teacher-only 5th `TeacherShell` tab, no admin equivalent).
