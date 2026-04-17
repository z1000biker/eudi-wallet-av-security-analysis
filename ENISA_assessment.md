# ENISA-Style Security Risk Assessment
## EU Digital Identity Wallet – Android Age Verification Component

**Reference Implementation Security & GDPR Risk Analysis**

---

## Document Metadata

| Field | Value |
|---|---|
| Document type | Security Risk Assessment (Pre-Production Reference Implementation) |
| Framework alignment | ENISA Threat Landscape (adapted) · OWASP MASVS · GDPR risk-indicator mapping |
| Methodology | Manual static code analysis — no dynamic testing |
| Scope | Android Kotlin reference implementation |
| Author | Nikiforos Kontopoulos |
| Date | April 2026 |
| Disclosure model | Responsible disclosure via GitHub Issues ([#66–#77](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues)) |

---

## 1. Executive Summary

This assessment evaluates the security and privacy posture of the EU Digital Identity Wallet
Android age verification reference implementation (`av-app-android-wallet-ui`), published by
the European Commission under the EUPL 1.2 licence.

The system processes high-sensitivity biometric and identity data, including:

- Facial biometric images (live capture + passport-derived)
- PIN authentication material
- Cryptographic keys and session metadata
- Machine learning-based face verification models

### Key Findings

The analysis identifies systemic weaknesses across four critical domains:

- Local data protection controls
- Authentication state integrity
- Network and model supply chain trust
- Device integrity assumptions

### Overall Risk Classification

| Category | Risk Level |
|---|---|
| Confidentiality | 🔴 High |
| Integrity | 🔴 High |
| Authentication security | 🔴 High |
| Network trust | 🔴 High |
| Regulatory exposure (GDPR) | 🟠 Medium–High |
| Systemic identity assurance | 🟠 Medium–High |
| Availability | 🟢 Low |

---

## 2. Scope

### 2.1 In Scope

- Kotlin application source code
- Authentication and biometric modules
- Storage and cryptographic layers
- Network configuration and model download logic
- Android manifest and security configuration

### 2.2 Out of Scope

- Backend issuer/verifier infrastructure
- Production deployments
- iOS companion implementation
- Runtime binary instrumentation
- Social engineering attacks

---

## 3. System Description

The system implements:

- Biometric face verification (passport NFC + live selfie)
- PIN-based authentication mechanism
- OpenID4VCI / OpenID4VP credential flows
- ONNX-based facial embedding model execution
- Local credential and session storage

> The system is designated as a reference implementation. The repository explicitly states it
> is not intended for production use. However, reference implementations occupy a privileged
> position in software ecosystems: they define architectural patterns that downstream national
> wallet implementations, vendor integrations, and Member State deployments will likely inherit.
> Insecure decisions at this stage carry a disproportionate propagation risk.

---

## 4. Threat Model

### 4.1 Adversary Classes

| Actor | Capability | Relevant findings |
|---|---|---|
| Local attacker | App-level access, file inspection | #66, #68, #69, #74, #75 |
| Root attacker | Full filesystem and preference modification | #66, #67, #73 |
| Network attacker | MITM or rogue CA environment | #71, #72 |
| Supply chain attacker | Model or configuration manipulation | #71 |

### 4.2 Trust Boundaries

| Boundary | Trust assumption | Status |
|---|---|---|
| Device storage | Untrusted on rooted devices | ⚠️ Violated — see #66, #67, #68 |
| Network transport | Untrusted without pinning | ⚠️ Violated — see #71, #72 |
| ML model supply chain | Untrusted without integrity verification | ⚠️ Violated — see #71 |
| Android Keystore | Trusted hardware-backed subsystem | ✅ Not challenged |
| OS sandbox | Trusted only on non-compromised devices | ⚠️ Conditionally violated — see #73 |

### 4.3 Attack Surface Assumptions

- External storage is not trusted
- Network traffic is not trusted without pinning
- Device integrity may be compromised (root is within threat model)
- ML model distribution is not inherently trusted

---

## 5. Risk Derivation Methodology

### 5.1 Risk Model

```
Risk = Likelihood × Impact  (qualitative)
```

Both dimensions are assessed independently per finding using the rules below.

### 5.2 Impact Classification Rules

| Asset type | Impact baseline |
|---|---|
| Biometric data (face images, embeddings) | High |
| Authentication credentials (PIN, keys) | High |
| Cryptographic material (keys, IVs, aliases) | High |
| Network trust chain (TLS, model integrity) | High |
| Logging / metadata / UI data | Medium |
| Hardening gaps (no standalone exploit path) | Low–Medium |

### 5.3 Likelihood Classification Rules

| Attacker requirement | Likelihood |
|---|---|
| No privileges required | High |
| Local app access / normal device access | High |
| Rooted device required | Medium |
| Network positioning required | Medium |
| Multi-stage attack chain | Medium–Low (amplified if systemic) |

### 5.4 Risk Matrix

| Impact ↓ / Likelihood → | Low | Medium | High |
|---|---|---|---|
| High impact | Medium | **High** | **High** |
| Medium impact | Low | Medium | **High** |
| Low impact | Low | Low | Medium |

---

## 6. Assumption Violation Analysis

This section maps each violated system assumption to its corresponding findings.
ENISA reviewers use this structure to assess whether weaknesses are isolated or systemic.

### A1 — Device Security Assumption

**Assumption:** The Android OS sandbox prevents unauthorized cross-app access to sensitive files
and application state.

**Violations:**

- **#68** — Biometric images stored in `getExternalFilesDir()` — accessible to other apps on
  Android ≤ 9 via `READ_EXTERNAL_STORAGE`
- **#67** — PIN authentication counters stored in plaintext XML — modifiable via root shell

### A2 — Network Security Assumption

**Assumption:** TLS ensures confidentiality and integrity of remote communications and
externally sourced dependencies.

**Violations:**

- **#71** — ONNX embedding model downloadable over `http://` without post-download integrity
  verification
- **#72** — No certificate pinning configured — system CA store fully trusted for all endpoints

### A3 — Data Lifecycle Assumption

**Assumption:** Sensitive biometric artifacts are ephemeral and exist only for the duration of
a verification session.

**Violations:**

- **#68** — No deletion mechanism for live-capture face images in `CameraActivity`
- **#69** — Passport reference image deletion occurs only on success path — not on failure or
  exception
- **#77** — No retention enforcement mechanism, TTL policy, or background cleanup exists

### A4 — Cryptographic Protection Assumption

**Assumption:** Sensitive values stored locally are protected by cryptographic controls that
prevent recovery without the key.

**Violations:**

- **#66** — `setString()` applies deterministic Fisher-Yates shuffle with hardcoded public seed
  — not cryptographic; fully reversible from source code
- **#70** — `retrievePin()` decrypts and returns the full PIN as plaintext `String` to the
  caller; comparison uses non-constant-time `==` operator

---

## 7. Executive Security Assessment

The system demonstrates multiple independent trust boundary weaknesses that collectively enable:

- Extraction of biometric identity data from device storage
- Bypass of authentication rate limiting
- Manipulation of verification outcomes
- Compromise of identity assurance pipeline integrity

**Key structural issue:**

Security controls fail independently — but also fail **compositionally**. The storage layer,
authentication layer, and model integrity layer are each individually weak, and their weakness
is additive under a multi-stage attacker.

This creates a systemic identity assurance degradation risk, particularly relevant in
cross-border credential validation contexts where no single relying party can independently
verify the integrity of a presented credential's issuance chain.

---

## 8. Realistic Attack Scenarios

### Scenario 1 — Biometric Data Extraction

**Attacker:** Malicious app (legacy Android) or physical device access

**Conditions:**

- Face images stored unencrypted in `getExternalFilesDir(null)`
- No deletion lifecycle implemented in `CameraActivity`
- Cross-app accessibility on Android ≤ 9

**Impact:** Persistent biometric identifier exposure; GDPR Article 9 special-category data
breach conditions

---

### Scenario 2 — Identity Verification Bypass

**Attacker:** Network attacker (MITM) or supply chain actor controlling model URL

**Conditions:**

- ONNX model fetched over `http://` or unverified `https://`
- No SHA-256 checksum or signature validation post-download
- Model URL configurable via `FaceMatchConfig`

**Impact:** Arbitrary substitution of verification logic; biometric check always returns
positive match

---

### Scenario 3 — PIN Protection Bypass

**Attacker:** User-level access on rooted device

**Conditions:**

- `PinFailedAttempts` and `PinLockoutUntil` stored as raw integers in plaintext XML
- Direct modification via root shell

**Impact:** Unlimited brute-force attempts; progressive lockout (1 min → 8 hrs) rendered
ineffective

---

## 9. Combined Exploit Scenario — Trust Boundary Collapse

This section demonstrates how independently exploitable weaknesses combine into a
**systemic identity assurance failure** — not a chain of bugs, but a breakdown of independence
assumptions between security layers.

### Step 1 — Storage boundary breach (#68)

- **Asset:** Biometric face frames
- **Boundary:** Device external storage
- **Condition:** No encryption + no deletion lifecycle

➡ **Result:** Persistent biometric extraction capability established

### Step 2 — Authentication state manipulation (#67)

- **Asset:** Lockout enforcement state
- **Boundary:** Local application persistence
- **Condition:** Rooted device access + plaintext storage

➡ **Result:** Removal of rate-limiting protection; unlimited authentication attempts

### Step 3 — Model supply chain compromise (#71)

- **Asset:** Biometric verification model (ONNX)
- **Boundary:** Network / configuration layer
- **Condition:** Missing integrity verification + HTTP support

➡ **Result:** Substitution of decision logic; verification always returns positive

### Cross-boundary result

The system fails across three independent trust domains simultaneously:

- Storage trust boundary
- Authentication control boundary
- Model integrity boundary

**Outcome:** Complete identity verification bypass without triggering any detection mechanism
currently present in the codebase.

### System-level interpretation

This is classified as:

> **Identity assurance failure due to multi-layer trust collapse**

Not a chain of bugs — but a breakdown of independence assumptions between security layers.
Each layer's failure amplifies the impact of the others.

---

## 10. Findings Overview

### Severity Normalization Statement

Severity classification is derived from a combination of:

- Asset criticality (per Section 5.2)
- Attacker capability requirements (per Section 5.3)
- Exploit feasibility under real-world device conditions
- Cross-boundary impact amplification potential (per Section 9)

### Findings Summary Table

| ID | Finding | Category | Severity |
|---|---|---|---|
| [#66](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/66) | Weak SharedPreferences obfuscation | Cryptography | 🔴 High |
| [#67](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/67) | PIN lockout bypass via plaintext state | Authentication | 🔴 High |
| [#68](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/68) | Biometric images stored externally, never deleted | Data protection | 🔴 High |
| [#69](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/69) | Passport image retention on failure path | Data lifecycle | 🟠 Medium |
| [#70](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/70) | Non-constant-time PIN comparison | Cryptography | 🟠 Medium |
| [#71](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/71) | Model download without integrity verification | Supply chain | 🔴 High |
| [#72](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/72) | No TLS certificate pinning | Network security | 🔴 High |
| [#73](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/73) | No device integrity / root detection | Resilience | 🟠 Medium–High |
| [#74](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/74) | Sensitive data logged to shareable files | Information leakage | 🟠 Medium |
| [#75](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/75) | Deep link intent without validation | Input validation | 🟠 Medium |
| [#76](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/76) | Single AES key, no rotation strategy | Key management | 🟠 Medium |
| [#77](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues/77) | Missing consent mechanism and retention controls | GDPR compliance | 🟠 Medium |

---

## 11. Detailed Technical Findings (Abbreviated)

For full technical detail including affected file paths, source code excerpts, and Kotlin/XML
fix proposals, refer to the linked GitHub issues above.

### #66 — Weak Cryptographic Design in SharedPreferences

- Deterministic shuffle-based obfuscation with hardcoded public seed
- No cryptographic confidentiality guarantee
- Full reversibility from publicly available source code
- **Fix:** Replace with Jetpack Security `EncryptedSharedPreferences` (AES-256-GCM)

### #67 — PIN Lockout State Stored in Plaintext

- Authentication counters (`PinFailedAttempts`, `PinLockoutUntil`) stored as raw integers
- Modifiable via root shell — bypasses progressive lockout entirely
- **Fix:** Encrypted storage + HMAC integrity check over lockout state

### #68 — External Storage Biometric Leakage

- Captured face images written to `getExternalFilesDir(null)` without encryption
- No deletion mechanism exists — files persist indefinitely
- Cross-app accessibility on Android ≤ 9
- **Fix:** Use `filesDir` (internal); delete in `finally` block unconditionally

### #69 — Conditional Deletion Failure Path

- `deleteTempFile()` called only on success — not on failure or exception
- Passport face photograph persists after every failed verification attempt
- **Fix:** Move deletion to `finally` block

### #70 — Unsafe PIN Comparison

- `isPinValid()` uses non-constant-time `==` operator
- `retrievePin()` exposes decrypted PIN as heap-resident `String`
- **Fix:** `MessageDigest.isEqual()`; remove `retrievePin()` from public interface

### #71 — Model Integrity Failure

- ONNX model fetchable over `http://` — no rejection of insecure scheme
- No SHA-256 or signature verification post-download
- Enables model substitution via MITM or remote configuration manipulation
- **Fix:** Reject `http://`; verify SHA-256 checksum before model is loaded

### #72 — Missing Certificate Pinning

- TLS trust chain fully delegated to system CA store
- Susceptible to rogue CA interception in enterprise or adversarial network environments
- **Fix:** Add `<pin-set>` in `network_security_config.xml` for all backend endpoints

### #73 — No Device Integrity Verification

- No root detection, emulator detection, or hardware attestation
- All local storage protections negated on rooted devices
- **Fix:** Multi-layer approach: Play Integrity API (GMS) + Hardware-Backed Key Attestation
  (non-GMS with TEE) + heuristic fallback (universal); policy: warn, not hard-block, to
  preserve EU accessibility and non-discrimination principles

### #74 — Sensitive Logging Exposure

- Full filesystem paths of biometric images logged at DEBUG level
- Ktor HTTP client logs full request/response bodies in debug builds
- Log files shareable via `FileProvider`
- **Fix:** Remove path logging; reduce Ktor to `LogLevel.INFO`

### #75 — Deep Link Injection Risk

- Eight intent filters declared; raw `Intent` passed to navigation without whitelist validation
- **Fix:** Validate scheme and host against allowlist before routing

### #76 — Single Key, No Rotation

- One AES-256-GCM key handles all non-biometric operations
- No key rotation or versioning mechanism
- **Fix:** Separate keys per purpose; versioned alias scheme

### #77 — Missing Consent and Retention Controls

- No consent screen or flag before biometric processing pipeline is invoked
- No Art. 13 information presented to user at capture time
- No TTL, background cleanup, or maximum age policy for biometric files
- **Fix:** Explicit consent gate; `WorkManager` periodic cleanup; onboarding privacy notice

---

## 12. GDPR Risk Indicator Mapping

This section identifies GDPR risk indicators based on technical implementation patterns.
Final legal classification depends on:

- Defined data controller identity
- Lawful basis for biometric processing
- Deployment architecture (on-device vs. backend processing)
- National implementation of eIDAS 2.0 framework

| Article | Risk Indicator | Finding(s) |
|---|---|---|
| Art. 5(1)(c) — Data minimisation | Raw biometric images stored instead of derived embeddings | #68 |
| Art. 5(1)(e) — Storage limitation | No deletion lifecycle for biometric data | #68, #69, #77 |
| Art. 9 — Special category data | Biometric processing without explicit consent gate | #77 |
| Art. 13 — Transparency | No processing information presented to user at capture | #77 |
| Art. 25 — Privacy by design | External storage used for biometric data | #68 |
| Art. 35 — DPIA | Large-scale biometric identification processing | #77 |

> These represent DPIA-triggering indicators rather than compliance determinations.
> Legal conclusions require full system context, controller identification, and legal basis
> analysis beyond the scope of this static assessment.

---

## 13. Positive Security Controls Observed

| Control | Implementation |
|---|---|
| Biometric authentication level | `BIOMETRIC_STRONG` (Class 3) enforced |
| Key generation | AES-256-GCM via AndroidKeyStore |
| Key protection | `setUserAuthenticationRequired(true)` + `setInvalidatedByBiometricEnrollment(true)` |
| Backup prevention | `android:allowBackup="false"` |
| Network baseline | `cleartextTrafficPermitted="false"` globally enforced |
| FileProvider | `android:exported="false"` — correct configuration |
| Dependency scanning | OWASP Dependency Check Plugin integrated in CI |
| PIN lockout design | Progressive schedule conceptually sound (1 min → 8 hrs) |

---

## 14. Prioritised Recommendations

### Priority 1 — Critical (address before any production path)

| Action | Addresses |
|---|---|
| Replace SharedPreferences with `EncryptedSharedPreferences` (AES-256-GCM) | #66, #67 |
| Move biometric storage to `filesDir`; enforce deletion in `finally` block | #68, #69 |
| Add SHA-256 or signature verification for ONNX model; reject `http://` URLs | #71 |
| Implement TLS certificate pinning for all backend endpoints | #72 |

### Priority 2 — High (address before production release)

| Action | Addresses |
|---|---|
| Implement Play Integrity API + Hardware-Backed Key Attestation + heuristic fallback | #73 |
| Replace PIN comparison with `MessageDigest.isEqual()`; remove plaintext PIN from API | #70 |
| Add explicit consent gate and Art. 13 information screen before biometric capture | #77 |
| Add `WorkManager` periodic cleanup for residual biometric files | #77 |

### Priority 3 — Hardening (security hardening phase)

| Action | Addresses |
|---|---|
| Remove sensitive file paths and HTTP bodies from log output | #74 |
| Add scheme/host whitelist validation before intent routing | #75 |
| Introduce versioned key alias scheme enabling key rotation | #76 |

---

## 15. Risk Evaluation Matrix

| Dimension | Rating | Primary drivers |
|---|---|---|
| Confidentiality | 🔴 High | #66, #68, #71, #72 |
| Integrity | 🔴 High | #67, #71, #73 |
| Availability | 🟢 Low | No findings affect service availability |
| Regulatory exposure | 🟠 Medium–High | #68, #69, #77 |
| Systemic trust | 🟠 Medium–High | Combined exploit scenario (Section 9) |

---

## 16. Disclosure Timeline

| Date | Action |
|---|---|
| April 2026 | Static analysis completed |
| April 16, 2026 | Issues [#66–#77](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues) submitted to official repository |
| April 2026 | Public disclosure initiated via LinkedIn and GitHub |

All findings were reported to the maintainers via the official GitHub issue tracker prior to
any public communication, in accordance with responsible disclosure principles.

---

## 17. Methodology

- Manual static code review only
- No runtime testing, dynamic analysis, or binary instrumentation performed
- No production systems, backend services, or user data accessed
- Analysis scope: Kotlin source files, Android configuration, build pipeline, CI configuration

**Standards referenced:**

- OWASP MASVS (MSTG-STORAGE-1, MSTG-RESILIENCE-1, MSTG-RESILIENCE-4, MSTG-NETWORK-4)
- OWASP MSTG
- ENISA Threat Landscape methodology (adapted for mobile reference implementation context)
- GDPR Articles 5, 9, 13, 25, 35

---

## 18. Systemic Risk Consideration

The system forms part of a foundational identity infrastructure expected to operate across EU
member states under eIDAS 2.0 (Regulation (EU) 2024/1183), with planned deployment to all EU
citizens by December 2026.

Failures in biometric verification integrity, authentication lifecycle security, and model
supply chain trust do not affect only individual users — they may propagate into cross-border
identity assurance degradation, affecting relying parties including:

- Financial institutions (mandatory acceptance under eIDAS 2.0)
- Government services and public administration
- Border control and travel document verification
- Very Large Online Platforms (DSA gatekeepers)

Reference implementations additionally define patterns inherited by Member State implementations.
Architectural weaknesses identified at this stage carry a disproportionate risk of propagation
relative to equivalent weaknesses found in a production-only context.

---

## 19. Legal & Ethical Notice

- Analysis performed on publicly available source code licensed under EUPL 1.2
- No proprietary, confidential, or restricted information was accessed
- No compiled binary, live service, production environment, or user data was tested or accessed
- All findings reported to maintainers via GitHub prior to public disclosure
- No source code from the analysed repository is reproduced in this document
- This document is intended solely to support the secure and privacy-conscious development of
  public digital infrastructure

---

## About the Author

**Nikiforos Kontopoulos** — IT Educator, Security Researcher, Author

📎 [GitHub Issues #66–77](https://github.com/eu-digital-identity-wallet/av-app-android-wallet-ui/issues)

---

*This assessment is provided for security research and public-interest purposes only.
No liability is assumed for decisions made on the basis of this document.
Risk ratings represent the author's independent assessment and do not constitute
a formal audit opinion or legal determination.*
