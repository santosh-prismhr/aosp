# Appendix F — 100-Day AOSP Curriculum
This portfolio maps directly to the rubric in [L10 §10.4](./level-10-staff-mindset.md) and is sufficient to walk into Senior/Staff AOSP loops at Tier-1 OEMs.

1. Signed delta OTA + apply/rollback *(Day 95)*
2. CTS-on-GSI + VTS GTests for own HAL *(Day 92)*
3. Vendor daemon with sepolicy + KeyMint attestation *(Day 87)*
4. Boot-time reduction with perfetto report *(Day 80)*
5. Custom VHAL property + Car app *(Day 73)*
6. Virtual sensor through DT → driver → HAL → app *(Day 65)*
7. Full AIDL HAL + framework consumer + UI slider *(Day 55)*
8. `IMyService` exposed via `dumpsys` *(Day 40)*
9. `system/` daemon + APK + APEX *(Day 25)*
10. Custom `init` service + sepolicy + system property *(Day 14)*

## Cumulative Capstone Portfolio (what you can show at Day 100)

---

| 100 | **Mock loop:** 4 × 45-min sessions across phases | [L10 §10.3](./level-10-staff-mindset.md) | — | Pass/no-pass with feedback |
| 99 | Interview bank sprint: drill 50 Qs | [Appx A](./appendix-a-interview-bank.md) | — | Self-grade matrix |
| 98 | System design — "Design AOSP feature X" rubric | [L10 §10.2](./level-10-staff-mindset.md) | — | 60-min mock written design |
| 97 | ADRs, RFCs, design reviews | [L10 §10.1](./level-10-staff-mindset.md) | — | 1 ADR for your VHAL prop |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 11 — Staff Mindset & Interview Sprint (Days 97–100) 🟤

| 96 | Fuzzing & sanitizers in CI (HWASan, MTE, libfuzzer) | [L9 §9.4](./level-09-production-release.md) | — | A fuzzer on parser code, MTE on |
| 95 | Apply delta + rollback drill | [L9 §9.3](./level-09-production-release.md) | `labs/ota-delta/` | Apply, reboot, verify slot flip |
| 94 | OTA: full + delta + Virtual A/B snapshot | [L9 §9.2](./level-09-production-release.md) | `labs/ota-delta/` | Generate signed delta on CF |
| 93 | Branching strategy, Mainline (APEX) release flow | [L9 §9.1](./level-09-production-release.md) | — | Diagram: monthly OTA train |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 10 — Production Release (Days 93–96) ⚫

| 92 | **Capstone:** CTS-on-GSI run + 5 VTS GTests for own HAL | L8+L8a | — | Green CTS module list + VTS report |
| 91 | Tradefed sharding, retry, golden runs | [L8a §8A.2](./level-08a-deep-dive-tradefed.md) | — | Shard CTS across 4 CF instances |
| 90 | GTS / STS / MTS / xTS distinctions | [L8 §8.3](./level-08-testing-compatibility.md) | — | Map each to its purpose & owner |
| 89 | VTS: testing the HAL surface | [L8 §8.2](./level-08-testing-compatibility.md) | — | Write a GTest VTS case for `ILight` |
| 88 | CTS architecture, modules, Tradefed harness | [L8 §8.1](./level-08-testing-compatibility.md), [L8a §8A.1](./level-08a-deep-dive-tradefed.md) | — | Run a single CTS module on CF |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 9 — Testing & Compatibility (Days 88–92) ⚫

| 87 | **Capstone:** Vendor daemon with full sepolicy + attested key | L7+L7a | `labs/sepolicy-vendor/` | Daemon proves possession via attestation |
| 86 | TEE: TrustZone, Trusty, GP TA primer | [L7a §7A.3](./level-07a-deep-dive-security-tee.md) | — | Read Trusty IPC docs; diagram |
| 85 | Key attestation chain validation | [L7a §7A.2](./level-07a-deep-dive-security-tee.md) | — | Verify attestation cert offline |
| 84 | Keystore2 + KeyMint architecture | [L7 §7.4](./level-07-security.md), [L7a §7A.1](./level-07a-deep-dive-security-tee.md) | — | Generate hardware-backed key |
| 83 | sepolicy build flow: `system/sepolicy`, vendor split | [L7 §7.2.3](./level-07-security.md) | `labs/sepolicy-vendor/` | Pass `checkneverallows` |
| 82 | SELinux deep: types, attributes, neverallow | [L7 §7.2](./level-07-security.md) | `labs/sepolicy-vendor/` | Author full `.te` for vendor daemon |
| 81 | Verified Boot (AVB 2.0) end-to-end | [L7 §7.1](./level-07-security.md) | `labs/avb-resign/` | Custom-rooted CF still boots |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 8 — Security (Days 81–87) 🔴

