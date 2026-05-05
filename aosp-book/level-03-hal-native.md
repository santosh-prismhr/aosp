# Level 3 — HAL & Native Layer (Senior Engineer)

> *"The framework is what users see. The HAL is what shipping is. Every device launch lives or dies on HAL stability and VINTF compliance."*

---

## Chapter 3.1 — HAL Evolution: Legacy → HIDL → AIDL

### 3.1.1 Legacy HAL (pre‑Android 8)

A HAL was a `.so` shared library `libhardware/<module>.so` loaded **in-process** by the framework via `hw_get_module()`. Examples: `audio.primary.<board>.so` loaded into `audioserver`.

Problems:
- Tight ABI coupling — every framework upgrade required HAL recompilation.
- No process isolation — a HAL bug crashed the framework process.
- No versioning — silent ABI breaks.

This model is **deprecated** but still present (e.g., `gralloc` legacy, some OEM trees). New HALs **must not** use it.

### 3.1.2 HIDL (Android 8 → 12, deprecated 13+)

HIDL = HAL Interface Definition Language. Solved Treble's goals:

- HALs run **in their own process**, in the **vendor partition**.
- IPC is **hwbinder** (`/dev/hwbinder`), separate from app binder.
- Versioned interfaces: `android.hardware.audio@6.0`.
- Framework and HAL can be built independently.

Status: deprecated for new interfaces in Android 13+. Existing HIDL HALs continue to work but new development is **AIDL only**.

### 3.1.3 AIDL HAL (Android 11+, mandatory for new HALs Android 13+)

Same AIDL you use for Java services, with `stability: "vintf"`. Runs over `/dev/binder` (with separate SELinux types for vendor processes). Cleaner tooling, single language for framework and HAL.

| Feature | Legacy | HIDL | AIDL HAL |
|---------|--------|------|----------|
| Process isolation | ❌ | ✅ | ✅ |
| Versioning | ❌ | ✅ | ✅ (frozen versions) |
| Backend | C | C++/Java | C++/Java/Rust/NDK |
| Status | Deprecated | Deprecated | **Current** |
| Driver | direct | `/dev/hwbinder` | `/dev/binder` |

🎯 **Staff‑Level Insight:** Even on Android 14+, you will encounter HIDL in BSPs from 2020-era SoCs. Knowing both is non-negotiable. Many bring-up bugs are AIDL-shim-over-HIDL translations.

---

## Chapter 3.2 — Vendor vs System: The Treble Boundary

```
┌────────────────────────────────────────────────────────┐
│                  /system + /system_ext                 │
│   (framework, system_server, system apps)              │
│   Built from AOSP. Signed by OEM with system key.      │
│   Updated as one unit (OTA).                           │
└─────────────────────────┬──────────────────────────────┘
                          │  STABLE AIDL (VINTF)
                          │  ── compatibility_matrix ──
                          │  ── manifest ──
┌─────────────────────────▼──────────────────────────────┐
│              /vendor + /odm                            │
│   (HALs, vendor libs, kernel modules in /vendor/lib)   │
│   Built from SoC BSP + OEM customizations.             │
│   Versioned independently. May NOT depend on /system.  │
└────────────────────────────────────────────────────────┘
```

Rules enforced by Soong + VTS:
1. Vendor processes **must not** dlopen `/system/lib*` (linker namespaces enforce).
2. Vendor processes **must not** call non-VINTF AIDL on framework services.
3. Framework **must not** dlopen `/vendor/lib*`.
4. Vendor SELinux domains live in `device/<oem>/sepolicy/vendor/` — they cannot use `system_server` types.

### 3.2.1 Variants of `cc_library`

```blueprint
cc_library_shared {
    name: "libfoo",
    vendor: true,                 // /vendor/lib only
}
cc_library_shared {
    name: "libbar",
    vendor_available: true,       // builds twice: /system/lib AND /vendor/lib
}
cc_library_shared {
    name: "libbaz",
    system_ext_specific: true,    // /system_ext/lib
}
cc_library_shared {
    name: "libqux",
    product_specific: true,       // /product/lib
}
```

The duplicated build for `vendor_available` exists because vendor and system may use **different VNDK** versions. The vendor copy is built against the **VNDK snapshot** for that vendor’s target API level.

