# Level 5 — Android Automotive OS (AAOS)

> *"AAOS is not 'Android in a car.' It is Android with a Vehicle HAL, multi-display from day one, multi-user from day one, and ASIL-aware engineering throughout the stack."*

---

## Chapter 5.1 — AAOS Architecture Differences

### 5.1.1 What Changes vs Phone Android

| Subsystem | Phone | AAOS |
|-----------|-------|------|
| Primary input | touch | touch + rotary + voice + steering wheel |
| Displays | 1 | 2–8 (cluster, IVI, HUD, RSE × 2+) |
| Users | 1 active | driver + passengers, possibly all simultaneous |
| Boot time req | < 30 s | < **2 s to backup camera** (regulatory, FMVSS 111) |
| Connectivity | always-on | intermittent (cellular only) |
| HW interface | sensors, radios | **CAN, LIN, FlexRay, Ethernet AVB**, vehicle bus |
| Update | OTA over WiFi | OTA + dealer flash; fail-safe required |
| Safety | best-effort | ASIL-B+ for some flows |
| Audio | 2-channel | 8–16 channel, beamforming, ducking, focus |
| HVAC, seats, lighting | n/a | **first-class APIs** |

### 5.1.2 Layered Architecture

```
┌────────────────────────────────────────────────────────┐
│  Car Apps (CarUi, Car Launcher, Media, Dialer, ...)      │
├────────────────────────────────────────────────────────┤
│  Car API (android.car.* — system signature)            │
├────────────────────────────────────────────────────────┤
│  Car Service (CarService, packages/services/Car/)      │
│   ├─ CarPropertyService   ─ talks to VHAL              │
│   ├─ CarAudioService                                   │
│   ├─ CarPowerManagementService                         │
│   ├─ CarUserService                                    │
│   ├─ CarOccupantZoneService                            │
│   └─ ClusterHomeService                                │
├────────────────────────────────────────────────────────┤
│  Vehicle HAL (VHAL)  ─ AIDL (Android 14+) / HIDL       │
├────────────────────────────────────────────────────────┤
│  Vehicle Network (CAN/LIN/Ethernet-AVB)                │
└────────────────────────────────────────────────────────┘
```

Source roots:
- `packages/services/Car/` — Car Service, Car API, system apps.
- `hardware/interfaces/automotive/` — VHAL, EVS, audio control AIDL.
- `device/google/cuttlefish/shared/auto/` — Cuttlefish auto variant.

### 5.1.3 Key Subsystems

- **CarService** — central system service, runs in `system_server` (until Android 13) or its own process (Android 14+, separate `com.android.car` system app).
- **VHAL** — only abstraction over the vehicle bus. Exposes ~300 standardized properties.
- **Car Audio** — context-based zones (driver, rear-left, rear-right) + audio focus integrated with VHAL audio control.
- **EVS (Exterior View System)** — sub-second-boot path for backup camera, bypasses Java framework.
- **OccupantZone** — maps users to displays and zones.

---

## Chapter 5.2 — Vehicle HAL (VHAL) Deep Dive

### 5.2.1 The Property Model

Everything is a **vehicle property**. A property is a tuple of:

- `propId` — 32-bit ID (`VehicleProperty` enum, e.g. `HVAC_TEMPERATURE_SET = 0x05000503`).
- `value` — typed (`int32`, `int64`, `float`, `string`, `byte[]`, mixed).
- `areaId` — bitmask of **areas** (e.g., row 1 left seat, row 1 right). Per `VehicleAreaSeat`, `VehicleAreaWindow`, etc.
- `status` — `AVAILABLE` / `UNAVAILABLE` / `ERROR`.
- `timestamp` (ns).
- `accessibility` — READ / WRITE / READ_WRITE.
- `changeMode` — `STATIC` / `ON_CHANGE` / `CONTINUOUS`.

### 5.2.2 The AIDL Interface (Android 14+)

`hardware/interfaces/automotive/vehicle/aidl/android/hardware/automotive/vehicle/IVehicle.aidl`:

```aidl
@VintfStability
interface IVehicle {
    VehiclePropConfigs getAllPropConfigs();
    VehiclePropConfigs getPropConfigs(in int[] propIds);
    void getValues(in IVehicleCallback callback, in GetValueRequests requests);
    void setValues(in IVehicleCallback callback, in SetValueRequests requests);
    void subscribe(in IVehicleCallback callback,
                   in SubscribeOptions[] options, int maxSharedMemoryFileCount);
    void unsubscribe(in IVehicleCallback callback, in int[] propIds);
}
```

Property values flow over **shared memory** for high-rate continuous properties (e.g., wheel speed at 100 Hz).

### 5.2.3 VHAL Reference Implementation

`hardware/interfaces/automotive/vehicle/aidl/impl/default_config/` defines mock properties used by the default VHAL service running on Cuttlefish.

