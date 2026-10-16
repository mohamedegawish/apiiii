# MISSING_UI_REPORT.md

> Backend endpoints (in scope for the student/parent app) that currently have **no** — or **incomplete** — UI. Ordered by value.

## 1. Endpoints with a repository but no screen (data reachable, UI missing)
| Endpoint | Repo support | Suggested UI |
|---|---|---|
| GET `/student/sessions` | not yet in repo | "جلسات الحلقة" list under the student shell (circle sessions calendar). |
| GET `/student/learning-records` | not yet in repo | "سجل التعلّم" timeline (type + surah + score). |
| GET `/parent/my-children/{student}/sessions` | `ApiParentRepository` can add `getChildSessions` | child sessions list on `ChildProfileScreen`. |
| POST `/notifications/read-all` | reachable via repo | "تعليم الكل كمقروء" button on `NotificationsScreen`. |

## 2. Endpoints with partial UI needing completion
| Endpoint | Gap | Needed |
|---|---|---|
| POST `/parent/evaluations` | The `EvaluationScreen` collects behavior/prayer only; backend **requires `circle_session_id` + 1–5 scores + `homework_followup_score`**. | Add a **session picker** (from `/parent/my-children/{student}/sessions`) and a homework-follow-up score field, then map RatingLevel/PrayerRating → 1–5 before POST. Currently `saveEvaluation` throws `feature_unavailable` until this UI exists. |
| PUT `/parent/evaluations/{id}` | no edit UI | edit action on an existing evaluation row. |
| GET `/parent/dashboard` | fetched but engagement counts under-displayed | surface counts (evaluations submitted, sessions attended…) on `ParentHomeScreen`. |

## 3. Auth/device management endpoints with no UI
| Endpoint | Needed |
|---|---|
| GET `/auth/devices`, DELETE `/auth/devices/{token}`, PUT `/auth/device-name`, POST `/auth/logout-all` | a "الأجهزة والجلسات" screen under Profile: list active tokens, rename current device, revoke a device, log out of all devices. |
| GET/POST/DELETE `/devices` (push registration) | push-notification registration on login (FCM/APNs token) + a toggle in settings. Requires adding a push SDK (out of current dependency set). |

## 4. Notes
- **Gamification UI** (certificates, payments, badges, behavior analytics, prayer tracking) is **not** listed here because the backend has no endpoints for it — see `REMOVED_FEATURES_REPORT.md`. These are "extra UI", not "missing UI".
- The student `Student` model carries gamification fields (`points`, `rank`, `levelTitle`, `badges`) the backend can't populate; they render with neutral defaults. If the product wants them, the **backend** must add the endpoints first.
