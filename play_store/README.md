# Google Play submission pack — الطبري (Itqan platform)

| App | Package | Audience |
|---|---|---|
| `admin_app` — administration & teachers | `com.itqan.itqan` | Center staff |
| `student_app` — students & guardians | `com.altabari.altabari` | Students (possibly minors) & guardians |

**Overall status: READY WITH ACTION REQUIRED** — the code, configuration, legal
pages and account deletion are in place and tested, but the apps cannot be
submitted until the developer supplies the upload keystore, the operator's contact
and legal details, public policy/deletion URLs, confirmation of the production API
host, reviewer accounts, store graphics and the Play Console declarations.

| Document | Contents |
|---|---|
| [COMPLIANCE_AUDIT.md](COMPLIANCE_AUDIT.md) | SDK/target API, manifest, permissions, SDKs, network, branding — with AAB evidence |
| [DATA_SAFETY.md](DATA_SAFETY.md) | Data inventory table and suggested Data safety answers per app |
| [ACCOUNT_DELETION.md](ACCOUNT_DELETION.md) | Backend endpoint, what is deleted / retained, app flows, web request procedure |
| [APP_ACCESS_REVIEWER.md](APP_ACCESS_REVIEWER.md) | How reviewers sign in; safe demo-account strategy; console instructions text |
| [STORE_LISTING.md](STORE_LISTING.md) | Titles, short/full descriptions (AR/EN), category, permissions, graphics list |
| [CONTENT_RATING_AND_DECLARATIONS.md](CONTENT_RATING_AND_DECLARATIONS.md) | IARC, target audience/Families, ads, financial features, UGC, other declarations |
| [RELEASE_CHECKLIST.md](RELEASE_CHECKLIST.md) | Build/sign/verify steps, results of this run, READY vs ACTION REQUIRED checklist |
| [legal/](legal/) | Web-hostable Arabic privacy policy, account deletion, terms, about, contact — generated from the in-app source by `admin_app/tool/export_legal_markdown.dart` |

Placeholders the developer must replace are written `[مطلوب: …]` (Arabic content)
or `[ACTION REQUIRED]` (documents). Nothing in this pack invents a company name,
address, phone, e-mail, certification or data practice.