### 3.2.2 VNDK (Vendor Native Development Kit)

VNDK is a **frozen snapshot** of the AOSP shared libraries (e.g., `libutils`, `libbase`, `libcutils`) that vendor code can link against. It enables vendors to ship `/vendor` images compiled against an older VNDK while `/system` updates to a newer Android.

> **Version note:** VNDK is being **deprecated in Android 15+** (replaced by stable C/Rust APIs and the LL-NDK contract). Cuttlefish on `main` may not have VNDK directories. For Android 13/14 you still see them; treat VNDK as legacy when designing new code.

---

## Chapter 3.3 — VINTF: Vendor Interface Manifest & Compatibility Matrix

### 3.3.1 The Two Files

- **Compatibility Matrix** (`compatibility_matrix.<level>.xml`): the **framework's requirements** — what HALs and versions the framework expects to find.
- **Manifest** (`manifest.xml`): the **vendor's offer** — what HALs and versions vendor provides.

Boot-time check: `libvintf` validates that *manifest* satisfies *matrix*. If it doesn't, `vts_treble_vintf_test` fails and the device cannot pass CTS.

Files live in:
```
/system/etc/vintf/manifest.xml          # framework manifest
/system/etc/vintf/compatibility_matrix.xml
/vendor/etc/vintf/manifest.xml          # vendor manifest
/vendor/etc/vintf/compatibility_matrix.xml
```

### 3.3.2 Example — Vendor Manifest Entry

```xml
<manifest version="2.0" type="device" target-level="8">
    <hal format="aidl">
        <name>android.hardware.light</name>
        <version>1</version>
        <fqname>ILights/default</fqname>
    </hal>
    <hal format="hidl">
        <name>android.hardware.audio</name>
        <transport>hwbinder</transport>
        <version>6.0</version>
        <interface>
            <name>IDevicesFactory</name>
            <instance>default</instance>
        </interface>
    </hal>
</manifest>
```

`target-level="8"` declares the **FCM (Framework Compatibility Matrix) level** the vendor implements. Each Android release has an FCM level (Android 14 = 8, Android 15 = 9). Vendors can **lag** the framework by N levels — Treble’s whole point.

### 3.3.3 Tools

```bash
adb shell vintf                              # current device VINTF state
adb shell cat /system/etc/vintf/manifest.xml
out/host/linux-x86/bin/checkvintf <files>   # offline check
```

🐞 **Common Production Bug:** Adding a new AIDL HAL but forgetting to register it in vendor manifest. Symptoms: HAL process runs, but `IServiceManager.waitForService()` from framework returns null after 5 s, framework feature degrades silently. Fix: add to `device/<oem>/<product>/manifest.xml`.

---

## Chapter 3.4 — Writing a Production-Grade AIDL HAL

We will build `android.hardware.example.IFoo` — a vendor HAL that exposes a sensor‑like read.

### 3.4.1 Layout

```
hardware/interfaces/example/aidl/
├── android/hardware/example/
│   ├── IFoo.aidl
│   └── FooData.aidl
├── default/                       # reference impl
│   ├── Foo.cpp
│   ├── Foo.h
│   ├── service.cpp
│   ├── android.hardware.example-service.rc
│   ├── android.hardware.example-service.xml
│   └── Android.bp
└── Android.bp
```

### 3.4.2 The AIDL

```aidl
// android/hardware/example/IFoo.aidl
package android.hardware.example;
import android.hardware.example.FooData;

@VintfStability
interface IFoo {
    FooData read();
    void setSamplingRateHz(int hz);
    const int MAX_RATE_HZ = 1000;
}
```
```aidl
// android/hardware/example/FooData.aidl
package android.hardware.example;

@VintfStability
parcelable FooData {
    long timestampNs;
    float[] values;
}
```

`@VintfStability` enforces the stability contract at build time.

### 3.4.3 `Android.bp`

Top-level:
```blueprint
aidl_interface {
    name: "android.hardware.example",
    vendor_available: true,
    srcs: ["android/hardware/example/*.aidl"],
    stability: "vintf",
    backend: {
        cpp:  { enabled: false },
        ndk:  { enabled: true  },
        java: { enabled: false },
        rust: { enabled: true  },
    },
    versions_with_info: [
        { version: "1", imports: [] },
    ],
    frozen: true,
}
```

