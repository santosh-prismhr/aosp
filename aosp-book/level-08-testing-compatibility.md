# Level 8 — Testing & Compatibility

> *"Compatibility is not a milestone. It is the contract that lets a billion devices run the same APK. Every line of vendor code you ship is implicitly promising to keep that contract."*

This Level covers the testing universe an Android product must pass to ship: **CTS, VTS, GTS, CTS-Verifier, MTS, STS**, plus the day-to-day craft of writing tests, running them on Cuttlefish, and debugging the failures that always appear two weeks before launch.

---

## Chapter 8.1 — The Test Suite Map

```
┌──────────────────────────────────────────────────────────────────┐
│ CTS  Compatibility Test Suite     — Java/Kotlin API + behavior   │
│ VTS  Vendor Test Suite            — HAL + kernel + Treble        │
│ GTS  Google Test Suite (closed)   — GMS / Play certification     │
│ CTS-Verifier                      — manual tests requiring human │
│ MTS  Mainline Test Suite          — APEX modules (per-module CTS)│
│ STS  Security Test Suite          — known-CVE regressions        │
│ BTS  Build Test Suite (internal)  — sanity for CI                │
└──────────────────────────────────────────────────────────────────┘
```

Pass requirements for shipping:

| Product type | Required |
|--------------|----------|
| AOSP-only device (no Play) | none mandatory; CTS strongly advised for app compat |
| GMS phone/tablet | CTS + VTS + GTS + STS + CTS-Verifier + MTS |
| Android Automotive (with Google Auto) | all of the above + AAOS-specific subsets |
| Android TV | CTS + GTS + TV-specific |
| Wear OS | CTS + GTS + Wear-specific |

🎯 **Staff‑Level Insight:** "We pass CTS" is not the same as "we are compatible." CTS samples behavior; field bugs live in the un-sampled space. Treat CTS as the **floor**, not the **ceiling**.

---

## Chapter 8.2 — CTS — How It Actually Works

### 8.2.1 Architecture

CTS is built on **Tradefed (Trade Federation)** — an AOSP test harness in `tools/tradefederation/`. Tests are JUnit / Java but coordinated by Tradefed which:

- Selects target devices (parallel sharding across N devices).
- Pushes the test APK + test data.
- Runs `am instrument` and collects results over adb.
- Aggregates into a **report** (HTML + XML) at `android-cts/results/`.

Source layout:

```
cts/
  apps/             # CTS helper apps installed on device
  hostsidetests/    # tests that drive the device from host (Tradefed)
  tests/            # device-side instrumentation tests
  tools/            # cts-tradefed launcher
  common/           # shared utilities
```

### 8.2.2 Downloading and Running

```bash
# Download CTS for the matching API level (Android 14 example)
# https://source.android.com/compatibility/cts/downloads

unzip android-cts-14_r1-linux_x86-arm.zip
cd android-cts/tools

# Cuttlefish must be running with userdebug build matching the API
./cts-tradefed
> run cts                              # full run (~12-24h on real device)
> run cts --module CtsAccelerationTestCases
> run cts --shard-count 4              # parallel across 4 devices
> retry --retry 0                      # only failed tests of session 0
> list results
```

Reports land in `android-cts/results/<timestamp>/`:

- `test_result.html` — human view
- `test_result.xml` — machine view, must be uploaded for certification

### 8.2.3 Anatomy of a CTS Test

A small representative test, `cts/tests/tests/os/src/.../BuildTest.java`:

```java
public class BuildTest {
  @Test
  public void testFingerprint() {
    assertNotNull(Build.FINGERPRINT);
    assertTrue("fingerprint must not contain spaces",
               !Build.FINGERPRINT.contains(" "));
    assertTrue(Build.FINGERPRINT.length() <= 91);  // CDD §3.2.2
  }
}
```

Note the **CDD reference**. Every CTS test maps to a clause in the **Compatibility Definition Document** (`https://source.android.com/compatibility/cdd`). When a test fails, the CDD clause tells you *why* the requirement exists.

### 8.2.4 Writing a Custom CTS-style Test

🛠️ **Hands‑On — host-side test for a vendor service:**

```java
// cts/hostsidetests/myvendor/src/com/foo/cts/MyVendorTest.java
@RunWith(DeviceJUnit4ClassRunner.class)
public class MyVendorTest extends BaseHostJUnit4Test {
    @Test
    public void serviceIsRunning() throws Exception {
        String out = getDevice().executeShellCommand("service list");
        assertTrue("my_vendor_service should be registered",
                   out.contains("my_vendor_service"));
    }

    @Test
    public void serviceRespondsUnderLoad() throws Exception {
        for (int i = 0; i < 1000; i++) {
            CommandResult r = getDevice().executeShellV2Command(
                "cmd my_vendor_service ping");
            assertEquals(0, (int) r.getExitCode());
        }
    }
}
```

`Android.bp`:

```bp
java_test_host {
    name: "MyVendorCtsTests",
    srcs: ["src/**/*.java"],
    libs: ["tradefed", "compatibility-host-util"],
    test_suites: ["cts", "general-tests"],
}
```

Build and run:

```bash
m MyVendorCtsTests
atest MyVendorCtsTests
```

### 8.2.5 Reading a Failure

Typical CTS failure record:

```
android.security.cts.PackageSignatureTest#testPackageSignatures
  java.lang.AssertionError: Hash mismatch for com.android.providers.media
    expected:<...> but was:<...>
```

This means: a Mainline module was modified locally — the system signature no longer matches the APEX signature catalog. Fix: don't fork Mainline modules; or ship as a custom OEM build that does not claim GMS compatibility.

🐞 **Common Production Bug:** A vendor adds an extra permission to a system app's manifest "for convenience." `CtsPermissionTestCases` fails because the app now requests permissions not allowed for its location. Lesson: **never modify system apps without checking the privapp allowlist and CTS signatures**.

---

## Chapter 8.3 — VTS — Vendor Test Suite

VTS validates the **vendor side** of the Treble boundary: HALs, kernel, init scripts, SELinux, Treble compliance.

### 8.3.1 What VTS Tests

| Test family | Validates |
|-------------|-----------|
| `VtsHalXxxTargetTest` | HAL behavior conformance (per AIDL/HIDL spec) |
| `vts_kernel_*` | Kernel config, syscalls, ABI |
| `VtsTreble*` | VINTF manifests, /vendor on /system independence |
| `VtsSecurity*` | SELinux neverallow, sepolicy version |
| `VtsTrebleSysProp` | sysprop ownership (vendor cannot set system props) |

### 8.3.2 Running VTS

```bash
unzip android-vts-14_r1-linux_x86_64.zip
cd android-vts/tools
./vts-tradefed
> run vts -m VtsHalLightTargetTest
> run vts --skip-preconditions
```

VTS often uses `gtest` binaries on the device — for HALs the test calls into the HAL via the same AIDL/HIDL the framework uses, so a passing VTS proves the *interface* is correct (not necessarily that the *behavior* is fully correct beyond what the spec mandates).

### 8.3.3 Treble Compliance — VTS Fingerprint Tests

VTS includes tests that **boot a Generic System Image (GSI)** and verify the device still functions. This is the litmus test: if your vendor partition can pair with an unmodified Google GSI, you are Treble-compliant.

```bash
# Flash GSI on Cuttlefish
fastboot -w
fastboot flash system gsi_arm64-system.img
fastboot --disable-verification flash vbmeta vbmeta.img
```

⚠️ **OEM Pitfall:** Vendor adds a system property read by a vendor HAL (e.g., `ro.vendor.camera.foo` set in `system/build.prop`). GSI doesn't set this property → HAL crashes → boot loop. The fix: vendor must own all properties it depends on; place them in `vendor.prop`.

---

## Chapter 8.4 — CTS-Verifier — The Manual Suite

Some compatibility requirements cannot be automated: camera physical orientation, NFC tap, sensor calibration, biometric enrollment. **CTS-Verifier** is an APK installed on the device; a human operator follows on-screen instructions and the app records pass/fail.

```bash
adb install android-cts-verifier/CtsVerifier.apk
adb shell am start -n com.android.cts.verifier/.TestListActivity
# … run through ~80 tests, export report …
adb pull /sdcard/verifierReports
```

Always plan **2–3 engineer-days per device variant** for CTS-Verifier. It is not parallelizable.

---

## Chapter 8.5 — STS — Security Test Suite

STS validates that known CVEs are **patched on this build**. Each CVE has a regression test that exploits the issue; if the exploit succeeds, the patch is missing.

```bash
unzip android-sts-14_r1.zip
cd android-sts/tools
./sts-tradefed
> run sts-dynamic-develop
```

The **Android Security Patch Level (SPL)** in `Build.VERSION.SECURITY_PATCH` declares the date through which patches are applied. STS verifies that declaration is truthful. Faking SPL → automatic CTS failure + GMS de-listing.

🐞 **Common Production Bug:** Vendor cherry-picks 80% of an Android Security Bulletin's patches but bumps SPL to that month anyway. STS catches an unpatched kernel bug; certification rejected; release slips by 4 weeks.

---

## Chapter 8.6 — MTS — Mainline Test Suite

For each Mainline module (APEX), there is a **per-module CTS+VTS** subset. When Google publishes an update, OEMs run MTS to certify the module on their builds.

```bash
./mts-tradefed
> run mts-tethering
> run mts-permission
```

Modules updated through Play Store still require the OEM to keep MTS green on the underlying platform — Mainline does not absolve you of testing.

---

## Chapter 8.7 — Cuttlefish-Specific Test Workflow

Cuttlefish is the **canonical** CTS execution environment for AOSP development because:

- It boots fast, parallelizes (launch N instances on one host).
- Identical kernel/userspace to a real device per arch.
- CI-friendly (no physical device, no flake from cables).