| 80 | **Capstone:** Cut Cuttlefish cold boot by 1 s; produce a perfetto report | L6+L6a | `labs/perfetto-sql/` | Before/after trace + write-up |
| 79 | dma-buf accounting, gralloc4, leak hunt | [L6a §6A.3](./level-06a-deep-dive-perf-tracing.md) | — | `dmabuf_dump` annotated |
| 78 | LMKD v2, PSI thresholds, zRAM tuning | [L6 §6.3](./level-06-performance-memory.md) | — | Trigger LMK; explain decision |
| 77 | simpleperf: stat, record, FlameGraph | [L6 §6.2](./level-06-performance-memory.md) | — | CPU flame for `system_server` |
| 76 | systrace deprecated → perfetto; ftrace primer | [L6a §6A.2](./level-06a-deep-dive-perf-tracing.md) | — | Custom ftrace event from kernel module |
| 75 | Perfetto: traces, SQL, custom data sources | [L6a §6A.1](./level-06a-deep-dive-perf-tracing.md) | `labs/perfetto-sql/` | Write SQL for jank frames |
| 74 | Boot timeline: `bootchart`, `bootstat`, init parallelism | [L6 §6.1](./level-06-performance-memory.md) | — | Cut 500 ms off CF boot |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 7 — Performance & Memory (Days 74–80) 🔴

| 73 | **Capstone:** Custom VHAL prop + Car app reads it | L5+L5a | `labs/vhal-prop/` | `CarPropertyManager.getProperty(...)` returns live value |
| 72 | Rotary controller, focus traversal | [L5 §5.6](./level-05-aaos.md) | — | Rotary HMI demo app |
| 71 | SOME/IP, vsomeip, signal bridges | [L5a §5A.3](./level-05a-deep-dive-aaos-internals.md) | — | Subscribe a SOME/IP event into VHAL |
| 70 | CarAudio: zones, focus, ducking | [L5 §5.5](./level-05-aaos.md) | — | Per-zone routing config |
| 69 | EVS (Exterior View System), camera priority | [L5 §5.4](./level-05-aaos.md), [L5a §5A.2](./level-05a-deep-dive-aaos-internals.md) | — | Reverse-gear simulated → EVS on |
| 68 | Multi-display, multi-user, headless mode | [L5 §5.3](./level-05-aaos.md) | — | Cluster + center display routed |
| 67 | VHAL: AIDL VehicleHAL, properties model | [L5 §5.2](./level-05-aaos.md), [L5a §5A.1](./level-05a-deep-dive-aaos-internals.md) | `labs/vhal-prop/` | Add `CUSTOM_OEM_PROP` |
| 66 | AAOS architecture, CarService, Car APIs | [L5 §5.1](./level-05-aaos.md) | — | Switch to `aosp_cf_x86_64_auto`; boot |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 6 — AAOS (Days 66–73) 🟡

| 65 | **Capstone:** Add a virtual sensor end-to-end (DT + driver + HAL) | L4+L4a+L3a | `labs/dts-virt-sensor/` + `labs/sensors-stub/` | Sensor visible in `adb shell sensorservice` |
| 64 | Cuttlefish internals (`crosvm`, virtio HALs) | [L4 §4.6](./level-04-bsp-bringup.md) | — | `cvd` flag tour; multi-instance |
| 63 | Bring-up loop: serial console, fastboot, EDL | [L4 §4.5](./level-04-bsp-bringup.md) | — | Recover a "soft-bricked" CF instance |
| 62 | Vendor blobs, `vendor.img`, ODM partition | [L4 §4.4](./level-04-bsp-bringup.md) | — | `extract-files.sh` walkthrough |
| 61 | Kernel module — `hello.ko` | [L0a §0A.7](./level-00a-deep-dive-kernel.md) | `labs/kmod-hello/` | `insmod hello.ko` → `dmesg` greeting |
| 60 | Device Tree (`.dts/.dtsi`), overlays | [L4 §4.3](./level-04-bsp-bringup.md) | `labs/dts-virt-sensor/` | DT overlay for a virtual I²C sensor |
| 59 | GKI / KMI: kernel ABI freeze, vendor modules | [L4 §4.2](./level-04-bsp-bringup.md) | — | Build a GKI kernel; load OEM vendor module |
| 58 | AVB 2.0: vbmeta chain, hashtree | [L7 §7.1](./level-07-security.md), [L4a §4A.3](./level-04a-deep-dive-bootloader-kernel.md) | `labs/avb-resign/` | Resign `boot.img` with custom AVB key |
| 57 | Partitions: A/B, Virtual A/B, dynamic, super | [L4a §4A.2](./level-04a-deep-dive-bootloader-kernel.md) | — | `lpdump` on CF |
| 56 | Boot chain: BL1/2/PBL/ABL/U-Boot vs ABL | [L4a §4A.1](./level-04a-deep-dive-bootloader-kernel.md) | — | Diagram annotated for a Pixel-class SoC |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 5 — BSP & Bring-up (Days 56–65) 🟠

