# Level 5A — Deep Dive: AAOS Internals, EVS, Audio Routing & System Integration

> *Supplement to Level 5. Production-depth coverage of Android Automotive: the Exterior View System (backup camera), multi-zone audio architecture, cluster rendering, SOME/IP integration, and the vehicle abstraction layers that make AAOS different from every other Android form factor.*

---

## 5A.1 — Exterior View System (EVS) — Sub-Second Camera

### 5A.1.1 Why EVS Exists

FMVSS 111 (US) and ECE R46 (EU) require the backup camera image to be visible within **2 seconds** of engaging reverse gear. Android's standard camera stack (Camera2 API → CameraService → Camera HAL → ISP) takes 3–8 seconds through the full framework boot. EVS bypasses the entire Java framework.

### 5A.1.2 Architecture

```
[Gear = Reverse] ← VHAL property GEAR_SELECTION
      │
      ▼
EvsManager (native service, started by init very early)
      │
      ├──► IEvsCamera HAL (AIDL)
      │     Opens camera device directly (V4L2 / vendor driver)
      │     Returns frame buffers (DMA-BUF / GraphicBuffer)
      │
      ├──► IEvsDisplay HAL (AIDL)  
      │     Renders directly to a hardware overlay plane
      │     Bypasses SurfaceFlinger entirely
      │
      └──► EvsApp (native app, minimal UI — guidelines overlay, trajectory)
            Receives frames from IEvsCamera
            Optionally overlays parking guidelines
            Sends composed frames to IEvsDisplay
```

### 5A.1.3 Source Paths

```
hardware/interfaces/automotive/evs/          # AIDL interface definitions
packages/services/Car/evs/                   # EvsManager native service
packages/services/Car/evs/apps/              # Reference EVS app
device/google/cuttlefish/guest/hals/evs/     # Cuttlefish EVS HAL (virtual camera)
```

### 5A.1.4 The Boot-Fast Path

EVS is special because it starts **before system_server**:

```rc
# vendor init.rc
on early-init
    start evs_driver       # camera HAL process

on init
    start evsmanagerd      # EVS manager

# Gear=R triggers EvsApp immediately, no Java involved
```

Timeline on a well-tuned SoC:
```
T+0.0s    Power on
T+0.6s    Kernel + early init
T+0.8s    EVS camera HAL open()
T+1.0s    First frame captured
T+1.2s    Frame on display (via HWC overlay)
T+1.5s    Parking guidelines rendered
          ─── regulatory requirement met ───
T+8-15s   Full Android UI available
```

### 5A.1.5 Hands-On: EVS on Cuttlefish

```bash
lunch aosp_cf_x86_64_auto-trunk_staging-userdebug && m -j$(nproc)
launch_cvd

# Simulate gear reverse
adb shell cmd car_service inject-vhal-event 289408000 4  # GEAR_SELECTION = REVERSE(4)

# Check EVS state
adb shell dumpsys android.hardware.automotive.evs.IEvsEnumerator/default
```

### 5A.1.6 Production Considerations

- **Dedicated overlay plane:** EVS must not compete with SurfaceFlinger for display resources. Configure HWC to reserve one overlay plane for EVS.
- **Power sequencing:** Camera power rail must be always-on or have <100ms enable time.
- **Thermal:** Camera module generates heat; in hot climates (>85°C ambient in engine bay), thermal throttling must not disable the backup camera.
- **Fallback:** If EVS fails, a separate MCU-driven backup camera should take over (safety requirement at some OEMs).

---

## 5A.2 — Multi-Zone Audio Architecture (Production Depth)

### 5A.2.1 The Problem Statement

A vehicle with 4 occupants needs:
- Driver: navigation prompts + phone call + low background music
- Front passenger: podcast at moderate volume
- Rear-left child: cartoon video audio through headphones
- Rear-right passenger: gaming audio through headphones

All simultaneously, with proper ducking (nav ducks music), focus management, and zone isolation.

### 5A.2.2 The Configuration Stack

