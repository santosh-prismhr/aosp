# Level 7A — Deep Dive: Keystore, TrustZone, KeyMint, Gatekeeper, and Hardware Security

> *Supplement to Level 7. This chapter is the AOSP development manual for hardware-backed security: Keystore2, KeyMint, Gatekeeper, Weaver, Synthetic Password, FBE, attestation, StrongBox, TEE integration, debugging, CTS/VTS, Cuttlefish limitations, and OEM production workflows.*

This chapter assumes you already understand Level 7's security model: AVB, SELinux, app sandboxing, permissions, FBE, and Mainline. Here we go deeper into the security boundary that Android relies on when the normal Linux world is compromised: the **Trusted Execution Environment (TEE)** and, on some devices, a discrete **Secure Element / StrongBox**.

A Staff-level Android platform engineer should be able to answer three questions from first principles:

1. **Where is the key material?** If the answer is "in Android userspace," the design is usually wrong for high-value secrets.
2. **What proves the key belongs to a genuine, locked, up-to-date device?** This is attestation.
3. **What survives compromise of root in Android?** Hardware-backed keys, rollback-resistant state, and TEE-enforced authentication should survive; app data in a live unlocked session may not.

---

## 7A.1 — The Hardware Security Architecture

### 7A.1.1 The Two Worlds

ARM TrustZone partitions the SoC into two execution worlds:

- **Normal World / Non-Secure World** — Android Linux kernel, Android userspace, apps, system services.
- **Secure World** — TEE OS plus Trusted Applications (TAs), reached by Secure Monitor Calls (SMCs).

```
Normal World / Android                                      Secure World / TEE
──────────────────────                                      ──────────────────

 Apps / Framework APIs
  │
  │ Android Keystore API
  ▼
 system_server
  ├─ LockSettingsService
  ├─ BiometricService
  └─ vold / StorageManager integration
  │
  ▼
 keystore2 daemon  ──────────────────────────────┐
  │ Rust service                                  │ AIDL / Binder
  │ key namespaces, grants, auth tokens           │
  ▼                                               │
 KeyMint HAL / Gatekeeper HAL / Weaver HAL        │ Normal-world HAL process
  │                                               │
  │ ioctl / TEE driver / SMC                      │
  ▼                                               ▼
────────────────────────── TrustZone boundary ────────────────────────────────
  ▲
  │
 TEE OS: Trusty / QSEE / Kinibi / OP-TEE / vendor TEE
  ├─ KeyMint TA        — key generation, key use, attestation
  ├─ Gatekeeper TA     — credential verification and auth tokens
  ├─ Weaver TA         — hardware rate limiting / secret slots
  ├─ RPMB service      — replay-protected persistent storage
  ├─ DRM TA            — Widevine / media keys
  └─ OEM TAs           — device-specific secure functions
```

**Version note:** Android 12 introduced `keystore2` and AIDL KeyMint as the modern architecture. Older devices use Keystore 1.0 and Keymaster HIDL/legacy HALs. Android 13+ production launches should be KeyMint-first; Android 14/15 devices increasingly use Remote Key Provisioning (RKP) for attestation certificate provisioning.

### 7A.1.2 What Lives Where

| Component | Process / world | Source / interface | Responsibility |
|-----------|-----------------|--------------------|----------------|
| Android Keystore API | App process | `java.security.*`, `android.security.keystore.*` | Public app-facing key API |
| `LockSettingsService` | `system_server` | `frameworks/base/services/core/java/com/android/server/locksettings/` | Credential enrollment, unlock, synthetic password coordination |
| `keystore2` | Native Rust daemon | `system/security/keystore2/` | Key database, grants, operation lifecycle, HAL routing |
| KeyMint HAL | HAL process | `hardware/interfaces/security/keymint/aidl/` | Stable AIDL boundary to secure key operations |
| Gatekeeper HAL | HAL process | `hardware/interfaces/security/gatekeeper/aidl/` | Credential verification and auth token generation |
| Weaver HAL | HAL process / secure element | `hardware/interfaces/weaver/aidl/` | Rate-limited secret slots for credential protection |
| `vold` | Native daemon | `system/vold/` | FBE mount/unlock, CE/DE key install |
| TEE OS / TAs | Secure World | Vendor / Trusty / OP-TEE tree | Enforce secrets outside Android |
| StrongBox | Discrete secure element | KeyMint StrongBox HAL instance | Higher-assurance key storage and attestation |

