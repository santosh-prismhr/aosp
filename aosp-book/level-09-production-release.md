# Level 9 — Production & Release Engineering

> *"Shipping Android is not a build. It is a supply chain. Source code, build infra, signing keys, OTA servers, telemetry, support contracts — they all have to be operational on launch day, and on day 1825 (5-year automotive warranty) too."*

This Level covers the engineering disciplines that turn an AOSP tree into a product millions of people use, and that you can update safely for years afterward: **branching, merging, OTA, signing, telemetry, field debug, and the operational craft of release engineering**.

---

## Chapter 9.1 — The Lifecycle of a Shipping Android Product

```
┌──────────────────────────────────────────────────────────────────┐
│  T-18m  SoC selection, AOSP base picked (e.g. android-14.0.0_r5) │
│  T-15m  BSP bring-up, first boot                                 │
│  T-12m  Feature complete on engineering builds                   │
│  T-9m   Internal alpha; CTS green on lab devices                 │
│  T-6m   External beta; STS clean; GTS in progress                │
│  T-3m   GMS certification submission                             │
│  T-1m   Production sign-off; mass-production keys; OTA ready     │
│  T+0    Launch                                                   │
│  T+1m   First security patch OTA                                 │
│  T+12m  First major version OTA (e.g., 14 → 15)                  │
│  T+60m  End of security support (or longer for automotive)       │
└──────────────────────────────────────────────────────────────────┘
```

Every box on this timeline is owned by Release Engineering. Most teams discover this at T-3m and panic.

---

## Chapter 9.2 — Branching Strategy for AOSP Products

### 9.2.1 The Industry-Standard Layout

```
aosp-upstream                     ← read-only mirror of AOSP
  │
  ├── android-14.0.0_r5           ← AOSP release tag (frozen)
  │
oem/main                          ← integration; rebased onto next AOSP merge
  │
  ├── oem/14.0.0_r5-dev           ← active development
  │     │
  │     ├── feature/foo           ← short-lived feature branches
  │     ├── feature/bar
  │     │
  │     └── oem/14.0.0_r5-stab    ← stabilization (cherry-pick only)
  │           │
  │           └── oem/14.0.0_r5-mp ← mass production (signed builds only)
  │
  ├── oem/14.0.0_r5-mr1           ← maintenance release 1 (security patches)
  ├── oem/14.0.0_r5-mr2
  └── oem/14.0.0_r5-mr3
```

Rules:

- `mp` branches are **append-only**. Every commit is signed-off, change-ID'd, code-reviewed, and tied to a JIRA.
- Cherry-picks **only** from `dev` → `stab` → `mp`, never the other way.
- Each MR branch corresponds to one release build, with a unique `ro.build.fingerprint`.

### 9.2.2 Repo Manifests Are Your Source of Truth

A product is defined by its **manifest**, not by its tree. Lock down per-release:

```xml
<!-- manifests/oem/14.0.0_r5-mp.xml -->
<manifest>
  <remote name="aosp" fetch="https://android.googlesource.com/" />
  <remote name="oem"  fetch="ssh://git@oem.example.com/" />

  <default revision="refs/tags/android-14.0.0_r5" remote="aosp" sync-j="8" />

  <project path="device/oem/foo"   name="device/oem/foo"
           remote="oem" revision="oem/14.0.0_r5-mp" />
  <project path="vendor/oem"       name="vendor/oem"
           remote="oem" revision="oem/14.0.0_r5-mp" />
  <project path="kernel/msm-5.15"  name="kernel/msm-5.15"
           remote="oem" revision="oem/kernel/14.0.0_r5-mp" />

  <!-- explicitly pin every project that diverges from AOSP -->
  <project path="frameworks/base"  name="platform/frameworks/base"
           revision="<SHA1>" />
</manifest>
```

🎯 **Staff-Level Insight:** A manifest with floating branches (`revision="main"`) is a time bomb. **Always pin to SHAs or release tags for MP**. Otherwise re-building the exact MP image one year later — for forensic analysis or court — is impossible.

### 9.2.3 Merging a New AOSP Release

When AOSP publishes `android-14.0.0_r6`:

```bash
# 1. Sync the upstream mirror
cd ~/oem-mirror && repo sync

# 2. Create a merge branch
repo start merge-r6 --all

# 3. Per-project merge — script this
for proj in $(repo list -p); do
    cd $proj
    git fetch aosp android-14.0.0_r6
    git merge --no-ff aosp/android-14.0.0_r6 -m "Merge AOSP android-14.0.0_r6"
    cd -
done

# 4. Build, run smoke + CTS subset
m -j && atest cts-presubmit

# 5. Resolve conflicts; rebuild; full CTS in CI overnight
```