```
car_audio_configuration.xml (device overlay)
     │
     ▼ parsed by CarAudioService
┌───────────────────────────────────────────────────────┐
│ Zone 0: Primary (Driver)                              │
│   Audio Device Addresses:                             │
│     bus0_media_out → front speakers (L/R/C/Sub)       │
│     bus0_nav_out → front center speaker               │
│     bus0_phone_out → front speakers (ducked)          │
│   Usages → Device mapping:                            │
│     USAGE_MEDIA → bus0_media_out                      │
│     USAGE_ASSISTANCE_NAVIGATION_GUIDANCE → bus0_nav_out│
│     USAGE_VOICE_COMMUNICATION → bus0_phone_out        │
├───────────────────────────────────────────────────────┤
│ Zone 1: Rear-Left                                     │
│   Audio Device Addresses:                             │
│     bus1_media_out → rear-left headphone jack         │
│   Usages:                                             │
│     USAGE_MEDIA → bus1_media_out                      │
│     USAGE_GAME → bus1_media_out                       │
├───────────────────────────────────────────────────────┤
│ Zone 2: Rear-Right                                    │
│   Audio Device Addresses:                             │
│     bus2_media_out → rear-right headphone jack        │
│   Usages:                                             │
│     USAGE_MEDIA → bus2_media_out                      │
│     USAGE_GAME → bus2_media_out                       │
└───────────────────────────────────────────────────────┘
```

### 5A.2.3 Audio HAL Requirements for Multi-Zone

The Audio HAL must expose **multiple output streams** with distinct **device addresses**:

```xml
<!-- audio_policy_configuration.xml (vendor) -->
<module name="primary" halVersion="3.0">
    <mixPorts>
        <mixPort name="primary output" role="source" flags="AUDIO_OUTPUT_FLAG_PRIMARY">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </mixPort>
        <mixPort name="zone1 output" role="source">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </mixPort>
    </mixPorts>
    <devicePorts>
        <devicePort tagName="bus0_media_out" type="AUDIO_DEVICE_OUT_BUS" role="sink" address="bus0_media_out"/>
        <devicePort tagName="bus1_media_out" type="AUDIO_DEVICE_OUT_BUS" role="sink" address="bus1_media_out"/>
    </devicePorts>
    <routes>
        <route type="mix" sink="bus0_media_out" sources="primary output"/>
        <route type="mix" sink="bus1_media_out" sources="zone1 output"/>
    </routes>
</module>
```

### 5A.2.4 Audio Focus Across Zones

Each zone has its own focus stack. `CarAudioFocus` manages independently per zone:

- Zone 0 (driver) can have: NAV (ducking MEDIA) + PHONE (exclusive)
- Zone 1 (passenger) independently: MEDIA (exclusive in that zone)
- Cross-zone ducking: If driver takes a call, should passenger audio duck? **Configurable per OEM.**

```java
// CarAudioService.java
private final SparseArray<CarAudioZone> mCarAudioZones;

public int requestAudioFocusForZone(AudioFocusInfo info, int zoneId) {
    CarAudioZone zone = mCarAudioZones.get(zoneId);
    return zone.getCarAudioFocus().evaluateAndHandleFocusRequest(info);
}
```

### 5A.2.5 Debugging Audio on AAOS

```bash
adb shell dumpsys car_service --services CarAudioService
# Shows: zones, focus stacks per zone, current routing, volume groups

adb shell dumpsys audio
# Shows: AudioFlinger mixer state, connected devices, active tracks

adb shell dumpsys media.audio_policy
# Shows: audio policy configuration, engine state, connected devices

# Inject audio context change:
adb shell cmd car_service inject-vhal-event 289409283 1  # AUDIO_FOCUS state
```

---

## 5A.3 — Cluster Rendering (Instrument Cluster)

### 5A.3.1 The Architecture

The instrument cluster (speedometer, tachometer, warnings) can be rendered by:

1. **Non-Android** (separate MCU/QNX guest) — Android sends data via vsock/shared memory.
2. **Android as secondary display** — A dedicated display managed by `ClusterHomeService`.
3. **Hybrid** — Android renders navigation card; MCU renders gauges.

For case 2 (full Android cluster):

```
frameworks/base/services/core/.../ClusterHomeService.java
  │
  ▼ provides cluster activities to
InstrumentClusterRenderingService (vendor app)
  │ renders on cluster display
  ▼
DisplayManager → display ID for cluster
  │
  ▼
SurfaceFlinger → HWC → cluster display panel
```

### 5A.3.2 Navigation on Cluster

Turn-by-turn navigation info goes from the nav app to the cluster via:

```
NavigationApp (Google Maps, etc.)
  │ navigationStateChange() via CarNavigationStatusManager
  ▼
CarNavigationStatusService
  │ bundles NavigationState proto
  ▼
InstrumentClusterRenderingService (renders on cluster display)
```

The `NavigationState` proto contains: road name, distance to maneuver, maneuver type (turn left, exit, etc.), lane guidance.

### 5A.3.3 Cluster Timing Requirements

- Speedometer update: ≤100 ms latency from CAN signal to display pixel.
- Warning icons: ≤200 ms from ECU signal.
- Boot-to-cluster-visible: ≤2 s (if Android renders it; otherwise MCU handles it).

If Android cannot meet these, the **hybrid** approach is used: MCU renders critical gauges from CAN directly; Android renders the "media card" and "navigation card" areas on the cluster once booted.

---

## 5A.4 — SOME/IP and Automotive Ethernet Integration

### 5A.4.1 Beyond CAN: Ethernet in Vehicles

Modern vehicles increasingly use Ethernet (100BASE-T1, 1000BASE-T1) for:
- Camera streams (surround view: 4× cameras at 30fps = ~400 Mbps)
- ADAS sensor data (radar, lidar)
- Software updates to ECUs
- Diagnostics (DoIP — Diagnostics over IP)

The protocol stack on Ethernet:

```
Application Layer:
  SOME/IP (Service-Oriented Middleware over IP)
  DoIP (Diagnostics over IP)
  AVB (Audio/Video Bridging for time-sensitive streams)

Transport:
  UDP (SOME/IP, most automotive)
  TCP (DoIP, bulk transfers)

Network:
  IPv4 (automotive default; IPv6 emerging)

Data Link:
  Ethernet (100BASE-T1 for most; 1000BASE-T1 for cameras/ADAS)
```

### 5A.4.2 SOME/IP in AAOS

SOME/IP is a service discovery + RPC protocol used by AUTOSAR. Android's VHAL may need to talk to ECUs via SOME/IP instead of CAN.

Architecture:
```
VHAL (vendor)
  │ reads/writes vehicle properties
  ▼
SOME/IP middleware library (vsomeip / CommonAPI)
  │ serializes/deserializes service calls
  ▼
UDP socket to vehicle Ethernet
  │
  ▼
ECU (AUTOSAR, running SOME/IP service)
```

Integration pattern:
```cpp
// In vendor VHAL implementation:
class SomeIpVehicleHardware : public IVehicleHardware {
    void onPropertySubscribed(int32_t propId) override {
        // Register SOME/IP event subscription for this property
        someipClient.subscribe(propId_to_service_event_mapping[propId]);
    }
    
    void onSomeIpEvent(ServiceId svc, EventId evt, Payload data) {
        // Convert SOME/IP payload to VehiclePropValue
        VehiclePropValue val = convertToVhalValue(svc, evt, data);
        deliverPropertyEvent(val);
    }
};
```

### 5A.4.3 Time-Sensitive Networking (TSN/AVB)

For camera streams and audio, automotive Ethernet uses IEEE 802.1AS (time sync) and 802.1Qav (traffic shaping). Android's `SurroundView` service consumes camera frames delivered over AVB:

```
Camera ECU → Ethernet AVB stream → gPTP-synced capture → DMA → Android buffer → SurroundView HAL
```