Reference impl (`default/Android.bp`):
```blueprint
cc_binary {
    name: "android.hardware.example-service.default",
    relative_install_path: "hw",
    vendor: true,
    init_rc:    ["android.hardware.example-service.rc"],
    vintf_fragments: ["android.hardware.example-service.xml"],
    srcs: ["service.cpp", "Foo.cpp"],
    shared_libs: [
        "libbase", "libbinder_ndk", "liblog", "libutils",
        "android.hardware.example-V1-ndk",
    ],
}
```

### 3.4.4 The Service Entrypoint

`service.cpp`:
```cpp
#include <android-base/logging.h>
#include <android/binder_manager.h>
#include <android/binder_process.h>
#include "Foo.h"

using aidl::android::hardware::example::Foo;

int main() {
    ABinderProcess_setThreadPoolMaxThreadCount(4);
    auto foo = ndk::SharedRefBase::make<Foo>();
    const std::string instance =
        std::string(Foo::descriptor) + "/default";
    binder_status_t status =
        AServiceManager_addService(foo->asBinder().get(), instance.c_str());
    CHECK_EQ(status, STATUS_OK) << "addService failed: " << status;
    LOG(INFO) << "example HAL ready: " << instance;
    ABinderProcess_joinThreadPool();
    return EXIT_FAILURE;  // joinThreadPool should not return
}
```

### 3.4.5 The Implementation

`Foo.h`/`Foo.cpp`:
```cpp
// Foo.h
#pragma once
#include <aidl/android/hardware/example/BnFoo.h>
#include <atomic>

namespace aidl::android::hardware::example {
class Foo : public BnFoo {
public:
    ::ndk::ScopedAStatus read(FooData* out) override;
    ::ndk::ScopedAStatus setSamplingRateHz(int32_t hz) override;
private:
    std::atomic<int32_t> mRateHz{100};
};
}
```
```cpp
// Foo.cpp
#include "Foo.h"
#include <chrono>

namespace aidl::android::hardware::example {

::ndk::ScopedAStatus Foo::read(FooData* out) {
    out->timestampNs = std::chrono::steady_clock::now().time_since_epoch().count();
    out->values = {1.0f, 2.0f, 3.0f};
    return ndk::ScopedAStatus::ok();
}

::ndk::ScopedAStatus Foo::setSamplingRateHz(int32_t hz) {
    if (hz <= 0 || hz > IFoo::MAX_RATE_HZ) {
        return ndk::ScopedAStatus::fromExceptionCodeWithMessage(
            EX_ILLEGAL_ARGUMENT, "rate out of range");
    }
    mRateHz = hz;
    return ndk::ScopedAStatus::ok();
}
}
```

### 3.4.6 init.rc and VINTF Fragment

`android.hardware.example-service.rc`:
```rc
service vendor.example-default /vendor/bin/hw/android.hardware.example-service.default
    class hal
    user system
    group system
    capabilities
```

`android.hardware.example-service.xml`:
```xml
<manifest version="1.0" type="device">
    <hal format="aidl">
        <name>android.hardware.example</name>
        <version>1</version>
        <fqname>IFoo/default</fqname>
    </hal>
</manifest>
```

### 3.4.7 SELinux Policy (Vendor)

`device/google/cuttlefish/shared/sepolicy/vendor/hal_example_default.te`:
```
type hal_example_default, domain;
type hal_example_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_example_default)
binder_use(hal_example_default)
binder_call(hal_example_default, servicemanager)
add_service(hal_example_default, hal_example_service)
```

`device/google/cuttlefish/shared/sepolicy/vendor/service.te`:
```
type hal_example_service, hal_service_type, protected_service, service_manager_type;
```

`device/google/cuttlefish/shared/sepolicy/vendor/service_contexts`:
```
android.hardware.example.IFoo/default   u:object_r:hal_example_service:s0
```

### 3.4.8 Wiring into the Product

`device/google/cuttlefish/shared/auto/device_vendor.mk` (or your product mk):
```make
PRODUCT_PACKAGES += android.hardware.example-service.default
```

### 3.4.9 Build, Deploy, Verify