### 7A.1.3 TEE vs StrongBox

| Property | TEE KeyMint | StrongBox KeyMint |
|----------|-------------|-------------------|
| Location | Same SoC, Secure World | Separate secure element / security chip |
| Isolation | Isolated by TrustZone | Isolated by separate hardware and firmware |
| Performance | Faster | Slower, limited operation throughput |
| Key storage | Hardware-bound, TEE-protected | Stronger physical attack resistance |
| Typical use | Most app keys, FBE, auth-bound keys | Payment, identity, high-value enterprise/FIDO keys |
| Failure mode | TEE firmware compromise affects security | StrongBox compromise required separately |

🎯 **Staff-Level Insight:** StrongBox is not "faster Keystore." It is intentionally slower and more constrained. Use it only where the threat model needs physical attack resistance or stronger isolation. Blindly requesting StrongBox for every app key causes performance problems and unnecessary user-visible failures on devices without StrongBox.

---

## 7A.2 — Boot, Root of Trust, and the Security Boundary

### 7A.2.1 Security Chain at Boot

```
BootROM
  │ verifies bootloader using fused key hash
  ▼
Bootloader / ABL / U-Boot
  │ verifies vbmeta and partitions via AVB
  │ loads TEE firmware and secure monitor
  ▼
TEE OS starts
  │ initializes secure storage, crypto engines, RPMB
  │ exposes TAs to Normal World through SMC interface
  ▼
Linux kernel starts
  │ loads Android userspace
  ▼
init → keystore2 / gatekeeper / vold / system_server
```

The TEE must know the device's verified boot state because attestation certificates include it. The bootloader passes verified boot information to the TEE through vendor-specific secure channels and to Android through `androidboot.*` kernel command-line parameters.

Relevant properties:

```bash
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop ro.boot.vbmeta.device_state
adb shell getprop ro.boot.veritymode
adb shell getprop ro.boot.flash.locked
adb shell getprop ro.boot.verifiedbootstate
```

Typical values:

| Property | Good production value |
|----------|-----------------------|
| `ro.boot.verifiedbootstate` | `green` |
| `ro.boot.vbmeta.device_state` | `locked` |
| `ro.boot.veritymode` | `enforcing` |
| `ro.boot.flash.locked` | `1` |

⚠️ **OEM Pitfall:** Shipping a device that reports `green` to Android while the TEE attestation state reports unlocked or self-signed. Enterprise and banking apps verify hardware attestation, not just Android properties. The two must be consistent.

### 7A.2.2 RPMB and Replay Protection

**RPMB** (Replay Protected Memory Block) is a special eMMC/UFS region with authenticated, monotonic writes. The TEE uses it to store small security-critical state:

- Gatekeeper password handles.
- Weaver slots.
- Rollback-resistant key metadata.
- KeyMint secure deletion metadata.
- Attestation provisioning state.

Why RPMB matters: normal flash can be cloned and rolled back. RPMB uses a device-unique key and write counter, preventing an attacker from restoring old secure state after too many failed PIN attempts or after key deletion.

🐞 **Common Production Bug:** Factory flashing wipes RPMB or fails to provision the RPMB key. Symptoms appear only after lockscreen enrollment or FBE unlock: Gatekeeper returns errors, Weaver slots fail, or KeyMint rollback resistance is unavailable. Factory line must include RPMB provisioning verification.

