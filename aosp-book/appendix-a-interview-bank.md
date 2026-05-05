# Appendix A — Staff‑Level AOSP / AAOS Interview Question Bank

> *"Interviews don't test what you know. They test how fast you can show what you know, under pressure, in a way that lets the interviewer write 'hire' with confidence. This appendix is the drill ground."*

This appendix is a **curated question bank** with model answers, ordered by Level. Every question is one I have either **been asked**, **asked others**, or **seen in a Staff debrief**. The depth of each model answer reflects the bar at a serious OEM (Volvo, GM, Stellantis, Ford, BMW, Hyundai), Tier‑1 (Bosch, Continental, Aptiv, Harman, LG VS), platform vendor (Qualcomm, NVIDIA, Samsung S.LSI), or platform-level FAANG (Google AOSP / Android Auto / Pixel, Meta Reality Labs, Amazon Devices).

Use it three ways:

1. **Flashcard mode.** Cover the answer, force a 2-minute response, compare.
2. **Mock mode.** Pick 5 random questions across levels; answer aloud as if interviewing.
3. **Depth audit.** If you cannot give a *model-answer-grade* response to ≥ 80% in any level's section, that level is your weak point. Re-study before the loop.

---

## A.0 — Foundations

### A.0.1 What is the difference between a Linux process and an Android app process?

