# Implementation Plan: Fingerprint / Biometric Login for the Android App

> **Note:** This repository currently contains only a GitHub profile README — there is no
> Android application source in it yet. This document is the design/implementation **plan**
> for adding biometric login to the Android app. The primary plan targets **native Android
> (Kotlin)** using AndroidX Biometric. A short mapping to other stacks (.NET MAUI, React
> Native, Flutter) is included at the end so it can be adapted to whatever the real app uses.

## 1. Goal & User Story

**As a user**, after I log in with my username/password, I want the app to **offer** to enable
biometric login (fingerprint or face/device biometrics). If I accept, subsequent logins can be
completed with a biometric prompt instead of typing my password. I can **disable** biometric
login at any time from **Settings**.

### Acceptance criteria
1. After a successful credential login, if the device supports biometrics **and** biometric login
   is not already enabled, show a one-time prompt: *"Enable fingerprint / biometric login?"*
   with **Enable** / **Not now** actions.
2. On **Enable**, authenticate with a biometric prompt and securely persist the credential/refresh
   token so future logins can be unlocked biometrically.
3. On the login screen, if biometric login is enabled, show a **"Login with fingerprint"** entry
   point that authenticates and logs the user in without a password.
4. **Settings** contains a **"Biometric login"** toggle. Turning it **off** deletes the stored
   secret and disables the feature. Turning it **on** re-runs the enrollment flow.
5. Graceful handling for: no biometric hardware, no enrolled biometrics, lockout, user cancel,
   and OS-level biometric changes (invalidate stored key).

## 2. Key Design Decisions

### What do we store, and how?
We do **not** store the user's raw password. We store a **secret that lets us re-authenticate**:
- **Preferred:** the backend **refresh token** (or a long-lived session token) issued on login.
- **Alternative:** an opaque "biometric login token" the backend mints specifically for this
  purpose (best practice — lets the server revoke biometric sessions independently).

The secret is stored **encrypted at rest** and released **only** after a successful biometric
authentication, by binding decryption to a key in the Android **Keystore** that requires user
authentication.

### Security model (native Android)
- Generate an AES key in the **Android Keystore** with `setUserAuthenticationRequired(true)` and
  `setInvalidatedByBiometricEnrollment(true)`. The key becomes usable only inside a
  `BiometricPrompt` authentication result, and is **auto-invalidated if the user adds/changes a
  fingerprint** (prevents an attacker who enrolls a new fingerprint from unlocking the secret).
- Use `BiometricPrompt` with a `CryptoObject` wrapping a `Cipher` initialized from that key.
  Encrypt the secret on enrollment; decrypt it on login. The crypto binding guarantees the
  biometric actually happened (defeats simple "return success" tampering).
- Store the ciphertext + IV in `EncryptedSharedPreferences` (Jetpack Security) as defense in depth.
- Set `BIOMETRIC_STRONG` (Class 3) as the required authenticator so only strong biometrics qualify.

### Terminology in the UI
Android device auth spans fingerprint, face, and iris. The system `BiometricPrompt` shows whatever
the device supports. We label the feature **"Biometric login"** in Settings, but the post-login
prompt can say **"fingerprint / biometric"** to match the user's wording. (See open question in §9
about offering a fingerprint-vs-biometric choice.)

## 3. Dependencies (native Android)

```kotlin
// app/build.gradle(.kts)
dependencies {
    implementation("androidx.biometric:biometric:1.1.0")            // BiometricPrompt + BiometricManager
    implementation("androidx.security:security-crypto:1.1.0-alpha06") // EncryptedSharedPreferences
    // (existing) coroutines, lifecycle, DI (Hilt/Koin), networking (Retrofit/Ktor)
}
```

`minSdk` 23+ is required for Keystore-backed biometric crypto; `BiometricPrompt` unifies the API
across versions. No manifest permission is needed for `androidx.biometric` (the legacy
`USE_FINGERPRINT` / `USE_BIOMETRIC` permissions are normal and handled by the library).

## 4. Component Breakdown

