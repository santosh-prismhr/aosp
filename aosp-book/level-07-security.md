# Level 7 — Security (Staff Level)

> *"Security on Android is not a feature. It is a property of the entire stack — bootloader, kernel, SELinux, framework, app sandbox, keystore, attestation — and it holds only as long as the weakest link holds. Staff engineers think in attack trees, not checklists."*

This Level teaches the Android security model from the silicon up: **Verified Boot**, **SELinux**, **the permission and UID model**, **Keystore and the TEE**, and the day-to-day craft of writing policy, debugging denials, and reasoning about threat models on shipping products.

---

## Chapter 7.1 — The Android Security Model in One Picture

```
┌──────────────────────────────────────────────────────────────────────┐
│  Hardware Root of Trust (eFuses, ROM, OTP keys)                      │
│   └─► Bootloader (verifies next stage with public key in ROM)        │
│        └─► AVB (vbmeta → boot, system, vendor, product, system_ext)  │
│             └─► Kernel (dm-verity for /system, /vendor, /product)    │
│                  └─► init + SELinux policy load (in enforcing)       │
│                       └─► Zygote → app sandbox (UID, seccomp, SELinux)│
│                            └─► Keystore2 ↔ Keymint TA (in TEE)       │
└──────────────────────────────────────────────────────────────────────┘
```

Every layer **measures and verifies** the next, and **constrains** what runs on top. Compromise at layer N gives the attacker layer N+1's privilege — never more, if the model holds.

🎯 **Staff‑Level Insight:** When asked "is X secure?" the answer is always **"against what threat model?"** A debugger attached over USB defeats most defenses; a remote attacker over Wi-Fi has a very different reach. State the threat model before the mitigation.

---

## Chapter 7.2 — Verified Boot (AVB 2.0)

### 7.2.1 What AVB Guarantees

**Android Verified Boot 2.0** guarantees that:

1. The bootloader runs only code signed by the OEM root key burned in eFuse / OTP.
2. Each partition is hashed and the hash tree is signed; tampering is detected at read time (dm-verity) or boot time (hash descriptor).
3. **Rollback protection** prevents booting an older (vulnerable) image even if signed.

It does **not** guarantee that the running code is unmodifiable at runtime — that's the job of SELinux + kernel hardening.

### 7.2.2 The vbmeta Chain

```
vbmeta.img
  ├─ hash descriptor for boot.img        (small partitions: full hash)
  ├─ hashtree descriptor for system.img  (large partitions: dm-verity merkle tree)
  ├─ hashtree descriptor for vendor.img
  ├─ chain partition descriptor → vbmeta_system.img (signed by Google for GSI)
  └─ rollback index location: 0  (per-slot rollback counter)
```

Inspect on a device:

```bash
adb shell avbctl get-verity            # returns enforcing / disabled
adb shell avbctl get-verification      # returns enforcing / disabled
# On host:
out/host/linux-x86/bin/avbtool info_image --image out/.../vbmeta.img
```

### 7.2.3 Boot States (the colored screen)

| State | Meaning | Screen |
|-------|---------|--------|
| GREEN | Verified with OEM key | none |
| YELLOW | Verified with user-installed key | yellow warning, 5 s |
| ORANGE | Bootloader unlocked | orange warning, 5 s |
| RED | Verification failed | red, halt |

The state is bound to attestation: a YELLOW/ORANGE device cannot pass **hardware-backed key attestation**, which is how Play Integrity and many enterprise apps detect tampering.

### 7.2.4 Rolling Your Own Keys (Cuttlefish)

🛠️ **Hands‑On:**

```bash
# Generate a 4096-bit RSA AVB key
out/host/linux-x86/bin/avbtool generate_test_image \
    --key external/avb/test/data/testkey_rsa4096.pem \
    --algorithm SHA256_RSA4096 ...

# Sign vbmeta with that key
avbtool make_vbmeta_image \
  --output vbmeta.img \
  --key my_avb_key.pem --algorithm SHA256_RSA4096 \
  --include_descriptors_from_image boot.img \
  --include_descriptors_from_image system.img \
  --rollback_index 0
```

