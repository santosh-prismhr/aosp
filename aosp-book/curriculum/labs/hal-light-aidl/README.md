# Lab — `hal-light-aidl`
- `init` user `system` works for demo; production usually uses a dedicated `light` user.
- Forgetting `<vintf_fragments>` makes the HAL register but `vintf check-compat` will fail at OTA time.
- `frozen: true` in the AIDL `Android.bp` will fail your build until you run `m android.hardware.booklight-update-api`.

## Pitfalls observed in the field

---

- App slider moves; logcat shows `setBrightness id=0 b=...`
- `dumpsys -l | grep booklight` shows your manager service
- `lshal` lists `android.hardware.booklight.IBookLight/default`
🧪 **Verifying:**

```
cf:# logcat -s BookLight:V booklight:V
cf:# cmd booklight set 0 128         # via your shell stub (optional)
cf:# service list | grep booklight   # AIDL HAL appears in service_manager
cf:# lshal | grep booklight
```bash

```
$ launch_cvd
$ m -j$(nproc)               # full image rebuild
$ m android.hardware.booklight-service android.hardware.booklight-V1-ndk
$ source build/envsetup.sh && lunch aosp_cf_x86_64_phone-userdebug
```bash

## 6. Build & verify

---

```
}
    }
            })
                override fun onStopTrackingTouch(sb: SeekBar?) {}
                override fun onStartTrackingTouch(sb: SeekBar?) {}
                    mgr.setBrightness(0, p)
                override fun onProgressChanged(sb: SeekBar?, p: Int, fromUser: Boolean) =
            object : SeekBar.OnSeekBarChangeListener {
        findViewById<SeekBar>(R.id.slider).setOnSeekBarChangeListener(
        val mgr = getSystemService("booklight") as BookLightManager  // OEM-shimmed
        super.onCreate(s); setContentView(R.layout.main)
    override fun onCreate(s: Bundle?) {
class MainActivity : AppCompatActivity() {
```kotlin

## 5. App slider (Kotlin)

---

Register from `SystemServer.startOtherServices()` (OEM tree only) or expose via a regular SystemService stub.

```
}
    }
        catch (Exception e) { Slog.w(TAG, "setBrightness", e); }
        try { mHal.setBrightness(id, level, BrightnessMode.USER); }
    public void setBrightness(int id, int level) {

    }
        mHal = IBookLight.Stub.asInterface(b);
        IBinder b = ServiceManager.waitForDeclaredService(SERVICE);
    public BookLightManagerService() {

    private final IBookLight mHal;
    private static final String SERVICE = "android.hardware.booklight.IBookLight/default";
    private static final String TAG = "BookLightMS";
public final class BookLightManagerService {

import android.util.Slog;
import android.os.ServiceManager;
import android.os.IBinder;
import android.hardware.booklight.BrightnessMode;
import android.hardware.booklight.IBookLight;

package com.android.server.booklight;
```java
**`framework/BookLightManagerService.java`** (skeleton — full SystemService wiring is per OEM)

## 4. Framework consumer

---

> 📦 **Version Note (Android 14):** Use `hwservice_contexts` for HIDL paths only; AIDL HALs use `service_contexts`. The Android 13+ split is enforced.

```
/vendor/bin/hw/android\.hardware\.booklight-service  u:object_r:booklight_exec:s0
```text
**`sepolicy/file_contexts`**

```
android.hardware.booklight.IBookLight/default u:object_r:booklight_service:s0
```text
**`sepolicy/service_contexts`**

```
# allow booklight sysfs_leds:file rw_file_perms;
# allow booklight sysfs_leds:dir r_dir_perms;
# Read /sys/class/leds for real impl (uncomment in production)

allow booklight self:capability { setuid setgid };
# Logging

hal_attribute_service(booklight, booklight_service)
init_daemon_domain(booklight)

type booklight_exec, exec_type, vendor_file_type, file_type;
type booklight, domain;
```te
**`sepolicy/booklight.te`**

## 3. SELinux policy

---

```
}
    return EXIT_FAILURE;  // joinThreadPool only returns on error
    ABinderProcess_joinThreadPool();
    LOG(INFO) << "BookLight HAL registered as " << name;
    CHECK_EQ(s, STATUS_OK) << "Failed to register " << name;
    binder_status_t s = AServiceManager_addService(svc->asBinder().get(), name.c_str());
    const std::string name = std::string(IBookLight::descriptor) + "/default";
    auto svc = ndk::SharedRefBase::make<BookLight>();
    ABinderProcess_setThreadPoolMaxThreadCount(2);
int main() {

using aidl::android::hardware::booklight::IBookLight;
using aidl::android::hardware::booklight::BookLight;

#include "BookLight.h"
#include <android/binder_process.h>
#include <android/binder_manager.h>
#include <android-base/logging.h>
```cpp
**`service/service.cpp`**

