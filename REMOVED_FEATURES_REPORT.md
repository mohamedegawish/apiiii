# REMOVED_FEATURES_REPORT.md

> Phase 5 outcome. Non-destructive default chosen: backend-unsupported screens now render an on-brand **"غير متاحة حالياً"** (`FeatureUnavailableScreen`) instead of fake data. Two dead/unrouted duplicate files were deleted. **All mock data was removed.**

## 1. Screens converted to "Feature not available yet" placeholders
Each still exists (class name + constructor preserved so routing/nav is intact) but no longer renders mock data.

| Screen | Route | Reason | Provider removed |
|---|---|---|---|
| `CertificatesScreen` | `/student/certificates` | No `/certificates` endpoint | `certificatesProvider` |
| `PaymentsScreen` (student) | `/student/payments` | No payments API | `paymentsProvider` |
| `AchievementsScreen` | `/student/achievements` | No badges/gamification API | `badgesProvider` |
| `BehaviorScreen` | `/parent/child/:id/behavior` | No behavior-analytics API | `behaviorProvider` |
| `PrayerScreen` | `/parent/child/:id/prayer` | No prayer-tracking API | `prayerProvider` |
| `ParentMemorizationScreen` | `/parent/child/:id/memorization` | No parent endpoint for a child's records | (used student provider — invalid for parent) |
| `ParentAssessmentsScreen` | `/parent/child/:id/assessments` | No parent endpoint for a child's records | (used student provider — invalid for parent) |
| `AttendanceScreen` (parent) | `/parent/child/:id/attendance` | No parent endpoint for a child's records | `childAttendanceProvider` |

## 2. Files deleted (dead / unrouted duplicates)
| File | Why |
|---|---|
| `features/parent/screens/parent_payments_screen.dart` | Not referenced by the router or any screen; drove fake family-payments data. |
| `features/parent/screens/parent_notifications_screen.dart` | Not routed; `/parent/notifications` already uses the shared `NotificationsScreen`. |

## 3. Mock data & fake sources deleted (Phase 4 — "no fake data")
| File | Contained |
|---|---|
| `data/datasources/mock_data_source.dart` | 650 lines of seeded fake students, records, payments, badges, evaluations, behavior/prayer. |
| `data/repositories/mock_auth_repository.dart` | Fake login/OTP. |
| `data/repositories/mock_student_repository.dart` | Fake student data. |
| `data/repositories/mock_parent_repository.dart` | Fake parent data + `MockBehaviorRepository`. |

## 4. Providers removed from `app_providers.dart`
`certificatesProvider, paymentsProvider, badgesProvider, familyPaymentsProvider, parentNotificationsProvider, behaviorProvider, prayerProvider, behaviorRepositoryProvider, childMemorizationProvider, childAttendanceProvider, childAssessmentsProvider, evaluationStateProvider, selectedRoleProvider` — all backed only by mock data or unused.

## 5. What was NOT removed
- `child_profile_screen.dart` (parent) — kept; now shows the **real** child name/status from `/parent/my-children` (drill-down record screens are placeholders).
- Domain models `Certificate/Payment/Badge/BehaviorData/PrayerData` — kept as types (unused, harmless) so the design can be re-activated if the backend gains these features.
- The rich UI/UX design system — untouched.

**Result:** `flutter analyze` → **0 issues**. No screen renders fabricated data; every list/detail is either a real API call or an explicit "not available" state.
