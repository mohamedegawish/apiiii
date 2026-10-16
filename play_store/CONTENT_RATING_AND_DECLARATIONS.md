# Content rating, target audience and policy declarations

Facts are verified from the code. Where Google's answer depends on a business or
legal decision, the item is marked **ACTION REQUIRED** and no answer is guessed.

## 1. Content rating (IARC questionnaire)

Category to pick: **Reference, News, or Educational** (both apps).

| Question area | Fact in the apps | Suggested answer |
|---|---|---|
| Violence, blood, sexual content, nudity, profanity, crude humour | None — educational records and Quran surah names only | No |
| Drugs, alcohol, tobacco, gambling, simulated gambling | None | No |
| Fear / horror | None | No |
| Users can interact or exchange content with each other (chat, public posts) | **No chat, no public content.** Staff enter records about students; guardians send private evaluation scores + notes to the center only | No — **ACTION REQUIRED: confirm**; the guardian free-text note is private to the center, not visible to other users |
| Shares user location | No location access | No |
| Digital purchases | No in-app purchases, no payment processing | No |
| Unrestricted internet / web browser | No WebView, no browser | No |
| Religious content | Quran memorization context (surah names, ranges); no Quran text reader | Answer "No" to objectionable-content questions; religious educational context is not a rated descriptor — **ACTION REQUIRED: developer confirms while filling the live questionnaire** |

Expected result: lowest rating tier (e.g. PEGI 3 / Everyone / IARC 3+), to be
confirmed by the certificate Play issues.

## 2. Target audience and content (Families)

| App | Facts | Recommendation | Decision |
|---|---|---|---|
| admin_app | Used by center administrators and teachers; staff-only login | Target age **18+** only; not appealing to children | **ACTION REQUIRED (confirm)** |
| student_app | Used by students (who may be children) and their guardians; accounts issued by the center; no ads; no analytics/crash SDKs; no location; no camera; one runtime font download from Google Fonts | If any student users are **under 13**, you must select the matching age groups and the app becomes subject to the **Families Policy** (privacy policy, no non-certified ad SDKs, appropriate content, data minimisation, COPPA/GDPR-K style consent handled by the center). If the app is intended to be used by guardians on behalf of young children only, you may target 13+/18+ — but that must match reality and the store listing. | **ACTION REQUIRED — must be decided by the operator** (do not guess) |

Before choosing an under-13 audience for student_app, also:
- bundle the Tajawal font and disable Google Fonts runtime fetching (removes the only
  third-party network call) — recommended;
- confirm the center obtains guardian consent before creating a child's account
  (the privacy policy states the center is responsible for this);
- keep the store listing and screenshots child-appropriate.

## 3. Ads

Verified: no ads SDK (no `google_mobile_ads`, AdMob, Facebook Audience Network, etc.)
in either `pubspec.yaml`, and no ad component in either merged manifest.
Declaration: **"No, my app does not contain ads"** — both apps. READY.

## 4. Financial features declaration

Facts: admin_app **records** tuition fees, cash payments, expenses and staff payroll
for the center's bookkeeping; student_app **displays** a child's fees/payments. Neither
app moves money, offers payment processing, loans, banking, crypto, trading, insurance
or stores card/bank data.

Suggested answer: **"My app doesn't provide any financial features"** —
**ACTION REQUIRED: developer confirms** (Play's categories target financial
services, not bookkeeping of fees).

## 5. User-generated content (UGC)

No public or shared-with-other-users content, no chat, no uploads, no profiles
visible to strangers. Guardian evaluation notes and staff notes are private
records visible only to the center (and, for records about a child, to that child's
guardian/student). Suggested: **not a UGC app** — **ACTION REQUIRED: confirm.**

## 6. Other App content declarations

| Declaration | Answer | Basis |
|---|---|---|
| News app | No | |
| COVID-19 contact tracing / status | No | |
| Government app | No — **ACTION REQUIRED** if the operator is a government body | |
| Health apps | No | |
| Data safety | See `DATA_SAFETY.md` | |
| App access | See `APP_ACCESS_REVIEWER.md` | |
| Photo and video permissions | Not applicable — no `READ_MEDIA_*` / storage permission | merged manifests |
| Foreground service permissions | Not applicable — none declared | merged manifests |
| Full-screen intent / exact alarm | Not applicable — none declared | merged manifests |
| Advertising ID | **No** — `AD_ID` permission not present in either merged manifest; no ads/analytics SDK | merged manifests |
| Account deletion | Provided in app + web URL (**ACTION REQUIRED** publish URL) | `ACCOUNT_DELETION.md` |
