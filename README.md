# EUDI Wallet Android — Security & GDPR Analysis

**Independent security and privacy assessment of the Age Verification Android Wallet UI**
Part of the EU Digital Identity Wallet initiative under the eIDAS 2.0 framework

**Document type:** Security Risk Assessment (Pre-Production Reference Implementation)
**Framework reference:** ENISA Threat Landscape (adapted) · OWASP MASVS · GDPR contextual mapping
**Author:** Nikiforos Kontopoulos
**Date:** April 2026
**Methodology:** Manual static analysis · Secure code review · GDPR-oriented assessment
**Disclosure type:** Responsible public disclosure via GitHub Issues

---

## Table of Contents

- [Background](#background)
- [Repository Under Analysis](#repository-under-analysis)
- [Overall Risk Posture](#overall-risk-posture)
- [Protected Assets](#protected-assets)
- [Threat Model](#threat-model)
- [Executive Security Assessment](#executive-security-assessment)
- [Realistic Attack Scenarios](#realistic-attack-scenarios)
- [Combined Exploit Scenario](#combined-exploit-scenario--cross-layer-failure)
- [Findings Overview](#findings-overview)
- [Detailed Technical Findings](#detailed-technical-findings)
- [GDPR Considerations](#gdpr-considerations)
- [Positive Observations](#positive-observations)
- [Prioritised Recommendations](#prioritised-recommendations)
- [Risk Evaluation Matrix](#risk-evaluation-matrix)
- [Disclosure Timeline](#disclosure-timeline)
- [Methodology](#methodology)
- [Systemic Risk Consideration](#systemic-risk-consideration)
- [Legal & Ethical Notice](#legal--ethical-notice)
- [About the Author](#about-the-author)

---

## Background

The EU Digital Identity Wallet (EUDI Wallet) is defined under Regulation (EU) 2024/1183 (eIDAS 2.0)
and is scheduled for rollout to all EU citizens by December 2026. It is intended to function as a
core digital identity layer across the European Union, supporting authentication, credential
issuance and presentation, electronic signatures, and age verification.

The repository analysed (`av-app-android-wallet-ui`) is the official Android reference
implementation for age verification, published by the European Commission under the EUPL 1.2
licence. It includes biometric face verification (passport NFC + live selfie), PIN-based
authentication, and OpenID4VCI / OpenID4VP credential flows.

Given the sensitivity of the data involved — biometric photographs, travel document data, and
cryptographic authentication material — this analysis focuses on identifying security and data
protection risks **prior to production deployment**.

Reference implementations occupy a unique position in software ecosystems: they define patterns
that downstream national wallet implementations, vendor integrations, and Member State deployments
will likely inherit. Insecure architectural decisions at this stage carry a disproportionate risk
of propagation into production systems across multiple jurisdictions.

---

## Repository Under Analysis

| Property | Value |
|---|---|
| Repository | [`eu-digital-identity-wallet/av-app-android-wallet-ui`](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui) |
| Language | Kotlin / Android |
| Licence | EUPL 1.2 |
| Analysed version | `main` branch, April 2026 |
| Status | Pre-production / demonstration release |

> The repository explicitly states: *"This is an initial version of the software, developed solely
> for the purpose of demonstrating the business flow of the solution. It is not intended for
> production use."* This context is acknowledged throughout the analysis.

---

## Overall Risk Posture

| Risk Category | Level |
|---|---|
| Data protection | 🔴 High |
| Local device compromise | 🔴 High |
| Network interception | 🔴 High |
| Authentication bypass | 🔴 High |
| Systemic trust | 🟠 Medium–High |
| Availability | 🟢 Low |
| Regulatory exposure | 🟠 Medium–High |

---

## Protected Assets

| Asset | Sensitivity | At risk |
|---|---|---|
| Facial biometric images (live capture) | 🔴 Very High | #68, #73 |
| Passport-derived face image | 🔴 Very High | #69 |
| PIN authentication material | 🔴 High | #66, #67 |
| Biometric embedding ONNX model | 🔴 High | #71 |
| Credential issuance session data | 🔴 Very High | #72 |
| Cryptographic key alias (AndroidKeyStore) | 🟠 High | #66 |
| Lockout enforcement state | 🟠 High | #67 |

---

## Threat Model

This analysis is scoped to the following threat model. Findings are only claimed where they
fall within these attacker capabilities and trust boundaries.

### Device Types in Scope

| Device class | Included | Notes |
|---|---|---|
| Stock Android (GMS, API 28+) | ✅ | Primary target platform |
| Rooted stock Android | ✅ | Realistic for targeted attacks |
| Android ≤ 9 with external storage access | ✅ | `READ_EXTERNAL_STORAGE` available to other apps |
| De-Googled / custom ROM (GrapheneOS, /e/OS) | ✅ | Covered via Layer 2/3 integrity controls |
| Emulated environment | ✅ | CI/testing exposure |

### Attacker Classes

| Class | Capability | Relevant findings |
|---|---|---|
| **Local attacker** | Physical or app-level device access | #66, #67, #68, #69, #73 |
| **Network attacker (MITM)** | Intercept TLS on shared/rogue network | #71, #72 |
| **Supply chain / insider** | Influence model URL or build configuration | #71 |

### Trust Boundaries

| Boundary | Assumed state |
|---|---|
| Device storage | **Not trusted** on rooted devices; weakly protected on stock |
| Network transport | **Not trusted** without certificate pinning |
| ML model supply chain | **Not trusted** without integrity verification |
| AndroidKeyStore | **Trusted** — findings do not challenge hardware security module integrity |
| OS-level sandbox | **Trusted** on non-rooted stock Android only |

### Out of Scope

- Backend services, issuer infrastructure, and verifier endpoints
- iOS companion application
- Compiled APK binary analysis or runtime instrumentation
- Social engineering or phishing scenarios

---

## Executive Security Assessment

- Extraction of biometric identity data from device storage
- Bypass of local authentication protections
- Manipulation of identity verification outcomes
- Interception of sensitive identity flows under controlled conditions

The primary risk emerges from the combination of weak local storage protections, absence of device
integrity enforcement, and insufficient data lifecycle controls. This creates realistic conditions
where biometric and authentication data may be exposed on compromised or semi-trusted devices.

From a data protection perspective, these patterns would likely conflict with GDPR requirements —
particularly regarding storage limitation, data minimisation, and privacy by design — if present
in a production system.

---

## Realistic Attack Scenarios

### Scenario 1 — Biometric Data Extraction

**Attacker capability:** Malicious app (legacy Android) or attacker with physical device access

**Attack chain:**
1. Face images written to `getExternalFilesDir(null)` — accessible to other apps on Android ≤ 9
2. No deletion mechanism exists in `CameraActivity`
3. No encryption applied to stored images
4. No device integrity checks to detect compromise

**Impact:** Persistent storage of biometric face images; unauthorized access to special-category
personal data under GDPR Article 9

---

### Scenario 2 — Identity Verification Bypass

**Attacker capability:** Network attacker (MITM) or rogue Wi-Fi operator

**Attack chain:**
1. ONNX embedding model configured with an `http://` URL
2. No SHA-256 checksum or signature verification after download
3. Attacker substitutes a malicious model that always returns a positive match

**Impact:** Complete bypass of biometric face verification; potential unauthorized credential
issuance to any person presenting any face

---

### Scenario 3 — PIN Protection Bypass

**Attacker capability:** User-level attacker on a rooted device

**Attack chain:**
1. `PinFailedAttempts` and `PinLockoutUntil` stored as raw integers in plaintext XML
2. Attacker edits `eudi-wallet.xml` directly via root shell
3. Counters reset to zero between brute-force attempts
4. Progressive lockout (1 min → 8 hrs) rendered ineffective

**Impact:** Unlimited PIN attempts with no rate limiting; full authentication bypass

---

## Combined Exploit Scenario — Cross-Layer Failure

Individual findings can be dismissed as demo-stage gaps. The following scenario demonstrates
how three of them **chain into a complete identity fraud path** that survives each individual
mitigation unless all are addressed together.

**Attacker profile:** Insider threat, stolen unlocked device, or malicious app on Android ≤ 9

**Preconditions:**

| Requirement | Attacker capability required |
|---|---|
| Device access for image harvest | Physical access OR malicious app on Android ≤ 9 |
| Root access for lockout reset | Rooted device (no SE enforcement) |
| Model substitution | Control of model URL (remote config) OR MITM on HTTP connection |

**Step 1 — Biometric harvest** *(exploits #68)*

Face images from previous verification sessions are stored unencrypted in
`getExternalFilesDir(null)` and never deleted. The attacker retrieves one or more
`captured_frame_*.png` files — high-quality lossless PNGs of the victim's face.

**Step 2 — PIN lockout suppression** *(exploits #67)*

`PinFailedAttempts` and `PinLockoutUntil` are plaintext integers in `eudi-wallet.xml`.
The attacker resets both to zero between brute-force rounds, eliminating all rate limiting.
With no device integrity checks (#73), this requires only root access — available on a
significant fraction of devices in circulation.

**Step 3 — Verification pipeline compromise** *(exploits #71)*

The ONNX face embedding model URL is supplied via `FaceMatchConfig` — a configuration object
that can be set at integration time or, depending on deployment, via remote configuration.
The `ModelDownloader` accepts both `http://` and `https://` URLs with no post-download integrity
check. Two realistic vectors exist: (a) an attacker controlling remote configuration substitutes
the model URL; (b) a MITM on an unprotected `http://` connection intercepts and replaces the
model file in transit. In either case, the attacker substitutes a model that extracts embeddings
matching the harvested reference image. The live verification step now accepts the attacker's
face as the victim's.

**Result:**

> An attacker with device access and network positioning can harvest biometric identity
> data, eliminate authentication rate limiting, and corrupt the verification pipeline —
> enabling unauthorized credential presentation **without triggering any detection mechanism
> currently present in the codebase**.

This is not three bugs. This is a trust model failure.

**Why this matters at scale:** The EUDI Wallet credential, once issued, is intended for
acceptance at banks, border crossings, and public services across all EU member states.
A compromised verification pipeline does not affect one user — it affects the **relying
parties' trust in every credential issued through that pipeline**.

---

## Findings Overview

### Findings Summary Table

| ID | Finding | Category | Severity |
|---|---|---|---|
| [#66](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/66) | SharedPreferences obfuscation — not encryption | Cryptography | 🔴 High |
| [#67](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/67) | PIN lockout counters stored in plaintext | Authentication | 🔴 High |
| [#68](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/68) | Biometric images stored externally, never deleted | Data protection | 🔴 High |
| [#69](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/69) | Passport image not deleted on failure | Data lifecycle | 🟠 Medium |
| [#70](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/70) | Non-constant-time PIN comparison | Cryptography | 🟠 Medium |
| [#71](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/71) | ONNX model download without integrity verification | Supply chain | 🔴 High |
| [#72](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/72) | No TLS certificate pinning | Network security | 🔴 High |
| [#73](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/73) | No device integrity / root detection | Resilience | 🟠 Medium–High |
| [#74](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/74) | Sensitive data logged to shareable files | Information leakage | 🟠 Medium |
| [#75](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/75) | Deep link intent without validation | Input validation | 🟠 Medium |
| [#76](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/76) | Single AES key, no rotation | Key management | 🟠 Medium |
| [#77](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/77) | Missing consent mechanism and retention controls | GDPR compliance | 🟠 Medium |

### Count by Severity

| Severity | Count |
|---|---|
| 🔴 High — Exploitable / Design Weakness | 5 |
| 🟠 Medium–High / Medium | 6 |
| 🔵 Hardening Gap | 1 |
| **Total** | **12** |

### GDPR Risk Indicators

| Area | Finding |
|---|---|
| Storage limitation | Biometric images never deleted |
| Data minimisation | Raw images stored instead of derived embeddings |
| Transparency | No user-facing processing information at capture |
| Privacy by design | Externally accessible storage used for biometric data |
| Consent | No explicit consent gate before biometric processing |
| Retention enforcement | No systematic TTL or background cleanup |

👉 **[All issues on GitHub: #66–77](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues)**

---

## Detailed Technical Findings

---

### 🔴 [#66] Insecure SharedPreferences "encryption" — hardcoded shuffle

**Severity:** High (Design Weakness)
**Files:**
- `business-logic/src/main/java/eu/europa/ec/businesslogic/extension/StringExtensions.kt`
- `business-logic/src/main/java/eu/europa/ec/businesslogic/controller/storage/PrefsController.kt`

#### Description

`PrefsControllerImpl.setString()` applies a Fisher-Yates shuffle with a **hardcoded, publicly
known seed** as its "encryption". This is a deterministic, reversible character permutation — not
cryptography. Inline comments describe it as "encrypted", which is misleading.

```kotlin
// StringExtensions.kt
private val FY_SEED = intArrayOf(1, 3, 5, 7, 9, 2, 4, 6, 8)

internal fun String.shuffle(seed: IntArray = FY_SEED): String =
    encodeToBase64().computeShuffle(seed)
```

All strings stored via `setString()` — including `BiometricAuthentication` JSON, `PinEnc`,
`PinIv`, and `CryptoAlias` — are recoverable by anyone with access to the public source code.

#### Fix

Replace with Jetpack Security `EncryptedSharedPreferences`:

```kotlin
private fun getSharedPrefs(): SharedPreferences {
    val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()
    return EncryptedSharedPreferences.create(
        context,
        "eudi-wallet",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
}
```

---

### 🔴 [#67] PIN lockout counters stored in plaintext — bypassable on rooted devices

**Severity:** High (Design Weakness)
**File:** `authentication-logic/src/main/java/eu/europa/ec/authenticationlogic/storage/PrefsPinStorageProvider.kt`

#### Description

`PinFailedAttempts` and `PinLockoutUntil` are written via `setInt()` / `setLong()`, which apply
**no obfuscation at all** — raw values in plaintext XML. The progressive lockout scheme (4
attempts → 1 min, through 9+ attempts → 8 hrs) is entirely bypassable on a rooted device.

```kotlin
override fun incrementFailedAttempts(): Int {
    val currentAttempts = getFailedAttempts() + 1
    prefsController.setInt(KEY_FAILED_ATTEMPTS, currentAttempts) // plaintext
    return currentAttempts
}

override fun setLockoutUntil(timestampMillis: Long) {
    prefsController.setLong(KEY_LOCKOUT_UNTIL, timestampMillis) // plaintext
}
```

#### Fix

Use `EncryptedSharedPreferences` (see #66) and add an HMAC integrity check over the lockout state:

```kotlin
fun setLockoutUntil(timestampMillis: Long) {
    val hmac = computeHmac(timestampMillis.toString(), keystoreKey)
    prefsController.setLong(KEY_LOCKOUT_UNTIL, timestampMillis)
    prefsController.setString(KEY_LOCKOUT_HMAC, hmac)
}
```

---

### 🔴 [#68] Biometric face images stored in external storage and never deleted `[GDPR]`

**Severity:** High
**File:** `passport-scanner/src/main/java/kl/open/fmandroid/CameraActivity.kt`

#### Description

`CameraActivity` saves each captured selfie frame as a lossless PNG to `getExternalFilesDir(null)`.
There is **no `delete()` call anywhere in the class**. Three or more PNG files accumulate per
session (the decision engine requires 3 samples). On Android ≤ 9, `getExternalFilesDir` is
readable by any app with `READ_EXTERNAL_STORAGE`.

```kotlin
val finalFile = File(
    getExternalFilesDir(null), // ← external storage, no encryption
    "captured_frame_${System.currentTimeMillis()}.png"
)
// ... file is written, processed, and passed downstream
// ← no finalFile.delete() exists anywhere in this class
```

#### Fix

Use internal storage and delete in a `finally` block:

```kotlin
private fun captureFrame() {
    val finalFile = File(filesDir, "face_frame_${System.currentTimeMillis()}.png")
    val callback = object : ImageCapture.OnImageCapturedCallback() {
        override fun onCaptureSuccess(imageProxy: ImageProxy) {
            try {
                // save and process
            } finally {
                finalFile.delete() // always runs
            }
        }
    }
}
```

---

### 🟠 [#69] Passport reference image not deleted on failed face match `[GDPR]`

**Severity:** Medium
**File:** `onboarding-feature/src/main/java/eu/europa/ec/onboardingfeature/ui/passport/passportlivevideo/PassportLiveVideoViewModel.kt`

#### Description

`deleteTempFile(config.faceImageTempPath)` is called **only in the success branch**. On failure
or exception, the passport face photograph remains on device indefinitely.

```kotlin
is FaceMatchPartialState.Success -> {
    deleteTempFile(config.faceImageTempPath)  // ✅ deleted on success
    navigateToNextScreen()
}
is FaceMatchPartialState.Failure -> {
    showError(matchResult.error)              // ❌ file NOT deleted
}
// catch block also does not delete
```

#### Fix

```kotlin
try {
    when (val result = passportLiveVideoInteractor.captureAndMatchFace(...)) {
        is FaceMatchPartialState.Success -> navigateToNextScreen()
        is FaceMatchPartialState.Failure -> showError(result.error)
    }
} catch (e: Exception) {
    showError(e.message ?: "Unexpected error")
} finally {
    deleteTempFile(config.faceImageTempPath) // always runs
}
```

---

### 🟠 [#70] Non-constant-time PIN comparison; decrypted PIN exposed in heap

**Severity:** Medium
**File:** `authentication-logic/src/main/java/eu/europa/ec/authenticationlogic/storage/PrefsPinStorageProvider.kt`

#### Description

`isPinValid()` uses the `==` operator (non-constant-time). `retrievePin()` is a public method
that returns the full decrypted PIN as a `String`, leaving it in the JVM heap.

```kotlin
override fun isPinValid(pin: String): Boolean = retrievePin() == pin
// retrievePin() decrypts and returns the full PIN as plaintext String
```

#### Fix

```kotlin
override fun isPinValid(pin: String): Boolean {
    val stored = decryptedAndLoad().toByteArray(Charsets.UTF_8)
    try {
        return MessageDigest.isEqual(stored, pin.toByteArray(Charsets.UTF_8))
    } finally {
        stored.fill(0) // zero out after use
    }
}
```

Remove `retrievePin()` from the public interface — callers have no legitimate need for the raw PIN.

---

### 🔴 [#71] Face embedding ONNX model downloadable over HTTP without integrity verification

**Severity:** High
**Files:** `passport-scanner/src/main/java/eu/europa/ec/passportscanner/face/ModelDownloader.kt`,
`AVFaceMatchSdkImpl.kt`

#### Description

`ModelDownloader` accepts `http://` URLs for the face embedding model. There is no checksum or
signature verification after download. A MITM attacker can substitute a model that always returns
a positive match result.

```kotlin
// http:// is explicitly handled — no rejection, no hash verification
if (config.embeddingExtractorModel.startsWith("http://") ||
    config.embeddingExtractorModel.startsWith("https://")) {
    downloadModelFromUrl(...) // file used directly after download
}
```

#### Fix

```kotlin
// Reject http:// at configuration level
require(!modelPath.startsWith("http://")) { "Insecure HTTP not permitted for model download" }

// Verify SHA-256 after download
val actualHash = MessageDigest.getInstance("SHA-256")
    .digest(destFile.readBytes())
    .toHexString()
check(actualHash == config.expectedModelHash) {
    destFile.delete()
    "Model integrity check failed"
}
```

---

### 🔴 [#72] No certificate pinning for backend endpoints

**Severity:** High
**File:** `network-logic/src/main/res/xml/network_security_config.xml`

#### Description

`network_security_config.xml` only disables cleartext traffic. No `<pin-set>` is configured
for any domain. An attacker with a rogue CA certificate can intercept all TLS traffic.

```xml
<!-- Current configuration -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false" />
    <!-- No <domain-config> with <pin-set> -->
</network-security-config>
```

#### Fix

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false" />
    <domain-config>
        <domain includeSubdomains="true">issuer.example.eu</domain>
        <pin-set expiration="2027-01-01">
            <pin digest="SHA-256">AAAA...primary pin...=</pin>
            <pin digest="SHA-256">BBBB...backup pin...=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

---

### 🔵 [#73] No root / tampered-device detection

**Severity:** Hardening Gap
**Area:** Application-wide

#### Description

No root detection, emulator detection, or integrity attestation found anywhere in the codebase.
On a rooted device, all storage-level protections are negated.

#### Fix

Multi-layered approach covering all Android ecosystems — no user excluded by design:

**Layer 1 — Google Play Integrity API** *(GMS devices)*
```kotlin
val integrityManager = IntegrityManagerFactory.create(context)
integrityManager.requestIntegrityToken(
    IntegrityTokenRequest.newBuilder().setNonce(generateNonce()).build()
).addOnSuccessListener { response ->
    val verdict = decodeAndVerifyToken(response.token())
    if (!verdict.deviceIntegrity.deviceRecognitionVerdict
            .contains(DeviceRecognitionVerdict.MEETS_DEVICE_INTEGRITY)) {
        warnUser(IntegrityLevel.UNTRUSTED)
    }
}
```

**Layer 2 — Hardware-Backed Key Attestation** *(non-GMS devices with TEE)*
```kotlin
fun verifyKeyAttestation(alias: String): Boolean {
    val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
    val chain = keyStore.getCertificateChain(alias)
    // Verify: rootOfTrust.deviceLocked == true
    // Verify: verifiedBootState == VERIFIED
    // Verify: attestationSecurityLevel == TRUSTED_ENVIRONMENT or STRONG_BOX
    return parseAndVerifyAttestationCertificate(chain)
}
```

**Layer 3 — Heuristic detection** *(universal fallback)*
```kotlin
val rootBeer = RootBeer(context)
if (rootBeer.isRooted) { warnUser(IntegrityLevel.LOW) }
```

**Policy: warn, do not hard-block.** Surface the integrity level to the relying party via
presentation metadata — allow verifiers to set their own acceptable risk threshold. This approach
is also preferable from a governance perspective: risk-signaling controls rather than exclusionary
ones remain aligned with EU accessibility and non-discrimination principles applicable to public
digital services.

---

### 🔵 [#74] Sensitive data and file paths logged; log files shareable via FileProvider

**Severity:** Medium
**Files:** `CameraActivity.kt`, `NetworkModule.kt`, `LogController.kt`

#### Description

In debug builds: full filesystem paths of captured face images are logged at DEBUG level; Ktor
is configured with `LogLevel.BODY`, writing complete HTTP request/response bodies to log files.
Log files (up to 10 × 5 MB) are shareable via `FileProvider`.

#### Fix

- Remove `absolutePath` from `CameraActivity` log messages
- Reduce Ktor to `LogLevel.INFO` for all build types
- Restrict FileProvider URI grants to specific consumers

---

### 🔵 [#75] Deep link intent passed to navigation without scheme/host validation

**Severity:** Medium
**File:** `assembly-logic/src/main/java/eu/europa/ec/assemblylogic/ui/MainActivity.kt`

#### Description

Eight intent filters are declared. The raw `Intent` is passed directly to the Compose navigation
graph without whitelist validation. A malicious app can craft a deep link targeting an
attacker-controlled endpoint.

#### Fix

```kotlin
private fun Intent.isValidWalletIntent(): Boolean {
    val allowedSchemes = setOf("openid4vp", "mdoc-openid4vp", "eudi-openid4vp", "credential-offer")
    val scheme = data?.scheme ?: return action == "android.intent.action.MAIN"
    return scheme in allowedSchemes && isAllowedHost(data?.host)
}
```

---

### 🔵 [#76] Single AES key for all operations; no key rotation

**Severity:** Medium
**File:** `business-logic/src/main/java/eu/europa/ec/businesslogic/controller/crypto/KeystoreController.kt`

#### Description

One AES-256-GCM key handles all non-biometric operations. No rotation mechanism exists.

#### Fix

Separate keys per purpose + versioned alias scheme:

```kotlin
private fun aliasForVersion(version: Int) = "eudi_wallet_key_v$version"

fun rotateKey() {
    val newVersion = currentVersion + 1
    // Re-encrypt data under new key, delete old key from KeyStore
}
```

---

### 🔵 [#77] Missing consent mechanism and data retention enforcement `[GDPR]`

**Severity:** Medium
**Area:** `onboarding-feature/` — no consent screen before biometric capture

#### Description

Three GDPR obligations are unaddressed at code level:

- **Art. 9(2)(a):** No consent screen or consent flag before biometric processing
- **Art. 13:** No UI component presenting required information (controller, purpose, retention, rights)
- **Art. 5(1)(e):** No TTL, background cleanup, or maximum age policy for biometric files

#### Fix

```kotlin
// Consent gate before biometric capture
if (!consentRepository.hasBiometricConsentBeenGiven()) {
    showBiometricConsentScreen(
        onAccepted = { consentRepository.recordConsent(); proceedWithCapture() },
        onDeclined = { navigateBack() }
    )
    return
}

// WorkManager periodic cleanup
class BiometricFileCleanupWorker(...) : CoroutineWorker(...) {
    override suspend fun doWork(): Result {
        val maxAge = TimeUnit.HOURS.toMillis(24)
        filesDir.walkTopDown()
            .filter { it.name.startsWith("face_frame_") }
            .filter { System.currentTimeMillis() - it.lastModified() > maxAge }
            .forEach { it.delete() }
        return Result.success()
    }
}
```

---

## GDPR Considerations

This analysis does not represent legal conclusions. It identifies **implementation-level patterns
that may introduce compliance risk** if retained in a production system processing citizen
biometric data at EU scale.

Several GDPR obligations depend on system-level context that is outside the scope of static
analysis — specifically: the identity of the data controller (the Commission, a Member State
authority, or a wallet provider), the precise legal basis for biometric processing (consent,
public task, or another basis under Art. 9(2)), and whether biometric data ever leaves the
device or is processed solely on-device. These determinations affect which obligations apply
and how. The findings below are presented as **technical risk indicators**, not as confirmed
violations.

| Article | Concern | Finding |
|---|---|---|
| Art. 5(1)(c) — Data minimisation | Raw PNG images stored instead of derived embeddings | #68 |
| Art. 5(1)(e) — Storage limitation | No deletion mechanism for biometric face images | #68, #69 |
| Art. 9(2)(a) — Explicit consent | No consent gate or flag before biometric processing | #77 |
| Art. 13 — Transparency | No processing information presented to user at capture time | #77 |
| Art. 25 — Privacy by Design | Externally accessible storage used for biometric data | #68 |
| Art. 35 — DPIA | Large-scale biometric processing for identification purposes mandates assessment | #77 |

Full compliance assessment requires architectural documentation, a system-level DPIA, and
legal review of the applicable national implementation — none of which are available from
source code alone.

---

## Positive Observations

| Area | Implementation |
|---|---|
| Biometric level | `BIOMETRIC_STRONG` (Class 3) enforced |
| Key generation | AES-256-GCM via AndroidKeyStore |
| Key protection | `setUserAuthenticationRequired(true)` + `setInvalidatedByBiometricEnrollment(true)` |
| Backup | `android:allowBackup="false"` |
| Network baseline | `cleartextTrafficPermitted="false"` globally |
| FileProvider | `android:exported="false"` |
| Dependency scanning | OWASP Dependency Check Plugin in CI |
| PIN lockout | Progressive schedule well-designed (1 min → 8 hrs) |

---

## Prioritised Recommendations

### Priority 1 — Critical (address before any production path)

| Action | Addresses |
|---|---|
| Replace SharedPreferences with `EncryptedSharedPreferences` (AES-256-GCM) | #66, #67 |
| Move biometric images to `filesDir`; delete in `finally` block unconditionally | #68, #69 |
| Add SHA-256 checksum verification for ONNX model post-download; reject `http://` | #71 |
| Implement TLS certificate pinning for all backend endpoints | #72 |

### Priority 2 — High (address before production release)

| Action | Addresses |
|---|---|
| Implement Play Integrity API + Hardware-Backed Key Attestation + heuristic fallback | #73 |
| Replace PIN comparison with `MessageDigest.isEqual()`; remove `retrievePin()` from public API | #70 |
| Implement explicit consent gate and Art. 13 information screen before biometric capture | #77 |
| Add `WorkManager` periodic cleanup for residual biometric files | #77 |

### Priority 3 — Hardening (security hardening phase)

| Action | Addresses |
|---|---|
| Remove sensitive file paths and HTTP bodies from log output | #74 |
| Add scheme/host whitelist validation before intent routing | #75 |
| Introduce versioned key alias scheme enabling key rotation | #76 |

---

## Risk Evaluation Matrix

| Dimension | Rating | Primary drivers |
|---|---|---|
| Confidentiality | 🔴 High | #66, #68, #71, #72 |
| Integrity | 🔴 High | #67, #71, #73 |
| Availability | 🟢 Low | No findings affect service availability |
| Regulatory exposure | 🟠 Medium–High | #68, #69, #77 |
| Systemic / cross-border trust | 🟠 Medium–High | Combined exploit scenario |

---

| Date | Action |
|---|---|
| April 2026 | Source code analysis conducted |
| April 16, 2026 | Issues [#66](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/66)–[#77](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/77) submitted |
| April 2026 | Public disclosure via LinkedIn and this repository |

All findings were reported to maintainers via the official GitHub issue tracker before any
public communication, in accordance with responsible disclosure principles.

---

## Methodology

This assessment was conducted exclusively through **manual static analysis** of publicly
available source code. No dynamic testing, instrumentation, reverse engineering of compiled
binaries, or access to any live service or user data was performed.

**Scope:**
- Kotlin source files across authentication, storage, crypto, biometric, and networking modules
- `AndroidManifest.xml`, `network_security_config.xml`, build configuration, CI pipelines

**Standards referenced:**
- OWASP MASVS (MSTG-STORAGE-1, MSTG-RESILIENCE-1, MSTG-RESILIENCE-4, MSTG-NETWORK-4)
- OWASP MSTG
- GDPR Articles 5, 9, 13, 25, 30, 33, 35

---

## Systemic Risk Consideration

The EUDI Wallet is intended to operate as foundational identity infrastructure for hundreds of
millions of EU citizens. Weaknesses in biometric verification, secure data storage, and device
trust assumptions do not affect only individual users — they may scale into broader risks
affecting cross-border identity trust, credential reliability, and citizen data protection
across the Union.

This is why pre-production security analysis of reference implementations matters: issues
identified and corrected now cost orders of magnitude less than the same issues discovered
after citizen data is involved.

---

## Legal & Ethical Notice

- Analysis performed on publicly available source code licensed under EUPL 1.2
- No proprietary, confidential, or restricted information was accessed
- No compiled binary, live service, production environment, or user data was tested or accessed
- All findings reported to maintainers via GitHub before public disclosure
- No source code from the analysed repository is reproduced in this repository
- This work is intended solely to support the secure and privacy-conscious development of
  public digital infrastructure

---

## About the Author

**Nikiforos Kontopoulos** — IT Educator, Security Researcher, Author

📎 [GitHub Issues #66–77](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues)

---

*This analysis is provided for educational and public-interest purposes only.
No liability is assumed for decisions made based on this report.*
