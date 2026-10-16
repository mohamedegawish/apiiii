# Store listing drafts

Every feature named here exists in the release build (verified in code). Features
that show a "not available" state in the app (student achievements, certificates,
student payments) are **not** advertised. Do not add claims of encryption at rest,
certifications, awards, user counts or partner organisations.

Limits: title ≤ 30 characters, short description ≤ 80, full description ≤ 4000.
The UI is **Arabic only (RTL)**; list Arabic as the default language. An English
listing is optional — if used, say clearly that the app interface is Arabic.

Naming: both apps currently show the launcher label «الطبري». Distinct store titles
avoid confusion for families who install both. **ACTION REQUIRED:** confirm titles
(and whether "Itqan"/«إتقان» should appear — the shipped branding is «الطبري — رواية
ودراية»; the admin package id is `com.itqan.itqan`).

---

## 1. admin_app — `com.itqan.itqan`

| Field | Draft |
|---|---|
| Title (AR) | الطبري — إدارة التحفيظ |
| Title (EN, optional) | Al-Tabari — Quran Center Admin |
| Category | **Education** |
| Tags suggestion | Education, Productivity/Management |
| Target audience | 18+ (center staff) — see `CONTENT_RATING_AND_DECLARATIONS.md` |
| Contains ads | No |
| In-app purchases | No |
| Privacy policy URL | `[ACTION REQUIRED — publish legal/privacy_policy_ar.md]` |
| Account deletion URL | `[ACTION REQUIRED — publish legal/account_deletion_ar.md]` |
| Website / support URL | `[ACTION REQUIRED]` |
| Support e-mail (mandatory in Play Console) | `[ACTION REQUIRED]` |

**Short description (AR, 60 chars):**
إدارة حلقات تحفيظ القرآن: الطلاب والحضور والحفظ والتقارير

**Short description (EN):**
Manage Quran memorization circles: students, attendance, progress.

**Full description (AR):**

تطبيق «الطبري — رواية ودراية» لإدارة مراكز وحلقات تحفيظ القرآن الكريم، مخصّص لإدارة
المركز والمحفّظين.

للمحفّظ:
• جدول جلسات اليوم وفتح الجلسة وتسجيل الحضور والغياب.
• مسح رمز QR الخاص بالطالب لتسجيل حضوره بسرعة.
• تسجيل الحفظ والمراجعة والمتون ونطاقاتها ودرجاتها، والتقييمات والواجبات.
• العمل دون اتصال بالإنترنت ثم مزامنة التغييرات تلقائياً عند عودة الاتصال.

لإدارة المركز (حسب صلاحية الحساب):
• إدارة الطلاب وأولياء الأمور والمحفّظين والحلقات وجداولها.
• متابعة الجلسات والتقارير الشهرية لتقدّم الطلاب.
• الرسوم والمدفوعات والمصروفات ومسيرات رواتب المحفّظين.
• التقارير وسجلّ التدقيق.

ملاحظات:
• الحسابات يُنشئها المركز؛ لا يوجد تسجيل ذاتي.
• واجهة التطبيق باللغة العربية.
• لا إعلانات، ولا مشتريات داخل التطبيق.
• يمكن حذف الحساب من داخل التطبيق: «المزيد» ← «حذف الحساب».

**Full description (EN, optional):**

Al-Tabari is a management app for Quran memorization centers, for center
administrators and teachers. The app interface is in Arabic.

Teachers: today's sessions, attendance, scanning a student's QR code, recording
memorization, revision and text (matn) progress with grades, assessments and
homework, and offline work with automatic sync.

Center management (depending on account role): students, guardians, teachers,
circles and schedules, monthly progress reports, fees, payments, expenses and
teacher payroll, reports and audit log.

Accounts are created by the center — there is no public sign-up. No ads, no
in-app purchases. Accounts can be deleted in the app.

**Permissions explanation (for the listing / reviewer notes):**

