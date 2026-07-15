# Implementation Plan: Fingerprint / Biometric Login (SEMX Android app)

> **Target app:** `Hossein-Basiri/ai-powered-smart-expense-manager` → `src/ClientApp`.
> This is a **Capacitor 8 + React 19 + TypeScript (Vite)** web app wrapped as a native
> Android app — **not** native Kotlin or .NET MAUI. So biometric login is implemented in
> TypeScript against a Capacitor biometric plugin, reusing the app's existing refresh-token
> session model. (An earlier draft of this file targeted native Kotlin; it has been rewritten
> for the actual stack.)

## 1. User story & acceptance criteria
**After a successful login** (password or Google), if the device supports biometrics and it
isn't already set up, **ask the user** whether to enable fingerprint / biometric login. If they
accept, future logins can be completed with a biometric prompt instead of a password. The user
can **disable** it from **Settings**.

1. Post-login, in the native app only: if biometrics are available and not yet enrolled, show a
   prompt *"Enable fingerprint / biometric login?"* (**Enable** / **Not now**).
2. On **Enable** → biometric prompt → securely persist the **refresh token** so future logins can
   mint a fresh session without a password.
3. Login screen shows a **"Log in with fingerprint"** button when biometric login is enrolled.
   It authenticates, reads the stored refresh token, calls `/users/refresh`, and establishes the
   session.
4. **Settings** gets a **"Biometric login"** card with a toggle. Off → delete the stored secret.
   On → run enrollment (requires an active session).
5. Graceful handling: no hardware, none enrolled at OS level, lockout, cancel, and refresh token
   expired/revoked → clear state and fall back to password.

## 2. Why the refresh token (not the password)
The app already stores a `TokenPair { accessToken, refreshToken }` in `localStorage` via
`src/auth/session.ts`, and `src/api/client.ts` already exchanges a refresh token at
`POST /users/refresh` for a fresh pair. Biometric login **reuses that exact mechanism**:

- On enrollment we copy the current `refreshToken` into **biometric-protected secure storage**
  (OS Keystore), releasing it only after a successful biometric authentication.
- On biometric login we retrieve it, POST it to `/users/refresh`, store the returned pair in
  `Session`, load `/users/api/users/me`, and we're in — the same shape as
  `AuthContext.establishSession`.

No raw password is ever stored. **No backend changes are required** — `/users/refresh` already
exists. (Optional hardening in §8.)

## 3. Plugin choice (needs a Capacitor 8 compatibility check)
The app is on `@capacitor/core@^8.4.1`. Recommended:

- **`@aparajita/capacitor-biometric-auth`** — `checkBiometry()` (availability + type) and
  `authenticate()` (the biometric prompt). Actively maintained, typed, promise-based.
- **`@aparajita/capacitor-secure-storage`** — Keystore/Keychain-backed key-value store for the
  refresh token.

**Fallback:** `capacitor-native-biometric`, which bundles `verifyIdentity()` +
`setCredentials/getCredentials/deleteCredentials` in one plugin.

> **Action item:** before coding, confirm the chosen plugin publishes a build compatible with
> Capacitor 8 (peer deps). If not, use the fallback or pin a compatible major. This is the one
> real unknown in the plan.

Install (indicative):
```bash
cd src/ClientApp
npm i @aparajita/capacitor-biometric-auth @aparajita/capacitor-secure-storage
npm run android:sync   # build + cap sync android
```
Android specifics: `minSdkVersion` ≥ 23 (check `android/variables.gradle`); the plugin adds the
`USE_BIOMETRIC` permission via its own manifest. No Google Play policy issue — biometric data
never leaves the device; the plugin only returns success + lets us unlock the stored token.

## 4. New / changed files
| File | Change |
|---|---|
| `src/lib/biometric.ts` | **New.** Thin native wrapper (mirrors `lib/googleNative.ts`): `biometricAvailability()`, `promptAndStore(key, value)`, `promptAndRead(key)`, `remove(key)`. Lazy-imports the plugin; all calls gated by `isNativeApp()`. |
| `src/auth/biometric.ts` | **New.** Feature glue: `REFRESH_KEY` constant, `isBiometricEnabled()` (a `semx.biometric` localStorage flag), `enable(refreshToken)`, `disable()`. The **flag** lives in localStorage; the **secret** lives in secure storage. |
| `src/auth/AuthContext.tsx` | Add `biometricEnabled`, `enrollBiometric()`, `loginWithBiometric()`, `disableBiometric()` to the context. `enrollBiometric` reads `Session.tokens.refreshToken`; `loginWithBiometric` reads the stored token → `/users/refresh` → reuse the existing session-establish path; `logout()` should **not** wipe the enrollment (so biometric login survives logout) — clearing is Settings-only. |
| `src/features/auth/BiometricEnrollPrompt.tsx` | **New.** The post-login "Enable fingerprint / biometric login?" dialog (reuse the existing `Modal` component). |
| `src/features/auth/LoginPage.tsx` | After `finishLogin` → `'ok'` in the native app, if available && !enrolled, show `BiometricEnrollPrompt` before navigating. Add a **"Log in with fingerprint"** button (native + enrolled) that calls `loginWithBiometric()`. |
| `src/features/settings/BiometricCard.tsx` | **New.** Settings card with the toggle, modeled on the existing Two-factor card. Rendered from `SettingsPage` only when `isNativeApp()`. |
| `src/features/settings/SettingsPage.tsx` | Render `<BiometricCard />` (native only). |

