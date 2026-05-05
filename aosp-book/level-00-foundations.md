# Level 0 — Foundations (Pre‑AOSP)

> *"You cannot debug Android until you can debug Linux. Android is not a Linux distribution — it is a userspace built on the Linux kernel with radically different conventions."*

This Level closes the gap between a Linux backend engineer and an Android platform engineer. If you skip it, you will misdiagnose `init` failures, SELinux denials, and property-system bugs for the rest of your career.

---

## Chapter 0.1 — Linux Kernel vs Android Userspace

### 0.1.1 What Android **Is**

Android is:

1. **The Linux kernel** (with Android-specific patches: `binder`, `ashmem`/`memfd`, `low memory killer` userspace daemon hooks, wakelocks/`epoll` PM, `sync_file`, `incremental fs`, `f2fs` tuning).
2. **A bionic-based userspace** (not glibc).
3. **A custom init system** (`/system/bin/init`, not systemd, not SysV).
4. **A managed runtime** (ART) running on top of native services.
5. **A specific security model** built on SELinux in *enforcing* mode by default, plus a per‑app UID sandbox.
6. **A Hardware Abstraction Layer (HAL)** decoupled from the framework via Binder (HIDL/AIDL).

### 0.1.2 What Android Is **Not**

- It is **not** GNU. There is no `glibc`, no `bash` (Toybox provides `sh`), no `coreutils`. Tools you take for granted (`ldd`, `strace -f`, `gdb`) exist but with caveats.
- It is **not** a typical Linux distribution. There is no package manager for native binaries on the device. Updates ship as **images** (system, vendor, boot, etc.), not packages.
- It is **not** a single-root filesystem. Android uses **partition-as-namespace**: `/system`, `/vendor`, `/product`, `/system_ext`, `/odm`, `/data`, `/apex`. Each has different ownership, build provenance, and update cadence.

### 0.1.3 The Layered Model

```
┌──────────────────────────────────────────────────────────┐
│                    Apps (APK / Java/Kotlin)              │
├──────────────────────────────────────────────────────────┤
│       Java Framework (android.*)  — system_server        │
├──────────────────────────────────────────────────────────┤
│   Native Services (surfaceflinger, audioserver, …)       │
├──────────────────────────────────────────────────────────┤
│   HAL (AIDL/HIDL)  ↔   Vendor Processes                  │
├──────────────────────────────────────────────────────────┤
│   Bionic libc / ART runtime / Binder driver              │
├──────────────────────────────────────────────────────────┤
│              Linux Kernel (+ Android patches)            │
└──────────────────────────────────────────────────────────┘
```

🎯 **Staff‑Level Insight:** The horizontal lines above are not just diagrams — they are **ABI contracts**. Project Treble (Android 8+) made the line between "framework" and "HAL/vendor" a hard, versioned interface (VINTF). Crossing it incorrectly is the #1 cause of OEM merge pain.

---

## Chapter 0.2 — `init`: The First Userspace Process

`init` (PID 1) is the heart of Android boot. Source: `system/core/init/`.

### 0.2.1 Why Android Has Its Own init

Traditional `systemd`/SysV‑init do not work because:

- Android needs **deterministic boot ordering** for tens of native services with fast startup (<5s on phones).
- Android requires **SELinux labeling at process creation time**.
- Android needs **property triggers** — start a service when a property changes.
- Android must run very early before any filesystem (raw `initramfs`) and bootstrap dynamic partitions.

### 0.2.2 The `.rc` Language

`init` reads `*.rc` files from:

- `/system/etc/init/` (framework services)
- `/vendor/etc/init/` (vendor services — HALs)
- `/odm/etc/init/`, `/product/etc/init/`, `/system_ext/etc/init/`
- `/apex/<name>/etc/`

Anatomy:

```rc
# /system/etc/init/surfaceflinger.rc
service surfaceflinger /system/bin/surfaceflinger
    class core animation
    user system
    group graphics drmrpc readproc
    capabilities SYS_NICE
    onrestart restart --only-if-running zygote
    task_profiles HighPerformance

on property:vendor.display.enable=1
    start surfaceflinger
```

Key constructs:

- `service` — defines a process.
- `class` — startup class (`core`, `main`, `late_start`, `hal`).
- `on <trigger>` — action block. Triggers: `early-init`, `init`, `late-init`, `boot`, `property:foo=bar`, `post-fs-data`.
- `setprop`, `start`, `stop`, `restart`, `chmod`, `chown`, `mkdir`, `mount`, `write` are built‑in commands.

### 0.2.3 The Boot Phase Order

1. `early-init`
2. `init`
3. `late-init` (typically triggers everything else via `trigger early-fs`, `trigger fs`, `trigger post-fs`, `trigger post-fs-data`, `trigger zygote-start`, `trigger boot`)
4. `boot`

🐞 **Common Production Bug:** A vendor HAL is started in `class core` instead of `class hal`. On a slow boot path (cold flash, first boot), the HAL races SELinux relabeling on `/data`, and the service crash‑loops. Fix: use `class hal` and add appropriate `on property:sys.boot_completed=1` gating for non‑critical work.

### 0.2.4 Reading `init` Logs

```bash
adb shell dmesg | grep init:
adb logcat -b all -d | grep -E "init|SELinux"
adb shell cat /proc/1/status   # PID 1 = init
```

🛠️ **Hands‑On (Cuttlefish):**

```bash
# After booting Cuttlefish (see Level 1)
adb shell getprop init.svc.surfaceflinger     # running
adb shell getprop ro.boottime.surfaceflinger  # ns since boot
adb shell setprop ctl.restart surfaceflinger  # ask init to restart it
```

---

## Chapter 0.3 — The Property System

The property system is Android’s **global, typed, access‑controlled key‑value store**, accessible from C, C++, Java, and shell. Source: `system/core/init/property_service.cpp`, `bionic/libc/bionic/system_properties/`.

### 0.3.1 Mental Model

- A **shared memory area** (`/dev/__properties__/`) mapped read-only into every process.
- Only `init` (via the `property_service` socket `/dev/socket/property_service`) can write.
- Property names are dot‑separated and **namespaced**: `ro.`, `persist.`, `sys.`, `vendor.`, `ctl.`, `dalvik.`, `debug.`.

| Prefix | Meaning |
|--------|---------|
| `ro.` | Read‑only after first set. Used for build constants. |
| `persist.` | Persisted to `/data/property/` across reboots. |
| `sys.` | System runtime, framework writable. |
| `vendor.` | Vendor partition properties. |
| `ctl.` | Control properties — write to `ctl.start`/`ctl.stop` to control services. |
| `debug.` | Engineer/dev knobs, often guarded by `ro.debuggable`. |

### 0.3.2 SELinux + Property ACLs

Every property is mapped to an SELinux **type** in `property_contexts` files:

```
# system/sepolicy/private/property_contexts
persist.sys.locale         u:object_r:exported_system_prop:s0 exact string
vendor.audio.              u:object_r:vendor_audio_prop:s0
```

A process can read/write only properties whose type its domain has been allowed to access. **You cannot get around this with root.** This is enforced by the kernel's SELinux LSM.

🐞 **Common Production Bug:** "My new property `vendor.foo.bar` cannot be set from my HAL — `init: sys_prop: permission denied`." Cause: missing entry in `vendor/etc/selinux/vendor_property_contexts` plus a missing `set_prop(your_hal_domain, vendor_foo_prop)` rule.

### 0.3.3 APIs

C/C++:
```cpp
#include <sys/system_properties.h>
char buf[PROP_VALUE_MAX];
__system_property_get("ro.build.version.sdk", buf);
// Modern API:
#include <android-base/properties.h>
auto sdk = android::base::GetIntProperty("ro.build.version.sdk", 0);
android::base::SetProperty("persist.example.flag", "1");
```