---

## 7A.3 — Keystore2 Internals

### 7A.3.1 Why Keystore2 Exists

Pre-Android 12 Keystore had several limitations:

- C++ daemon with legacy Keymaster assumptions.
- Harder Mainline integration.
- Awkward namespacing for system components.
- Limited multi-user and grant modeling.

`keystore2` improves this:

- Written in Rust for memory safety.
- AIDL interface to clients and HALs.
- Key namespaces and grants are explicit.
- Better operation tracking and crash recovery.
- Designed for KeyMint and Remote Key Provisioning.

Source:

```text
system/security/keystore2/
├── src/
│   ├── authorization.rs
│   ├── database.rs
│   ├── operation.rs
│   ├── permission.rs
│   ├── security_level.rs
│   └── service.rs
├── aidl/android/system/keystore2/
└── tests/
```

### 7A.3.2 Keystore2 Data Model

A key has:

| Field | Meaning |
|-------|---------|
| Domain | Where the key is scoped: app, SELinux namespace, blob, grant |
| Namespace | App UID or system namespace |
| Alias | Human-readable key name |
| Key blob | Opaque wrapped key returned by KeyMint |
| Parameters | Purpose, algorithm, auth requirements, rollback resistance |
| Security level | Software, TEE, or StrongBox |
| Grants | Temporary delegated access to another UID/process |

Common domains:

| Domain | Used by |
|--------|---------|
| `APP` | Normal app keys under app UID |
| `SELINUX` | System components like `vold`, `wifi`, `bluetooth` |
| `GRANT` | Temporarily shared key access |
| `BLOB` | Raw key blob without database ownership |

### 7A.3.3 Key Operation Lifecycle

```
Client
  │ createOperation(key, purpose, params)
  ▼
keystore2
  │ checks permissions, loads key blob, chooses security level
  ▼
KeyMint HAL begin()
  │ returns operation handle
  ▼
Client streams data
  │ update() one or more times
  ▼
finish()
  │ KeyMint signs/encrypts/decrypts/MACs inside secure hardware
  ▼
Result returned to client
```

A key operation is stateful. `keystore2` tracks active operations to avoid resource leaks and to enforce per-UID limits. If a client dies, Binder death cleanup aborts the operation.

### 7A.3.4 Auth-Bound Keys

Auth-bound keys require recent user authentication:

```kotlin
val spec = KeyGenParameterSpec.Builder(
    "payment_key",
    KeyProperties.PURPOSE_SIGN or KeyProperties.PURPOSE_VERIFY
)
    .setDigests(KeyProperties.DIGEST_SHA256)
    .setUserAuthenticationRequired(true)
    .setUserAuthenticationParameters(
        30,
        KeyProperties.AUTH_BIOMETRIC_STRONG or KeyProperties.AUTH_DEVICE_CREDENTIAL
    )
    .build()
```

Flow:

1. User authenticates via PIN/password or strong biometric.
2. Gatekeeper / biometric HAL emits a **HardwareAuthToken**.
3. `keystore2` receives / caches the token through auth token plumbing.
4. KeyMint validates token HMAC, timestamp, user SID, and authenticator type.
5. Key operation proceeds if token satisfies the key's auth policy.

**SID (Secure User ID):** A stable secure identifier associated with the user's credential. When the credential changes, the SID changes. Keys bound to the old SID become permanently unusable.

🎯 **Staff-Level Insight:** "User changed PIN and now the app's key is invalid" is expected if the key was auth-bound to the previous SID. This is not data loss; it is the security contract.

---

## 7A.4 — KeyMint Manual

### 7A.4.1 KeyMint HAL Instances

A device may expose multiple KeyMint instances:

```text
android.hardware.security.keymint.IKeyMintDevice/default
android.hardware.security.keymint.IKeyMintDevice/strongbox
```

Check on device:

```bash
adb shell service list | grep -i keymint
adb shell lshal | grep -i keymaster      # older HIDL devices
adb shell dumpsys keystore2
```

AIDL source:

```text
hardware/interfaces/security/keymint/aidl/android/hardware/security/keymint/
├── IKeyMintDevice.aidl
├── IKeyMintOperation.aidl
├── KeyParameter.aidl
├── KeyPurpose.aidl
├── SecurityLevel.aidl
├── Tag.aidl
└── ...
```

### 7A.4.2 KeyMint Versions

| Android | HAL generation | Notes |
|---------|----------------|-------|
| Android 9–11 | Keymaster 4.x | HIDL / legacy keystore |
| Android 12 | KeyMint 1 | AIDL, keystore2 |
| Android 13 | KeyMint 2 | RKP integration improves |
| Android 14 | KeyMint 3 | More robust attestation / versioning |
| Android 15+ | KeyMint evolves | Check AIDL frozen versions in source |

Version details evolve; the source tree is authoritative. Always inspect `hardware/interfaces/security/keymint/aidl/aidl_api/` for frozen API versions.

### 7A.4.3 Key Parameters

KeyMint keys are defined by tags:

| Tag category | Examples |
|--------------|----------|
| Algorithm | AES, RSA, EC, HMAC, 3DES legacy |
| Purpose | ENCRYPT, DECRYPT, SIGN, VERIFY, AGREE_KEY, WRAP_KEY |
| Digest | SHA_2_256, SHA_2_512 |
| Padding | RSA_PSS, RSA_PKCS1, PKCS7 |
| Block mode | GCM, CBC, CTR, ECB |
| Auth | USER_AUTH_TYPE, AUTH_TIMEOUT, USER_SECURE_ID |
| Validity | ACTIVE_DATETIME, ORIGINATION_EXPIRE_DATETIME |
| Attestation | ATTESTATION_CHALLENGE, INCLUDE_UNIQUE_ID |
| Security | ROLLBACK_RESISTANCE, EARLY_BOOT_ONLY, UNLOCKED_DEVICE_REQUIRED |

### 7A.4.4 Rollback Resistance and Secure Deletion

Rollback resistance means an attacker cannot restore an old key blob after deletion or invalidation. Implementations use secure storage such as RPMB to track key versions or deletion state.

Common requirement: enterprise or payment keys must be rollback-resistant if the device claims support.

⚠️ **OEM Pitfall:** Advertising rollback resistance in KeyMint but implementing it only in normal-world storage. VTS and security reviews can catch this, and remote attestation consumers may reject the device class.

### 7A.4.5 Early-Boot-Only Keys

Early boot keys are usable only before the device is fully unlocked/booted. They support use cases such as encrypted notifications or boot-time services. Once Android marks boot complete or leaves early boot state, KeyMint refuses further operations with these keys.

---

## 7A.5 — Gatekeeper, Weaver, and Credential Verification

### 7A.5.1 Gatekeeper Responsibilities

Gatekeeper is not the lockscreen UI. It is the secure verifier for the user's credential.

```
User enters PIN
  ▼
LockSettingsService
  ▼
Gatekeeper HAL verify(userId, challenge, enrolledHandle, credential)
  ▼
Gatekeeper TA checks credential inside TEE / secure hardware
  ▼
Returns:
  - success/failure
  - retry timeout if rate-limited
  - HardwareAuthToken on success
```

A **password handle** contains metadata required for verification. It is stored by Android, but integrity is protected by Gatekeeper/TEE. The raw credential is never stored.

### 7A.5.2 HardwareAuthToken

A successful authentication returns a token conceptually like:

```text
HardwareAuthToken {
  challenge
  userId / secureUserId
  authenticatorId
  authenticatorType
  timestamp
  mac = HMAC(shared_auth_token_key, fields)
}
```

KeyMint trusts the token only if it can verify the HMAC using a key shared with Gatekeeper/biometric secure components.