| 55 | **Capstone:** Ship `ILight` HAL + framework manager + app slider | L3+L3a | `labs/hal-light-aidl/` | App slider changes a virtual LED via full stack |
| 54 | NetworkStack module, IpClient, DHCPv6, ConnectivityService | [L3b §3B.3](./level-03b-deep-dive-connectivity.md) | — | IpClient trace of CF DHCP |
| 53 | Wi-Fi & `wificond`, Bluetooth & `bluedroid`/Gabeldorsche | [L3b §3B.1–3B.2](./level-03b-deep-dive-connectivity.md) | — | `wificond` `cmd` queries |
| 52 | NNAPI / Neural Networks HAL | [L3a §3A.6](./level-03a-deep-dive-hal-subsystems.md) | — | Run sample model on CPU vendor driver |
| 51 | GNSS AIDL HAL | [L3a §3A.4](./level-03a-deep-dive-hal-subsystems.md) | — | NMEA injector → `LocationManager` fix |
| 50 | Sensors AIDL HAL | [L3a §3A.3](./level-03a-deep-dive-hal-subsystems.md) | `labs/sensors-stub/` | Custom `SENSOR_TYPE_GYROSCOPE` reports |
| 49 | Audio HAL: tinyalsa, audio_policy_configuration.xml | [L3a §3A.2](./level-03a-deep-dive-hal-subsystems.md) | — | Add a virtual output device |
| 48 | Camera HAL3 architecture (CamX/Chi optional) | [L3a §3A.1](./level-03a-deep-dive-hal-subsystems.md) | `labs/camera-stub/` | Stub captures one synthetic frame |
| 47 | HIDL legacy & migration | [L3 §3.7](./level-03-hal-native.md) | — | Identify HIDL leftovers in CF |
| 46 | Framework consumer: SystemService → HAL | [L3 §3.6](./level-03-hal-native.md) | `labs/hal-light-aidl/` | `LightManagerService` calls our HAL |
| 45 | HAL service init.rc, sepolicy contexts | [L3 §3.5](./level-03-hal-native.md), [L7 §7.3](./level-07-security.md) | `labs/hal-light-aidl/` | HAL boots, policy clean |
| 44 | AIDL HAL impl in C++ (`ndk::SharedRefBase`) | [L3 §3.4](./level-03-hal-native.md) | `labs/hal-light-aidl/` | Service registers with `AServiceManager_addService` |
| 43 | AIDL HAL: `.aidl` → `aidl_interface` Soong module | [L3 §3.3](./level-03-hal-native.md) | `labs/hal-light-aidl/` | New `ILight` AIDL interface compiled |
| 42 | VINTF: manifests, compatibility matrices, `vintf_object` | [L3 §3.2](./level-03-hal-native.md) | — | `vintf check-compat` on CF |
| 41 | Treble architecture, `vendor/` partitioning | [L3 §3.1](./level-03-hal-native.md) | — | Diagram: framework ↔ HAL ↔ kernel split |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 4 — HAL, Native & Connectivity (Days 41–55) 🟣