```
default_config_test_properties.json     # for tests
DefaultProperties.json                  # the catalog
fake_impl/                              # fake VehicleHardware
android.hardware.automotive.vehicle@V3-default-service.rc
```

### 5.2.4 Adding a Custom Vehicle Property

🛠️ **Hands-On (Cuttlefish):** Add a custom OEM property `OEM_DRIVER_FATIGUE_LEVEL`.

#### Step 1 — Define the Property ID

OEM-extensible properties use the `VehiclePropertyGroup.VENDOR` group. Format:
```
0x?XXX0400  ← VENDOR group, GLOBAL area, INT32
```
Example ID: `0x21100400` (vendor int32 global).

`hardware/interfaces/automotive/vehicle/aidl_property/.../VehicleProperty.aidl`:
```aidl
const int OEM_DRIVER_FATIGUE_LEVEL = 0x21100400;
```

(In real OEM trees, vendor properties live in `vendor/<oem>/automotive/.../IVehicleVendorProperty.aidl`.)

#### Step 2 — Register in DefaultProperties.json

```json
{
  "property": "VehicleProperty::OEM_DRIVER_FATIGUE_LEVEL",
  "defaultValue": { "int32Values": [0] },
  "access": "READ_WRITE",
  "changeMode": "ON_CHANGE",
  "areas": [{ "areaId": 0 }]
}
```

#### Step 3 — Rebuild and Restart VHAL

```bash
m android.hardware.automotive.vehicle@V3-default-service
adb root && adb remount
adb sync vendor
adb shell stop && adb shell start
adb shell lshal | grep automotive.vehicle
```

#### Step 4 — Read/Write from Shell

`packages/services/Car/tools/emulator/` provides `vhal_emulator.py`. Or use `dumpsys`:

```bash
adb shell dumpsys android.hardware.automotive.vehicle.IVehicle/default
adb shell cmd car_service inject-vhal-event 0x21100400 7
adb shell dumpsys car_service --hal | grep 0x21100400
```

#### Step 5 — Java App Access

```java
Car car = Car.createCar(context);
CarPropertyManager cpm = (CarPropertyManager) car.getCarManager(Car.PROPERTY_SERVICE);
int level = cpm.getIntProperty(0x21100400, VehicleAreaType.VEHICLE_AREA_TYPE_GLOBAL);
cpm.setIntProperty(0x21100400, VehicleAreaType.VEHICLE_AREA_TYPE_GLOBAL, 5);
cpm.registerCallback(callback, 0x21100400, CarPropertyManager.SENSOR_RATE_NORMAL);
```

The app needs the `android.car.permission.CAR_VENDOR_EXTENSION` signature permission (or a custom one declared by OEM).

⚠️ **OEM Pitfall:** Vendor properties bypass the AAOS CDD. They are **not portable across OEMs** and not guaranteed across Android versions. Use them only for genuine OEM-specific telemetry; everything else should go through standard properties.

🐞 **Common Production Bug:** A vendor adds a `CONTINUOUS` property at 200 Hz without shared memory; binder thread pool saturates; CarService becomes unresponsive; ANR. Fix: use `maxSharedMemoryFileCount > 0` in subscribe and ensure VHAL impl publishes via the shmem path.

---

## Chapter 5.3 — Multi-Display & Multi-User

### 5.3.1 Display Topology

AAOS distinguishes:

- **Default display** (driver IVI head unit).
- **Cluster display** (instrument cluster).
- **Rear-seat displays** (RSE).
- **Head-up display (HUD)**.

Configured in `device/<oem>/<device>/overlay/frameworks/base/core/res/res/values/config.xml` and via `IDisplay` enumeration. The `OccupantZoneService` maps each display to an **occupant zone**.

```xml
<!-- Example overlay -->
<integer name="config_defaultUiModeType">3</integer>  <!-- car -->
<bool name="config_supportsMultiDisplay">true</bool>
<integer-array name="config_occupant_display_mapping">
    <item>0:0</item>  <!-- zone 0 (driver) → display 0 -->
    <item>1:1</item>  <!-- zone 1 (front passenger) → display 1 -->
</integer-array>
```

### 5.3.2 Multi-User Models

AAOS supports **Headless System User Mode (HSUM, Android 13+)** as the default:

- User 0 = "system user," runs CarService and persistent infra. **Never visible.**
- User 10+ = real human users. The **first foreground user** is auto-created on first boot.
- Multiple users can be visible **simultaneously** on different displays (Android 14 "Multi-user, Multi-display" / `MUMD`).

```bash
adb shell pm list users
adb shell cmd car_service create-user "Driver"
adb shell cmd car_service switch-user 10
adb shell cmd car_service start-user-in-background-on-display 11 1
```

Source: `packages/services/Car/service/src/com/android/car/user/CarUserService.java`.