### 7A.5.3 Weaver

Weaver provides hardware rate-limited slots:

- `write(slot, key, value)` — enroll a key/value.
- `read(slot, key)` — returns value if key matches, or timeout if wrong.

Wrong guesses increase timeout in secure hardware, not in Android. This prevents offline brute force if flash is cloned.

**Devices without Weaver:** Android can still use Gatekeeper rate limiting, but the security properties may differ. Strong devices use a secure element or TEE-backed Weaver-like primitive.

---

## 7A.6 — Synthetic Password and FBE Unlock Flow

### 7A.6.1 Why Synthetic Password Exists

Android does not directly derive file encryption keys from the user's PIN. A PIN has low entropy. Instead:

1. Android generates a high-entropy random **Synthetic Password (SP)**.
2. SP protects user secret material.
3. User credential + hardware factors unwrap SP.
4. SP unwraps or derives the keys needed to unlock CE storage.

Benefits:

- Changing PIN does not re-encrypt all user data.
- Multiple unlock methods can wrap the same SP.
- Hardware rate limiting prevents offline brute force.
- Enterprise escrow/recovery can be modeled safely where supported.

### 7A.6.2 Boot to User Unlock

```
Boot
  │
  ├─ DE storage unlocked automatically
  │   /data/user_de/<userId>/ available
  │   Direct Boot apps may run
  │
  └─ CE storage locked
      /data/user/<userId>/ unavailable

User enters credential
  ▼
LockSettingsService verifies with Gatekeeper / Weaver
  ▼
Synthetic Password unwrapped
  ▼
vold receives CE key material / token
  ▼
fscrypt installs per-user CE keys into kernel
  ▼
/data/user/<userId>/ becomes available
  ▼
ACTION_USER_UNLOCKED broadcast
```

Commands:

```bash
adb shell dumpsys lock_settings
adb shell dumpsys user
adb shell dumpsys vold
adb shell ls /data/user_de/0
adb shell ls /data/user/0        # requires root and unlocked CE
```

### 7A.6.3 Credential Change

When user changes PIN:

1. Old credential verifies and unwraps SP.
2. New credential is enrolled in Gatekeeper/Weaver.
3. SP is re-wrapped with the new credential path.
4. FBE file contents remain unchanged.
5. Auth-bound keys tied to old SID may be invalidated.

### 7A.6.4 Factory Reset

Factory reset destroys the metadata needed to unwrap keys. The data blocks may remain on flash temporarily, but they are cryptographically unrecoverable.

This is why Android factory reset is fast: it is a **crypto erase**, not a full disk overwrite.

---

## 7A.7 — Attestation Manual

### 7A.7.1 What Attestation Proves

Hardware key attestation is a certificate chain proving:

- Key was generated in hardware-backed KeyMint / StrongBox.
- Key parameters: algorithm, purpose, digest, auth policy.
- Device state: locked/unlocked, verified boot state.
- OS version and patch level.
- Vendor patch level / boot patch level where supported.
- Challenge supplied by remote verifier is embedded in cert.

### 7A.7.2 App-Side Example

```kotlin
val challenge = SecureRandom().generateSeed(32)
val spec = KeyGenParameterSpec.Builder(
    "attested_key",
    KeyProperties.PURPOSE_SIGN or KeyProperties.PURPOSE_VERIFY
)
    .setAlgorithmParameterSpec(ECGenParameterSpec("secp256r1"))
    .setDigests(KeyProperties.DIGEST_SHA256)
    .setAttestationChallenge(challenge)
    .build()

val kpg = KeyPairGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_EC,
    "AndroidKeyStore"
)
kpg.initialize(spec)
val kp = kpg.generateKeyPair()

val ks = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
val chain = ks.getCertificateChain("attested_key")
```

### 7A.7.3 Server-Side Verification Checklist

A verifier should check:

