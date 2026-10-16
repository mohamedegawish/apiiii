# AUTH_AUDIT.md

Authentication is now backed entirely by the real Laravel Sanctum API.

## Flows

| Flow | Status | Implementation |
|------|--------|----------------|
| **Login** | ✅ Real | `LoginScreen` → `AppState.signIn` → `AuthRepository.login` → `POST /auth/login`. Returns user + token; both stored securely. Loading state, inline error, and field-level 422 messages. Quick-login buttons use the seeded demo accounts. |
| **Logout** | ✅ Real | Drawer footer & More screen → confirm dialog → `AuthRepository.logout` → `POST /auth/logout`, then secure storage cleared. Clears even if the network call fails. |
| **Auto-login / session persistence** | ✅ | On launch `AppState.bootstrap` reads the stored token, shows the cached user optimistically, then validates via `GET /auth/me` in the background. |
| **Auto-logout on unauthorized** | ✅ | `ApiClient.onUnauthorized` fires on any `401`; `AppState._handleUnauthorized` clears the session and routes to login — but only when a real token existed (design-preview mode is never kicked out). |
| **Forgot / Reset password / OTP** | ⚠️ No backend | The API exposes only `login`, `logout`, `me`. The forgot screen now shows an honest "contact your center admin" notice instead of a fake OTP flow. Ready to wire when endpoints are added. |

## Token storage (security)

- Stored via **`flutter_secure_storage`** (`SecureTokenStorage`) → Android Keystore / iOS Keychain / platform-encrypted store. Not in plain SharedPreferences.
- Keys: `itqan.auth.token`, `itqan.auth.user`.
- Bearer token injected on every request by `ApiClient`; never logged.
- `InMemoryTokenStorage` provided for tests (no platform channel).

## Session behavior

- `Authorization: Bearer <token>` + `Accept: application/json` on all authenticated requests.
- `me()` refreshes and re-caches the user on each successful validation.
- Deactivated accounts are enforced server-side by the `active` middleware; the client surfaces the resulting error.

## Tests
- `auth_test.dart`: login stores token+user and returns session; logout clears storage even on network failure.
- `live_api_test.dart`: real login for both `super_admin` and `teacher`; `401` for anonymous requests; `/auth/me` returns the correct role.
