# Level 2 — Framework Internals (Mid-Level Engineer)

> *"`system_server` is Android. If you cannot trace a request from app → Binder → service → HAL, you cannot ship Android."*

This Level builds the mental model of **what runs**, **in what process**, **talking to what**, **through which IPC**.

---

## Chapter 2.1 — Boot Sequence: Kernel → Init → Zygote → SystemServer

### 2.1.1 The Whole Picture

```
[BootROM]
    │
    ▼
[Bootloader] (e.g., U-Boot, ABL, fastboot, Cuttlefish's crosvm bootloader)
    │   verifies boot.img via AVB
    ▼
[Linux Kernel]
    │   kernel cmdline includes androidboot.* params
    ▼
[/init from generic_ramdisk]   PID 1
    │   parses /init.rc, mounts /system, /vendor via super.img + dm-verity
    │   transitions context: u:r:init:s0 → fork() → setexeccon()
    ▼
[init: trigger early-init → init → late-init → fs → post-fs → post-fs-data → boot]
    │   starts native services (servicemanager, hwservicemanager removed in 14+,
    │   vold, netd, surfaceflinger, audioserver, mediaserver, ...)
    ▼
[zygote] (app_process forked from init)
    │   preloads classes & resources, opens socket /dev/socket/zygote
    │   forks system_server as the FIRST child
    ▼
[system_server]
    │   bootstraps ~100 Java services (AMS, PMS, WMS, PowerMS, ...)
    │   publishes them to ServiceManager
    │   sends BOOT_COMPLETED broadcast
    ▼
[Launcher / SystemUI / first user-facing UI]
```

### 2.1.2 Detailed Phases (init perspective)

`system/core/rootdir/init.rc` is the master script. Key triggers:

```rc
on early-init
    write /proc/1/oom_score_adj -1000
    mkdir /mnt/vendor 0755 root root

on init
    sysclktz 0
    mount cgroup2 ...
    # set up cgroups, /dev, /sys

on late-init
    trigger early-fs
    trigger fs
    trigger post-fs
    trigger late-fs
    trigger post-fs-data
    trigger zygote-start
    trigger early-boot
    trigger boot

on post-fs-data
    mkdir /data/system 0775 system system
    # decrypt /data via vold
    setprop vold.post_fs_data_done 1

on zygote-start && property:ro.crypto.state=encrypted && property:ro.crypto.type=file
    start zygote
    start zygote_secondary

on boot
    # set defaults, start late_start services
```

🐞 **Common Production Bug:** Vendor adds `start my_hal` under `on boot` instead of `on post-fs-data` and the HAL needs `/data/vendor/...` which isn't mounted yet. Symptom: `open: ENOENT` in vendor logs only on first boot. Fix: gate on `post-fs-data` or add `disabled` + `start` from a property trigger.

### 2.1.3 Zygote: The Process Factory

Every app process is a **fork** of `zygote`. Zygote pre-loads:

- All framework classes (`/system/framework/framework.jar`)
- All shared resources (`framework-res.apk`)
- The ART runtime
- Common JNI libraries (`libandroid_runtime.so`, etc.)

When you launch an app, AMS asks zygote to fork; the child closes the zygote socket, sets its UID/GID, applies seccomp, then `execve` is **not** called — it just runs `ActivityThread.main()`. This is why app startup is fast (no class loading from scratch).

Source: `frameworks/base/core/java/com/android/internal/os/Zygote*.java`, `frameworks/base/cmds/app_process/`.

🎯 **Staff-Level Insight:** The "Unspecialized App Process Pool" (USAP, Android 10+) keeps a few pre-forked but un-specialized processes ready, cutting cold-launch latency by ~20–80 ms. If you change zygote preloading, you measurably move SystemServer boot latency. Use `frameworks/base/services/core/java/com/android/server/SystemServer.java`'s timing logs (`BOOT_PROGRESS_*` events) to validate.

### 2.1.4 SystemServer Boot Timeline

`SystemServer.run()` proceeds in three phases:

1. **Bootstrap services** — ActivityManager, PowerManager, PackageManager, BatteryStats, Sensors.
2. **Core services** — BatteryService, UsageStatsService, WebViewUpdateService.
3. **Other services** — ~80+ services: Wifi, Telephony, Location, Display, Window, JobScheduler, etc.

