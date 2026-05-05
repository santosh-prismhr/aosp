# Appendix E — AOSP / AAOS Development Manual

> *"The levels teach the system. This manual teaches how to work in the system every day."*

This appendix is the operational manual for AOSP/AAOS development. It turns the book into a **single source of truth** by organizing the material around the real workflows a platform engineer performs: setting up a tree, making a framework change, adding a HAL, debugging boot, shipping AAOS features, writing SELinux, running CTS/VTS, and preparing a release.

Use this appendix when you need to answer: **"What do I do next, and where in the book do I go deeper?"**

---

## E.1 — The AOSP Engineer Workflow Map

```
Requirement / Bug / Incident
        │
        ▼
Clarify layer and ownership
        │
        ├── Framework/API? ───────────────► Level 2, Appendix C Project 1
        ├── Native service? ──────────────► Level 2A, Level 6
        ├── HAL/vendor? ─────────────────► Level 3, Appendix C Project 2
        ├── BSP/kernel/boot? ────────────► Level 4, Level 4A
        ├── AAOS/vehicle? ───────────────► Level 5, Level 5A
        ├── Security/keys/SELinux? ──────► Level 7, Level 7A
        ├── Compatibility failure? ──────► Level 8
        └── OTA/release/field issue? ───► Level 9
        │
        ▼
Design smallest compatible change
        │
        ▼
Implement behind correct boundary
        │
        ▼
Build, flash/sync, test on Cuttlefish
        │
        ▼
Run focused tests + relevant CTS/VTS
        │
        ▼
Write debugging/rollback notes
        │
        ▼
Code review + merge + release tracking
```

🎯 **Staff-Level Insight:** The first decision is never "what code do I write?" It is **which boundary owns the behavior**: app/framework, native service, HAL, kernel, secure world, or release infra. Most expensive Android mistakes are boundary mistakes.

---

## E.2 — Environment Baseline Checklist

Before starting real work, a developer workstation should satisfy:

- [ ] Ubuntu 22.04/24.04 or known-good Linux build environment.
- [ ] KVM available for Cuttlefish.
- [ ] AOSP synced to a known tag or branch.
- [ ] `repo manifest -r` snapshot saved for reproducibility.
- [ ] `source build/envsetup.sh` works.
- [ ] `lunch aosp_cf_x86_64_phone-trunk_staging-userdebug` works for phone work.
- [ ] `lunch aosp_cf_x86_64_auto-trunk_staging-userdebug` works for AAOS work.
- [ ] Clean build passes at least once.
- [ ] Cuttlefish boots and `adb shell` works.
- [ ] `adb root && adb remount` works on userdebug.
- [ ] `atest` can run a trivial test.

Deep references:

- Setup: [Level 1](./level-01-aosp-basics.md)
- Daily commands: [Appendix B](./appendix-b-daily-reference.md)
- Build internals: [Level 1A](./level-01a-deep-dive-build-system.md)

---

## E.3 — Workflow: Make a Framework Change

Use this for changes in `frameworks/base`, `packages/modules/*`, or Java system services.

### E.3.1 Decision Contract

Before editing code, answer:

1. Is this public API, `@SystemApi`, `@hide`, or internal-only?
2. Does it affect app compatibility or target SDK behavior?
3. Does it need a permission?
4. Does it need multi-user/multi-display behavior?
5. Does it belong in `system_server`, a Mainline module, or a separate process?

### E.3.2 Implementation Steps

1. Find owner:
   ```bash
   grep -rn "class FooService" frameworks/base packages/modules
   repo info frameworks/base
   ```
2. Read existing service pattern.
3. Add API / internal method.
4. Add permission checks before privileged action.
5. Add logging only where actionable.
6. Add unit or host test.
7. Build focused modules:
   ```bash
   m services framework-minus-apex
   ```
8. Sync and restart:
   ```bash
   adb root
   adb remount
   adb sync system
   adb shell stop
   adb shell start
   ```
9. Verify with `dumpsys`, logs, and a small client.
10. Run focused `atest` / CTS module.

Deep references:

- Framework internals: [Level 2](./level-02-framework-internals.md)
- Binder/native internals: [Level 2A](./level-02a-deep-dive-binder-native.md)
- Custom service project: [Appendix C](./appendix-c-project-walkthroughs.md)