Latency budget: <33 ms end-to-end (one frame at 30 fps). Achievable with proper TSN configuration.

---

## 5A.5 — AAOS Power Management — Complete State Machine

### 5A.5.1 The Full State Diagram

```
                         ┌──────────────┐
                    ┌───►│     ON       │◄──────────────────────────┐
                    │    │ (full run)   │                            │
                    │    └──────┬───────┘                            │
                    │           │ User turns off ignition            │
                    │           │ (VHAL: AP_POWER_STATE_REQ =        │
                    │           │  SHUTDOWN_PREPARE)                  │
                    │           ▼                                     │
                    │    ┌──────────────────┐                        │
                    │    │ SHUTDOWN_PREPARE │                        │
                    │    │ (Garage Mode)    │                        │
                    │    │ • OTA apply      │                        │
                    │    │ • dexopt         │                        │
                    │    │ • log upload     │                        │
                    │    │ • job scheduler  │                        │
                    │    └───────┬──────────┘                        │
                    │            │                                    │
                    │     ┌──────┴───────────────┐                   │
                    │     │                      │                   │
                    │     ▼                      ▼                   │
                    │  ┌──────────┐     ┌───────────────┐           │
                    │  │ SUSPEND  │     │   SHUTDOWN    │           │
                    │  │ (S2RAM)  │     │ (power off)   │           │
                    │  │ ~100ms   │     │ requires full │           │
                    │  │ resume   │     │ boot to wake  │           │
                    │  └────┬─────┘     └───────────────┘           │
                    │       │                                        │
                    │       │ Wake trigger (door open, key fob,     │
                    │       │ scheduled wake, CAN wakeup)            │
                    └───────┘                                        │
                                                                     │
                    ┌────────────────┐                               │
                    │ HIBERNATE      │  (S2D, Android 13+)           │
                    │ RAM → flash    │                               │
                    │ 0 mW standby   │                               │
                    │ ~5-15s resume  ├───────────────────────────────┘
                    └────────────────┘
```

### 5A.5.2 Garage Mode Budget Negotiation

```
Android (CarPowerManagementService)          Vehicle MCU
──────────────────────────────────           ───────────
                                             AP_POWER_STATE_REQ = SHUTDOWN_PREPARE
                                               (includes max_garage_mode_ms = 600000)
  Enters Garage Mode
  Schedules work (OTA, dexopt, upload)
  ...working...
                                             
  AP_POWER_STATE_REPORT = SHUTDOWN_PREPARE   
    (periodically: "I need more time")       
                                             
  ...work complete or budget expired...      
                                             
  AP_POWER_STATE_REPORT = DEEP_SLEEP_ENTRY   
  (or SHUTDOWN_START)                        
                                             MCU cuts power to AP domain
```

### 5A.5.3 CarPowerPolicy — Component-Level Power Control

`CarPowerPolicyDaemon` (native, started very early) manages which hardware components are powered in each state:

```xml
<!-- /vendor/etc/automotive/power_policy.xml -->
<powerPolicy>
    <policyGroups>
        <policyGroup id="SuspendPrep">
            <defaultPolicy id="policy_suspend_prep"/>
        </policyGroup>
    </policyGroups>
    <policies>
        <policy id="policy_suspend_prep">
            <otherComponents>
                <component id="POWER_COMPONENT_WIFI" state="off"/>
                <component id="POWER_COMPONENT_BLUETOOTH" state="on"/>
                <component id="POWER_COMPONENT_DISPLAY" state="off"/>
                <component id="POWER_COMPONENT_AUDIO" state="off"/>
                <component id="POWER_COMPONENT_NFC" state="off"/>
            </otherComponents>
        </policy>
    </policies>
</powerPolicy>
```

Apps register for power policy changes:
```java
CarPowerManager cpm = (CarPowerManager) car.getCarManager(Car.POWER_SERVICE);
cpm.addPowerPolicyListener(executor, POWER_COMPONENT_DISPLAY, (policy) -> {
    if (!policy.isComponentEnabled(POWER_COMPONENT_DISPLAY)) {
        // Display is off — pause rendering, release GPU resources
    }
});
```