⚠️ **OEM Pitfall:** Squashing the merge to "clean up history." This breaks `git log --first-parent` accounting that auditors and security teams use to prove patches landed. **Use `--no-ff` always.**

---

## Chapter 9.3 — Build Infrastructure

### 9.3.1 What "CI" Means for AOSP

AOSP is a **150 GB checkout**, **45-minute clean build on a 32-core machine**. Naive CI bankrupts you on AWS. Patterns:

- **Workspace cache** — pre-synced AOSP tree on a fast SSD, snapshot per branch, mounted by build workers.
- **`ccache` (large, persistent, distributed)** — reduces incremental build to 5-10 min.
- **Soong's RBE (Remote Build Execution)** — distributes compile actions across a build farm. Required at scale.
- **Pre-submit vs post-submit:** pre-submit on every CL builds `aosp_cf_x86_64_phone-userdebug` only and runs a smoke test. Post-submit (per merge) builds all targets and runs CTS-subset.

### 9.3.2 The Critical Pipelines

```
┌──────────────────────────────────────────────────────────────────┐
│ Pipeline                Trigger      Duration   Gates            │
│──────────────────────────────────────────────────────────────────│
│ presubmit-fastboot      every CL     12 min     compile, lint    │
│ presubmit-cts-smoke     every CL     45 min     CTS smoke (50)   │
│ postsubmit-full         on merge     6 h        full CTS         │
│ nightly-vts             nightly      8 h        VTS + STS        │
│ nightly-soak            nightly      24 h       Monkey, mem      │
│ release-build           manual       2 h        signed, OTA pkg  │
└──────────────────────────────────────────────────────────────────┘
```

### 9.3.3 Build Reproducibility

A *truly* reproducible build means: same source SHA + same toolchain + same build host = byte-identical output. AOSP supports this; OEMs frequently break it. Common breakers:

- Embedding `$(date)` in build artifacts (`ro.build.date.utc`) — accepted, normalized.
- Embedding hostname / username (banned).
- Network access during build pulling unpinned dependencies (banned).
- Soong randomizes some orderings — pin via `BUILD_NUMBER`, `BUILD_DATETIME`.

🎯 **Staff-Level Insight:** Reproducibility is not pedantry. When a regulator (FDA, NHTSA, EU CRA) asks you to prove that the binary on the recalled device matches the source you'll publish, the only acceptable answer is a reproducible build receipt. Set this up at T-15m, not T-3m.

---

## Chapter 9.4 — Signing and Key Management

### 9.4.1 The Keys Inventory

Every Android product holds at least these keys:

| Key | Purpose | Storage |
|-----|---------|---------|
| AVB / vbmeta key | Verified Boot signing | HSM |
| Bootloader key | Bootloader signature (varies by SoC) | HSM |
| Kernel modsign key | Kernel module signing | HSM |
| OTA package key | OTA `META-INF/com/android/otacert` | HSM |
| `platform`, `media`, `shared`, `networkstack` | APK signing for system apps | HSM |
| APEX key per module | APEX file signing | HSM |
| Attestation provisioning | Per-device attestation key | provisioning service |

The AOSP defaults under `build/target/product/security/` are **publicly known**. Shipping with them is shipping a master key for your fleet.

### 9.4.2 Signing Workflow

```
build artifacts (target_files.zip, unsigned)
   │
   ▼
sign_target_files_apks  ─── HSM (PKCS#11 pkcs11-tool)
   │   --replace_ota_keys --key_mapping=...
   ▼
target_files-signed.zip
   │
   ▼
img_from_target_files  ──►  factory image (.img)
ota_from_target_files  ──►  full + incremental OTAs
```

🛠️ **Hands-On — signing on Cuttlefish (test keys, not for production):**

```bash
out/host/linux-x86/bin/sign_target_files_apks \
  --replace_ota_keys \
  --default_key_mappings build/target/product/security \
  out/dist/aosp_cf_x86_64_phone-target_files-eng.santosh.zip \
  signed-target_files.zip

out/host/linux-x86/bin/img_from_target_files signed-target_files.zip signed-img.zip
out/host/linux-x86/bin/ota_from_target_files \
  --block signed-target_files.zip ota.zip
```

⚠️ **OEM Pitfall:** Allowing release-engineering laptops to hold private keys "for convenience." A single stolen laptop equals a compromised fleet. Production keys live **only** in HSMs accessed by signing services with audit logs and m-of-n approval.

### 9.4.3 Key Rotation

If a key is compromised — or just on regular cycle:

- **APK signing keys:** rotate via APK signature scheme v3 lineage. Old signature still valid for upgrades; new one going forward.
- **AVB key:** much harder. Bootloader trust is in eFuse; rotation often requires next-gen hardware. This is why the AVB root key is the most-protected secret in the company.
- **OTA key:** can rotate by issuing an OTA that includes the new public key in `META-INF/com/android/otacert`. **The old key must sign that transition OTA**.