### E.3.3 Review Checklist

- [ ] No synchronous Binder call on main/UI thread.
- [ ] No lock held across Binder/HAL call.
- [ ] Permission enforced at service boundary.
- [ ] Multi-user behavior explicit (`userId`, `UserHandle`, visible background user if AAOS).
- [ ] Multi-display behavior explicit if UI/window/display involved.
- [ ] Test covers permission denial and success path.
- [ ] `dumpsys` output added for supportability if service state matters.

---

## E.4 — Workflow: Add a Native Daemon

Use this for native services under `/system/bin`, `/vendor/bin`, `/system_ext/bin`, or `/product/bin`.

### E.4.1 Correct Partition Decision

| If daemon is... | Put it in... |
|-----------------|-------------|
| AOSP/framework-owned | `/system` |
| OEM system extension | `/system_ext` |
| Product/SKU behavior | `/product` |
| SoC/vendor hardware integration | `/vendor` |
| ODM board-specific | `/odm` |

### E.4.2 Required Artifacts

- `Android.bp` module.
- `.rc` service file.
- SELinux domain and exec type.
- `file_contexts` entry.
- Optional property contexts.
- Tests or smoke command.

### E.4.3 Minimal Pattern

```blueprint
cc_binary {
    name: "oem_foo_daemon",
    vendor: true,
    srcs: ["main.cpp"],
    shared_libs: ["libbase", "liblog"],
    init_rc: ["oem_foo_daemon.rc"],
}
```

```rc
service vendor.oem_foo /vendor/bin/oem_foo_daemon
    class late_start
    user system
    group system
    oneshot
```

```text
# file_contexts
/vendor/bin/oem_foo_daemon  u:object_r:oem_foo_exec:s0
```

```te
type oem_foo, domain;
type oem_foo_exec, exec_type, vendor_file_type, file_type;
init_daemon_domain(oem_foo)
```

Deep references:

- init and services: [Level 0](./level-00-foundations.md)
- SELinux: [Level 7](./level-07-security.md)
- SELinux project: [Appendix C](./appendix-c-project-walkthroughs.md)

---

## E.5 — Workflow: Add an AIDL HAL

Use this for new vendor-facing hardware abstractions.

### E.5.1 HAL Design Rules

- Prefer **AIDL HAL** for new work.
- Mark VINTF-stable HALs with `@VintfStability`.
- Keep policy out of HAL. HAL reports capabilities/state; framework decides behavior.
- Version by adding methods or parcelable fields compatibly; do not mutate frozen versions.
- Add VTS tests at the same time as the HAL.

### E.5.2 Required Artifacts

```text
hardware/interfaces/<domain>/<hal>/aidl/
├── android/hardware/<domain>/<hal>/IYourHal.aidl
├── Android.bp                    # aidl_interface
├── default/
│   ├── service.cpp
│   ├── YourHal.cpp/.h
│   ├── Android.bp                # cc_binary
│   ├── your-hal-service.rc
│   └── your-hal-service.xml      # VINTF fragment
└── vts/
    └── VtsHalYourHalTargetTest.cpp
```

### E.5.3 Bring-Up Commands

```bash
m android.hardware.yourhal-service.default
adb root
adb remount
adb sync vendor
adb shell stop
adb shell start
adb shell service list | grep yourhal
adb shell lshal | grep yourhal
atest VtsHalYourHalTargetTest
```

Deep references:

- HAL and VINTF: [Level 3](./level-03-hal-native.md)
- HAL project: [Appendix C](./appendix-c-project-walkthroughs.md)
- VTS: [Level 8](./level-08-testing-compatibility.md)

---

## E.6 — Workflow: Debug a Boot Failure

### E.6.1 Triage by Boot Phase

| Last visible phase | Likely layer |
|--------------------|-------------|
| No UART / no bootloader | BootROM / board power / flash strap |
| Bootloader only | Kernel image, DTB, AVB |
| Kernel logs, no init | Ramdisk, kernel panic, init missing |
| init starts, no mounts | fstab, super, dm-verity, AVB |
| zygote crash loop | ART/framework libraries, SELinux, missing jars |
| system_server watchdog | service deadlock, HAL hang, permission issue |
| UI not visible | SurfaceFlinger, HWC, WMS, SystemUI |