Java:
```java
android.os.SystemProperties.get("ro.build.id");
android.os.SystemProperties.set("persist.example.flag", "1"); // hidden API
```

Shell: `getprop`, `setprop`, `watchprops`.

🎯 **Staff‑Level Insight:** Avoid `persist.` properties for high‑frequency writes — they hit `/data/property/` synchronously. Use `DeviceConfig` or app `SharedPreferences` instead. A common foot‑gun is using `persist.` for telemetry counters and burning UFS write cycles.

---

## Chapter 0.4 — Namespaces, cgroups, and the Android Sandbox

Android's app sandbox is built on standard Linux primitives:

| Primitive | Used For |
|-----------|----------|
| **UID per app** | Filesystem isolation (`/data/data/<pkg>` is owned by the app's UID) |
| **`mount` namespaces** | Per‑app storage views (`/storage/emulated/0`), Bionic linker namespaces |
| **`pid` namespace** | Not heavily used; framework relies on UID |
| **`net` namespace** | VPN apps, tethering offload, per‑app firewalling (Android 10+ `INetd`) |
| **cgroups v1 + v2** | CPU scheduling (`task_profiles`), freezer (cached app freezer Android 12+), memcg |
| **seccomp‑bpf** | Per‑process syscall filters (zygote applies) |
| **Capabilities** | Drop CAP_SYS_ADMIN etc. on most services |

### 0.4.1 Linker Namespaces (bionic)

Bionic's dynamic linker (`/system/bin/linker64`) supports **namespaces**: a process can only `dlopen()` libraries from a whitelisted set of paths. This enforces the **system / vendor / APEX** library boundary.

Configured via `/system/etc/ld.config.<version>.txt` (and `/linkerconfig` generated at boot).

```
# Excerpt of ld.config.txt
[system]
namespace.default.search.paths = /system/${LIB}:/system_ext/${LIB}
namespace.default.permitted.paths = /system/${LIB}
namespace.default.links = vndk,art
namespace.vndk.search.paths = /apex/com.android.vndk.v32/${LIB}
```

🐞 **Common Production Bug:** A vendor binary tries to `dlopen("/system/lib64/libfoo.so")` and gets `dlopen failed: library "libfoo.so" not found`. Reason: vendor processes run in the `vendor` linker namespace which cannot see arbitrary `/system/lib`. Fix: either add the lib to LL‑NDK / VNDK or move it into `/vendor/lib64` or an APEX.

### 0.4.2 cgroups & `task_profiles`

Defined in `system/core/libprocessgroup/profiles/task_profiles.json`. The framework moves apps between `top‑app`, `foreground`, `background`, `system‑background` cgroups based on lifecycle. The **freezer** (cgroup v2) suspends cached apps to save battery.

🔗 See **Level 6 — Performance & Memory** for deep dives on LMKD, freezer, and PSI.

---

## Chapter 0.5 — SELinux Fundamentals (Level 0 Primer)

This is a primer; full treatment is in **Level 7 — Security**.

### 0.5.1 Why SELinux

DAC (Unix permissions) cannot stop a compromised root process. **MAC** (Mandatory Access Control) labels every object (file, socket, property, binder service) with a type, and policy declares what each domain (process type) may do.

> Android has run SELinux **enforcing for everything** since Android 5.0 (Lollipop). Permissive mode in production is a **launch blocker** for CTS.

### 0.5.2 The Vocabulary

- **Subject** — a domain (process). E.g., `untrusted_app`, `system_server`, `vendor.audio_hal`.
- **Object** — anything labeled. Files have types like `system_file`, `vendor_file`, `shell_data_file`. Properties have `*_prop` types. Binder services have `*_service` types.
- **Class** — type of object (`file`, `dir`, `chr_file`, `binder`, `property_service`, `service_manager`).
- **Permission** — verb (`read`, `write`, `ioctl`, `find`, `add`, `call`).
- **Allow rule** — `allow <domain> <type>:<class> { perms };`
- **Neverallow** — compile‑time assertion; CTS enforces these.

### 0.5.3 Reading a Denial

```
avc: denied { read } for pid=1234 comm="my_hal" name="config.bin"
  dev="dm-2" ino=99 scontext=u:r:vendor_my_hal:s0
  tcontext=u:object_r:vendor_configs_file:s0 tclass=file permissive=0
```

Decode:
- *Domain* `vendor_my_hal` wanted to *read* a *file* labeled `vendor_configs_file`.
- It was denied (`permissive=0` = enforcing).
- Fix: add `allow vendor_my_hal vendor_configs_file:file r_file_perms;` (or relabel the file, depending on intent).

### 0.5.4 The `audit2allow` Trap

`audit2allow` generates rules from denials. **Never paste its output blindly into policy.** It will:
- Allow over‑broad permissions.
- Hide real bugs (a denial often indicates a *misconfigured file*, not a missing rule).
- Trigger neverallow violations.

🎯 **Staff‑Level Insight:** Treat every denial as a design question: *"Should this domain access this object at all? Or should I relabel, refactor, or move data?"*

---

## Chapter 0.6 — Android vs Embedded Linux: Where the Roads Diverge

If you come from Yocto/Buildroot/OpenEmbedded:

| Topic | Embedded Linux | Android |
|-------|---------------|---------|
| Package mgmt | `opkg`/`rpm`/`apt` | None on device; image-based OTA |
| Init | `systemd` / SysV / BusyBox | Custom `init` + `.rc` |
| libc | glibc / musl | bionic |
| Service IPC | D-Bus, sockets | Binder + ServiceManager |
| Update | rootfs swap, RAUC, swupdate | A/B + dm-verity + AVB |
| HAL | direct `/dev` access from app | HAL process behind Binder |
| Apps | native binaries | APK + ART + Zygote |
| Filesystem | single rootfs (often) | Partitioned (system/vendor/data/…) |
| Build | bitbake / make | Soong (`Android.bp`) + Make legacy |
| Versioning | per-package | per-image, per-partition (VINTF) |

⚠️ **OEM Pitfall:** Teams transitioning from Yocto often try to "just `apt-get` install a tool on the device." There is no `apt`. Either build the binary into the system image (`PRODUCT_PACKAGES`) or push it for debugging only via `adb push` to `/data/local/tmp` (never `/system` — verity will reject it).

---

## Chapter 0.7 — Toolchain & Build System Preview

You will use:

- **Soong** — Android's build system, configured by `Android.bp` files (Blueprint syntax, JSON-like).
- **Make** — legacy `Android.mk` files; being phased out but still present.
- **Bazel** — being introduced for parts of the platform; not yet primary.
- **Kati** — converts `Android.mk` → Ninja so Soong can drive it.
- **Ninja** — actual build executor.
- **`m`, `mm`, `mma`, `mmm`** — wrapper commands set up by `build/envsetup.sh`.

🔗 Full coverage in **Level 1, Chapter 1.4 — The Build System**.

---

## Chapter 0.8 — Verifying Your Foundations

Before moving to Level 1, you should be able to answer:

1. *Why does Android run its own init instead of systemd?* (deterministic, SELinux‑aware, property triggers, early init)
2. *What is the difference between `ro.`, `persist.`, and `sys.` properties?*
3. *What is a linker namespace and why does it matter for vendor code?*
4. *Read this denial and tell me what to fix:* (use 0.5.3 example)
5. *What is the difference between system, vendor, and product partitions, and why do they exist?* (Project Treble, modular updates)
6. *Why is there no `apt` on Android?*

If any of these are fuzzy, re-read the relevant chapter. **Level 1 assumes you have absorbed all of Level 0.**

---

➡️ Continue to **[Level 1 — AOSP Basics](./level-01-aosp-basics.md)**