## 5. Flows (mapped to existing code)

### 5.1 Enrollment — after `establishSession` returns `'ok'`
```
login('ok')  →  navigate happens in LoginPage.finishLogin
  ├─ if isNativeApp() && biometricAvailable && !isBiometricEnabled():
  │     show BiometricEnrollPrompt
  │       Enable → enrollBiometric():
  │                  token = Session.tokens.refreshToken
  │                  biometric.promptAndStore(REFRESH_KEY, token)   // biometric prompt here
  │                  set semx.biometric = '1'
  │                  toast "Biometric login enabled"
  │       Not now → set a dismissed marker (re-offer only from Settings)
  └─ then navigate(from)
```

### 5.2 Login with biometrics — LoginPage button
```
tap "Log in with fingerprint"
  └─ loginWithBiometric():
       refreshToken = biometric.promptAndRead(REFRESH_KEY)   // biometric prompt here
       response = fetch POST /users/refresh { refreshToken }
       if !ok → disable(); throw "Please sign in with your password"
       Session.tokens = response
       profile = api('/users/api/users/me')
       Session.profile = profile; setUser(profile); setRoles(...)
       navigate home
```

### 5.3 Settings toggle
```
OFF → disableBiometric(): biometric.remove(REFRESH_KEY); clear semx.biometric flag; toast
ON  → requires Session.tokens (user is logged in) → enrollBiometric() (5.1 body)
```

## 6. Edge cases
| Case | Handling |
|---|---|
| Not a native build (browser/PWA) | Feature fully hidden — every entry point is behind `isNativeApp()`. |
| No biometric hardware / none enrolled at OS level | Never show the offer or the login button. In Settings, show the toggle disabled with a hint ("Set up a fingerprint in your device settings first"). |
| User cancels the biometric prompt | Treat as no-op (same `/cancel/i` convention as `takeReceiptPhoto`/`nativeGoogleIdToken`); keep password login available. |
| Lockout / too many attempts | Surface the plugin error message; fall back to password. |
| Stored refresh token expired or revoked (`/users/refresh` 4xx) | `disableBiometric()` + prompt to sign in with password and re-enroll. |
| New fingerprint enrolled on device | If using the plugin's key-invalidation option, reads fail → treat as the expired-token case. |
| Logout | Keep the enrollment (biometric login is a convenience across logouts). Only Settings-off or a failed refresh clears it. |

## 7. Tests (Vitest — matches existing `*.test.tsx`)
- `lib/biometric.test.ts`: mock the plugin; availability mapping; store/read/remove; cancel path.
- `auth/biometric.test.ts`: enable sets flag + stores token; disable clears both.
- `AuthContext` test: `loginWithBiometric` refreshes and establishes a session (mock `fetch`/`api`).
- `LoginPage.test.tsx`: fingerprint button appears only when native + enrolled; enroll prompt
  appears after login when available + not enrolled.
- `BiometricCard` test: toggle enable/disable calls the right context methods.
- Existing web tests must stay green (all new behavior is `isNativeApp()`-gated; jsdom → false).
- Manual QA on device/emulator: `adb -e emu finger touch 1` to simulate a fingerprint.

## 8. Optional backend hardening (not required for v1)
Reusing the login refresh token works today. For stronger isolation later, `UserService` could
mint a **dedicated, independently-revocable biometric refresh token** at enrollment and add a
"revoke biometric sessions" action — so disabling biometrics (or losing a device) can be enforced
server-side, not just client-side. Out of scope for the first cut.

## 9. Decisions to confirm before implementing
1. **"Fingerprint" vs "Biometrics" as two choices.** On Android, a Capacitor biometric prompt
   uses whatever strong biometric the device offers — an app can't force *fingerprint-only* on a
   face-capable phone. So offering two options is **label-only**. Options: (a) one toggle
   "Biometric login" (recommended), or (b) show both labels in the enroll dialog, both mapping to
   the same prompt. Your request said "ask if they want a fingerprint login or biometrics login" —
   I'll implement (b)'s wording (a dialog that says fingerprint / biometrics) over the single
   underlying mechanism unless you prefer (a).
2. **Re-offer cadence** after "Not now": never again until Settings (recommended), or re-offer
   after N logins?
3. **Store the login refresh token** (v1, zero backend work) vs. the dedicated biometric token
   from §8 (more work, better revocation)?

## 10. Implementation checklist (in order)
- [ ] Confirm plugin ↔ Capacitor 8 compatibility; install plugin(s); `cap sync`.
- [ ] `src/lib/biometric.ts` (native wrapper) + tests.
- [ ] `src/auth/biometric.ts` (enable/disable/isEnabled + secure-storage key) + tests.
- [ ] `AuthContext` additions (`enrollBiometric`, `loginWithBiometric`, `disableBiometric`, flags).
- [ ] `BiometricEnrollPrompt` + wire into `LoginPage` post-login.
- [ ] "Log in with fingerprint" button on `LoginPage`.
- [ ] `BiometricCard` in `SettingsPage` (native only).
- [ ] Edge-case handling per §6.
- [ ] Tests green (`npm test`, `npm run lint`, `npm run typecheck`); manual device QA.