A Linux process is just a `task_struct` with its own address space. An **Android app process** is *also* a Linux process, but additionally:
- Forked (not exec'd) from **Zygote**, inheriting a pre-loaded Java runtime, framework classes, and many warm pages (CoW saves hundreds of ms and tens of MB per app).
- Confined by an **app-specific UID** (10000+) and by **SELinux** domain `untrusted_app_X` (per-target-SDK variants).
- Has a **seccomp-bpf** filter narrowing the syscall surface.
- Joined to **cgroups** (cpu, memory, freezer) controlled by `system_server`.
- Lives in a **PID namespace** scoped per-user via `user namespaces`-equivalent isolation (Android does not enable Linux user namespaces; isolation is at SELinux + UID level instead).
- Will be **killed by LMKD** under memory pressure, ranked by `oom_score_adj` set by `system_server` based on app importance.

A Staff answer connects this to *why*: Zygote saves cold-start time, SELinux contains exploits, LMKD enables the "every app is potentially backgroundable" UX.

### A.0.2 Walk me through `init` from kernel handoff to first userspace service.

Kernel finishes early init, mounts initramfs, executes `/init` (PID 1, the AOSP `init` binary in `system/core/init/`). That binary then:
1. Mounts `tmpfs`, `proc`, `sysfs`, `selinuxfs`, `cgroup`, `debugfs`.
2. Sets up kernel log reader.
3. Loads SELinux policy (`/sys/fs/selinux/load`); transitions itself out of `kernel` domain.
4. Reads first-stage `/init.rc` (in ramdisk).
5. Switches root to `/system` via `switch_root` (when using system-as-root).
6. Re-execs itself in second stage; loads `/system/etc/init/*.rc`, `/vendor/etc/init/*.rc`, `/odm/etc/init/*.rc`.
7. Starts the **property service** (a separate thread).
8. Walks the action queue, executing triggers (`on early-init`, `on init`, `on fs`, `on post-fs`, `on post-fs-data`, `on boot`).
9. Forks `service` entries — `servicemanager`, `hwservicemanager` (legacy), `vold`, `surfaceflinger`, `zygote`, `system_server` (via Zygote), etc.
10. Enters supervisor loop: reaps SIGCHLD, restarts services per their `oneshot`/`critical`/`restart` flags.

A common interviewer follow-up: *"What happens if `init` itself dies?"* → Kernel panics with `Attempted to kill init`. There is no init recovery. This is why `init`'s crash hardening matters.

### A.0.3 Explain Android's property system. Why isn't it just a config file?

Android props are a **shared-memory key-value store** (`/dev/__properties__`) writeable only by the property service (a thread inside `init`), readable by anyone. Clients use `__system_property_get()` which mmaps the area and reads atomically.

Why not a config file?
- **Atomicity & speed.** Reads are wait-free; thousands of consumers read continuously without I/O.
- **Notification.** Clients can subscribe to changes (`__system_property_wait`).
- **Access control via SELinux.** Property *names* are mapped to SELinux types in `property_contexts`; only allowed domains may set them. This makes `setprop persist.sys.foo` from an app fail at the kernel level, not in userspace.
- **Persistence semantics.** Names prefixed `persist.*` are written to `/data/property/`; `ro.*` are immutable post-boot; `ctl.*` are control commands to `init` (`ctl.start`, `ctl.stop`).

⚠️ Common Staff probe: *"What's the security risk of widening a property type?"* — A `vendor` process gaining `set` on a `system`-owned property can influence platform behavior across the Treble boundary. CTS `VtsTrebleSysProp` enforces ownership.

### A.0.4 Cgroups in Android — concrete uses.

- **cpu cgroup (`/dev/cpuctl`).** Foreground apps in `top-app`, background in `background`. Different `cpu.shares` and CPU set (foreground gets all cores; background often pinned to little cluster).
- **cpuset.** Background apps restricted to `cpuset.cpus = 0-3` (little cores) on big.LITTLE; foreground gets `0-7`.
- **memory cgroup (legacy v1, v2 on newer).** Tracks per-app RSS for LMKD scoring; PSI signals.
- **freezer (`/sys/fs/cgroup/freezer/...` or v2 `cgroup.freeze`).** **Critical for App Standby and Cached App Freezer (Android 12+).** Frozen apps consume zero CPU and zero binder transactions get processed (queued instead).
- **blkio / io.** Throttle background I/O so foreground stays responsive.

The **Cached App Freezer** is a frequent Staff question: *"How do you debug an app that doesn't receive a binder reply when frozen?"* Answer: ATOMIC freezer logic in `binder.c` returns `-EAGAIN` to senders if recipient is frozen; the sender must opt into deferred delivery (`oneway` only) or expect the call to fail.

### A.0.5 Android vs Embedded Linux — name three architectural differences and three consequences.

Differences:
1. **No glibc.** Bionic libc — smaller, no full POSIX locale, different threading details, `pthread_cancel` is absent. Means: porting Linux daemons often requires patching.
2. **Init system is custom, not systemd.** No `systemctl`, no socket activation in the systemd sense, but Android `init` does service supervision and lazy-start.
3. **Userspace is non-FHS.** `/system`, `/vendor`, `/product`, `/system_ext`, `/data` instead of `/usr`, `/etc`, `/var`. Read-only system; immutable in production.

Consequences:
1. Vendor C/C++ daemons compiled against glibc need a Bionic port.
2. Service supervision idioms (auto-restart, dependencies) translate but require `.rc` rewrites.
3. Configuration must live in writable `/data` or per-partition; `/etc` *is* read-only in Android.

---

## A.1 — AOSP Basics

### A.1.1 What's in the manifest, and why is `repo` not just `git submodule`?

A `repo` manifest is XML listing **projects**, each `(name, path, remote, revision)`. `repo sync` clones each project into its specified path. Why not submodules?
- AOSP has ~700+ projects; submodule operations are O(N) and slow.
- Manifests support **`<copyfile>` / `<linkfile>`**, **groups** (e.g., `device,vendor`), and **conditional sync** (`--all`, group-based).
- Manifests are themselves git-versioned in their own repo (`.repo/manifests`); a tagged manifest = a precise tree state.

Staff follow-up: *"How do you guarantee a 5-year-later rebuild produces the same bytes?"* → manifest pinned to project SHAs (not branches), toolchain version pinned in `prebuilts/clang/`, mirrored manifest repo, container image of the build environment, and an archived `out/` for the gold build to bit-compare.

### A.1.2 Soong vs Make in AOSP — what's the actual delta?

Soong (`build/soong/`) is the modern AOSP build system. It reads `Android.bp` (Blueprint, a JSON-like declarative format) and generates a **Ninja** build graph. Why migrate from Make?
- **Hermetic.** No arbitrary shell snippets in build files.
- **Parallel.** Module evaluation is parallel and deterministic.
- **Incremental.** Ninja's dependency model is more accurate than Make's; avoids spurious rebuilds.
- **Validated.** Module types are typed; misconfigurations fail at parse time, not link time.

Make is still present (`Android.mk`) for legacy modules; Soong handles ~99% of new code. There is a third layer: `kati` translates remaining Makefiles to Ninja so the whole tree is one Ninja invocation.

Staff probe: *"What does `m` actually do?"* → `m` is a bash function from `build/envsetup.sh` that calls `build/soong/soong_ui.bash --make-mode <targets>`. Soong UI runs Soong, runs kati for legacy makefiles, merges, then invokes Ninja.

### A.1.3 First build hangs at "host_javac" — what's the diagnostic path?

1. `top` / `htop` — is it CPU-bound (compiling) or stuck (D-state)?
2. `ps -ef | grep java` — how many javac/jvm processes? Memory?
3. `df -h` — `/tmp`, `out/`, `~`. Common: `/tmp` full from prior failed builds.
4. `ulimit -n` — file descriptor exhaustion on big builds (`8192+` recommended).
5. `dmesg | tail` — OOM killer hits javac.
6. Disable parallelism: `m -j1` to isolate.
7. `out/soong.log`, `out/build.log`, `out/error.log` for the actual stack.
8. Check Java heap: `_JAVA_OPTIONS` or `ANDROID_JACK_VM_ARGS` set elsewhere.

Staff move: name the **observability** path before the **fix** path. Anyone can guess; only seniors triage methodically.

### A.1.4 `lunch` in 60 seconds.

`lunch <product>-<variant>` sets:
- `TARGET_PRODUCT` — drives `device/<vendor>/<product>/`, deciding which makefiles, sepolicy, packages.
- `TARGET_BUILD_VARIANT` — `eng` (debuggable, all modules), `userdebug` (debuggable, slimmer), `user` (production; non-debuggable).
- Implicit `TARGET_ARCH`, `TARGET_CPU_ABI`.
- Exports `ANDROID_PRODUCT_OUT`, etc.

Staff trivia: variants change *what is built*, not just signing. `user` strips `dumpstate`'s extra fields, removes `adb root`, and enables some compile-time security flags.

---

## A.2 — Framework Internals

### A.2.1 ServiceManager — how does a process find a service?

`servicemanager` is a tiny native daemon (`frameworks/native/cmds/servicemanager/`) holding a `vector<Service>` mapping name → Binder handle. It is the **only** Binder service whose handle is well-known: handle `0`. Everything else is found via `IServiceManager.getService(name)` which is a Binder call to handle `0`.

Steps for a client to talk to `activity`:
1. Open `/dev/binder` (already done by `ProcessState`).
2. Construct `Parcel` with the string `"activity"`.
3. `BC_TRANSACTION` to handle `0`, code `GET_SERVICE_TRANSACTION`.
4. Driver delivers to `servicemanager`'s waiting thread.
5. `servicemanager` looks up in its map, returns Binder handle (with `flat_binder_object` of type `BINDER_TYPE_HANDLE`).
6. Driver translates handle in receiver's space → fresh handle in caller's space.
7. Caller wraps handle in `BpBinder`, returns to Java as `IBinder`.
8. Subsequent `IActivityManager.foo()` calls go directly to that handle.

Staff probe: *"What if the service hasn't registered yet?"* → `getService` returns null; common AOSP idiom is `waitForService` which polls + sleeps for ~5 s. *"Why isn't there race-free registration?"* → there is, since Android 11: `IServiceManager.waitForService` blocks until registered.

### A.2.2 Binder one-way vs two-way — when do you choose which?

| | Two-way | One-way |
|--|---------|---------|
| Caller blocks? | Until reply | Returns immediately on enqueue |
| Buffer | Shared 1 MB pool | Separate, smaller async queue |
| Ordering | N/A | Per-target ordered |
| Failure visibility | Exception/result | Silent unless explicit ack |
| Use for | Read APIs, anything needing result | Notifications, callbacks, fire-and-forget |

Choose **one-way**:
- Callbacks from system_server to apps (avoids system_server blocking on a janky app).
- HAL → framework events (e.g., key events from input HAL).
- Cross-process broadcasts where loss is recoverable.

Choose **two-way**:
- Read APIs (`getCallingUid`, `getRunningTasks`).
- Authorization checks where caller must wait for verdict.

Staff trap: *"What happens if a one-way call's recipient is frozen (Cached App Freezer)?"* → enqueued in driver; delivered on unfreeze. *"What if the queue overflows?"* → driver drops with `BR_TRANSACTION_PENDING_FROZEN` and sender gets failure on a subsequent call.

### A.2.3 `system_server` is one process holding ~80 services. Why? Pros and cons.

**Why one process:** binder calls within `system_server` short-circuit (in-process, no IPC); memory shared across services (e.g., `PackageManager`'s package cache served to `ActivityManager` without copy); single watchdog.

**Pros:**
- Fast inter-service communication.
- Coherent memory view.
- Single restart point.

**Cons:**
- One service's bug → whole `system_server` reboots → device UI restart (~5–10 s, sometimes treated as a soft reboot).
- One service's leak → entire `system_server` OOM.
- Hard to isolate ownership for partner-shipped code.

Staff move: discuss the trend — `network_stack`, `permission_controller`, `media_provider` have moved **out** of `system_server` into APEX/separate processes for blast-radius control. This is the *direction*, even if the bulk is still monolithic.

### A.2.4 The Zygote fork model — why `fork()` and not `exec()`?

`fork()` without `exec()` preserves the parent's address space (CoW). Zygote pre-loads the JVM, framework classes, common resources. App startup then = fork + light reset (clear logging, refresh thread name, drop caps, set UID, set seccomp filter, start app's `Application.onCreate`). Skipping `exec` saves:
- ~150 ms of JVM startup.
- ~20–40 MB of pre-loaded classes (CoW shared with all children).
- File descriptors are explicitly closed; capabilities explicitly dropped.

Risks: CoW unsharing under heavy memory pressure (every write to a shared page makes a copy), and **fork safety** of native libraries — anything holding a lock at fork time is undefined in the child. Hence the long list of `Zygote.preload()` rules and the prohibition on starting threads in `static` initializers loaded by Zygote.

Staff trivia: there are **multiple Zygotes** — `zygote64`, `zygote32`, `webview_zygote` (for isolated WebView renderers), `app_zygote` (for isolated services). Each preloads a different set tailored to its workload.

### A.2.5 Write a custom system service end-to-end. (Whiteboard.)

```
1. AIDL: frameworks/base/core/java/android/foo/IFooService.aidl
   interface IFooService { String hello(); }
2. Manager class: frameworks/base/core/java/android/foo/FooManager.java
   public class FooManager {
     private final IFooService mSvc;
     public FooManager(IFooService s) { mSvc = s; }
     public String hello() { return mSvc.hello(); }
   }
3. Service impl: frameworks/base/services/core/java/com/android/server/foo/FooService.java
   public class FooService extends IFooService.Stub {
     @Override public String hello() {
       // Permission check
       getContext().enforceCallingPermission("android.permission.FOO", "hello");
       return "hi from " + Binder.getCallingUid();
     }
   }
4. Register in SystemServer.java:
   FooService foo = new FooService(context);
   ServiceManager.addService("foo", foo);
5. Bind manager in SystemServiceRegistry:
   registerService(Context.FOO_SERVICE, FooManager.class, ...);
6. Permission: declare in frameworks/base/core/res/AndroidManifest.xml
7. SELinux: add foo_service to service_contexts; allow callers binder_call(...);
8. CTS test: hostsidetests or device-side instrumentation hitting the API.
9. Build: m services framework-minus-apex && m
```

Staff candidates also state: *"This adds an `@SystemApi`. I'll need API council review, an `@hide` lint exemption if needed, and a CDD entry if it changes compatibility."*

### A.2.6 Watchdog kills `system_server`. What was it watching?

`com.android.server.Watchdog` runs a thread that pings each registered `HandlerThread` every 30 s. If a handler doesn't reply within **60 seconds** (configurable), Watchdog dumps stacks of every thread in `system_server` plus held locks, writes to `/data/anr/`, and calls `Process.killProcess(myPid())`.

Common causes of Watchdog kill:
1. **Lock held across binder call.** Service A holds `mLock`, calls into vendor HAL synchronously, HAL hangs.
2. **Disk I/O on main handler.** `PackageManagerService` scanning a malformed APK on main thread.
3. **Native deadlock.** JNI call into a library taking a process-global lock that another thread holds via the framework.

Investigation: read the `system_server_watchdog` dropbox entry, find the stuck thread, walk the lock graph. Look at the *holder*, not the *waiter*.

---

## A.3 — HAL & Native

### A.3.1 HIDL vs AIDL HAL — when, why, and what's actually different?

HIDL (Android 8–11) was a custom IDL with its own runtime (`libhidltransport`, `hwservicemanager`). AIDL HAL (Android 12+) reuses the existing AIDL framework with **stability annotations** (`@VintfStability`).

Concrete differences:
- **Runtime.** HIDL: `hwservicemanager` + `libhidlbase`. AIDL: `servicemanager` (same as framework) + `libbinder_ndk`.
- **Versioning.** HIDL: `@1.0`, `@1.1` interface suffixes; vendor freezes at major version. AIDL: snapshot frozen via `aidl_api/<iface>/<ver>/`; CI fails on unfrozen change.
- **Generation.** HIDL: bespoke C++ codegen. AIDL: same backend as framework (Java, NDK, Rust).
- **Memory.** AIDL HAL's NDK backend is leaner; less per-call overhead.
- **Marshalling.** HIDL had `safe_union` and unique semantics; AIDL has `union` (Android 12+) and parcelables.

Why migrate: tooling (Rust support, fuzzing scaffolding), ABI consistency, killing a parallel IDL universe. As of Android 13+, **no new HIDL HALs are accepted upstream**. Existing ones stay until shadow-deprecated.

### A.3.2 What is VINTF and how does it gate boot?

VINTF = Vendor Interface. Two XML files anchor it:

- `compatibility_matrix.<ver>.xml` (system) — declares HALs the system **requires**, with allowed version ranges and feature flags.
- `manifest.xml` (vendor) — declares HALs the vendor **provides**, with versions.

At boot, `init` (or earlier, the `apexd`/`vintf` checker) runs `libvintf::checkCompatibility()`. If vendor doesn't satisfy system's matrix → boot **halts** with an error (or in a more permissive build, logs and continues).

Concrete failure: system matrix requires `android.hardware.audio@2.0-7.0` interface; vendor provides only `@5.0`. New system requires `@6.0`; mismatch → boot fail. Fix: bump vendor to provide `@6.0` HAL or pin system to require ≤ `@5.0` (only feasible in pre-launch).

Staff probe: *"How do GSI tests rely on this?"* GSI = Generic System Image with the **latest** matrix. Pairing it with an OEM vendor partition validates that vendor's manifest covers the GSI's matrix. This is the on-device proof of Treble compliance.

### A.3.3 Vendor HAL crashes repeatedly. What's the recovery and what does the framework see?

`init` restarts the HAL service per its `.rc` (`restart_period 5s`, `oneshot off`). After N consecutive crashes within M seconds, init may give up depending on `critical` flag.

Framework side: a `IHwBinder.DeathRecipient` (HIDL) or `AIBinder_DeathRecipient` (AIDL) registered by the framework client fires. The client typically:
1. Logs `dropbox` event with HAL name.
2. Marks feature unavailable until rebind.
3. Rebinds via `getService(name)` with retry.

Staff probe: *"What if the framework client itself blocks waiting for rebind?"* → it must not. The death callback runs on a binder thread; rebind on a worker. Synchronous rebind on a callback thread risks priority inversion and is a common cause of `system_server` Watchdog hits when a HAL dies.

### A.3.4 Design a thermal mitigation HAL with a framework feedback loop.

**Goal.** Vendor reads thermal sensors; framework decides throttling policy; HAL applies cooling actions (CPU freq cap, GPU cap, fan control).

**Design.**

```
Sensor sysfs ─► thermal HAL (vendor) ─AIDL─► ThermalService (framework)
                                                   │ policy (CDD-defined zones)
                                                   ▼
                                            HardwarePropertiesManager + apps
                                                   │ recommendation
                                                   ▼
                                          thermal HAL .applyThrottle(...)
                                                   ▼
                                          devfreq, cpufreq, GPU governor
```

Decisions:
- **Vendor reports temperatures, system decides actions.** Avoid policy in HAL (10.3 anti-pattern).
- **AIDL HAL with `@VintfStability`** so vendor freeze applies.
- **Callback model** for sensor events (`IThermalChangedCallback`); framework subscribes once, no polling.
- **Framework computes throttle level** based on thermal zones, current workload, user state (gaming/charging), regional regulation.
- **HAL has `applyThrottle(zone, level)`** which translates to vendor-specific knobs.
- **Failure mode:** HAL reports `THROTTLING_FAILED`; framework escalates to next mitigation (e.g., shut down camera).
- **CTS:** framework throttling behavior (CDD §3.X). **VTS:** HAL interface conformance.

This is the answer for "design a HAL" in any subsystem — replace "thermal" with audio/camera/sensor and the structure holds.

---

## A.4 — BSP / Bring-Up

### A.4.1 You receive a new SoC + reference board. Outline the bring-up plan.

```
Week 0–1: SoC vendor BSP delivered (kernel tree + binary blobs + tooling)
Week 1–2: Build their kernel; boot to console; verify storage, USB, UART
Week 2–4: Integrate AOSP device/<vendor>/<board>/; first init shell
Week 4–6: Display, touch, basic input working; can see SystemUI
Week 6–10: Cellular (if applicable), Wi-Fi, BT, audio, camera (ISP basics)
Week 10–14: Sensors, GNSS, NFC, biometrics
Week 14–18: SELinux from permissive → enforcing; CTS-on-bench begins
Week 18–24: VTS pass; pre-cert CTS pass
Week 24–32: Performance tuning, power, thermal
Week 32–40: GMS cert submission, beta program
```

Concrete artifacts to lay down on day 1:
- `device/<vendor>/<board>/BoardConfig.mk` — partition sizes, kernel cmdline, AVB key, AB.
- `device/<vendor>/<board>/<board>.mk` — `PRODUCT_PACKAGES`, copyfiles for firmware.
- `vendor/<vendor>/<board>/proprietary/` — extracted blobs.
- `kernel/<vendor>/<chip>` — kernel tree pinned to a SHA.
- `device/<vendor>/<board>/sepolicy/` — empty initially, grown by hand.

Staff candidates: *"What blocks I expect."* Driver upstreaming gaps; vendor ships out-of-tree drivers that haven't been tested with newer LTS kernels; device tree overlays inconsistent with Treble (vendor must own its DT subset).

### A.4.2 Why is `/vendor` a separate partition?

Treble. Vendor partition contains anything SoC/board-specific:
- Vendor HALs.
- Vendor `.rc` and `init` extensions.
- Firmware blobs.
- Vendor SELinux labels.
- Vendor manifest XML.

System partition can be replaced (GSI, major OS upgrade) without touching vendor. The interface between them is enforced by VINTF + frozen AIDL/HIDL + sepolicy public API.

Staff trivia: there's also `/odm` (ODM differentiation, e.g., a board built by an ODM with vendor SoC), `/product` (carrier/SKU customization), `/system_ext` (system extensions that are still part of "system" but separately built). Knowing which goes where is a Staff-level partition discipline question.

### A.4.3 What is dm-verity, and how does it interact with OTAs?

`dm-verity` is a Linux device-mapper target that verifies every disk block on read against a Merkle tree whose root hash is signed by the AVB key. Read of a tampered block → I/O error → process killed; in production, often a kernel panic.

Interaction with OTA:
- OTA writes to inactive slot, then re-generates the Merkle tree, hashes it, embeds root in `vbmeta` for that slot.
- On reboot to new slot, bootloader verifies vbmeta signature, kernel sets up dm-verity with the new root.
- `update_engine` cannot do partial in-place writes to the **active** partition; A/B is necessary because verity makes mutation impossible.

For non-A/B (legacy), `dm-verity` is bypassed for `/data` only. `/system` mutations would invalidate verity → impossible to write while booted from it.

### A.4.4 What changes between Cuttlefish and a real SoC?

| Aspect | Cuttlefish | Real SoC |
|--------|-----------|----------|
| Kernel | upstream LTS + Cuttlefish virt drivers | vendor fork with hundreds of out-of-tree patches |
| HALs | software defaults (AOSP `default` impls) | vendor implementations talking to real silicon |
| Display | virtio-gpu | dedicated display controller, MIPI/HDMI, multiple planes |
| Storage | virtio-blk | UFS or eMMC with vendor-specific quirks |
| Power | none | PMIC, DVFS, thermal, battery |
| Boot loader | crosvm minimal | EDK2/U-Boot/SoC ROM with eFuse keys |
| Cell/GNSS/sensors | absent or simulated | real radios with regulatory constraints |

This is why "passes on Cuttlefish" ≠ "ready to ship" but is the right floor for **AOSP correctness** of the system half. Vendor half must be tested on hardware.

---

## A.5 — Android Automotive

### A.5.1 The full bring-up flow for a new VHAL property — narrate it.

1. **Decide.** System property in `VehicleProperty.aidl`, or vendor property with `VEHICLE_PROPERTY_GROUP_VENDOR` bit. New use case → vendor; pattern matches an existing system property → use it.
2. **Define.** If vendor: `propId = 0x2000 | TYPE | AREA_TYPE | VEHICLE_PROPERTY_GROUP_VENDOR`. Pick `change mode`, `access`, `area config`, `min/max`.
3. **Implement** in vendor VHAL service (e.g., subclass of `IVehicleHardware`): `getValues`, `setValues`, `subscribe`, `unsubscribe`. Wire to underlying transport (CAN, SOME-IP, sysfs).
4. **Register** in `DefaultConfig.json` or vendor equivalent so the VHAL service exposes the property at `getAllPropConfigs`.
5. **Allowlist** in `CarPropertyService` — vendor properties default to system-only; if exposing to non-system app, add to permissions XML.
6. **SELinux** — vendor VHAL's domain already covers the binder service; if reading new sysfs node, label it.
7. **Test** — `cmd car_service inject-vhal-event <propId> <areaId> <value>` for unit-level; `MockedVehicleHal` in CarService tests; on-device with the actual ECU.
8. **Documentation** — internal API docs; if a system property addition, propose to AOSP.

Staff probe: *"What if the property's data shape changes next year?"* Don't. Define a new propId. Stable propIds are a contract.

### A.5.2 Multi-display + multi-user + audio routing — wire it together.

Goal: passenger watches video on rear display with their headphones; driver listens to navigation on front speakers; both are simultaneously logged-in users.

Mechanisms:
- **`CarOccupantZoneService`** maps display, audio zone, input device → occupant zone. Configured in `car_occupant_zone_configuration.xml`.
- **`UserManager` with visible background users.** User 11 (passenger) is started but not "the" current user (which remains user 10, driver). Both are running.
- **`CarUserService`** assigns user 11 to passenger zone via `assignVisibleUserToOccupantZone`.
- **`CarAudioService`** honors per-zone routing. Audio streams from user 11's apps tagged with zone 1; routed to audio HAL device addressed `bus1` per `car_audio_configuration.xml`.
- **Activity launch.** Launcher in user 11 launches activities with `setLaunchDisplayId(rearDisplayId)`. WindowManager honors and SurfaceFlinger composes on that display.
- **Audio HAL** must implement `getDevicesForStream` per zone. Multi-zone aware HAL is a vendor responsibility.

Staff trap: if your audio HAL is single-zone, all of the above won't help — passenger audio will mix into driver speakers. The HAL is the bottleneck.

### A.5.3 ASIL — what is in/out of Android's safety boundary?

ASIL ratings (ISO 26262): QM, A, B, C, D. Higher = more safety-critical. Determined by hazard analysis (severity × exposure × controllability).

In Android: QM (no safety claim) for AAOS in general; some component suppliers claim ASIL-A for narrowly scoped safety-supportive functions.

Out of Android (in a separate ECU/MCU/safety domain):
- ABS, ESC, airbags, steering, braking — ASIL D.
- Powertrain safety logic — ASIL C/D.
- ADAS perception/decision (depends; up to ASIL D for AEB).
- Cluster speedometer when used as primary speed indicator — typically ASIL B (legal requirement to display correct speed).

Pattern: **hypervisor split.** A safety-certified Type-1 hypervisor (Greenhills INTEGRITY, BlackBerry QNX, ACRN) hosts an Android guest plus an AUTOSAR/QNX guest for the cluster and safety functions. Communication via shared memory or vsock. Android cannot violate the safety domain; safety domain treats Android as untrusted.

If asked *"Why can't Android display the speedometer?"* — it can, as a *secondary* display. The *primary* speedometer must be either non-Android or backed by an ASIL-rated fallback that takes over on Android failure (kernel panic detection from the safety MCU).

### A.5.4 Why is `CarService` not in `system_server`?

Crash isolation. `CarService` runs in `com.android.car` as a separate process. Reasons:
- A vendor-shipped Car module can crash without taking down `system_server` (and thus the UI).
- It allows OEM extension: vendor `.jar`s loaded into `com.android.car` via `ICarServiceHelper`, isolated from framework state.
- Different lifecycle: `CarService` participates in `CarPowerManagementService` state machine (S2RAM, SHUTDOWN_PREPARE), which is car-specific and differs from phone's Doze model.

It still binds to `system_server` for coordination (`ICarServiceHelper`), but its services live behind their own binder bindings (`car_service` registered in service manager, `cmd car_service` for shell).

### A.5.5 Garage Mode — what runs and why is its budget bounded?

Garage Mode is the post-shutdown task window between user "turn off" and full sleep. Runs:
- `update_engine` apply (A/B finalize).
- `dexopt` of newly installed APKs.
- `CarTelemetryService` log batch upload.
- `BackupManagerService` periodic backup.
- OEM-registered `CarPowerStateListener` workers (e.g., model update for ML features).

Budget enforced by VHAL `AP_POWER_STATE_REPORT` — Android tells vehicle "I need N more minutes"; vehicle ECU grants a budget based on 12V battery state, scheduled wake time, regulatory constraints. When budget elapses, Android transitions to sleep regardless of incomplete work.

Staff candidates know: this budget is **per-vehicle-event**, not infinite. An OTA spanning Garage Mode budget + multiple ignition cycles must be *resumable* — `update_engine` is, but custom OEM workers often aren't. That's a class of bug.

---

## A.6 — Performance & Memory

### A.6.1 Methodology for reducing cold-start time of an app.

1. **Measure first.** Cold start = process start to first frame (`am start -W` reports `TotalTime`). Capture Perfetto trace covering before-`am start` to first frame (`-a <pkg> sched freq am wm view binder`).
2. **Decompose.** Look for: process fork (Zygote), `bindApplication`, `Application.onCreate`, `Activity.onCreate`, view inflation, first measure/layout/draw, GPU upload, first present.
3. **Common wins (in observed frequency order):**
   - `Application.onCreate` doing synchronous work (DB init, SharedPreferences, network bootstrap, analytics SDK init). Defer to background.
   - Static initializers loading the world. Lazy-init.
   - Heavy XML inflation. Migrate to compose, or use `AsyncLayoutInflater`.
   - First-frame BinderService calls. Cache or move off critical path.
   - dexopt missing → app interpreted on first launch. Verify `pm compile -m speed-profile`.
   - GPU shader compile. Pre-warm with the first scene's shaders.
4. **Validate.** Re-trace, confirm gain, regression-test.
5. **Lock the gain.** Add a startup-time benchmark in CI; alert on regression.

Staff candidates state numbers: *"baseline 1200 ms, target <800 ms; biggest single win was 220 ms by deferring Firebase init from `Application.onCreate` to a `WorkManager` job at first idle."*

### A.6.2 LMKD scoring — what determines who dies first?

LMKD ranks processes by `oom_score_adj` (OSA), set by `system_server` (`ProcessList`). Ranges:
- `-1000` — `init`, never killed.
- `-900` — `system_server`, `surfaceflinger`.
- `0` — foreground app.
- `100–200` — visible / perceptible.
- `200–500` — backgrounded.
- `900+` — cached, killed first.

Under memory pressure (PSI signals on modern Android, `vmpressure` legacy), LMKD walks processes in OSA-descending order, killing until pressure abates. It chooses based on:
1. OSA bucket.
2. Within bucket, RSS (kill the biggest first to free more).
3. Some recency heuristics for thrashing avoidance.

Staff probe: *"Why might a foreground app still be killed?"* Severe pressure (e.g., camera capture spike); LMKD will kill foreground rather than let the kernel OOM-killer fire (which is less predictable). Also: the foreground app may have been recently moved to perceptible because of a UI transition.

### A.6.3 Perfetto categories you reach for daily, and what they show.

- **`sched`** — runqueue, who ran when, on which CPU. Find priority inversion, big.LITTLE migration cost.
- **`freq`** — CPU/GPU frequency. Find DVFS lag (frequency too low when needed).
- **`binder`** — every transaction with sender/receiver, code, duration. Find unexpected sync calls, IPC storms.
- **`am`** — ActivityManager events: process start, lifecycle. Cold-start spans.
- **`wm`** — WindowManager: window add/remove, focus, transitions.
- **`view`** — Choreographer, draw, measure/layout. Jank analysis.
- **`gfx`** — SurfaceFlinger frame composition, GPU buffer queue.
- **`memory`** — RSS sampling, ION/dmabuf allocations.
- **`power`** — wakelocks, suspend states.
- **`disk`** — I/O latency, queue depth.

Staff move: enumerate the *5* you would enable for *this* class of bug, not all 30. Demonstrates you can target.

### A.6.4 Boot-time optimization — name three structural levers.

1. **Parallelize service start in `init`.** Default is sequential per-trigger. Use `class_start` with parallel-friendly classes; mark non-critical services `oneshot` and `disabled`, start later.
2. **Defer dexopt of non-essential apps.** `pm bg-dexopt-job` runs in idle; ensure first-boot critical apps are pre-opt'd in build (`PRODUCT_DEXPREOPT`), defer the rest.
3. **Trim Zygote preload.** `frameworks/base/config/preloaded-classes` — every class loaded costs CoW pages. Audit and remove unused.

Bonus: **filesystem layout.** Frequently-read files (boot anim, fonts) co-located reduces seek time on eMMC. F2FS atomic-write for `/data` reduces fsck time on dirty shutdown. Compressing `/system` with EROFS reduces I/O at boot at the cost of CPU decompression — net win on flash-bound boards.

### A.6.5 OOM in `system_server` — investigation playbook.

1. Pull `dropbox` `system_server_lowmem`, `system_server_native_crash`.
2. From `bugreport`: `dumpsys meminfo --proto`, locate `system_server` PSS, native heap, dalvik heap, graphics.
3. Pull heap dump: `am dumpheap system_server /data/local/tmp/ss.hprof`.
4. Analyze with Android Studio Memory Profiler or `hat`. Look for:
   - Strong references holding contexts.
   - Listener registries growing unboundedly (apps registering, never unregistering).
   - Bitmap caches (icon cache, thumbnail cache).
   - PackageManager package list duplication on reinstall.
5. Cross-check with `dumpsys procstats --hours 24` — services growing over time = leak; spikes = burst load.
6. Reproduce on Cuttlefish with `am-restrict-background-data` + heavy app churn; confirm.

Staff move: distinguish *leak* (monotonic growth) from *bloat* (high steady-state). Different fixes.

---

## A.7 — Security

### A.7.1 Walk me through verified boot from power-on to first user activity.

1. Power-on; CPU executes ROM code (immutable, in silicon). ROM verifies bootloader signature against eFused public key hash.
2. Bootloader runs; verifies its own configuration (anti-rollback fuse vs version stored in storage).
3. Bootloader loads `vbmeta`; verifies signature against AVB key (also fused).
4. `vbmeta` contains hash descriptors for `boot.img` (kernel + ramdisk), and hashtree descriptors for `system`, `vendor`, `product`, etc.
5. Kernel boots; `init` runs; mounts root with dm-verity using root hash from vbmeta. Any tampered block → I/O error → halt.
6. SELinux loads; `init` re-execs into restricted domain.
7. `zygote`, `system_server`, `surfaceflinger`, etc. start.
8. First user activity (lock screen) draws.

Each step verifies the next. The chain is broken by:
- Unlocking bootloader (ORANGE state; subsequent images verified with whatever key, allowed to boot but tagged).
- Replacing AVB key with developer key (YELLOW; warning shown).
- Tampering with verified partition (RED; halt).

Staff probe: *"What stops an attacker who has root from disabling verity?"* Verity is enforced by kernel, not userspace. Disabling requires kernel modification, which requires re-signing `boot.img`, which requires the AVB key. With root but no key, you cannot persist — next reboot restores integrity.

### A.7.2 SELinux denial: write a `.te` rule to fix it (correctly).

```
avc: denied { read } for pid=812 comm="my_hal" name="config" dev="vendor"
   ino=12345 scontext=u:r:my_hal:s0 tcontext=u:object_r:vendor_file:s0 tclass=file
```

**Wrong fix:** `allow my_hal vendor_file:file r_file_perms;` — opens all of `/vendor` to this HAL.

**Right fix:**

```
# 1. Label the specific config file.
# in device/<vendor>/<board>/sepolicy/file_contexts:
/vendor/etc/my_hal_config\.xml   u:object_r:my_hal_config_file:s0

# 2. Declare the new type.
# in device/<vendor>/<board>/sepolicy/my_hal.te:
type my_hal_config_file, file_type, vendor_file_type;

allow my_hal my_hal_config_file:file r_file_perms;
allow my_hal my_hal_config_file:dir search;
```

Staff candidates explain: *"I narrow the type because (a) sepolicy is a least-privilege contract; (b) `vendor_file` covers thousands of files I have no business reading; (c) `neverallow` rules forbid widening."* Then verify by re-running and seeing the denial replaced by success, *not* by enabling permissive.

### A.7.3 What's a synthetic password and why does it exist?

The synthetic password (SP) is an internal secret derived from the user's credential (PIN/pattern/password) **plus** a Keymint-bound key, used to wrap FBE keys.

Why: separates "user can change credential" from "data must be re-encrypted." Without SP, changing PIN would require re-wrapping all FBE keys. With SP:
- Credential change → unwrap SP with old credential, re-wrap with new. FBE keys unchanged.
- Reset escrow → SP can be reconstructed by enterprise admin via escrow key (with policy).
- Forgotten credential → SP cannot be reconstructed → CE storage permanently lost. By design.

This also makes "factory reset" cryptographic: destroy SP wrappings → ciphertext on disk is unrecoverable even with Keymint cooperation.

### A.7.4 Mainline security advantages and risks.

Advantages:
- Patches ship via Play in days vs months for full OTA.
- A 0-day in `conscrypt` (TLS) gets fixed across the fleet without OEM coordination.
- Reduces "patch gap" between ASB publication and device install.

Risks:
- A buggy module update can affect *every* device with that APEX. Google has a staged rollout, but mistakes happen.
- OEMs that fork Mainline modules opt out — they get the full responsibility back.
- APEX signing key compromise = fleet compromise. Hence Google's hardware-backed signing for these.

Staff move: Mainline is *not free*. CTS and MTS run on every module update; the OEM still validates against their build. Skipping MTS = ship-with-bug risk.

### A.7.5 Design question: detect a rooted device.

There is no perfect detection, only heuristic layers:
1. **Hardware-backed key attestation.** Cleanest. Server gets attestation chain; if root cert is not Google attestation root, or if `verified_boot_state` is not `VERIFIED`, device is suspect. This is what banking apps do.
2. **`SafetyNet` / Play Integrity** — runtime checks bundled in Google's library, attests to device integrity to a Google service.
3. **Filesystem checks** — `su` binary present, `/system` writable, etc. Easily defeated by Magisk and friends.
4. **Process tree checks** — `zygote` parent, suspicious LD_PRELOAD. Easily defeated.

Best answer: layer 1 (hardware attestation) is the only one that scales. Layers 3–4 are anti-pattern (cat-and-mouse). Don't write your own; use Play Integrity if available, otherwise raw KeyMint attestation.

Staff probe: *"What if the OEM doesn't ship Play?"* Then you ship your own server-side attestation verifier, rooted in the OEM's prod attestation key (provisioned per-device at the factory). No shortcut.

---

## A.8 — Testing & Compatibility

### A.8.1 Difference between CTS, VTS, GTS, MTS, STS — in two sentences each.

- **CTS** validates the public Android API behavior; required for GMS; ~1 M tests across modules.
- **VTS** validates the vendor side: HAL conformance to AIDL/HIDL spec, kernel config, Treble compliance.
- **GTS** is closed-source Google Test Suite for GMS-specific behavior (Play services, Google apps integration).
- **MTS** validates Mainline modules; per-module test suites that gate Play-pushed module updates.
- **STS** validates that known CVE patches are applied; STS-dynamic for monthly-rotating set, STS-pre-submit for older.

Staff move: name what each does *not* cover. CTS doesn't cover OEM customization or carrier-specific behavior. VTS doesn't cover HAL *quality*, only conformance. STS doesn't cover *unknown* vulnerabilities (that's where pen-test and fuzzing come in).