---

## 5A.6 — User Experience Architecture: Rotary Input & Voice

### 5A.6.1 Rotary Controller Input

Many vehicles use a rotary dial (BMW iDrive style) instead of touch:

```
Physical rotary encoder → kernel input device (/dev/input/eventN)
  │ EV_REL/REL_WHEEL events
  ▼
InputFlinger → CarInputService
  │ Translates rotary events to focus navigation
  ▼
FocusArea / FocusParkingView (Car UI Library)
  │ Manages spatial focus across the UI
  ▼
App receives standard focus/key events
```

The **Car UI Library** (`packages/apps/Car/libs/car-ui-lib/`) provides:
- `FocusArea` — defines navigation regions
- `FocusParkingView` — invisible view that "parks" focus when no focusable view is appropriate
- `RotaryController` — translates rotary input to focus traversal

### 5A.6.2 Voice Interaction

AAOS voice architecture:

```
Microphone array → Audio HAL (record from driver zone)
  │
  ▼
VoiceInteractionService (Google Assistant / OEM assistant)
  │ Always-on hotword detection (runs in DSP if available)
  │
  ▼ recognized intent
  │
CarVoiceInteractionSession
  │ Maps voice commands to car actions:
  │   "Set temperature to 72" → CarPropertyManager.setFloatProperty(HVAC_TEMPERATURE_SET, ...)
  │   "Navigate to home" → launches nav intent
  │   "Call John" → telephony intent
  │
  └─► CarUxRestrictions checked: voice always allowed while driving
```

---

## 5A.7 — AAOS System Integration Testing

### 5A.7.1 AAOS-Specific CTS Modules

```
CtsCarTestCases
CtsCarHostTestCases  
CtsCarBuiltinAppHostTestCases
CtsCarMediaHostTestCases
CtsVehicleHalTestCases
```

### 5A.7.2 Vehicle HAL Testing Framework

```java
// packages/services/Car/tests/vehiclehal_test/
// Uses MockedVehicleHal to simulate VHAL without real hardware:

@Test
public void testHvacTemperatureSet() {
    mMockedVehicleHal.injectEvent(
        VehiclePropValueBuilder.newBuilder(VehicleProperty.HVAC_TEMPERATURE_SET)
            .setAreaId(VehicleAreaSeat.ROW_1_LEFT)
            .addFloatValues(22.5f)
            .build());
    
    CarPropertyValue<Float> value = mCarPropertyManager.getProperty(
        VehicleProperty.HVAC_TEMPERATURE_SET, VehicleAreaSeat.ROW_1_LEFT);
    assertEquals(22.5f, value.getValue(), 0.01f);
}
```

### 5A.7.3 End-to-End Vehicle Simulation

For pre-production testing without a real vehicle:

```
Vehicle Simulator (PC application or HIL bench)
  │ sends CAN/SOME-IP frames
  ▼
CAN interface (SocketCAN on Linux, or USB-CAN adapter)
  │
  ▼
Vendor VHAL (reads CAN, populates properties)
  │
  ▼
CarService → Apps
```

On Cuttlefish, this is simulated entirely in software via the default VHAL + `inject-vhal-event` commands.

---

## 5A.8 — Verifying This Deep Dive

You should be able to:

1. Explain why EVS bypasses the Java framework and how it achieves <2s boot-to-camera.
2. Configure a 3-zone audio system in `car_audio_configuration.xml`.
3. Describe the Garage Mode budget negotiation protocol between Android and the vehicle MCU.
4. Explain the role of SOME/IP in a modern vehicle's Android integration.
5. Design a cluster rendering path that meets <100ms speedometer update latency.
6. Write a VHAL test using `MockedVehicleHal`.

---

⬅️ Back to **[Level 5 — AAOS](./level-05-aaos.md)** | ➡️ **[Level 6 — Performance & Memory](./level-06-performance-memory.md)**