Each service receives lifecycle callbacks: `PHASE_WAIT_FOR_DEFAULT_DISPLAY`, `PHASE_LOCK_SETTINGS_READY`, `PHASE_SYSTEM_SERVICES_READY`, `PHASE_ACTIVITY_MANAGER_READY`, `PHASE_THIRD_PARTY_APPS_CAN_START`, `PHASE_BOOT_COMPLETED`.

```bash
adb shell dumpsys SurfaceFlinger --latency
adb logcat -d | grep -E 'SystemServer|boot_progress'
adb shell cat /proc/bootstat   # if BootStat is enabled
```

---

## Chapter 2.2 — Binder IPC Deep Dive

Binder is the heart of Android. **All cross-process calls** go through it.

### 2.2.1 Why Binder (Not Sockets / D-Bus / gRPC)

- **Synchronous by default** with deferred async option.
- **Reference counting across processes** (death notifications).
- **Capability passing** — you can hand an object to another process and they get a usable proxy.
- **Thread pool managed by kernel driver** (`/dev/binder`, `/dev/hwbinder`, `/dev/vndbinder`).
- **Single-copy** payloads via `ashmem`/`memfd` for large data.
- **Parcel** serialization is platform-internal, fast, and stable-per-version.

### 2.2.2 The Three Domains

| Domain | Driver | Used For |
|--------|--------|----------|
| `/dev/binder` | binder | App ↔ Framework |
| `/dev/hwbinder` | hwbinder | HIDL HALs (legacy) |
| `/dev/vndbinder` | vndbinder | Vendor ↔ Vendor |

Android 14+ pushes towards **AIDL HALs over `/dev/binder`** (still labeled separately by SELinux); `hwbinder` is deprecated.

### 2.2.3 Anatomy of a Binder Call

```
App Process (client)             system_server (server)
─────────────────────            ──────────────────────
ActivityManager.getTasks()
  └─ IActivityManagerProxy.getTasks()
       └─ Parcel data, reply
            └─ ioctl(/dev/binder, BC_TRANSACTION, ...)
                 │
                 ▼  kernel
            binder_thread_write()
            wakes target thread in server
                 │
                 ▼
            BR_TRANSACTION delivered
            └─ IActivityManagerStub.onTransact(code, data, reply)
                 └─ ActivityManagerService.getTasks()
                 └─ reply.writeTypedList(...)
            └─ ioctl(BC_REPLY, ...)
                 │
                 ▼
            client wakes, parses reply
       └─ returns to caller
```

### 2.2.4 AIDL — How You Actually Define Interfaces

`frameworks/base/core/java/android/content/pm/IPackageManager.aidl` (excerpt):

```aidl
package android.content.pm;
import android.content.pm.PackageInfo;

interface IPackageManager {
    PackageInfo getPackageInfo(String packageName, long flags, int userId);
    String[] getPackagesForUid(int uid);
    void grantRuntimePermission(String packageName, String permName, int userId);
    @nullable ParceledListSlice<ResolveInfo> queryIntentActivities(
            in Intent intent, String resolvedType, long flags, int userId);
}
```

The build system runs `aidl` which generates `IPackageManager.Stub` and `IPackageManager.Stub.Proxy`. The Stub lives in the server, the Proxy in the client.

### 2.2.5 Stable AIDL (Android 11+)

Stable AIDL adds versioning. Used for **HAL** AIDL and Mainline modules:

```blueprint
aidl_interface {
    name: "android.hardware.example",
    vendor_available: true,
    srcs: ["android/hardware/example/IFoo.aidl"],
    stability: "vintf",          // or "system_ext", omitted for unstable
    backend: { ndk: { enabled: true }, cpp: { enabled: false } },
    versions: ["1", "2"],        // frozen versions in aidl_api/
}
```

Once a version is frozen, **you cannot change it**. New methods go in the next version. CTS `vts_treble_vintf_test` enforces this.

### 2.2.6 Death Recipients

A client must handle the server dying:

```java
binder.linkToDeath(new IBinder.DeathRecipient() {
    @Override public void binderDied() {
        // re-acquire service, recover state
    }
}, 0);
```

🐞 **Common Production Bug:** `system_server` crash leaves apps with stale binder proxies. Apps that don't implement `linkToDeath` keep calling `DeadObjectException`. Always re-bind on death for long-lived service references.

### 2.2.7 Performance: Oneway, Buffers, Threads

- **`oneway`** methods do not block the caller; they consume one slot in the server's 1 MB buffer per process. Saturating the buffer = `TransactionTooLargeException`.
- The default Binder buffer per process is **1 MB minus 8 KB**. A 1 MB Parcel will fail.
- The default thread pool size is **15** (set by `ProcessState::setThreadPoolMaxThreadCount`). For `system_server` it's higher.

🎯 **Staff-Level Insight:** When you see "Binder transaction failed" or `TransactionTooLargeException`, the fix is rarely "increase the buffer." The real fix is: paginate (cursor-style), use `ParcelFileDescriptor` for large blobs, or stream via shared memory (`HardwareBuffer`, `Ashmem`).

### 2.2.8 Tools

```bash
adb shell dumpsys -l                   # list all services
adb shell dumpsys activity service     # AM details
adb shell ps -A | grep -i binder
adb shell cat /sys/kernel/debug/binder/state
adb shell cat /sys/kernel/debug/binder/transactions
adb shell strace -p <pid> -e ioctl     # see binder ioctls
```

---

## Chapter 2.3 — ServiceManager

ServiceManager is **the Binder of Binders** — the rendezvous point. It is itself a Binder service at handle `0`.

Source: `frameworks/native/cmds/servicemanager/`.

### 2.3.1 What It Does

- Holds a map: `name (String) → IBinder`.
- Three methods: `addService`, `getService`, `listServices`.
- Enforces SELinux on every operation: a domain must be allowed to `add` or `find` a specific service type (`*_service`).

`service_contexts` files map service names to types:

```
# system/sepolicy/private/service_contexts
activity                 u:object_r:activity_service:s0
package                  u:object_r:package_service:s0
SurfaceFlinger           u:object_r:surfaceflinger_service:s0
```

### 2.3.2 Code Flow

Server (Java):
```java
ServiceManager.addService("my_service", new MyService());
```

Client:
```java
IBinder b = ServiceManager.getService("my_service");
IMyService svc = IMyService.Stub.asInterface(b);
```

C++ equivalents in `libbinder` use `defaultServiceManager()`.

🐞 **Common Production Bug:** Adding a new service and getting `service "foo" not found` from clients. 90% of the time: you forgot to update `service_contexts` (SELinux); the call was silently denied with a non-fatal denial in dmesg.

---

## Chapter 2.4 — Core Services Tour

### 2.4.1 ActivityManagerService (AMS)

Source: `frameworks/base/services/core/java/com/android/server/am/`.

Responsibilities:
- Process lifecycle (start, kill, OOM adj).
- Activity stack (now `ActivityTaskManagerService` since Android 10).
- Broadcasts (queues, ordered/unordered).
- ContentProvider acquisition.
- Permissions enforcement at runtime.

Key classes: `ActivityManagerService`, `ProcessRecord`, `BroadcastQueue`, `OomAdjuster`.

🛠️ **Hands-On:**
```bash
adb shell dumpsys activity processes      # all running, oom_adj per process
adb shell dumpsys activity broadcasts     # broadcast queue state
adb shell dumpsys activity services       # bound services
adb shell am force-stop com.example.foo
adb shell am start-activity -n com.example.foo/.MainActivity
```

### 2.4.2 PackageManagerService (PMS)

Source: `frameworks/base/services/core/java/com/android/server/pm/`.

Responsibilities:
- Parses `AndroidManifest.xml` for every installed APK.
- Maintains `/data/system/packages.xml` (the truth-of-record for installed packages).
- Resolves intents (`queryIntentActivities`).
- Manages permissions (in conjunction with `PermissionManagerService` since Android 10+).
- Drives `Installer` (native daemon at `/system/bin/installd`).