### A.8.2 You inherit a Tradefed test suite that is "flaky." Methodology to fix.

1. **Quantify.** Run the suite N=100 times; rank tests by failure rate. Flaky ≠ broken; >5% intermittent = flaky.
2. **Categorize causes:**
   - **Test ordering.** State leaked from prior test. Fix: per-test reset; `@Before`/`@After` setUp/tearDown.
   - **Timing.** `Thread.sleep(500)` racing real condition. Fix: `await condition`, idling resources.
   - **Resource contention.** Shared device state (single Bluetooth radio, single camera). Fix: serialize, mock.
   - **External dependencies.** Network, NTP. Fix: mock, or mark `@RequiresNetwork` and gate.
   - **Test infra.** Tradefed sharding bug, adb instability. Fix: harness investigation.
3. **Quarantine.** Tag flaky tests `@Flaky`; exclude from blocking signal until fixed. Track in JIRA.
4. **Burn-down.** Per quarter, fix top-10 flakiest. Measurable SLO: "<2% test flake rate."

Staff move: *"Flake is debt; treat it like a deprecation campaign with deadlines, not a 'keep retrying' habit."*

### A.8.3 Write a CTS test for a hypothetical platform feature.

(See Level 8 §8.2.4. The interview wants you to walk Android.bp + Java + assertions + CDD reference + how to run. State *out loud*: "I would also add this test to the relevant CTS module's `AndroidTest.xml` with the right `<option name="test-suite-tag" value="cts">` so it runs with `run cts`.")