1. Certificate chain roots in an accepted attestation root.
2. Attestation challenge equals server nonce.
3. Key purpose and algorithm match expected policy.
4. `verifiedBootState == Verified` / green equivalent.
5. Device locked state is true.
6. OS patch level is recent enough.
7. App identity / package info is checked separately where needed.
8. Certificate validity and revocation policy are enforced.

🎯 **Staff-Level Insight:** Attestation tells you about the key and device state at key generation time. If your server requires freshness, use a new challenge and a newly generated key or an attestation flow designed for freshness. Do not treat an old attestation certificate as a live health signal.

### 7A.7.4 Remote Key Provisioning (RKP)

Traditional attestation used factory-provisioned attestation keys. RKP improves privacy and scalability by provisioning attestation certificate chains remotely through Google infrastructure.

High-level flow:

```
Device generates CSR / key material in secure hardware
  ▼
RKP service verifies device eligibility
  ▼
Device receives short-lived attestation certificates
  ▼
KeyMint uses these certs for app key attestation
```

AOSP components live under areas such as:

```text
packages/modules/RemoteKeyProvisioning/
system/security/keystore2/
hardware/interfaces/security/rkp/
```

---

## 7A.8 — Cuttlefish Limitations and Development Strategy

Cuttlefish is excellent for AOSP framework development, but it cannot emulate all hardware security properties.

| Feature | Cuttlefish behavior | Real device behavior |
|---------|--------------------|----------------------|
| TEE | Simulated / software-backed depending on build | Secure World firmware |
| StrongBox | Usually absent | Discrete secure element on supported devices |
| RPMB | Simulated | Hardware replay-protected storage |
| Attestation root | Test/dev chain | Google/OEM attestation root |
| Weaver | Simulated or absent | Hardware rate-limited slots |
| Verified boot | Test keys | Production AVB keys and fused root |

Use Cuttlefish for:

- Keystore2 framework integration.
- Permission and SELinux policy work.
- API behavior tests.
- CTS-style test development.
- Basic FBE/DE/CE behavior.

Use real hardware for:

- Attestation acceptance by production servers.
- StrongBox performance and failure modes.
- RPMB persistence and rollback tests.
- Factory provisioning validation.
- TEE firmware / TA integration.

---

## 7A.9 — Debugging Keystore and TEE Issues

### 7A.9.1 First Commands

```bash
adb shell dumpsys keystore2
adb logcat -b all -s keystore2 KeyMint Keymaster Gatekeeper Weaver vold LockSettingsService
adb shell service list | grep -Ei 'keymint|gatekeeper|weaver|keystore'
adb shell ps -A -Z | grep -Ei 'keystore|keymint|gatekeeper|weaver'
adb shell getprop | grep -Ei 'boot|verified|keymint|keystore'
```

### 7A.9.2 Common Failure Patterns

| Symptom | Likely cause | Where to look |
|---------|--------------|---------------|
| `KeyPermanentlyInvalidatedException` | Credential changed, biometric enrollment changed, SID mismatch | App logs, LockSettings, KeyMint response |
| `StrongBoxUnavailableException` | Device lacks StrongBox or HAL not declared | VINTF manifest, service list |
| Attestation rejected by server | Unlocked bootloader, stale patch level, wrong root, RKP issue | Cert chain fields, `ro.boot.*` |
| CE storage not unlocking | Gatekeeper/Weaver/SP failure | `LockSettingsService`, `vold`, Gatekeeper logs |
| Works on `userdebug`, fails on `user` | Test keys, debug properties, missing permissions, SELinux denial | SELinux, signing, privileged permissions |
| Keystore operation hangs | HAL/TEE blocked, operation leak | `dumpsys keystore2`, HAL tombstone/logs |

### 7A.9.3 SELinux Domains

Common labels:

```bash
adb shell ps -A -Z | grep keystore
adb shell ps -A -Z | grep keymint
adb shell ls -Z /data/misc/keystore
```