```bash
m android.hardware.example-service.default
adb root && adb remount
adb push out/target/product/vsoc_x86_64/vendor/bin/hw/android.hardware.example-service.default /vendor/bin/hw/
adb shell stop && adb shell start
adb shell lshal | grep example
adb shell dumpsys -l | grep example
```

🛠️ **Hands‑On — Calling from a client:**
```cpp
auto svc = IFoo::fromBinder(ndk::SpAIBinder(
    AServiceManager_waitForService("android.hardware.example.IFoo/default")));
FooData d;
svc->read(&d);
LOG(INFO) << "ts=" << d.timestampNs;
```

---

## Chapter 3.5 — Framework ↔ HAL Integration

A typical flow when framework code wants to call a HAL:

1. **Service discovery**: `AServiceManager_waitForService` (NDK) or `IServiceManager.waitForDeclaredService` (Java).
2. **Stable AIDL proxy**: framework links against the *system* variant of the AIDL stub.
3. **Death recipient**: HAL crashes are common; framework must reconnect.
4. **Threading**: HAL calls happen on Binder threads; do not block UI thread.
5. **Versioning**: query `getInterfaceVersion()` and feature-flag accordingly.

Example: `frameworks/base/services/core/java/com/android/server/lights/LightsService.java` consumes `android.hardware.light.ILights`:

```java
ILights als = ILights.Stub.asInterface(
    ServiceManager.waitForDeclaredService(
        "android.hardware.light.ILights/default"));
als.setLightState(id, state);
```

🎯 **Staff‑Level Insight:** Always use `waitForDeclaredService` (returns null if not in VINTF manifest) over `waitForService` (blocks forever) on optional HALs. The latter is the source of countless watchdog timeouts on minimal devices.

---

## Chapter 3.6 — HAL Crash Debugging

A HAL crash manifests as:
- `vendor.example-default` not in `lshal`
- `init`: `service vendor.example-default exited; restarting`
- Tombstone in `/data/tombstones/`
- Framework feature degraded

### 3.6.1 Workflow

```bash
# 1. Find tombstone
adb shell ls -lt /data/tombstones/
adb pull /data/tombstones/tombstone_00 .

# 2. Symbolize using out/.../symbols
development/scripts/stack tombstone_00

# 3. logcat correlated
adb logcat -b crash -d
adb logcat -d | grep -E 'vendor.example|hal_example'

# 4. Crash counter (init exits 4 times in 4 s = persistent stop)
adb shell getprop init.svc.vendor.example-default
# stopped → service is in 'crashloop' state
```

### 3.6.2 The Four Failure Categories

1. **Permissions / SELinux** — `avc: denied` in `dmesg` correlates with crash.
2. **VINTF mismatch** — service starts but fails `addService` (no SELinux entry for service type).
3. **Driver / ioctl** — kernel returns `-EINVAL`; check vendor logs.
4. **Threading / lifecycle** — race during shutdown, double-destroy. Use `-fsanitize=address` builds.

🐞 **Common Production Bug:** "HAL works on `eng` but crashes on `user` build." 9/10 cases: a debug-only library or `/data/local/tmp` path is hardcoded. `user` builds strip those.

### 3.6.3 Debugging Tools

```bash
adb shell lshal --init-vintf > /tmp/manifest.xml   # generate manifest from running HALs
adb shell lshal debug android.hardware.example.IFoo/default
adb shell strace -f -p $(pidof android.hardware.example-service.default)
adb shell logwrapper /vendor/bin/hw/android.hardware.example-service.default
adb shell dumpstate -z      # full snapshot
```

For ASan/HWASan vendor builds:
```bash
SANITIZE_TARGET=address m android.hardware.example-service.default
```

---

## Chapter 3.7 — Verifying Level 3

You should be able to:

1. Explain why HIDL exists, why AIDL replaced it, and what `@VintfStability` means.
2. Write a new AIDL HAL end-to-end including SELinux and VINTF.
3. Diagnose a HAL crash from a tombstone in <15 minutes.
4. Read a `compatibility_matrix.xml` and explain target FCM level.
5. Describe linker namespace boundaries and why vendor code cannot dlopen `/system/lib`.

---

➡️ Continue to **[Level 4 — BSP & Device Bring‑Up](./level-04-bsp-bringup.md)**