### A.8.4 Treble VTS regression: GSI fails to boot. Triage.

1. `dmesg | grep -i vintf` — VINTF compatibility check failure usually says which interface mismatched.
2. `vintf check-compat` on host with vendor manifest + GSI matrix.
3. Compare HAL versions in vendor manifest vs GSI matrix requirements.
4. If a vendor HAL is at older minor version than GSI requires: bump vendor or get a waiver (rare).
5. If a vendor HAL is at newer major version than GSI matrix accepts: GSI is too old; this is a vendor-future-proof bug — vendor cannot mandate a brand-new HAL until system matrices catch up.
6. If sepolicy mismatch (vendor uses a system type that doesn't exist in old policy): violation of Treble — vendor must only use `public/` types of the targeted version.

The Staff move: *"GSI failing is a feature, not a bug — it's the CI signal that Treble is broken. The fix is on the vendor side, never on GSI."*

---

## A.9 — Production & Release

### A.9.1 OTA pipeline — name every key signing event from source to device.

1. **CL signing** (Gerrit): every commit signed by author's GPG key, verified by Gerrit.
2. **Manifest tagging:** signed git tag (Gerrit-side or manual) on the manifest commit.
3. **Build artifact signing:** `target_files.zip` is unsigned at build time (built with default test keys, immediately re-signed by `sign_target_files_apks` with HSM-backed prod keys).
4. **APK signing:** v2/v3 signature scheme, per-app key (platform, media, shared, networkstack, OEM apps).
5. **APEX signing:** APEX file signing key + `apk-in-apex` per-app signing.
6. **Vbmeta signing:** AVB key, written into `vbmeta_*.img` partitions.
7. **OTA package signing:** `otacerts.zip` in `/system/etc/security/`; OTA package signed by OTA key whose pub-half is in that zip.
8. **Mainline updates:** signed by Google + OEM if forked.
9. **Device-side verification:** bootloader verifies vbmeta; recovery verifies OTA package signature against `otacerts.zip`; `update_engine` verifies per-block hashes.

Staff move: *"Every step here is on a chain of custody from HSM to device. Compromise of any single key has a different blast radius — vbmeta is fleet-wide, an APK key is one app. We size the protections accordingly."*

### A.9.2 Field crash spike post-OTA — your first hour.

1. **Confirm.** Telemetry dashboard: crash rate Δ vs prior 7-day baseline, scoped to new fingerprint. Statistical significance (not just one outlier).
2. **Halt rollout** at server. This is the single most important first action. Capping exposure beats any clever fix.
3. **Triage by signature.** Group crashes by tombstone signature / ANR signature / kernel panic OOPS. Top-1 usually accounts for >50%.
4. **Identify cohort.** Is it all devices? One SKU? One region? One carrier? One Mainline module version? Cohort tells you the cause class.
5. **Hypothesis.** Form one hypothesis grounded in the diff (`git log <prev>..<new>`). Don't read all 1000 commits; read commit titles and pick suspects.
6. **Communicate up.** One-page status to org leadership: scope, halt status, hypothesis, ETA to fix or rollback.
7. **Decide path:** hotfix incremental OTA in 24–48 h, or full rollback to prior build. Rollback is faster but burns your rollback fuse window.
8. **Postmortem after.** Always. Blameless. Action items tracked.

Staff move: *"In hour one I focus on bounding damage, not finding the bug. The bug investigation can take 48 hours; the rollout halt cannot wait."*

### A.9.3 Symbol archival — what exactly do you keep, for how long?

Per shipped fingerprint:
- `out/target/product/<board>/symbols/` — full unstripped binaries, ~5–10 GB.
- `out/target/product/<board>/obj/KERNEL_OBJ/vmlinux` — kernel symbols.
- All `.dwp` / `.dwarf` debug info for Soong-built modules.
- `system/build.prop`, `vendor/build.prop` — for `ro.build.fingerprint` matching.
- Toolchain manifest (`prebuilts/clang/host/linux-x86/clang-r<rev>/`) for re-symbolization with the exact compiler.
- Source manifest (manifest XML pinned to SHAs).

Retention: as long as you support the device. Automotive: 10–15 years. Phone: 5–7 years. Storage cost is irrelevant compared to a single un-debuggable field issue.

Staff move: *"I'd containerize the build environment too. In year 7, the host OS that built this image won't compile it anymore. I want a Docker image of the build host, archived alongside symbols."*

### A.9.4 A/B vs Virtual A/B — when do you choose which?

| | A/B | Virtual A/B |
|--|-----|-------------|
| Storage cost | ~3–5 GB (duplicated partitions) | ~0 (snapshot in `userdata`) |
| Implementation complexity | Lower | Higher (dm-snapshot + COW + merge) |
| Update apply time | Faster (direct write) | Slightly slower (snapshot) |
| Reboot time after update | Same | Slightly slower (merge first boot) |
| Required for | All Android 11+ launches | Compressed VAB on Android 12+ |

Choose **traditional A/B** when storage is plentiful (>32 GB devices) and you want simplicity. Choose **Virtual A/B** when storage is tight (entry-level phones, specific automotive).

Staff trivia: VAB has a "compressed" variant where the COW data is compressed (lz4/zstd) further reducing storage. Default for new launches.

---

## A.10 — Staff Mindset & Behavioral

### A.10.1 Tell me about a 10-year decision you've made.

(Behavioral; the interviewer is checking three-horizon thinking.)

Good answer template: name the decision, name the alternatives at the time, name the *predicted* second-order effects, name what *actually* happened years later, name what you learned. Include a measurable artifact.

Bad answer: "I used Kotlin instead of Java because Kotlin is the future." (No tradeoffs, no horizon, no measurable consequence.)

### A.10.2 You disagree with a Google AOSP design direction (e.g., a deprecation). What do you do?

1. **Read the rationale first.** Source.android.com release notes; commit messages; design docs if public. Often the reasoning is sound and unfamiliar to me.
2. **Stage your disagreement at the right venue.** Public AOSP issue tracker for a bug; AOSP mailing lists for direction; through your Google TAM if you have OEM relationship.
3. **State your case with data.** "This deprecation will affect N apps in our fleet; here is telemetry; here are migration costs."
4. **Propose alternatives.** Compatibility shim? Phased deprecation? Documentation?
5. **Accept the decision.** Google owns AOSP direction. If they decline, you build a downstream mitigation; you don't fork unless absolutely forced to (forking AOSP is a 10-year commitment).

Staff candidates show: *"I've disagreed with X. Here's how I raised it. Here's where it landed. Here's how I adapted."* Real specificity beats abstract grandstanding.

### A.10.3 Mentoring — describe a specific instance.

Not "I helped engineers." Specific: name (anonymized), starting state, what they were stuck on, the *intervention*, the *follow-up*, the *outcome* with metric.

> *"Engineer A, senior, stuck on a Binder deadlock for 3 days. I sat with them, didn't take the keyboard. Asked: what's the call graph? What does Watchdog dump show? What's the holder vs waiter? They identified a `system_server` lock held during a HAL call. We landed a fix in 2 hours. I then asked them to write a postmortem and to add a lint check that flags HAL calls under that lock. The lint shipped, caught two more occurrences in the next quarter, and Engineer A presented it at our platform offsite. Six months later they led the cached-app-freezer migration."*

Staff = the multiplier. Show it.

### A.10.4 You shipped a bug. Walk me through the postmortem.

Template that interviewers grade highly:

1. **Timeline.** UTC timestamps, every event from first symptom to resolution.
2. **Impact.** Quantified: # devices, # users, $ if known, regulatory exposure.
3. **Root cause.** *Technical* root cause + *process* root cause. The technical RC is "null deref"; the process RC is "we didn't have a test for the empty-case." Both must be named.
4. **Detection.** How did we find out? Customer report? Telemetry? CI? (CI is best; customer is worst.)
5. **Mitigation.** What we did to stop the bleeding.
6. **Action items.** Each owned, with deadline. *"Add test"* without owner is not an action.
7. **What went well.** Always include. (e.g., "Halted rollout in 12 minutes; on-call response was crisp.")
8. **Lessons.** Stated as principles applicable beyond this incident.

Staff candidates emphasize the **process** RC and the principle. *"The bug was a null deref. The lesson was: any code path that reads a vendor property must default-handle null because vendors will ship empty values. We codified it in our review checklist."*

### A.10.5 Why this company / this role?

(Standard, but easy to fumble at Staff level.)

Bad: "Great team, great mission." Generic.

Good: A specific architectural problem the company is facing, that you've thought about, and that you can articulate a Staff-level approach to. *"Your fleet just crossed 10 M vehicles; you're entering the maintenance phase where every architectural decision is paid back at scale. I've shipped that transition once before; I want to do it again with your team because of [specific thing they're doing]."*