---

## Chapter 9.5 — OTA Architecture

### 9.5.1 The Big Picture

```
   ┌───────────────────────────────────┐
   │ Update Server (Omaha-style)       │
   │  ─ knows fleet, A/B, throttling   │
   └────────────┬──────────────────────┘
                │ HTTPS metadata
                ▼
        ┌───────────────────┐
        │ update_engine     │ (system app: GoogleUpdater / OEM equivalent)
        │ (system/update_engine) │
        └─────────┬─────────┘
                  │
                  ▼ writes to inactive slot
        ┌───────────────────┐
        │ A/B partitions    │ boot_a / boot_b, system_a / system_b, ...
        └─────────┬─────────┘
                  │ next reboot
                  ▼
            bootloader picks higher-priority slot
                  │
                  ▼
        boot_completed → mark slot successful (post_install verification)
```

### 9.5.2 A/B vs Non-A/B

| Aspect | A/B | Non-A/B (Recovery) |
|--------|-----|---------------------|
| Disk overhead | ~+3-5 GB (duplicated partitions) | none |
| Update time, user-visible | ~0 s (only reboot) | 5-15 min in recovery |
| Rollback on failure | automatic (bootloader slot switch) | brick risk |
| Streaming update | yes | no |
| Required since | Android 11 (mandatory for new launches) | legacy only |

A/B is mandatory for GMS launches since Android 11. Automotive **requires** A/B + virtual A/B (compressed Android 11+) for reliable field updates.

### 9.5.3 Virtual A/B

Virtual A/B (VAB) puts the second slot's data into a snapshot of the first, materialized in `userdata`. This eliminates the storage cost of true A/B at the cost of complexity. Implemented via dm-snapshot + COW; merged on next reboot.

```
adb shell snapshotctl dump
adb shell update_engine_client --status
```

### 9.5.4 Incremental vs Full OTA

- **Full OTA:** complete partition images. Used at launch and when chain breaks.
- **Incremental OTA:** binary diff against a specific source build. Tiny (~50 MB vs ~2 GB). Computed by `ota_from_target_files --incremental_from`.

```bash
ota_from_target_files \
  --incremental_from PREVIOUS-target_files.zip \
  --block CURRENT-target_files.zip incr.zip
```

### 9.5.5 OTA Server Considerations

- **Throttle rollout** — never push to 100% on day 1. Stages: 1%, 5%, 25%, 100% over 7-14 days.
- **Cohort by device variant** — carrier, SKU, region; bad images touch only one cohort.
- **Halt on telemetry signal** — automated rollback when crash rate > threshold.
- **Bandwidth caps** — millions of devices simultaneously downloading 2 GB will saturate any CDN; coordinate with networking.

🐞 **Common Production Bug:** OEM rolls out an OTA at 100% on Friday afternoon. A timezone-related bug bricks devices in a specific region overnight. By Monday morning, support queue has 50,000 tickets. Lesson: **never deploy on Fridays; always staged rollout; have a kill switch**.

### 9.5.6 The Anti-Brick Discipline

Every OTA must satisfy:

1. **Idempotence** — applying twice equals applying once.
2. **Fallback** — failure on first boot → automatic slot switch to old image.
3. **Forward-only versions** — rollback prevented by AVB rollback index.
4. **Signed end-to-end** — package signature verified before any write.
5. **Battery / power gate** — won't apply on <30% battery, won't apply unplugged on automotive.

---

## Chapter 9.6 — Telemetry, Crash Reporting, Field Debug

### 9.6.1 The Layers

```
Per-device:
  ─ tombstones (/data/tombstones/) for native crashes
  ─ ANR traces (/data/anr/) for app/system_server stalls
  ─ dropbox (system/core/dropbox) for kernel panics, system errors
  ─ statsd (frameworks/base/cmds/statsd) for atom-based telemetry
  ─ Perfetto persistent traces (slow-frame, jank, ANR)

Backend:
  ─ Privacy filter (PII redaction)
  ─ Aggregation (Crashlytics-style or in-house)
  ─ Per-build crash rate, top crashers, regressions
```

### 9.6.2 Tombstones in Practice

```bash
adb shell ls /data/tombstones/
adb pull /data/tombstones/tombstone_00 .
# Symbolize:
development/scripts/stack tombstone_00 \
   --symbols-dir=out/target/product/.../symbols
```

A symbolized stack on a stripped production image requires the **symbols directory** from the build that produced the image. Archive `out/target/product/.../symbols/` for **every released build**, indexed by `ro.build.fingerprint`.