⚠️ **OEM Pitfall:** Reusing the same AVB key across SKUs and dev/prod. A leaked dev key compromises every shipped SKU that trusted it. Always: separate keys per product family, separate dev/prod, HSM-backed signing for prod.

🐞 **Common Production Bug:** OEM ships an OTA whose vbmeta `rollback_index` was forgotten at the prior value. Devices that already received the new image refuse to roll back during recovery — bricking the recovery path. Lesson: **rollback index is monotonic and append-only**; treat it like a fuse.

---

## Chapter 7.3 — SELinux on Android

### 7.3.1 Why SELinux

DAC (Linux UID/GID) says "user X can read file Y." MAC (SELinux) says "process in domain `mediaserver` can `read` files of type `media_data_file` and nothing else." When (not if) `mediaserver` is exploited, SELinux contains the blast radius. Every major Android security CVE since 2014 has been *contained* by SELinux even when *exploited* in code.

### 7.3.2 The Anatomy of a Policy

Files live under `system/sepolicy/` and `device/<vendor>/<product>/sepolicy/`:

```
system/sepolicy/
  public/      # API surface for vendor — stable across PLATFORM_SEPOLICY_VERSION
  private/     # platform-only rules
  vendor/      # rules for vendor processes
  prebuilts/api/<ver>/  # frozen previous-platform copies (Treble)
```

Every object on Android has a **security context**:

```
u:r:system_server:s0
│ │ │              │
│ │ │              └── MLS sensitivity (almost always s0 on AOSP)
│ │ └── type / domain
│ └── role (almost always r for processes, object_r for files)
└── user (almost always u)
```

```bash
adb shell ps -Z | head
adb shell ls -Z /data/misc/
```

### 7.3.3 The Five Statements You Will Write 95% of the Time

```
# 1. Declare a new domain
type myhal, domain;
type myhal_exec, exec_type, vendor_file_type, file_type;

# 2. Make it a domain that init can transition into
init_daemon_domain(myhal)

# 3. Allow it to do its job (allow rules)
allow myhal sysfs_thermal:dir r_dir_perms;
allow myhal sysfs_thermal:file r_file_perms;

# 4. Bind to a HwBinder service
hwbinder_use(myhal)
add_hwservice(myhal, hal_thermal_hwservice)

# 5. Forbid the bad thing forever (neverallow)
neverallow myhal { proc -proc_thermal }:file no_rw_file_perms;
```

`neverallow` is **compile-time** — your build fails if any rule, in any module, violates it. This is how AOSP enforces invariants across thousands of vendors.

### 7.3.4 Macros — Read These to Read Policy

`system/sepolicy/public/te_macros` is required reading. The most-used:

| Macro | Expands to |
|-------|-----------|
| `init_daemon_domain(D)` | type_transition from init, allow init to exec, allow setexec |
| `binder_use(D)` | open `/dev/binder`, ioctl, transfer fds |
| `hwbinder_use(D)` | same for `/dev/hwbinder` |
| `binder_call(C, S)` | client C may make binder calls into server S |
| `add_service(D, svc)` | D may register `svc` with servicemanager |
| `r_file_perms` | `{ getattr open read ioctl lock map }` |
| `rw_file_perms` | adds `{ write append }` |

### 7.3.5 Reading and Writing a Real Policy

🛠️ **Hands‑On — give a vendor service access to a sysfs node:**

```text
# device/google/cuttlefish/sepolicy/vendor/mythermal.te
type mythermal, domain;
type mythermal_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(mythermal)

# label the sysfs node
# in file_contexts:
# /sys/class/thermal/thermal_zone[0-9]+/temp  u:object_r:sysfs_thermal:s0

allow mythermal sysfs_thermal:dir search;
allow mythermal sysfs_thermal:file r_file_perms;

# binder
hwbinder_use(mythermal)
add_hwservice(mythermal, hal_thermal_hwservice)

# logging
allow mythermal kmsg_device:chr_file w_file_perms;
```