Demonstrates: research, judgment, ownership.

---

## A.11 — The Final Mock Loop

For self-testing. Time-bound. Speak aloud or write.

| # | Question | Time |
|---|----------|------|
| 1 | Walk through Android boot from `init` to first frame. | 8 min |
| 2 | Design a system-wide screen-recording service with privacy and AAOS support. | 45 min |
| 3 | A device boot-loops post-OTA on 0.1% of fleet. First hour. | 15 min |
| 4 | Why is `CarService` not in `system_server`? Where would you change that? | 8 min |
| 5 | Tell me about saying no to a senior leader. | 6 min |
| 6 | Implement a HAL property end-to-end (whiteboard). | 30 min |
| 7 | SELinux denial in vendor. Write the right fix and explain why. | 12 min |
| 8 | Debug a 10% jank regression in Android 14 → 15 on your fleet. | 15 min |
| 9 | What invariants would you write into your platform's CI? | 10 min |

**Pass bar:** all 9 answered with depth, specificity, and *with tradeoffs stated*. Two weak ones = re-prep that area before the loop.

---

## A.12 — Closing Note for the Reader

You will not be asked all of these. You will be asked **some**. The point of preparation is not to memorize answers; it is to develop the **reflex** to reason from invariants, name tradeoffs, and produce a recommendation in real time.

The Staff bar is calm depth under ambiguity. This appendix is your gym. Train it for two weeks. Then, on loop day, **be the engineer the team needs at 2 a.m.** — and let them see it.

⬅️ Back to **[Table of Contents](./README.md)**