### E.6.2 First Commands

```bash
adb wait-for-device
adb shell dmesg | tail -200
adb shell logcat -b all -d | tail -500
adb shell getprop | grep -E 'init.svc|ro.boot|vold|sys.boot'
adb shell cat /proc/cmdline
adb shell cat /proc/mounts
adb shell ls -l /data/tombstones /data/anr
```

If the device reboots repeatedly:

```bash
adb shell ls /sys/fs/pstore
adb shell cat /sys/fs/pstore/console-ramoops-0
```

Deep references:

- Boot sequence: [Level 2](./level-02-framework-internals.md)
- Bootloader/kernel: [Level 4A](./level-04a-deep-dive-bootloader-kernel.md)
- Security/AVB: [Level 7](./level-07-security.md)

---

## E.7 — Workflow: Make an AAOS Feature

### E.7.1 AAOS Design Questions

1. Is the data already represented by a standard VHAL property?
2. Is it vendor-specific? If yes, define a vendor property and document it.
3. Does it affect driver distraction rules?
4. Which user/occupant/display owns the UI?
5. Which audio zone is used?
6. Does it need to run in Garage Mode?
7. Is it safety-critical? If yes, Android may not be the right control path.

### E.7.2 Standard AAOS Implementation Paths

| Feature type | Primary path |
|--------------|--------------|
| Vehicle data read/write | VHAL → CarPropertyService → CarPropertyManager |
| Multi-display UI | DisplayManager / ActivityOptions / OccupantZone |
| Driver-safe UI | CarUxRestrictionsManager |
| Audio feature | CarAudioService + audio policy XML + Audio HAL |
| Backup camera | EVS, not Camera2 app |
| Power/garage work | CarPowerManagementService |
| Cluster nav | CarNavigationStatusManager / Cluster rendering service |

Deep references:

- AAOS basics: [Level 5](./level-05-aaos.md)
- AAOS internals: [Level 5A](./level-05a-deep-dive-aaos-internals.md)
- AAOS project: [Appendix C](./appendix-c-project-walkthroughs.md)

---

## E.8 — Workflow: Write SELinux Policy Correctly

### E.8.1 The Correct Loop

1. Label the object narrowly.
2. Create a domain if the process is new.
3. Start permissive only for that new domain in dev builds.
4. Capture denials.
5. Interpret each denial manually.
6. Add the narrowest allow rule.
7. Remove permissive.
8. Build policy and run VTS/CTS relevant tests.

### E.8.2 Anti-Patterns

- `allow mydomain vendor_file:file rw_file_perms;`
- `allow mydomain sysfs:file rw_file_perms;`
- Making `system_server` access vendor-private files directly.
- Adding `permissive` to a `user` build.
- Using `audit2allow` output without redesigning labels.

Deep references:

- SELinux basics: [Level 0](./level-00-foundations.md)
- SELinux deep dive: [Level 7](./level-07-security.md)
- Security HAL/TEE policy: [Level 7A](./level-07a-deep-dive-security-tee.md)

---

## E.9 — Workflow: Debug Performance

### E.9.1 Universal Method

```
Symptom → trace → identify resource axis → prove critical path → fix → re-trace → lock regression test
```

Do not optimize from logs alone. Use Perfetto for timing and causality.

### E.9.2 Choose the Right Trace

| Symptom | Trace/tools |
|---------|-------------|
| Boot slow | Perfetto boot trace, bootchart, `ro.boottime.*` |
| App start slow | Perfetto `am/wm/view/binder/sched`, `am start -W` |
| Jank | Perfetto `gfx/view/surfaceflinger`, `dumpsys gfxinfo` |
| Memory pressure | `dumpsys meminfo`, `procrank`, `dmabuf_dump`, PSI |
| LMKD kills | `logcat -s lmkd`, `dumpsys activity oom` |
| Power drain | `dumpsys batterystats`, wakeup_sources, Perfetto power |

Deep references:

- Performance: [Level 6](./level-06-performance-memory.md)
- Native services: [Level 2A](./level-02a-deep-dive-binder-native.md)

---

## E.10 — Workflow: CTS/VTS Failure Triage

### E.10.1 Triage Steps