| Component | Responsibility |
|---|---|
| `BiometricAvailability` | Wraps `BiometricManager.canAuthenticate(BIOMETRIC_STRONG)` → enum: `AVAILABLE`, `NO_HARDWARE`, `HW_UNAVAILABLE`, `NONE_ENROLLED`. |
| `CryptoManager` | Creates/loads the Keystore key; provides `getEncryptCipher()` / `getDecryptCipher(iv)`. |
| `BiometricSecretStore` | Persists/reads/clears the encrypted secret + IV in `EncryptedSharedPreferences`; exposes `isEnabled()`. |
| `BiometricAuthenticator` | Shows `BiometricPrompt` with a `CryptoObject`; returns the unlocked `Cipher` or a typed error. |
| `AuthRepository` | Existing credential login; adds `enrollBiometric(token)`, `loginWithBiometric()`, `disableBiometric()`. |
| `PostLoginBiometricPrompt` (UI) | The "Enable biometric login?" dialog shown after first successful login. |
| `LoginScreen` (UI) | Adds a "Login with fingerprint" button when enabled. |
| `SettingsScreen` (UI) | Adds the "Biometric login" toggle. |

## 5. Flows

### 5.1 Enrollment (after successful password login)
```
password login OK
  └─ token/refreshToken received from backend
       └─ if BiometricAvailability == AVAILABLE and !store.isEnabled():
            show "Enable fingerprint / biometric login?"  [Enable] [Not now]
              Enable →
                cipher = CryptoManager.getEncryptCipher()
                BiometricPrompt(CryptoObject(cipher)) → onSuccess:
                    encrypted = cipher.doFinal(token.bytes)
                    store.save(encrypted, cipher.iv)   // marks enabled
                    toast "Biometric login enabled"
              Not now → remember dismissal (don't nag every login; re-offer from Settings)
```
- If `NONE_ENROLLED`: optionally offer a deep link to system settings to enroll a fingerprint,
  or just skip the prompt.

### 5.2 Login with biometrics
```
LoginScreen loads
  └─ if store.isEnabled(): show "Login with fingerprint" button
        tap →
          (encrypted, iv) = store.load()
          cipher = CryptoManager.getDecryptCipher(iv)
          BiometricPrompt(CryptoObject(cipher)) → onSuccess:
              token = cipher.doFinal(encrypted)
              AuthRepository.loginWithBiometric(token)  // exchange refresh token → session
              navigate to Home
          onError(KEY_PERMANENTLY_INVALIDATED):  // biometrics changed on device
              store.clear(); show "Please log in with your password again"
```

### 5.3 Disable from Settings
```
Settings → toggle "Biometric login" OFF
  └─ store.clear()  (delete ciphertext + IV, delete Keystore key, set enabled=false)
Settings → toggle ON
  └─ requires an active/valid token → run enrollment flow (5.1). If no token in memory,
     prompt user to re-authenticate with password first.
```

## 6. Error & Edge-Case Handling

| Situation | Handling |
|---|---|
| No biometric hardware (`NO_HARDWARE`) | Never show the offer or the login button. Toggle in Settings shown disabled with an explanatory subtitle. |
| Hardware busy (`HW_UNAVAILABLE`) | Skip silently; fall back to password. |
| No biometrics enrolled (`NONE_ENROLLED`) | Offer deep link to `Settings.ACTION_BIOMETRIC_ENROLL`; otherwise fall back to password. |
| User cancels prompt | Return to the calling screen; keep password login available. |
| Too many attempts / lockout (`ERROR_LOCKOUT`, `ERROR_LOCKOUT_PERMANENT`) | Show message; fall back to password login. |
| New fingerprint enrolled / biometrics changed (`KEY_PERMANENTLY_INVALIDATED`) | Clear the stored secret, disable feature, ask user to log in with password and re-enroll. |
| Backend token expired/revoked | `loginWithBiometric` fails → clear secret, force password login. |
| App uninstalled/reinstalled | Keystore key + EncryptedSharedPreferences are wiped by the OS; feature simply starts disabled. |

## 7. Testing Plan
- **Unit:** `BiometricAvailability` mapping; `BiometricSecretStore` save/load/clear; repository token
  exchange (mock backend). 
- **Instrumented (androidTest):** enrollment persists ciphertext; decrypt returns original token;
  `store.clear()` removes the key; key invalidation path clears state.
- **Manual/QA matrix:** device with fingerprint, device with face only, device with no biometrics,
  emulator with enrolled fingerprint (`adb -e emu finger touch 1`), lockout by repeated failures,
  add-new-fingerprint invalidation, airplane-mode token-exchange failure.
- **Security review:** confirm no raw password stored; confirm key uses `BIOMETRIC_STRONG` +
  `setUserAuthenticationRequired(true)` + `setInvalidatedByBiometricEnrollment(true)`.

## 8. Rollout & Backend Coordination
- Backend: add/confirm a **refresh-token or biometric-token exchange** endpoint and a way to
  **revoke** it (server-side kill switch, and on logout). 