Build and verify:

```bash
m selinux_policy
adb push out/.../vendor/etc/selinux/* /vendor/etc/selinux/
adb reboot
adb shell getenforce        # Enforcing
```

### 7.3.6 Debugging Denials

Denials appear in `dmesg` and `logcat`:

```
avc: denied { read } for pid=812 comm="mythermal"
   name="temp" dev="sysfs" ino=12345 scontext=u:r:mythermal:s0
   tcontext=u:object_r:sysfs:s0 tclass=file permissive=0
```

Read it like this: *"The process in domain `mythermal` was denied `read` on a `file` of type `sysfs`."*

The fix is **never** `allow mythermal sysfs:file ...` — that opens too much. Find the **specific** file's expected type, label it precisely (`file_contexts`), then `allow` to that narrow type.

`audit2allow` will *generate* an allow rule from a denial:

```bash
adb logcat -b all -d | audit2allow -p out/target/product/.../root/sepolicy
```

⚠️ **OEM Pitfall:** Pasting `audit2allow` output blindly into policy. It frequently widens types (`sysfs`, `vendor_file`, `system_file`) and instantly fails CTS `neverallow` checks or adds attack surface. **Always narrow the type first.**

🐞 **Common Production Bug:** A vendor in a hurry sets `setenforce 0` in a vendor `.rc` "for debugging" and forgets to remove it. CTS fails, security audit fails, build is rejected weeks later. Add a build-time check: `BUILD_BROKEN_TREBLE_SYSPROP_NEVERALLOW := false` and grep for `setenforce 0` in CI.

### 7.3.7 The Treble Compatibility Matrix for SELinux

Vendor policy is versioned (`PLATFORM_SEPOLICY_VERSION`). System policy can be **newer** than vendor policy by up to 5 years (Project Treble). The `prebuilts/api/<ver>` directory freezes prior policy so a new platform doesn't break old vendor images.

🎯 **Staff‑Level Insight:** When you add a new type used at the system/vendor boundary, it must go in `public/`, not `private/`. Otherwise old vendor images cannot reference it and Treble breaks.

---

## Chapter 7.4 — The UID and Permission Model

### 7.4.1 UIDs Are the Sandbox

Every installed app gets a fresh UID at install time (range `10000–19999` for the first user; multi-user adds `100000 * userId`). The kernel enforces filesystem isolation between UIDs; SELinux refines it. Two apps signed by the same certificate can share a UID via `android:sharedUserId` (deprecated since API 29 — do not use in new code).

```bash
adb shell ps -A -o USER,PID,NAME | grep -i com.foo
adb shell dumpsys package com.foo | grep userId=
adb shell ls -lZ /data/data/com.foo
```

### 7.4.2 Permission Tiers

| Tier | Examples | Granted by |
|------|----------|------------|
| **normal** | INTERNET, VIBRATE | install-time, automatic |
| **dangerous** (runtime) | CAMERA, RECORD_AUDIO, READ_CONTACTS | user prompt |
| **signature** | BIND_VPN_SERVICE | only same-cert apps |
| **signatureOrSystem** (legacy) | many | system apps in `/system/priv-app` |
| **role-based** (Android 10+) | DIALER, ASSISTANT | RoleManager |
| **privileged** | INSTALL_PACKAGES | priv-app in priv-app-permissions allowlist |

Privileged apps **must** be allowlisted in `/system/etc/permissions/privapp-permissions-*.xml` (or `/vendor/etc/...`). Otherwise the system disables them at boot. This is enforced by CTS.

### 7.4.3 Reading the Permission Tree

```bash
adb shell dumpsys package com.foo | sed -n '/requested permissions/,/install permissions/p'
adb shell pm list permissions -g -d   # all dangerous, grouped
```

🐞 **Common Production Bug:** OEM bundles a "weather widget" pre-installed and grants it `LOCATION` via `default-permissions.xml` to skip the user prompt. Privacy audit catches it. Lesson: **never auto-grant runtime permissions** to non-essential apps; users must consent.