```bash
adb shell pm list packages -f
adb shell pm path com.android.settings
adb shell pm dump com.android.settings | head -40
adb shell pm grant com.example.foo android.permission.CAMERA
```

### 2.4.3 WindowManagerService (WMS)

Source: `frameworks/base/services/core/java/com/android/server/wm/`.

Responsibilities:
- The **window** is the unit of display real-estate (an Activity has one or more windows).
- Z-order, focus, input routing.
- Animations (transitions, recents).
- Display topology (multi-display since Android 10, freeform, picture-in-picture).

WMS talks to **SurfaceFlinger** (a native service, separate process) over Binder. SurfaceFlinger composes layers using HWC (Hardware Composer HAL) onto the display.

```
App ──draws into──▶ Surface ──BufferQueue──▶ SurfaceFlinger ──HWC HAL──▶ Display
                       ▲
                       │ allocated by ──── WindowManagerService
```

```bash
adb shell dumpsys window
adb shell dumpsys SurfaceFlinger
adb shell wm size                # display size
adb shell wm density             # DPI override
```

🎯 **Staff-Level Insight:** When jank happens, **never start at the app**. Start with `dumpsys SurfaceFlinger --latency <window>` and `perfetto` traces. ~70% of "app jank" reports are GPU contention, HWC fallback to GPU composition, or display VSYNC misconfiguration in the BSP.

---

## Chapter 2.5 — Writing a Custom System Service

We will build `MyEchoService`: a system service that echoes strings, demonstrating end-to-end framework wiring.

### 2.5.1 Goals

- AIDL interface accessible from any app (with a custom signature permission).
- Service registered in `system_server`.
- Java client API in `android.app.MyEchoManager`.
- SELinux policy.
- Manifest entries.

### 2.5.2 Define the AIDL

`frameworks/base/core/java/android/myecho/IMyEchoService.aidl`:
```aidl
package android.myecho;

interface IMyEchoService {
    String echo(String input);
    int getCallCount();
}
```

Add it to `frameworks/base/Android.bp` under the `framework-minus-apex` aidl srcs (Android 14 pattern; on older releases edit `Android.mk`).

### 2.5.3 Implement the Service

`frameworks/base/services/core/java/com/android/server/myecho/MyEchoService.java`:
```java
package com.android.server.myecho;

import android.content.Context;
import android.myecho.IMyEchoService;
import android.os.Binder;
import android.util.Slog;

import com.android.server.SystemService;

public final class MyEchoService extends SystemService {
    private static final String TAG = "MyEchoService";
    private final BinderImpl mImpl = new BinderImpl();
    private int mCallCount = 0;

    public MyEchoService(Context ctx) { super(ctx); }

    @Override public void onStart() {
        Slog.i(TAG, "Publishing my_echo");
        publishBinderService("my_echo", mImpl);
    }

    private final class BinderImpl extends IMyEchoService.Stub {
        @Override public String echo(String input) {
            getContext().enforceCallingOrSelfPermission(
                "android.permission.MY_ECHO", "echo");
            mCallCount++;
            return "echo: " + input;
        }
        @Override public int getCallCount() { return mCallCount; }
    }
}
```

### 2.5.4 Register in SystemServer

`frameworks/base/services/java/com/android/server/SystemServer.java`, in `startOtherServices`:

```java
t.traceBegin("StartMyEchoService");
mSystemServiceManager.startService(MyEchoService.class);
t.traceEnd();
```

### 2.5.5 Manager API for Apps

`frameworks/base/core/java/android/myecho/MyEchoManager.java`:
```java
package android.myecho;

import android.annotation.SystemService;
import android.content.Context;
import android.os.RemoteException;

@SystemService(Context.MY_ECHO_SERVICE)
public final class MyEchoManager {
    private final IMyEchoService mService;
    public MyEchoManager(IMyEchoService s) { mService = s; }
    public String echo(String s) {
        try { return mService.echo(s); }
        catch (RemoteException e) { throw e.rethrowFromSystemServer(); }
    }
}
```