1. Reproduce the exact module/test locally.
2. Check whether it fails on clean AOSP Cuttlefish.
3. If clean AOSP passes, diff your product changes.
4. Read the CDD clause behind the CTS test.
5. For VTS, inspect VINTF manifest/matrix and HAL behavior.
6. Fix root cause; do not exclude tests unless there is an approved waiver.
7. Retry only failed tests, then run the full affected module.

### E.10.2 Commands

```bash
atest CtsSomeModule
atest CtsSomeModule:SpecificTest
adb bugreportz
adb shell getprop ro.build.fingerprint
adb shell dmesg | grep avc:
adb shell lshal
adb shell cat /vendor/etc/vintf/manifest.xml
```

Deep references:

- Testing: [Level 8](./level-08-testing-compatibility.md)
- Daily reference: [Appendix B](./appendix-b-daily-reference.md)

---

## E.11 — Workflow: Prepare a Release / OTA

### E.11.1 Release Gate Checklist

- [ ] Manifest pinned to SHAs or immutable tags.
- [ ] Build reproducible from clean checkout.
- [ ] `target_files.zip` archived.
- [ ] Symbols archived.
- [ ] Kernel `vmlinux` archived.
- [ ] OTA package signed with production key through HSM path.
- [ ] AVB/vbmeta signed with production key.
- [ ] CTS/VTS/STS/MTS required suites green.
- [ ] Cuttlefish smoke passes.
- [ ] Real hardware smoke passes.
- [ ] Rollout kill switch tested.
- [ ] Rollback plan documented.
- [ ] Security patch level truthfully matches patches.

### E.11.2 OTA Commands

```bash
m dist
ota_from_target_files out/dist/<product>-target_files.zip ota_full.zip
ota_from_target_files --incremental_from prev-target_files.zip curr-target_files.zip ota_inc.zip
adb reboot sideload
adb sideload ota_full.zip
```

Deep references:

- Release engineering: [Level 9](./level-09-production-release.md)
- Security key custody: [Level 7](./level-07-security.md), [Level 7A](./level-07a-deep-dive-security-tee.md)

---

## E.12 — AOSP Change Review Template

Use this template for any non-trivial AOSP/AAOS change:

```text
Title:
  [subsystem] Short description

Problem:
  What user/product/platform problem does this solve?

Layer decision:
  Why is this in framework / HAL / vendor / kernel / AAOS service?

Compatibility:
  Public API? @SystemApi? VINTF? CDD? CTS impact?

Security:
  Permissions, SELinux, secrets, attestation, user data, privacy.

Multi-user / multi-display:
  User ID handling, profile handling, AAOS occupant/display handling.

Performance:
  Binder calls, boot path, memory, jank, power.

Testing:
  Unit, atest, CTS/VTS, manual Cuttlefish, real hardware if needed.

Rollback:
  Feature flag, config overlay, OTA rollback, safe defaults.

Debuggability:
  dumpsys, statsd atom, log tag, bugreport evidence.
```

🎯 **Staff-Level Insight:** If a CL description cannot answer these questions, the design probably is not ready for code review.

---

## E.13 — Reading Path by Role

| Role / Goal | Read in this order |
|-------------|--------------------|
| New AOSP engineer | Level 0 → Level 1 → Appendix B → Level 2 |
| Framework engineer | Level 2 → Level 2A → Level 6 → Level 8 |
| HAL engineer | Level 3 → Level 4 → Level 4A → Level 7 |
| BSP engineer | Level 0A → Level 4 → Level 4A → Level 9 |
| AAOS engineer | Level 5 → Level 5A → Level 3 → Level 8 |
| Security engineer | Level 7 → Level 7A → Level 9 → Level 8 |
| Release engineer | Level 1A → Level 8 → Level 9 → Appendix B |
| Staff interview prep | Level 10 → Appendix A → weak levels from Appendix A |

---

## E.14 — Final Principle

AOSP is too large to memorize. The goal is to build a **map**:

- Which layer owns the behavior.
- Which source directory owns the code.
- Which command proves the hypothesis.
- Which test protects the fix.
- Which release mechanism ships it safely.

This book is that map. Keep returning to it while reading the source tree — because in Android platform work, the source tree is the final authority.

⬅️ Back to **[Table of Contents](./README.md)**