### 7.4.4 App Signing — v1, v2, v3, v4

| Scheme | Year | Mechanism | Notes |
|--------|------|-----------|-------|
| v1 (JAR) | pre-N | per-file in META-INF | vulnerable to Janus-style attacks; deprecated |
| v2 | N | whole-APK signing block | mandatory minimum since API 30 for new uploads |
| v3 | P | adds key rotation lineage | required to rotate signing keys |
| v4 | R | streaming over fs-verity | enables Incremental Install |

OEM platform keys (`platform`, `media`, `shared`, `networkstack`) live under `build/target/product/security/`. **Every vendor must replace these for production.** The default keys in AOSP are publicly known and any device shipping with them is trivially rootable.

⚠️ **OEM Pitfall:** Forgetting to re-sign **APEX modules** with production keys. APEX is signed independently; default keys come from `system/apex/apexer/testdata/`. CTS catches this only for some modules.

---

## Chapter 7.5 — Keystore and the TEE

### 7.5.1 Architecture

```
   App                              system_server
    │                                    │
    │ Keystore2 AIDL                     │
    ▼                                    ▼
 ┌────────────────────────────────────────────┐
 │ keystore2 service (system/security/keystore2) │
 └───────────────┬───────────────────────────┘
                 │ HwBinder (AIDL HAL)
                 ▼
        ┌────────────────────┐         REE  (Normal World)
        │ KeyMint HAL        │
        └────────┬───────────┘
                 │ SMC / mailbox
═════════════════│═══════════════════════════ TEE boundary
                 ▼                            (Secure World)
        ┌────────────────────┐
        │ KeyMint TA (Trusty / QSEE / TEE-OS) │
        │  • root-of-trust bound to verified boot │
        │  • per-device HBK derives keys      │
        │  • attestation signing key (Google-rooted)│
        └────────────────────┘
```

Properties of TEE-resident keys:

- **Never extractable.** App can use, cannot copy.
- **Bound to verified-boot state.** A relocked or tampered device produces different attestation output.
- **Auth-bound.** Optional `setUserAuthenticationRequired(true)` makes use require recent biometric/PIN.
- **StrongBox** (Pixel Titan M, Samsung S.E.) — the keys live in a separate secure element, not just TEE.

### 7.5.2 Key Attestation

```kotlin
val spec = KeyGenParameterSpec.Builder("k", PURPOSE_SIGN)
    .setAttestationChallenge(serverNonce)
    .setDigests(DIGEST_SHA256)
    .build()
KeyPairGenerator.getInstance("EC", "AndroidKeyStore").run {
    initialize(spec); generateKeyPair()
}
val chain = ks.getCertificateChain("k")  // X.509 chain rooted in Google attestation root
```

The chain proves to a remote server: *this key lives in genuine Google-attested KeyMint, on a device with verified-boot state X, OS patch level Y, locked/unlocked Z*. This is the foundation of Play Integrity, banking app integrity, and FIDO2.

🎯 **Staff‑Level Insight:** The attestation root is the only thing that distinguishes a genuine Pixel from a userdebug emulator that *says* it is one. Protect the device's **attestation key provisioning** with the same care as the AVB root key.

---

## Chapter 7.6 — Kernel Hardening (What Staff Engineers Should Know)

A non-exhaustive list of mitigations that Android requires and you must keep enabled:

- **CONFIG_USER_NS=n on Android** (intentional — namespaces are an attack surface).
- **Seccomp-bpf** filters in zygote for app processes (`bionic/libc/seccomp/`).
- **CONFIG_BPF_JIT_ALWAYS_ON, CONFIG_STRICT_KERNEL_RWX, CONFIG_RANDOMIZE_BASE.**
- **CFI** (Control-Flow Integrity) — kernel and userspace, Clang LTO.
- **ShadowCallStack** (arm64 userspace).
- **MTE** (Memory Tagging Extension) on supported SoCs — async for app, sync for system\_server in debug builds.
- **PAN/PXN, KASAN/KCSAN** in dev builds.

