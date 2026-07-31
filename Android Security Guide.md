# Android Security Guide

Concise reference for securing Android apps: Keystore, biometric auth, Network Security Config, secure storage/IPC, credential & API key handling, cryptography, and Play Integrity.

> For Intent-specific vulnerabilities (redirection, exported component auditing), see the `android-intent-security` skill — this guide covers the broader security surface.

---

## Table of Contents

1. [Android Keystore](#1-android-keystore)
2. [Biometric Authentication](#2-biometric-authentication)
3. [Network Security Configuration](#3-network-security-configuration)
4. [Secure Data Storage](#4-secure-data-storage)
5. [Secure IPC](#5-secure-ipc)
6. [Credential & API Key Management](#6-credential--api-key-management)
7. [Cryptography Best Practices](#7-cryptography-best-practices)
8. [Play Integrity API](#8-play-integrity-api)
9. [WebView Security](#9-webview-security)
10. [Best Practices Checklist](#10-best-practices-checklist)
11. [Further Reading](#11-further-reading)

---

## 1. Android Keystore

Stores cryptographic keys so **key material never enters your app's process** — only cryptographic *operations* (sign/verify/encrypt/decrypt) cross the boundary.

```kotlin
val kpg = KeyPairGenerator.getInstance(KeyProperties.KEY_ALGORITHM_EC, "AndroidKeyStore")
kpg.initialize(
    KeyGenParameterSpec.Builder(alias, KeyProperties.PURPOSE_SIGN or KeyProperties.PURPOSE_VERIFY)
        .setDigests(KeyProperties.DIGEST_SHA256, KeyProperties.DIGEST_SHA512)
        .build()
)
val keyPair = kpg.generateKeyPair()
```

| Concept | Detail |
|---|---|
| Extraction prevention | Key material stays in a system process; can optionally be bound to secure hardware (TEE/StrongBox) so it never leaves even if the OS is compromised |
| **StrongBox KeyMint** | Dedicated secure element (own CPU, secure storage, true RNG) — API 28+ devices that support it. Slower + fewer concurrent ops; only use for high-threat-model apps. Enable via `setIsStrongBoxBacked(true)`; catch `StrongBoxUnavailableException` and retry without it |
| Key use authorization | Restrict a key to specific algorithms/modes, a validity time window, and/or **user authentication** |
| Check hardware-backing | `KeyInfo.getSecurityLevel()` (API 29+) or `isInsideSecureHardware()` (API ≤28) |

**Require user authentication to use a key:**
```kotlin
KeyGenParameterSpec.Builder(KEY_NAME, KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
    .setBlockModes(KeyProperties.BLOCK_MODE_CBC)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_PKCS7)
    .setUserAuthenticationRequired(true)
    .setInvalidatedByBiometricEnrollment(true) // invalidate key if a new fingerprint/face is enrolled
    .build()
```
Run all `AndroidKeyStore` operations **off the main thread** — they can be slow.

---

## 2. Biometric Authentication

```kotlin
// build.gradle.kts
implementation("androidx.biometric:biometric:1.1.0")
```

```kotlin
val promptInfo = BiometricPrompt.PromptInfo.Builder()
    .setTitle("Biometric login")
    .setSubtitle("Log in using your biometric credential")
    .setAllowedAuthenticators(BIOMETRIC_STRONG or DEVICE_CREDENTIAL) // Class 3 biometric OR PIN/pattern/password
    .build()

val biometricPrompt = BiometricPrompt(this, executor, object : BiometricPrompt.AuthenticationCallback() {
    override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) { /* proceed */ }
    override fun onAuthenticationError(errorCode: Int, errString: CharSequence) { /* handle */ }
})
biometricPrompt.authenticate(promptInfo)
```

| Authenticator | Meaning |
|---|---|
| `BIOMETRIC_STRONG` | Class 3 biometric (highest assurance) |
| `BIOMETRIC_WEAK` | Class 2 biometric |
| `DEVICE_CREDENTIAL` | PIN / pattern / password fallback |

- Check availability first: `BiometricManager.from(context).canAuthenticate(...)`; if `BIOMETRIC_ERROR_NONE_ENROLLED`, launch `Settings.ACTION_BIOMETRIC_ENROLL`.
- Can't combine `setNegativeButtonText()` with `DEVICE_CREDENTIAL` in `setAllowedAuthenticators()` — pick one fallback style.
- **Bind biometric auth to a crypto operation** for real protection (not just a UI gate) — pass a `BiometricPrompt.CryptoObject` wrapping a `Cipher`/`Signature`/`Mac` backed by a Keystore key with `setUserAuthenticationRequired(true)`:

```kotlin
cipher.init(Cipher.ENCRYPT_MODE, secretKey)
biometricPrompt.authenticate(promptInfo, BiometricPrompt.CryptoObject(cipher))
// onAuthenticationSucceeded: result.cryptoObject.cipher.doFinal(plaintext)
```
- `setConfirmationRequired(false)` — skip explicit user confirmation for low-risk actions (passive face/iris unlock); keep it `true` (default) for high-risk actions like payments.

---

## 3. Network Security Configuration

```xml
<!-- AndroidManifest.xml -->
<application android:networkSecurityConfig="@xml/network_security_config">
```

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">example.com</domain>
        <trust-anchors>
            <certificates src="@raw/my_ca" /> <!-- custom/internal CA -->
        </trust-anchors>
        <pin-set expiration="2027-01-01">
            <pin digest="SHA-256">7HIpactkIAq2Y49orFOOQKurWxmmSFZhBCoQYcRhJ3Y=</pin>
            <pin digest="SHA-256">fwza0LRMXouZHRC8Ei+4PyuldPDcf3UKgO/04cDM1oE=</pin> <!-- backup pin, required -->
        </pin-set>
    </domain-config>
    <debug-overrides> <!-- only active when android:debuggable="true" -->
        <trust-anchors><certificates src="@raw/debug_cas" /></trust-anchors>
    </debug-overrides>
</network-security-config>
```

| Capability | Purpose |
|---|---|
| Custom trust anchors | Trust a self-signed/internal CA, or restrict to a smaller CA set |
| Cleartext opt-out/in | Block or allow unencrypted HTTP per-domain |
| Certificate pinning | Only trust specific public keys (SPKI hash) for a domain — **always include a backup pin** |
| Certificate Transparency | API 36+: opt-in (disabled by default); **API 37+: enabled by default**, can opt out |
| Encrypted Client Hello (ECH) | API 37+: enabled by default if your networking stack supports it — encrypts the TLS SNI field |

**Cleartext defaults by target SDK:** disabled by default on API 28+; enabled by default on API 27 and below.

---

## 4. Secure Data Storage

| Storage | Default access | Notes |
|---|---|---|
| **Internal storage** | App-private | Never use deprecated `MODE_WORLD_WRITEABLE`/`MODE_WORLD_READABLE` |
| **External storage** | World readable/writable | Only non-sensitive data; validate input as untrusted; never load executables from here without signature verification |
| **Content Provider** | Configurable | Set `android:exported="false"` unless intentionally shared; use `signature` protection level for provider sharing within your own app family; always use **parameterized queries** (`query()`/`update()`/`delete()` args) to avoid SQL injection |

Encrypted local storage (e.g. `EncryptedSharedPreferences`/`security-crypto`) has had a complicated maintenance history — **verify the current recommended approach for your AndroidX BOM at implementation time** rather than assuming; DataStore + Keystore-backed encryption is often the more current pattern (see `Android Data Storage Guide.md`).

---

## 5. Secure IPC

| Component | Default | Rule |
|---|---|---|
| `Service` | Not exported (no `exported`) **unless** it has an intent filter, which exports it by default | Always **explicitly** set `android:exported` |
| `BroadcastReceiver` | Exported by default | Set `android:exported="false"` unless meant for cross-app use; apply a `signature`-level `<permission>` if sharing only with your own signed apps |
| `ContentProvider` | Must declare `android:exported` explicitly (required attribute since API 17) | Same signature-permission guidance |

- **Use explicit Intents** to start/bind `Service`s — implicit intents to `bindService()` throw on API 21+ and are a known hijack vector.
- Intent filters are **not** a security boundary — a component can still be invoked directly with a crafted explicit `Intent`; always validate input inside the receiver/service/activity itself.
- For a full Intent-redirection audit workflow (exported components, `getParcelableExtra` handling, PendingIntent mutability), use the **`android-intent-security`** skill.

---

## 6. Credential & API Key Management

**Authentication:**
- Prefer **Credential Manager** (passkeys, passwords, federated sign-in) for initial sign-in — see `verified-email` skill for a full OTP-less flow.
- Never store raw username/password on-device — authenticate once, then persist a short-lived, service-specific token.
- Rate-limit authentication attempts to blunt brute-force attacks.

**API keys:**
| Practice | Why |
|---|---|
| Never commit keys to source control | Use `secrets-gradle-plugin` or a remote secrets manager instead |
| Encrypt at rest (Keystore + Tink) | Decompiled APKs expose plain string constants trivially |
| Per-environment keys (dev/test/prod) | Isolate blast radius; disable a compromised key without touching prod |
| Restrict by package name + signing cert | Most provider consoles (e.g. Google Maps) support this |
| Rotate every 90 days–6 months | Reduces exposure window for undiscovered leaks |

---

## 7. Cryptography Best Practices

- **Never implement your own crypto primitives.** Use `Cipher`/`KeyGenerator` with vetted algorithms (AES, RSA, EC).
- AES: 256-bit for commercial use (128-bit minimum). EC: 224 or 256-bit curves.
- Use `SecureRandom` to seed any generated key — a weak RNG undermines the whole algorithm.
- Pick the right block mode: **GCM** (preferred, built-in integrity) or CBC/CTR **plus** an HMAC (SHA-256/512) for integrity — never encrypt without integrity protection.
- **Never reuse an IV/counter** in CTR mode.
- For network TLS, use `HttpsURLConnection`/`SSLSocket` (note: `SSLSocket` does **not** do hostname verification itself — verify separately) rather than hand-rolling a protocol.
- Store any reusable key in `KeyStore` — never as a raw byte array in memory/prefs/files.

---

## 8. Play Integrity API

Verifies that requests come from your **genuine, unmodified app** on a **genuine, certified device/Play Games PC instance** — not a fraud-prevention silver bullet, but a strong signal to combine with other checks.

| Verdict | Detects |
|---|---|
| `accountDetails` | Whether the user installed/paid for the app via Google Play |
| `appIntegrity` | Whether the binary matches what Play recognizes (tamper detection) |
| `deviceIntegrity` | Genuine certified device vs. emulator/rooted/unlocked bootloader |
| `appAccessRiskVerdict` (opt-in) | Risky accessibility/overlay/screen-capture apps running alongside yours |
| `playProtectVerdict` (opt-in) | Whether Play Protect is on / has flagged malware |
| `recentDeviceActivity` (opt-in) | Anomalously high request volume (bot/automation signal) |

| Request type | Latency | Use for |
|---|---|---|
| **Standard** | ~hundreds of ms | Frequent, on-demand checks (recommended default) |
| **Classic** | ~seconds | Infrequent, highest-value/sensitive actions only; you manage the `nonce` yourself |

**Key practices:** don't cache verdicts (enables proxy/replay abuse), bind requests with `requestHash`/`nonce`, use a **tiered enforcement strategy** (not binary allow/deny) based on device trust labels (`MEETS_STRONG_INTEGRITY` → `MEETS_BASIC_INTEGRITY`), and request the verdict as close as possible to the sensitive action.

---

## 9. WebView Security

- Don't call `setJavaScriptEnabled(true)` unless you actually need JS — disabled by default, which also disables XSS via script injection.
- `addJavascriptInterface()` exposes Android methods to page JS — only enable for content you fully control/trust (ideally bundled in your APK, never open web content).
- Call `clearCache()` / use `no-store` response headers when a `WebView` handles sensitive data.
- Don't trust data loaded over plain HTTP inside a `WebView` — validate as untrusted input.

---

## 10. Best Practices Checklist

- [ ] Sensitive keys generated/stored in `AndroidKeyStore`, never as raw constants or plain files
- [ ] Keys requiring authorization use `setUserAuthenticationRequired(true)` + biometric `CryptoObject` binding for real (not cosmetic) protection
- [ ] All network traffic over TLS; Network Security Config defines trust anchors + (where warranted) certificate pinning with a backup pin
- [ ] `android:exported` explicitly set on every `Service`/`Receiver`/`Provider` — never left to default inference
- [ ] Content provider queries are parameterized — never string-concatenated
- [ ] API keys excluded from source control, environment-scoped, and rotated periodically
- [ ] Crypto uses vetted primitives (`Cipher`/`KeyGenerator`) with GCM or CBC/CTR+HMAC — never hand-rolled
- [ ] `WebView` JavaScript and `addJavascriptInterface()` disabled unless explicitly required and content is trusted
- [ ] Play Integrity verdicts checked close to the sensitive action, never cached, and layered into a tiered (not binary) enforcement strategy
- [ ] Minimum necessary permissions requested; UUIDs (not IMEI/phone number) used for any unique identifiers

---

## 11. Further Reading

| Resource | Link |
|---|---|
| Android Keystore system | https://developer.android.com/privacy-and-security/keystore |
| Biometric authentication dialog | https://developer.android.com/identity/sign-in/biometric-auth |
| Network Security Configuration | https://developer.android.com/privacy-and-security/security-config |
| Security checklist | https://developer.android.com/privacy-and-security/security-tips |
| Play Integrity API overview | https://developer.android.com/google/play/integrity/overview |
| Local skill: `android-intent-security` | Intent redirection audits, exported component review |
| Local skill: `verified-email` | Credential Manager verified-email flow |

---

*Last Updated: July 2026 · Biometric library 1.1.0.*