Policy locations:

```text
system/sepolicy/private/keystore.te
system/sepolicy/private/vold.te
system/sepolicy/private/gatekeeperd.te
system/sepolicy/vendor/           # vendor HAL policy
```

⚠️ **OEM Pitfall:** Granting a vendor HAL broad access to `/data/misc/keystore` to "fix" a failure. Vendor HALs should not read Keystore's database directly. Access must go through stable HAL or framework APIs.

### 7A.9.4 HAL Crash Debugging

```bash
adb shell ls -lt /data/tombstones/
adb pull /data/tombstones/tombstone_00 .
development/scripts/stack tombstone_00 --symbols-dir=out/target/product/<device>/symbols
adb logcat -b crash -d
```

If the crash is inside vendor TEE client libraries, you need:

- Exact build fingerprint.
- Vendor symbols under NDA, if available.
- TEE firmware version.
- Secure OS logs, if exposed through vendor debug tools.
- Repro steps on engineering hardware.

---

## 7A.10 — CTS, VTS, and Security Test Coverage

Relevant test areas:

```bash
atest CtsKeystoreTestCases
atest CtsKeystoreWycheproofTestCases
atest CtsDevicePolicyManagerTestCases
atest VtsHalKeyMintTargetTest
atest VtsHalGatekeeperTargetTest
atest VtsHalWeaverTargetTest
```

Exact module names vary by Android release; use:

```bash
atest --list-modules | grep -Ei 'keymint|keystore|gatekeeper|weaver'
```

### 7A.10.1 What Tests Prove

| Suite | Proves |
|-------|--------|
| CTS Keystore | Framework API behavior and app-visible contract |
| Wycheproof | Cryptographic edge-case correctness |
| VTS KeyMint | HAL conformance and security-level behavior |
| VTS Gatekeeper | Credential verification HAL behavior |
| VTS Weaver | Slot behavior and rate limiting contract |
| STS | Known security regression coverage |

🎯 **Staff-Level Insight:** Passing CTS/VTS proves interface conformance, not full attack resistance. Production security review must also inspect factory provisioning, key custody, RPMB behavior, rollback resistance, debug unlock paths, and server-side attestation policy.

---

## 7A.11 — OEM Bring-Up Checklist

### 7A.11.1 Early Bring-Up

- [ ] KeyMint HAL instance declared in VINTF.
- [ ] Gatekeeper HAL instance declared in VINTF.
- [ ] Weaver HAL declared if supported.
- [ ] `keystore2` starts cleanly.
- [ ] Lock credential enroll/verify works.
- [ ] CE/DE FBE unlock works across reboot.
- [ ] Attestation API returns a certificate chain.
- [ ] Device reports consistent verified boot state to Android and TEE.
- [ ] SELinux enforcing with no denials in keystore/gatekeeper/vold paths.

### 7A.11.2 Factory Provisioning

- [ ] AVB production key fused / enrolled.
- [ ] RPMB key provisioned exactly once.
- [ ] Attestation key / RKP eligibility provisioned.
- [ ] Widevine / DRM keys provisioned if product needs GMS/media.
- [ ] Device lock state verified after final flash.
- [ ] Factory reset tested after provisioning.
- [ ] No test attestation roots on production build.
- [ ] Debug unlock path disabled or controlled by signed-token workflow.

### 7A.11.3 Launch Readiness

- [ ] CTS Keystore passes.
- [ ] VTS KeyMint/Gatekeeper/Weaver passes.
- [ ] STS passes for claimed SPL.
- [ ] Hardware attestation accepted by target services.
- [ ] StrongBox behavior validated if device advertises StrongBox.
- [ ] Credential change invalidation behavior documented for OEM apps.
- [ ] FBE unlock tested for multi-user and work profile cases.
- [ ] OTA does not regress keystore database or CE unlock.

---

## 7A.12 — Real Production Failure Scenarios