🐞 **Common Production Bug:** A vendor disables `CONFIG_STRICT_KERNEL_RWX` to load a proprietary out-of-tree ko at runtime. This destroys kernel W^X. CTS **vts-kernel** catches it; release blocked.

---

## Chapter 7.7 — Encryption at Rest

### 7.7.1 FBE — File-Based Encryption

Since Android 10, **FBE is mandatory**. Each user has two key classes:

- **CE (Credential Encrypted)** — unlocked when user enters PIN/pattern/password. `/data/user/<id>/`.
- **DE (Device Encrypted)** — available at boot, before user unlock. `/data/user_de/<id>/`. Limited data only (alarms, telephony pre-unlock).

Keys are derived in KeyMint from user credential + hardware-bound key. Without the credential, ciphertext is unrecoverable even with TEE access (synthetic password binding).

### 7.7.2 Metadata Encryption

The `/data` partition itself uses **metadata encryption** (`dm-default-key` / `fscrypt v2`) to protect filenames and metadata even before user unlock. Key wrapped by Keymint.

```bash
adb shell dumpsys vold
adb shell sm list-volumes
```

---

## Chapter 7.8 — Updatable Security Components (Mainline)

Since Android 10, security-critical modules ship as **APEX** or **APK-in-APEX**, updated via Play independently of the OS:

- `com.android.conscrypt` — TLS stack
- `com.android.media`, `com.android.mediaprovider`
- `com.android.tethering` (network stack)
- `com.android.permission` (permission controller)
- `com.android.cellbroadcast`

Implication: **a 0-day in the TLS stack is patched in days, not months**. As a Staff engineer, you must ensure your product participates in Mainline (you ship Play Services and don't fork these modules) or you accept the patching liability yourself.

---

## Chapter 7.9 — Real Production Security Scenarios

### 7.9.1 "We need a vendor backdoor for diagnostics"

You will be asked. The answer is: **no shared secret, no static credential, no `setenforce 0`**. Build a **signed-command** flow: diagnostic tool sends command signed by an OEM key; on-device service in its own SELinux domain verifies signature, executes a *narrow* action, logs to tamper-evident log. The signing key lives in an HSM. Revocation via on-device CRL fetched at boot.

### 7.9.2 "Field devices are rooting themselves via a vulnerable kernel driver"

1. Capture the exploit (kernel panic logs, EL2 traces if available).
2. Identify the driver — usually a vendor display/camera/modem ko with an `ioctl` taking a userspace pointer without `copy_from_user` bounds check.
3. Hotfix: tighten SELinux to deny the offending `ioctl` (`allow … chr_file ioctl` with `ioctlcmd` filter).
4. Permanent fix: kernel patch + Mainline-style update if driver is in an updatable module; otherwise OTA.
5. Burn the rollback fuse so old vulnerable images cannot re-flash.

### 7.9.3 "Our IVI sometimes boots in YELLOW state after factory rework"

The factory tool re-flashed `vbmeta` with a developer key. AVB sees a non-OEM key → YELLOW. Fix: factory station must use the prod signing service (HSM); engineering tool fingerprint must never reach the line. Add a **factory final-test gate**: read AVB state via `avbctl` and fail the unit if not GREEN with prod key fingerprint.

---

## Chapter 7.10 — Verifying Level 7

You should be able to:

1. Draw the verified-boot chain and explain rollback protection.
2. Read an `avc: denied` log and propose a *narrow* fix.
3. Write a `.te` file for a new vendor service from scratch, including macro usage.
4. Distinguish CE vs DE storage and explain when each is available during boot.
5. Explain how key attestation proves device integrity to a remote server.
6. List five kernel hardening flags that must remain enabled and why.
7. Critique a proposed "diagnostic backdoor" design using threat-model reasoning.

---

➡️ Continue to **[Level 8 — Testing & Compatibility](./level-08-testing-compatibility.md)**