🎯 **Staff-Level Insight:** MUMD changes deep assumptions across the framework. ActivityManager, WindowManager, and InputManager all needed per-display + per-user awareness. If you're modifying any of these for AAOS, read `frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java` and `DisplayContent.java` carefully.

---

## Chapter 5.4 — Car Services & APIs

### 5.4.1 The Major Car Managers

| Manager | What it does |
|---------|--------------|
| `CarPropertyManager` | Read/write/subscribe vehicle properties |
| `CarPowerManager` | STR (suspend-to-RAM) / hibernate / boot reasons |
| `CarUxRestrictionsManager` | Driving-state UI restrictions (no video, etc.) |
| `CarAudioManager` | Multi-zone audio routing |
| `CarOccupantZoneManager` | Map user ↔ display ↔ zone |
| `CarMediaManager` | Multi-user media app focus |
| `CarDrivingStateManager` | Parked / idling / moving |
| `CarBluetoothManager` | Multi-profile auto-connect (HFP, A2DP, MAP) |
| `CarNavigationStatusManager` | Send turn-by-turn to cluster |
| `CarDiagnosticManager` | OBD-II live & freeze frames |
| `ClusterHomeManager` | Render activity to cluster display |

### 5.4.2 UX Restrictions

The UX Restrictions Engine reads driving state (from VHAL `PERF_VEHICLE_SPEED` + `GEAR_SELECTION`) and broadcasts a restriction set:

- `UX_RESTRICTIONS_NO_VIDEO`
- `UX_RESTRICTIONS_NO_KEYBOARD`
- `UX_RESTRICTIONS_LIMIT_STRING_LENGTH`
- `UX_RESTRICTIONS_NO_DIALPAD`
- ...and ~10 more.

Apps must adapt UI when restrictions change (`OnUxRestrictionsChangedListener`). CTS-Auto enforces compliance for built-in apps.

### 5.4.3 Car Power Management

A vehicle has 3 power states the framework cares about:

- **ON** — full Android up.
- **Suspend-to-RAM (STR)** — kernel sleeps; instant resume on door open.
- **Suspend-to-Disk** (Hibernation, Android 13+) — RAM dumped to flash.

Source: `packages/services/Car/service/src/com/android/car/power/CarPowerManagementService.java`.

VHAL property `AP_POWER_STATE_REQ` and `AP_POWER_STATE_REPORT` orchestrate hand-off with the vehicle MCU. Garage Mode (background updates while parked) is a sub-state of ON.

🐞 **Common Production Bug:** Vehicle wakes Android with `AP_POWER_STATE_REQ = ON` but app processes from previous user weren't cleanly suspended; ART hits `failed to lock JIT` race. Fix: ensure your app obeys the `onPrepareShutdown` callback path.

---

## Chapter 5.5 — Cuttlefish AAOS

Cuttlefish has an automotive variant. Build:

```bash
source build/envsetup.sh
lunch aosp_cf_x86_64_auto-trunk_staging-userdebug
m -j$(nproc)
launch_cvd --num_instances=1 \
           --display0=width=1080,height=600,dpi=120 \
           --display1=width=400,height=600,dpi=120
```

You get an IVI display + a cluster display. SystemUI Auto, CarLauncher, and the default VHAL run.

🛠️ **Hands-On — Inject a vehicle event:**
```bash
adb shell cmd car_service inject-vhal-event 287310600 50  # PERF_VEHICLE_SPEED = 50
adb logcat -s CarUxR
```

---

## Chapter 5.6 — Car Service Debugging

```bash
adb shell dumpsys car_service                    # full state
adb shell dumpsys car_service --hal-types VEHICLE
adb shell dumpsys car_service --user
adb shell cmd car_service help

# VHAL-specific
adb shell lshal debug android.hardware.automotive.vehicle.IVehicle/default

# Property history (CarPropertyService keeps a ring buffer)
adb shell dumpsys car_service --services CarPropertyService

# Audio zones
adb shell dumpsys car_service --services CarAudioService

# CarPowerManagementService timing
adb logcat -s CarPowerManagementService
```

🎯 **Staff-Level Insight:** Most "phantom" Car bugs are property timing bugs. The bus delivers a property *late*, the framework caches the stale value, the UI shows the wrong gear. Always correlate `timestamp` from VHAL with framework consumption time. The `dumpsys car_service --services CarPropertyService` ring buffer is the diagnostic tool.

---

## Chapter 5.7 — Verifying Level 5

You should be able to:

1. List 5 architectural differences between AAOS and phone Android.
2. Define a custom vehicle property end-to-end and observe it from a Java client.
3. Explain HSUM and MUMD; describe how the same user can drive multiple displays.
4. Trace a UX restriction change from VHAL speed → UI app.
5. Describe the AAOS power state machine and Garage Mode.

---

➡️ Continue to **[Level 6 — Performance & Memory](./level-06-performance-memory.md)**