### 7A.12.1 Banking Apps Reject a New Device SKU

**Symptoms:** Apps using hardware attestation reject only one SKU.

**Likely causes:**

- SKU shipped with unlocked bootloader state.
- Attestation certificate chain rooted in test key.
- Patch level in attestation does not match claimed SPL.
- RKP provisioning failed for that SKU.
- Device model / brand not allowlisted by verifier.

**Triage:**

1. Generate an attested key with a known challenge.
2. Decode the certificate extension server-side.
3. Compare fields against a passing SKU.
4. Check factory provisioning logs.
5. Validate `ro.boot.*` vs attested boot state.

### 7A.12.2 Users Lose App Keys After PIN Change

If keys were generated with `setUserAuthenticationRequired(true)` and bound to the old SID, this is expected. App should handle `KeyPermanentlyInvalidatedException` by re-enrolling credentials or regenerating key material.

OEM responsibility: system apps must not silently lose unrecoverable user data. Backup or re-enrollment flow is required.

### 7A.12.3 OTA Causes CE Unlock Failure

Potential causes:

- `vold` changed FBE policy without migration.
- Gatekeeper/Weaver TA version changed and cannot read old handles.
- RPMB state lost or mismatched after firmware update.
- SELinux denial prevents `vold` or `keystore2` access.

Mitigation: test OTA across all credential states:

| State | Must test |
|-------|-----------|
| No credential | boot and DE access |
| PIN set | unlock after OTA |
| Pattern/password set | unlock after OTA |
| Work profile | parent and profile unlock |
| Multi-user | each user's CE unlock |

---

## 7A.13 — Staff/Principal Interview Prompts

1. **Why can root not extract a hardware-backed private key?**
   - Because raw key material never leaves KeyMint/TEE/StrongBox; Android stores only wrapped blobs. Root can call APIs while user/session policy permits, but cannot export private key bytes if hardware enforces non-exportability.

2. **What is the difference between TEE and StrongBox?**
   - TEE is TrustZone on the main SoC. StrongBox is a discrete secure element with stronger physical isolation, slower operations, and stricter resource constraints.

3. **Explain PIN entry to CE storage unlock.**
   - LockSettings verifies through Gatekeeper/Weaver, unwraps Synthetic Password, coordinates with `vold`, installs fscrypt CE keys into the kernel, broadcasts user unlocked.

4. **Why does attestation include a challenge?**
   - To prove freshness and bind the certificate to the server's request, preventing replay of old attestation chains.

5. **Why is RPMB required?**
   - To store small secure state with replay protection, preventing rollback of password attempt counters, key deletion state, or secure metadata.

---

## 7A.14 — Verifying This Deep Dive

You should be able to:

1. Draw the full flow from Android Keystore API to KeyMint TA.
2. Explain Keystore2 domains, namespaces, grants, and operation lifecycle.
3. Describe Gatekeeper, Weaver, SID, HardwareAuthToken, and Synthetic Password.
4. Explain CE vs DE unlock and the exact role of `vold`.
5. Generate and reason about an attestation certificate chain.
6. Debug common keystore, Gatekeeper, Weaver, and FBE failures.
7. List CTS/VTS modules relevant to hardware-backed security.
8. Build an OEM factory provisioning checklist for hardware security.
9. Explain what Cuttlefish can and cannot prove for hardware security.

---

🔗 Cross-references:
- AVB and SELinux fundamentals: [Level 7](./level-07-security.md)
- CTS/VTS/STS execution: [Level 8](./level-08-testing-compatibility.md)
- Signing, key custody, release workflows: [Level 9](./level-09-production-release.md)
- Staff interview reasoning: [Level 10](./level-10-staff-mindset.md)

⬅️ Back to **[Level 7 — Security](./level-07-security.md)** | ➡️ **[Level 8 — Testing & Compatibility](./level-08-testing-compatibility.md)**