| 40 | **Capstone:** Add `IMyService` to `system_server`, expose via `dumpsys` | L2+L2a | `labs/myservice/` | `dumpsys myservice` returns custom output; client app calls it via AIDL |
| 39 | Hidden API enforcement, signature verifications | [L2 §2.8](./level-02-framework-internals.md) | — | Trip a hidden-API access; observe log |
| 38 | ART GC: CC, generational, large-object space | [L2b §2B.2](./level-02b-deep-dive-art-runtime.md) | — | Run `am dumpheap`, read with hprof |
| 37 | ART runtime: dex2oat, JIT, AOT, profile-guided | [L2b §2B.1](./level-02b-deep-dive-art-runtime.md) | `labs/art-pgo/` | Force `speed-profile`; inspect `.oat` |
| 36 | AudioFlinger, AudioPolicy, audio HAL bridge | [L2 §2.7](./level-02-framework-internals.md), [L3a §3A.2](./level-03a-deep-dive-hal-subsystems.md) | — | Force route to BT SCO; observe in `dumpsys audio` |
| 35 | SurfaceFlinger, BufferQueue, HWC2 | [L2 §2.6](./level-02-framework-internals.md), [L3a §3A.5](./level-03a-deep-dive-hal-subsystems.md) | — | `dumpsys SurfaceFlinger` layer dump |
| 34 | WindowManagerService + InputDispatcher | [L2 §2.5](./level-02-framework-internals.md) | — | Trace a touch event end-to-end via perfetto |
| 33 | PackageManagerService (PMS), scanning, install flow | [L2 §2.4](./level-02-framework-internals.md) | — | `pm install` traced via `dumpsys package` |
| 32 | ActivityManagerService (AMS) lifecycle, broadcasts | [L2 §2.3](./level-02-framework-internals.md) | — | `dumpsys activity` annotated; force-trigger an LMK |
| 31 | RPC Binder (vsock, sockets) | [L2a §2A.5](./level-02a-deep-dive-binder-native.md) | — | RPC ping over vsock between CF & host |
| 30 | FMQ (Fast Message Queue), shared mem, ashmem | [L2a §2A.4](./level-02a-deep-dive-binder-native.md) | — | FMQ producer/consumer demo |
| 29 | `libbinder_ndk` (NDK Binder, AIDL stable) | [L2a §2A.3](./level-02a-deep-dive-binder-native.md) | — | Pure-NDK client of `IPower` |
| 28 | Binder native: `libbinder`, `BBinder`, `BpBinder` | [L2a §2A.2](./level-02a-deep-dive-binder-native.md) | — | Read & explain `IServiceManager.cpp` |
| 27 | `ServiceManager`, AIDL Java interfaces | [L2 §2.2](./level-02-framework-internals.md) | — | `service list` annotated |
| 26 | `Zygote`, `system_server` start, fork model | [L2 §2.1](./level-02-framework-internals.md) | — | Diagram: zygote→app process |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 3 — Framework Internals (Days 26–40) 🔵

| 25 | **Capstone:** Add a `system/` daemon + APK + APEX, all built by `m` | L1+L1a | `labs/hello-suite/` | Single `m` builds all three; appears in image |
| 24 | Adding an APK: `android_app`, signing configs | [L1 §1.6](./level-01-aosp-basics.md) | `labs/hello-apk/` | APK installed to `/system/app/HelloMyCo` |
| 23 | SDK snapshots, `module_sdk`, prebuilts | [L1a §1A.5](./level-01a-deep-dive-build-system.md) | — | Generate a stub SDK snapshot for own module |
| 22 | Product config: `PRODUCT_*` vars, makefile inheritance | [L1 §1.5](./level-01-aosp-basics.md) | — | New product `aosp_cf_x86_64_phone_myco` |
| 21 | Kati→Ninja pipeline, `out/` layout, `m showcommands` | [L1a §1A.4](./level-01a-deep-dive-build-system.md) | — | Trace a single `.cpp` from `bp` to ninja edge |
| 20 | Bazel/`b`: migration status, `BUILD.bazel` syntax | [L1a §1A.3](./level-01a-deep-dive-build-system.md) | — | Build the same module via `b build` |
| 19 | `m`, `mm`, `mma`, `mmm`; build artifacts; ccache hits | [L1 §1.4](./level-01-aosp-basics.md) | — | Time `mm` cold vs warm; ccache stats |
| 18 | Soong advanced: `cc_library_shared`, `java_library`, `apex` | [L1a §1A.2](./level-01a-deep-dive-build-system.md) | `labs/hello-bp/` | `.apex` bundle that drops in via `apex_set` |
| 17 | Soong/`Android.bp` — modules, defaults, visibility | [L1a §1A.1](./level-01a-deep-dive-build-system.md) | `labs/hello-bp/` | A `cc_binary` on `/system/bin` |
| 16 | `repo` deep dive: manifests, branches, local manifests | [L1 §1.2](./level-01-aosp-basics.md) | — | Pin a project to a fork via `local_manifests/` |
| 15 | Source tree map (`frameworks/`, `system/`, `hardware/`, `vendor/`) | [L1 §1.1](./level-01-aosp-basics.md) | — | Annotated tree diagram |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 2 — AOSP Basics & Build System (Days 15–25) 🔵

