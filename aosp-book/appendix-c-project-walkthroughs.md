# Appendix C — End-to-End Project Walkthroughs

> *"The difference between 'I understand AOSP' and 'I can ship AOSP' is having done a complete loop — from requirement to code to SELinux to build to CTS to OTA — at least three times."*

This appendix provides **four complete projects** that exercise every layer of the stack. Each is Cuttlefish-runnable and designed to be completed in 1–3 days.

---

## Project 1 — Custom System Service: Device Health Monitor

**Goal:** Build a `DeviceHealthService` in `system_server` that exposes battery temperature, memory pressure, and thermal state to apps via a new `DeviceHealthManager` class.

### P1.1 — Architecture

```
App
 │ DeviceHealthManager.getHealthSnapshot()
 ▼
system_server: DeviceHealthService (IDeviceHealthService.Stub)
 │ reads BatteryManager, /proc/pressure/memory, PowerManager thermal
 ▼
Returns HealthSnapshot parcelable
```

### P1.2 — AIDL

```aidl
// frameworks/base/core/java/android/health/IDeviceHealthService.aidl
package android.health;
import android.health.HealthSnapshot;

/** @hide */
interface IDeviceHealthService {
    HealthSnapshot getHealthSnapshot();
}
```

```aidl
// frameworks/base/core/java/android/health/HealthSnapshot.aidl
package android.health;
parcelable HealthSnapshot {
    float batteryTempC;
    int memoryPressurePsi;
    int thermalStatus;
    long uptimeMs;
}
```

### P1.3 — Service Implementation

```java
// frameworks/base/services/core/java/com/android/server/health/DeviceHealthService.java
package com.android.server.health;

import android.content.Context;
import android.health.HealthSnapshot;
import android.health.IDeviceHealthService;
import android.os.PowerManager;
import android.os.SystemClock;

public class DeviceHealthService extends IDeviceHealthService.Stub {
    private final Context mContext;

    public DeviceHealthService(Context context) { mContext = context; }

    @Override
    public HealthSnapshot getHealthSnapshot() {
        mContext.enforceCallingOrSelfPermission(
            "android.permission.DEVICE_HEALTH_READ", "getHealthSnapshot");
        HealthSnapshot s = new HealthSnapshot();
        s.uptimeMs = SystemClock.uptimeMillis();
        PowerManager pm = mContext.getSystemService(PowerManager.class);
        s.thermalStatus = pm.getCurrentThermalStatus();
        // Read PSI from /proc/pressure/memory (parse "some avg10=X.XX" line)
        // Read battery temp from BatteryManager
        return s;
    }
}
```

### P1.4 — Register in SystemServer

```java
// In SystemServer.java startOtherServices():
DeviceHealthService dhs = new DeviceHealthService(context);
ServiceManager.addService("device_health", dhs);
```

### P1.5 — Manager + SystemServiceRegistry

```java
// frameworks/base/core/java/android/health/DeviceHealthManager.java
package android.health;

public class DeviceHealthManager {
    private final IDeviceHealthService mService;
    public DeviceHealthManager(IDeviceHealthService svc) { mService = svc; }
    public HealthSnapshot getHealthSnapshot() {
        try { return mService.getHealthSnapshot(); }
        catch (android.os.RemoteException e) { throw e.rethrowFromSystemServer(); }
    }
}
```

Register in `SystemServiceRegistry.java`:
```java
registerService("device_health", DeviceHealthManager.class,
    (ctx, b) -> new DeviceHealthManager(IDeviceHealthService.Stub.asInterface(b)));
```

### P1.6 — Permission + SELinux

Permission in `AndroidManifest.xml`:
```xml
<permission android:name="android.permission.DEVICE_HEALTH_READ"
    android:protectionLevel="signature|privileged" />
```

SELinux in `system/sepolicy/private/service_contexts`:
```
device_health  u:object_r:device_health_service:s0
```

### P1.7 — Build and Verify

```bash
m -j$(nproc)
adb root && adb remount && adb sync system
adb shell stop && adb shell start
adb shell service check device_health
adb shell dumpsys device_health
```

---

## Project 2 — Custom AIDL HAL: Ambient Light Sensor

**Goal:** Build a vendor AIDL HAL (`IAlsSensor`) with a callback interface, integrate with a framework helper, write SELinux policy and VTS test.

### P2.1 — Interface

```aidl
// hardware/interfaces/example_als/aidl/android/hardware/example_als/IAlsSensor.aidl
package android.hardware.example_als;

@VintfStability
interface IAlsSensor {
    float readLux();
    void setSamplingIntervalMs(int intervalMs);
    void registerCallback(in IAlsCallback cb);
}
```

```aidl
@VintfStability
interface IAlsCallback {
    oneway void onLuxChanged(float lux, long timestampNs);
}
```

### P2.2 — Android.bp

```blueprint
aidl_interface {
    name: "android.hardware.example_als",
    vendor_available: true,
    srcs: ["android/hardware/example_als/*.aidl"],
    stability: "vintf",
    backend: { ndk: { enabled: true }, java: { enabled: true } },
    versions_with_info: [{ version: "1", imports: [] }],
    frozen: true,
}
```

### P2.3 — Implementation (Cuttlefish-friendly, simulated)

```cpp
// default/AlsSensor.cpp
ndk::ScopedAStatus AlsSensor::readLux(float* out) {
    // Simulate: in production read /sys/bus/iio/devices/iio:device0/in_illuminance_raw
    *out = 350.0f + (rand() % 100);
    return ndk::ScopedAStatus::ok();
}
```

### P2.4 — init.rc + VINTF fragment