```
}  // namespace

}
    return ScopedAStatus::ok();
    *out = kCount;
ScopedAStatus BookLight::getLightCount(int32_t* out) {

}
    return ScopedAStatus::ok();
    *out = levels_[id];
    std::lock_guard<std::mutex> g(mu_);
        return ScopedAStatus::fromServiceSpecificError(-1);
    if (id < 0 || id >= kCount)
ScopedAStatus BookLight::getBrightness(int32_t id, int32_t* out) {

}
    return ScopedAStatus::ok();
    // Real driver would write /sys/class/leds/...
              << " mode=" << static_cast<int>(m);
    LOG(INFO) << "booklight id=" << id << " b=" << b
    }
        levels_[id] = b;
        std::lock_guard<std::mutex> g(mu_);
    {
        return ScopedAStatus::fromServiceSpecificError(-2);
    if (b < 0 || b > 255)
        return ScopedAStatus::fromServiceSpecificError(-1);
    if (id < 0 || id >= kCount)
ScopedAStatus BookLight::setBrightness(int32_t id, int32_t b, BrightnessMode m) {

using ndk::ScopedAStatus;

namespace aidl::android::hardware::booklight {

#include <android-base/logging.h>
#include "BookLight.h"
```cpp
**`service/BookLight.cpp`**

```
}  // namespace aidl::android::hardware::booklight

};
    std::array<int32_t, kCount> levels_{};
    std::mutex mu_;
  private:
    ::ndk::ScopedAStatus getLightCount(int32_t* out) override;
    ::ndk::ScopedAStatus getBrightness(int32_t id, int32_t* out) override;
                                       BrightnessMode mode) override;
    ::ndk::ScopedAStatus setBrightness(int32_t id, int32_t brightness,
    static constexpr int kCount = 4;
  public:
class BookLight : public BnBookLight {

namespace aidl::android::hardware::booklight {

#include <mutex>
#include <array>
#include <aidl/android/hardware/booklight/BnBookLight.h>
#pragma once
```cpp
**`service/BookLight.h`**

```
</manifest>
    </hal>
        <fqname>IBookLight/default</fqname>
        <version>1</version>
        <name>android.hardware.booklight</name>
    <hal format="aidl">
<manifest version="1.0" type="device">
```xml
**`service/booklight-service.xml`**

```
    group system
    user system
    class hal
    interface aidl android.hardware.booklight.IBookLight/default
service vendor.booklight /vendor/bin/hw/android.hardware.booklight-service
```rc
**`service/booklight-service.rc`**

```
}
    ],
        "android.hardware.booklight-V1-ndk",
        "libbase", "libbinder_ndk", "liblog",
    shared_libs: [
    srcs: ["service.cpp", "BookLight.cpp"],
    vendor: true,
    vintf_fragments: ["booklight-service.xml"],
    init_rc: ["booklight-service.rc"],
    relative_install_path: "hw",
    name: "android.hardware.booklight-service",
cc_binary {
```bp
**`service/Android.bp`**

## 2. Vendor service

---

```
}
    frozen: true,
    versions_with_info: [{ version: "1", imports: [] }],
    },
        java: { sdk_version: "module_current" },
        ndk: { enabled: true },
        cpp: { enabled: false },
    backend: {
    stability: "vintf",
    srcs: ["android/hardware/booklight/*.aidl"],
    vendor_available: true,
    name: "android.hardware.booklight",
aidl_interface {
```bp
**`aidl/Android.bp`**

```
enum BrightnessMode { USER = 0, SENSOR = 1, LOW_POWER = 2 }
@Backing(type="int")
@VintfStability

package android.hardware.booklight;
```aidl
**`aidl/android/hardware/booklight/BrightnessMode.aidl`**

```
}
    int getLightCount();
    /** Number of independently addressable lights on this device. */

    int getBrightness(int id);
    /** Return current brightness 0..255. */

    void setBrightness(int id, int brightness, in BrightnessMode mode);
    /** Set 0..255 brightness on the given light id. */
interface IBookLight {
@VintfStability

import android.hardware.booklight.BrightnessMode;

package android.hardware.booklight;
```aidl
**`aidl/android/hardware/booklight/IBookLight.aidl`**

## 1. AIDL interface

---

```
    └── src/.../MainActivity.kt
    ├── Android.bp
└── app/
│   └── BookLightManagerService.java
│   ├── Android.bp
├── framework/
│   └── hwservice_contexts
│   ├── service_contexts
│   ├── file_contexts
│   ├── booklight.te
├── sepolicy/
│   └── BookLight.h
│   ├── BookLight.cpp
│   ├── service.cpp
│   ├── booklight-service.xml             ← VINTF fragment
│   ├── booklight-service.rc
│   ├── Android.bp
├── service/
│       └── BrightnessMode.aidl
│       ├── IBookLight.aidl
│   └── android/hardware/booklight/
├── aidl/
├── README.md                              ← this file
hal-light-aidl/
```

## Layout

---

End-to-end AIDL HAL: a custom `ILights`-style interface, a vendor service implementation, init.rc, sepolicy, VINTF fragment, framework consumer SystemService, and a slider app. **Day 43–46 + 55 capstone** of the [100-day curriculum](../../appendix-f-curriculum-100-day.md).