🛠️ **Hands‑On — running CTS on 4 parallel Cuttlefish instances:**

```bash
# Launch 4 instances
launch_cvd --num_instances=4 \
  --cpus=4 --memory_mb=4096 \
  --report_anonymous_usage_stats=n

# Verify
adb devices
# 0.0.0.0:6520  device
# 0.0.0.0:6521  device
# 0.0.0.0:6522  device
# 0.0.0.0:6523  device

cd android-cts/tools
./cts-tradefed
> run cts --shard-count 4
```

A full CTS run that takes 18 hours on a single device finishes in ~5 hours across 4 cuttlefish instances on a 32-core workstation.

⚠️ **OEM Pitfall:** CTS results from Cuttlefish are accepted by Google **only for AOSP changes you upstream**, not for product certification. Real-device CTS is mandatory before GMS approval.

---

## Chapter 8.8 — Debugging Real CTS Failures

### 8.8.1 Triage Workflow

```
Failure
  │
  ├─ Reproducible? ── No ──► flake; tag, retry; if persistent flake report at issuetracker
  │      │
  │      Yes
  │      ▼
  ├─ Caused by local change? (revert and re-run)
  │      │
  │      Yes ──► bisect; assign to author
  │      No
  │      ▼
  ├─ AOSP upstream regression? (run on tip-of-tree)
  │      │
  │      Yes ──► file at issuetracker.google.com, attach report
  │      No
  │      ▼
  └─ Hardware/vendor-specific (won't repro on Cuttlefish)
         └─► get device logs, kernel dmesg, sepolicy denials, attach to internal tracker
```

### 8.8.2 Worked Example — `CtsCameraTestCases#testJpegOrientation` fails

1. Read the failure: assertion expected `EXIF Orientation = 6` (90°), got `1` (0°).
2. Map to CDD §7.5.5: camera HAL must rotate JPEGs to match physical sensor orientation.
3. Inspect vendor camera HAL — `getCameraInfo().orientation` returns 0 instead of 90.
4. Fix: device-specific `camera_metadata` in vendor HAL must declare `SENSOR_ORIENTATION = 90`.
5. Re-run only the affected module: `run cts -m CtsCameraTestCases -t android.hardware.cts.CameraTest#testJpegOrientation`.

### 8.8.3 Worked Example — `CtsNetTestCases` fails on Cuttlefish only

Cuttlefish networking is virtualized; some captive-portal or DHCP edge cases fail. Tradefed has a `--exclude-filter` mechanism, **but** you cannot exclude tests for a certified run. The right answer: **fix or document via a Google-approved waiver**, never silently exclude.

### 8.8.4 The "Compatibility Test Failure Report" That Actually Helps

When you escalate a failure, include:

- Test name + class + module
- CTS version (`cts/tools/version.txt`)
- Build fingerprint (`adb shell getprop ro.build.fingerprint`)
- `bugreportz` from device after failure
- Tradefed `host_log_*.txt` (excellent thread-state info)
- `dmesg` if kernel-related
- SELinux denials from `dmesg | grep avc:`

Without these, the receiving engineer files it under "needs more info" and you lose 48 h.

---

## Chapter 8.9 — Writing Your Own Test Plan

For an OEM product, you do **not** rely solely on Google's suites. You build:

```
Product Test Plan
├── Inherited: CTS, VTS, GTS, STS, MTS, CTS-Verifier
├── OEM Functional:
│    ├─ Telephony scenarios (carrier-specific)
│    ├─ Camera quality (DXO-style image tests)
│    ├─ Audio quality (3GPP TS 26.131/132)
│    ├─ AAOS scenarios (parking brake, gear, driver-attention)
│    └─ OEM apps (Settings extensions, theme, launcher)
├── Performance:
│    ├─ Boot time (5 cold-boot samples, p50 < target)
│    ├─ App launch (top 30 apps, p90 < target)
│    └─ Memory soak (72 h)
├── Power:
│    ├─ Standby drain (<X mA over 8 h)
│    └─ Active workloads (call, video)
├── Stress:
│    ├─ Monkey (10M events)
│    └─ Reboot loop (1000×)
└── Field-quality / Beta program
```

🎯 **Staff‑Level Insight:** Any test that does not run on **every CL** in CI rots. Pick a small, fast **smoke** subset (15 min) that runs on every change; full CTS nightly; full product plan weekly. Velocity dies the day developers can't see test results in their CL.

---

## Chapter 8.10 — Verifying Level 8

You should be able to:

1. Explain the role of CTS, VTS, GTS, MTS, STS — and which are required to ship GMS.
2. Run a CTS module on Cuttlefish and read the HTML report.
3. Write a host-side Tradefed test from scratch and add it to a build.
4. Map a CTS failure back to a CDD clause and propose a fix.
5. Diagnose a Treble VTS failure (system-vendor independence).
6. Build an OEM test plan that goes beyond Google's suites.

---

➡️ Continue to **[Level 9 — Production & Release Engineering](./level-09-production-release.md)**