🎯 **Staff-Level Insight:** No symbols = no field debug. Disk is cheap. Keep symbols, kernel `vmlinux`, all `.so` debug info, and the exact toolchain version, for **at least the support window of the product** (5-10 years for automotive).

### 9.6.3 statsd and Atoms

`statsd` is Android's structured telemetry system. Atoms are typed events declared in `frameworks/proto_logging/stats/atoms.proto`. Apps and platform log atoms; statsd aggregates per a config and uploads.

```bash
adb shell cmd stats print-stats
adb shell cmd stats dump-report --include-current-bucket --proto > report.pb
```

For OEM telemetry, define your atoms, ensure user consent, and respect the **GMS data submission policy** (or your equivalent) — illegal collection equals certification revocation and regulatory liability.

### 9.6.4 ANR / Native Crash Triage

```bash
adb shell dumpsys dropbox --print
# look for "system_server_anr", "system_app_anr", "system_app_native_crash"
```

A sample ANR has stack traces of every Java thread. The interesting bits:

- Main thread blocked in binder.transact → identify the server it's calling
- Lock graph in `dumpsys activity providers` for `held by`, `waiting for`
- watchdog timeout → 60 s in `system_server`, 30 s in `app`

🐞 **Common Production Bug:** A vendor `system_server` extension takes a global lock and calls a vendor HAL; HAL deadlocks on hardware → watchdog kills `system_server` → device reboots. Field signature: silent reboots every 4-6 hours under load. Fix: HAL calls **must** be on a worker thread or one-way binder.

---

## Chapter 9.7 — Real Production Failure Scenarios

### 9.7.1 "OTA bricks a subset of devices"

1. **Halt rollout immediately** at the server.
2. Pull tombstones / kernel logs from any reachable bricked device (some boot to recovery and report).
3. Bisect by partition: was it `boot`, `vendor`, `system_ext`?
4. If `vendor` — vendor blob mismatch with new kernel ABI. Rebuild OTA pinning vendor.
5. **Recovery path:** ship a "rescue" OTA over the air to non-bricked devices in the affected cohort; for bricked devices, RMA or fastboot recovery via service center.

### 9.7.2 "Boot loop on first boot after OTA"

Almost always: `dexopt` running out of disk, or `vbmeta` slot mismatch. Logs in `/cache/recovery/last_log`. Ensure A/B `data` is preserved across the reboot, not wiped.

### 9.7.3 "CTS regressed in MR2 after a security patch backport"

Backport conflicted in `frameworks/base`. Re-run CTS module-by-module to find the regression. Check upstream AOSP for **fix-up commits** that landed alongside the security patch. Always cherry-pick the *series*, not a single commit.

### 9.7.4 "Field crash spike from one carrier"

Filter telemetry by `Build.FINGERPRINT` and `gsm.operator.numeric`. Often: a carrier IMS push tickles a code path not covered in pre-launch testing. Hotfix via **carrier config update** (which is updatable at runtime via `CarrierConfig` overlay) — often faster than an OTA.

### 9.7.5 "Automotive head-unit reboots after driver leaves vehicle"

Power transition: `S2RAM` (suspend-to-RAM) timing race with VHAL. Sleep entered before VHAL flushed pending property writes; wake up sees inconsistent state. Fix: VHAL must register suspend hooks and gate suspend on completion. This is the kind of bug that is *only* found in real-vehicle ride-along testing — invest in it.

---

## Chapter 9.8 — Long-Term Maintenance

For an automotive program shipping a 2026 model, support obligations may run to **2036**:

- Security patches monthly for at least the GMS-mandated window (3 years at a minimum, 4-7 typical for automotive).
- Toolchain preservation — Clang/LLVM, Kotlin, AIDL, Soong of the original launch must be buildable a decade later. Archive **container images** of the build environment, not just toolchain tarballs.
- Source archival — every shipped image's source tree (or its manifest + git mirror) must be re-creatable.
- Personnel — the engineers who shipped it will leave. Document **runbooks**: how to build, sign, OTA, debug. Treat the runbook as a deliverable equal to code.

🎯 **Staff-Level Insight:** The defining trait of a Staff release engineer is paranoia about year 5, not year 0. Anyone can ship once. Shipping a maintainable platform is rare and valuable.

---

## Chapter 9.9 — Verifying Level 9

You should be able to:

1. Design a branching strategy for a multi-SKU Android product.
2. Set up a manifest pinned for reproducible MP builds.
3. Explain the inventory of signing keys and where each must live.
4. Build a signed OTA (full and incremental) end-to-end.
5. Diagnose a tombstone from a stripped production image using archived symbols.
6. Design an OTA rollout strategy with kill-switch.
7. Lay out a 5-year maintenance plan including toolchain and source archival.

---

➡️ Continue to **[Level 10 — Staff / Principal Mindset](./level-10-staff-mindset.md)**