- Feature-flag the UI (remote config) so it can be dark-launched and rolled back.
- Analytics: track offer-shown, enabled, disabled, biometric-login-success/failure (no PII).
- Update privacy policy / store listing note that biometric data never leaves the device (Android
  handles the biometric; the app only receives success + a crypto object).

## 9. Open Questions / Decisions for the User
1. **"Fingerprint" vs "biometric" as separate choices.** The Android system prompt does not let an
   app force *only* fingerprint on a face-capable device — `BiometricPrompt` uses whatever strong
   biometric the device offers. Options:
   - (a) **Single toggle** labeled "Biometric login" (recommended, matches platform behavior), or
   - (b) show two labels in the post-login dialog ("Fingerprint" / "Biometrics") that both map to
     the same `BiometricPrompt` — cosmetic only.
   The plan assumes (a). Confirm if you want the two-option wording in the dialog.
2. **Which secret to store** — reuse the existing refresh token, or have the backend mint a
   dedicated, independently-revocable biometric token (recommended)?
3. **Re-offer cadence** after "Not now" — never again until Settings, or re-offer after N logins?

## 10. Task Checklist (implementation order)
- [ ] Add `androidx.biometric` + `androidx.security:security-crypto` dependencies.
- [ ] `BiometricAvailability` helper + unit tests.
- [ ] `CryptoManager` (Keystore key creation, encrypt/decrypt ciphers).
- [ ] `BiometricSecretStore` (EncryptedSharedPreferences wrapper) + tests.
- [ ] `BiometricAuthenticator` (BiometricPrompt + CryptoObject wrapper).
- [ ] `AuthRepository` additions: `enrollBiometric`, `loginWithBiometric`, `disableBiometric`.
- [ ] Post-login "Enable biometric login?" dialog + wiring after successful login.
- [ ] "Login with fingerprint" button on the login screen.
- [ ] "Biometric login" toggle in Settings (enable → enroll, disable → clear).
- [ ] Error/edge-case handling per §6.
- [ ] Instrumented tests + QA matrix pass.
- [ ] Backend token-exchange/revoke endpoint + feature flag + analytics.

---

## Appendix A — Reference snippets (native Android / Kotlin)

**Availability check**
```kotlin
fun biometricStatus(context: Context): Int =
    BiometricManager.from(context)
        .canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_STRONG)
// BIOMETRIC_SUCCESS / _ERROR_NO_HARDWARE / _ERROR_HW_UNAVAILABLE / _ERROR_NONE_ENROLLED
```

**Keystore key (auth-required, invalidated on new enrollment)**
```kotlin
val spec = KeyGenParameterSpec.Builder(KEY_ALIAS,
        KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .setUserAuthenticationRequired(true)
    .setInvalidatedByBiometricEnrollment(true)
    .build()
KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
    .apply { init(spec) }.generateKey()
```

**Prompt with CryptoObject**
```kotlin
val prompt = BiometricPrompt(activity, executor, object : BiometricPrompt.AuthenticationCallback() {
    override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
        val cipher = result.cryptoObject!!.cipher!!   // now unlocked
        // encrypt (enroll) or decrypt (login) the token
    }
    override fun onAuthenticationError(code: Int, msg: CharSequence) { /* handle §6 */ }
})
val info = BiometricPrompt.PromptInfo.Builder()
    .setTitle("Login with fingerprint / biometrics")
    .setNegativeButtonText("Use password")
    .setAllowedAuthenticators(BiometricManager.Authenticators.BIOMETRIC_STRONG)
    .build()
prompt.authenticate(info, BiometricPrompt.CryptoObject(cipher))
```

## Appendix B — Equivalent libraries for other stacks
| Stack | Biometric API | Secure storage |
|---|---|---|
| **Native Android (Kotlin/Java)** | `androidx.biometric:BiometricPrompt` + Keystore | `EncryptedSharedPreferences` |
| **.NET MAUI** | `Plugin.Maui.Biometric` or `Plugin.Fingerprint`; platform `KeyStore` via bindings | `SecureStorage` (Xamarin.Essentials) backed by Keystore |
| **React Native** | `react-native-biometrics` / `expo-local-authentication` | `react-native-keychain` / `expo-secure-store` |
| **Flutter** | `local_auth` | `flutter_secure_storage` |

The **flows, security model, and edge cases in §5–§6 are identical** across stacks; only the API
names change.