Register in `frameworks/base/core/java/android/app/SystemServiceRegistry.java`:
```java
registerService(Context.MY_ECHO_SERVICE, MyEchoManager.class,
    new CachedServiceFetcher<MyEchoManager>() {
        @Override public MyEchoManager createService(ContextImpl ctx) {
            IBinder b = ServiceManager.getServiceOrThrow(Context.MY_ECHO_SERVICE);
            return new MyEchoManager(IMyEchoService.Stub.asInterface(b));
        }
    });
```

Add `MY_ECHO_SERVICE = "my_echo"` constant in `Context.java`.

### 2.5.6 SELinux Policy

`system/sepolicy/private/service_contexts`:
```
my_echo    u:object_r:my_echo_service:s0
```

`system/sepolicy/public/service.te`:
```
type my_echo_service, system_server_service, service_manager_type;
```

`system/sepolicy/private/system_server.te`:
```
allow system_server my_echo_service:service_manager add;
```

### 2.5.7 Permission Definition

`frameworks/base/core/res/AndroidManifest.xml`:
```xml
<permission android:name="android.permission.MY_ECHO"
            android:protectionLevel="signature|privileged" />
```

### 2.5.8 Build & Test

```bash
m -j$(nproc)
adb sync && adb reboot
adb logcat -s MyEchoService
# from a privileged app or shell:
adb shell service call my_echo 1 s16 hello
```

### 2.5.9 Mistakes to Avoid

⚠️ **OEM Pitfall:** Adding the service in `vendor/` and trying to register it in `system_server` — you cannot. `system_server` lives on the system partition; services on vendor must run in their own vendor process and expose AIDL HAL with `stability: "vintf"`.

🐞 **Common Production Bug:** Forgetting to add the service to `compatibility_matrix.xml` / `manifest.xml` if it is part of the VINTF-stable surface; CTS will fail with `vts_treble_vintf_test`.

---

## Chapter 2.6 — Framework Modification Best Practices

### 2.6.1 Don't Fork — Hook

If you need vendor behavior, use:
- **Config overlays** (`device/<vendor>/<product>/overlay/`) for resources.
- **Runtime Resource Overlays (RROs)** for boot-time resource changes.
- **`SystemConfig` permissions** (`/etc/permissions/*.xml`) for feature/lib declarations.
- **`Optional<>` interfaces** with vendor implementations behind a flag.

Forking `frameworks/base` creates a permanent merge debt. Every monthly security patch becomes a multi-day exercise.

### 2.6.2 Patch Discipline

- One logical change per CL.
- `git format-patch` and `git rebase` are your friends.
- Use `Change-Id` (gerrit hook) so amendments preserve identity.
- Annotate vendor patches with `// VENDOR: <ticket>` comments — you will thank yourself in 18 months.

### 2.6.3 The "@hide" and "@SystemApi" Distinction

- `@hide` — internal to platform, can change any release.
- `@SystemApi` — stable API for **system apps** (privileged, signed with platform key).
- Public API — stable forever.

If you add API for an OEM app, prefer `@SystemApi` with a `@FlaggedApi` (Android 14+) flag.

### 2.6.4 Mainline (APEX) Considerations

Many subsystems are now Mainline modules: ART, Tethering, MediaProvider, Bluetooth, Wifi, Connectivity, Permission, IPsec, etc. They live in `packages/modules/<name>/` and ship as **APEXs** updated via Play.

⚠️ **OEM Pitfall:** Modifying a Mainline module's source breaks the Google-signed APEX. You will lose security updates for that subsystem on field devices. Customize via the documented extension points only (e.g., Bluetooth: `BluetoothInCallService` is overlay-able; the stack itself is not).

🎯 **Staff-Level Insight:** When asked to "add feature X to Bluetooth," push back: is X really framework, or is it an app on top of `BluetoothAdapter`? Mainline module modifications should be the last resort.

---

## Chapter 2.7 — Verifying Level 2

You should be able to:

1. Trace an app API call → Binder → service → HAL by name and file path.
2. Read a `dumpsys activity processes` and identify which app is at OOM_ADJ_FOREGROUND.
3. Define a stable AIDL interface and explain version freezing.
4. Add a custom system service end-to-end on Cuttlefish.
5. Explain why forking framework/base is bad and list 3 alternatives.

---

➡️ Continue to **[Level 3 — HAL & Native Layer](./level-03-hal-native.md)**