```rc
service vendor.als-default /vendor/bin/hw/android.hardware.example_als-service.default
    class hal
    user system
    group system
```

```xml
<manifest version="1.0" type="device">
    <hal format="aidl">
        <name>android.hardware.example_als</name>
        <version>1</version>
        <fqname>IAlsSensor/default</fqname>
    </hal>
</manifest>
```

### P2.5 — SELinux

```
type hal_als_default, domain;
type hal_als_default_exec, exec_type, vendor_file_type, file_type;
init_daemon_domain(hal_als_default)
binder_use(hal_als_default)
add_service(hal_als_default, hal_als_service)
```

### P2.6 — VTS Test

```cpp
TEST_F(AlsTest, ReadLuxPositive) {
    float lux = 0;
    ASSERT_TRUE(mHal->readLux(&lux).isOk());
    EXPECT_GT(lux, 0.f);
}
```

### P2.7 — Build and Verify

```bash
m android.hardware.example_als-service.default
adb root && adb remount && adb sync vendor
adb shell stop && adb shell start
adb shell lshal | grep als
atest VtsHalAlsTargetTest
```

---

## Project 3 — AAOS: Custom Vehicle Property + Car App + OTA

**Goal:** Define an OEM vendor property (`DRIVER_FATIGUE_LEVEL`), expose it through CarPropertyManager, build a Car app that displays it, and produce a signed incremental OTA.

### P3.1 — Define Property (in DefaultProperties.json)

```json
{
  "property": 557846528,
  "defaultValue": { "int32Values": [0] },
  "access": "READ_WRITE",
  "changeMode": "ON_CHANGE",
  "areas": [{ "areaId": 0 }],
  "comment": "OEM_DRIVER_FATIGUE: 0=alert, 1=mild, 2=drowsy, 3=critical"
}
```

### P3.2 — Car App (Kotlin)

```kotlin
class FatigueActivity : Activity() {
    override fun onCreate(b: Bundle?) {
        super.onCreate(b)
        setContentView(R.layout.main)
        val car = Car.createCar(this)
        val cpm = car.getCarManager(Car.PROPERTY_SERVICE) as CarPropertyManager
        cpm.registerCallback(object : CarPropertyManager.CarPropertyEventCallback {
            override fun onChangeEvent(ev: CarPropertyValue<*>) {
                runOnUiThread { findViewById<TextView>(R.id.level).text = "Level: ${ev.value}" }
            }
            override fun onErrorEvent(p: Int, z: Int) {}
        }, 557846528, CarPropertyManager.SENSOR_RATE_ONCHANGE)
    }
}
```

AndroidManifest must include:
```xml
<uses-permission android:name="android.car.permission.CAR_VENDOR_EXTENSION" />
<meta-data android:name="distractionOptimized" android:value="true" />
```

### P3.3 — Add to Product + Build OTA

```make
PRODUCT_PACKAGES += FatigueMonitor
```

```bash
# Build "before"
m dist && cp out/dist/*-target_files-*.zip ~/before.zip

# Add feature, rebuild
m dist && cp out/dist/*-target_files-*.zip ~/after.zip

# Generate incremental OTA
ota_from_target_files --incremental_from ~/before.zip ~/after.zip ~/ota.zip

# Sideload
adb reboot sideload && adb sideload ~/ota.zip
```

### P3.4 — Verify

```bash
adb shell pm list packages | grep fatigue
adb shell cmd car_service inject-vhal-event 557846528 2
adb logcat -s FatigueMonitor
```

---

## Project 4 — SELinux Policy From Scratch for a Vendor Daemon

**Goal:** Write and debug SELinux policy iteratively for a vendor daemon that reads sysfs and writes a property.

### P4.1 — The Daemon

```cpp
// vendor/oem/thermald/thermald.cpp
#include <android-base/properties.h>
#include <android-base/file.h>
#include <unistd.h>
int main() {
    while (true) {
        std::string temp;
        android::base::ReadFileToString("/sys/class/thermal/thermal_zone0/temp", &temp);
        android::base::SetProperty("vendor.thermal.temp_c", std::to_string(std::stoi(temp) / 1000));
        sleep(5);
    }
}
```

### P4.2 — Iterative Policy Development

**Round 1 — Permissive, capture denials:**
```
type oem_thermald, domain;
type oem_thermald_exec, exec_type, vendor_file_type, file_type;
init_daemon_domain(oem_thermald)
permissive oem_thermald;
```

```bash
adb shell dmesg | grep "oem_thermald" | grep "avc:"
```

**Round 2 — Add narrow allow rules for each denial:**
```
allow oem_thermald sysfs_thermal:dir search;
allow oem_thermald sysfs_thermal:file r_file_perms;
set_prop(oem_thermald, vendor_thermal_prop)
```

**Round 3 — Remove `permissive`, verify zero denials:**
```bash
adb shell getenforce   # Enforcing
adb shell dmesg | grep "oem_thermald" | grep "avc:"   # empty = success
adb shell getprop vendor.thermal.temp_c                # shows temperature
```

### P4.3 — File and Property Contexts

```
# file_contexts
/vendor/bin/oem_thermald   u:object_r:oem_thermald_exec:s0

# property_contexts
vendor.thermal.   u:object_r:vendor_thermal_prop:s0
```

---

## How to Use These Projects

| Your level | Do this |
|-----------|---------|
| Mid (L2–L3) | Project 1 + 4 — system service + SELinux |
| Senior (L3–L4) | Project 2 — HAL + VTS |
| Staff-prep (L5+) | Project 3 — full AAOS + OTA loop |
| All levels | All four in 2 weeks |

---

⬅️ Back to **[Table of Contents](./README.md)**