| 14 | **Capstone:** Custom init service + sepolicy + property | L0 + L7 | `labs/init-myhello/` | Service starts on boot, denies cleanly without sepolicy, allows with it |
| 13 | Binder kernel driver (mmap'd ring, transactions) | [L2a §2A.1](./level-02a-deep-dive-binder-native.md) | — | Hand-trace `BC_TRANSACTION → BR_REPLY` |
| 12 | Memory: zRAM, lmkd v2, dma-buf, ION/gralloc | [L0a §0A.5](./level-00a-deep-dive-kernel.md), [L6 §6.3](./level-06-performance-memory.md) | — | `dumpsys meminfo` annotated by region |
| 11 | Filesystems: ext4, f2fs, erofs, fs-verity | [L0a §0A.6](./level-00a-deep-dive-kernel.md) | — | `mount` table on CF; identify each partition |
| 10 | SELinux primer, MAC vs DAC, contexts | [L7 §7.2](./level-07-security.md) | `labs/sepolicy-vendor/` | Read `/dev/null` denial → write the allow rule |
| 9 | logd, logcat buffers, tombstoned, dropbox | [L0 §0.5](./level-00-foundations.md) | — | Capture a tombstone from a forced SIGSEGV |
| 8 | Bionic libc internals: linker, TLS, fortify | [L0 §0.4](./level-00-foundations.md) | — | Strace a hello-world; identify `linker64` calls |
| 7 | System properties (persistent, ro, ctl); `property_service` | [L0 §0.3](./level-00-foundations.md) | — | `getprop`/`setprop` walkthrough; sepolicy for a custom prop |
| 6 | `init` language: services, triggers, actions | [L0 §0.2](./level-00-foundations.md) | `labs/init-myhello/` | Custom `init.myhello.rc` service runs at `late-init` |
| 5 | Android boot chain (BL → kernel → init) | [L0 §0.1](./level-00-foundations.md), [L4a §4A.1](./level-04a-deep-dive-bootloader-kernel.md) | — | Annotated `dmesg` from CF boot |
| 4 | Linux process model, fork/exec, namespaces, cgroups v2 | [L0a §0A.1–0A.3](./level-00a-deep-dive-kernel.md) | — | Diagram: PID/UID/mount namespaces in Android |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 1 — Linux & Android Foundations (Days 4–14) 🟢

| 3 | **First build:** `lunch aosp_cf_x86_64_phone-userdebug` + `m`; launch CF | [L1 §1.3](./level-01-aosp-basics.md), [L4 §4.6](./level-04-bsp-bringup.md) | — | Cuttlefish boots to home screen |
| 2 | `repo init`, manifests, partial sync, ccache 100 GB | [L1 §1.2](./level-01-aosp-basics.md) | `labs/devcontainer/` | `aosp/` synced at tag `android-15.0.0_r*` |
| 1 | Host setup, Ubuntu/WSL2, packages, JDK | [Appx G §1](./appendix-g-tooling-and-devcontainer.md) | `labs/devcontainer/` | `repo --version`, `javac -version`, 100 GB free |
|---|---|---|---|---|
| Day | Topic | Anchor | Lab | Deliverable |

## Phase 0 — Setup (Days 1–3) 🟢

---

Legend: 🟢 Foundations · 🔵 Build/Framework · 🟣 HAL/Native · 🟠 BSP · 🟡 AAOS · 🔴 Perf/Sec · ⚫ Test/Release · 🟤 Staff

> **Setup prerequisite:** Complete [Appendix G — Tooling & Devcontainer](./appendix-g-tooling-and-devcontainer.md) before Day 1.
>
> **Primary target:** Android 15 on Cuttlefish (`aosp_cf_x86_64_phone-userdebug`). Switch to `aosp_cf_x86_64_auto-userdebug` for Phase 6 (AAOS).
>
> **How to use:** One row = one day = ~3–5 hours of focused work (1h theory, 2–4h hands-on). Skip nothing in **bold**; they are gating capstones. Every day links to (a) the chapter section, (b) the lab folder, and (c) the deliverable you should be able to demo.