| Permission | Why | When requested |
|---|---|---|
| Internet | Talk to the center's server | Install time (normal permission) |
| Network state | Detect offline mode to queue attendance and sync later | Install time (normal) |
| Camera | **Only** to scan a student's QR code; decoded on the device, no photos saved or uploaded | Runtime prompt the first time a QR scanner screen opens; an Arabic explanation is shown if denied |

---

## 2. student_app — `com.altabari.altabari`

| Field | Draft |
|---|---|
| Title (AR) | الطبري — الطالب وولي الأمر |
| Title (EN, optional) | Al-Tabari — Student & Parent |
| Category | **Education** |
| Target audience | **ACTION REQUIRED** (students may be minors) |
| Contains ads | No |
| In-app purchases | No |
| Privacy policy URL | same URL as admin_app |
| Account deletion URL | same URL as admin_app |

**Short description (AR, 57 chars):**
متابعة حفظ القرآن والحضور والواجبات للطالب ووليّ الأمر

**Short description (EN):**
Follow Quran memorization, attendance and homework — students & parents.

**Full description (AR):**

تطبيق «الطبري — رواية ودراية» للطلاب وأولياء أمورهم في مراكز تحفيظ القرآن الكريم
التي تستخدم منصّة الطبري.

للطالب:
• متابعة سجلّ الحفظ والمراجعة وتقييمات المحفّظ.
• سجلّ الحضور وجلسات الحلقة والجلسة القادمة.
• الواجبات والاختبارات.
• رمز QR الخاص بالطالب لتسجيل الحضور لدى المحفّظ.
• الإشعارات داخل التطبيق.

لوليّ الأمر:
• متابعة جميع الأبناء المرتبطين بحسابه.
• تقرير متابعة لكل ابن: الحضور والحفظ والتقييمات.
• عرض الرسوم والمدفوعات الخاصة بكل ابن.
• إرسال تقييم السلوك والصلاة ومتابعة الواجب للمركز.

ملاحظات:
• يحصل الطالب ووليّ الأمر على الحساب من إدارة المركز؛ لا يوجد تسجيل ذاتي.
• الدخول برقم الجوال أو رمز الطالب وكلمة المرور.
• واجهة التطبيق باللغة العربية، مع الوضع الليلي.
• لا إعلانات، ولا مشتريات داخل التطبيق، ولا دفع إلكتروني.
• يمكن حذف الحساب من داخل التطبيق: «حسابي» ← «حذف الحساب».

**Full description (EN, optional):**

For students and parents at Quran memorization centers that use the Al-Tabari
platform. The app interface is in Arabic.

Students: memorization and revision records with teacher evaluations, attendance
and circle sessions, homework and assessments, a personal QR code for attendance,
in-app notifications.

Parents: all linked children, per-child progress report (attendance, memorization,
evaluations), fees and payments view, and behaviour/prayer/homework follow-up
evaluations sent to the center.

Accounts are issued by the center. Sign in with mobile number or student code.
No ads, no in-app purchases, no online payments. Accounts can be deleted in the app.

**Permissions:** Internet only. No camera, location, contacts, storage or
notification permission.

---

## 3. Graphics — ACTION REQUIRED

| Asset | Spec | Status |
|---|---|---|
| App icon | 512×512 PNG, 32-bit, ≤1 MB | Source exists: `assets/brand/icon.png` (official «الطبري» logo). Export at 512×512 — **ACTION REQUIRED** |
| Feature graphic | 1024×500 JPG/PNG, no transparency | **Missing — ACTION REQUIRED** |
| Phone screenshots | 2–8, 16:9 or 9:16, min 320 px, max 3840 px | **Missing — ACTION REQUIRED.** Use demo data only (no real students' names, faces or phone numbers) |
| 7″/10″ tablet screenshots | Optional (admin app has a tablet layout) | Recommended for admin_app |

Suggested screenshot sets — admin_app: welcome, Today, session attendance, QR scan,
memorization sheet, student profile, reports (admin), privacy & info hub.
student_app: welcome, student home, memorization records, attendance, my QR,
parent home, child report, account screen with privacy/deletion.
