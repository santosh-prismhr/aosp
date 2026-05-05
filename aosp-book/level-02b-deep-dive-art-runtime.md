# Level 2B — Deep Dive: ART Runtime
🔗 Continue to [Level 3 — HAL & Native](./level-03-hal-native.md) and [L3A Deep Dive: HAL Subsystems](./level-03a-deep-dive-hal-subsystems.md).

5. Trace a `pm install` to dexopt to first launch on a stopwatch.
4. Diagnose a bitmap-fragmentation case with `dumpsys meminfo` and `am dumpheap`.
3. Force a profile, recompile with it, and prove via `oatdump` that hot methods are AOT'd.
2. Read an `art:I` GC log line and identify pause time vs total time.
1. Explain the difference between `verify`, `speed-profile`, `speed` and pick the right one for: a system app, a third-party app on a low-RAM device, a boot-critical service.

You can finish Phase 3 of the curriculum when you can:

## ✅ Verifying this chapter

---

```
cmd package bg-dexopt-job
profman --dump-only --reference-profile-file=...
/data/misc/profiles/ref/<pkg>/primary.prof
/data/misc/profiles/cur/<uid>/<pkg>/primary.prof
```text
### 📋 Cheat-sheet

1. **[Staff]** How would you detect that `bg-dexopt-job` regressed in the field? *Telemetry on first vs subsequent launches; `dumpsys package dexopt`; OTAs that disable `dexopt-secondary` masking results.*
### 🎓 Interview Questions

- A pinned `cur.prof` from a beta build into a release — bad hot-method set, regresses launch.
- Shipping `dirty-image-objects` referencing internal classes that get renamed — ART falls back to slow path silently.
### ⚠️ Pitfalls

```
cf:# pm compile -m speed-profile -f com.example.app
cf:# chown system:system /data/misc/profiles/ref/com.example.app/primary.prof
$ adb push primary.prof /data/misc/profiles/ref/com.example.app/primary.prof
          --reference-profile-file=primary.prof
          --apk=app.apk --dex-location=base.apk \
$ profman --create-profile-from=methods.txt \

EOF
HSPLcom/example/Foo;->baz(I)Ljava/lang/String;
HSPLcom/example/Foo;->bar()V
$ cat > methods.txt <<'EOF'
```bash

### 🛠️ Code Lab — Seed a profile manually

Cloud Profiles (Play): Play Store ships an aggregate `cloud.prof` with the APK; `installd` seeds local `ref.prof` on install.

4. Next launch: `.odex` contains AOT'd hot methods; cold methods stay in DEX/JIT.
3. The `bg-dexopt-job` (system idle, plugged in, > 10% battery) invokes `dex2oat -m speed-profile --profile-file=ref.prof`.
2. `profman` merges `cur.prof` → `ref.prof` periodically.
1. While the user runs the app, ART JIT collects a **current profile** (`cur.prof`).

### 📐 Concept

PGO is what makes Android apps feel snappier at week 2 than week 1, **without** the OEM doing anything. Misconfiguring it (or shipping stale profiles) silently disables it.
### 🟦 Why it matters

## §2B.3 Profile-Guided Optimization in Practice

---

```
adb shell setprop dalvik.vm.usejit false   # disable JIT (debug)
adb logcat -s art:I                        # GC log lines
adb shell am dumpheap -n <pid> <out>       # native heap (libc malloc)
adb shell am dumpheap <pkg> <out>          # Java heap
adb shell dumpsys meminfo <pkg>            # heap, native, GL, ashmem
```text
### 📋 Cheat-sheet

3. **[Staff]** A bitmap-heavy app shows fragmentation pressure on Android 11. Diagnose. *Check LOS occupancy via `dumpsys meminfo`; consider `BitmapFactory.Options.inBitmap` reuse; `setHasFixedSize` for RecyclerView; ensure CC generational is on (`dalvik.vm.useartservice`).*
2. **[Senior]** Why is generational mode only viable with concurrent copying? *Need a forwarding mechanism for moved young-gen objects without an STW pause.*
1. **[Senior]** What changed when ART moved from CMS to CC? *Eliminated full-heap STW; moving collector enables compaction and TLAB; lower fragmentation.*
### 🎓 Interview Questions

- JNI `NewGlobalRef` leaks — invisible to Java profilers; need `dumpsys meminfo` + `am dumpheap -n` (native).
- Holding bitmaps in `static` fields — they become roots of the image space, never collected.
- Calling `System.gc()` in production — bypasses heuristics, schedules a non-generational pass.
### ⚠️ Pitfalls

```
cf:# logcat -d -s art:I | tail
cf:# am send-trim-memory com.android.calculator2 RUNNING_CRITICAL
```bash
Force-trigger and observe:

```
# Open calc-std.hprof in Android Studio → Profiler → Memory
$ hprof-conv calc.hprof calc-std.hprof          # tool from Android SDK
$ adb pull /data/local/tmp/calc.hprof
cf:# am dumpheap com.android.calculator2 /data/local/tmp/calc.hprof
```bash

### 🛠️ Code Lab — Heap dump & analysis

Decode: paused = STW pause (good if < 5 ms); total = wall time (CC overlaps with mutator); 33% free is post-GC.
```
I art   : Background concurrent copying GC freed 8765(1024KB) AllocSpace objects, 12(48KB) LOS objects, 33% free, 16MB/24MB, paused 1.234ms total 78.901ms
```
Sample log line:

- `CollectorTransition` — switching strategies
- `NativeAlloc` — `RegisterNativeAllocation` pressure
- `Explicit` — `System.gc()`
- `Background` — heuristic, idle
- `Alloc` — heap exhaustion at allocation
GC trigger types (seen in `logcat`):

```
+---------------------+
|   bump-pointer TLAB |  per-thread fast alloc
+---------------------+
|   main (region)     |  young + old regions, copying
+---------------------+
|   image space       |  boot.art + app.art (ZygoteSpace)
+---------------------+
|   non-moving        |  classes, intern strings
+---------------------+  large object space (LOS, mmap discontiguous)
```

ART today defaults to **Concurrent Copying (CC)** GC with a generational mode (Android 11+). Heap layout:

### 📐 Concept

Jank at scroll-time on a midrange device is *almost always* GC. Knowing which collector ran, why, and how to read its log line in `logcat` is table stakes for performance work.
### 🟦 Why it matters

## §2B.2 ART Garbage Collection

---

```
adb logcat -b all -s installd dex2oat profman
adb shell dumpsys package dexopt
ls /data/dalvik-cache/$(getprop ro.product.cpu.abi)/
dexdump -d <.dex>                           # readable bytecode
profman --dump-only --reference-profile-file=<.prof> --apk=<.apk>
oatdump --oat-file=<.odex> --header-only    # filter, isa, deps
cmd package bg-dexopt-job                   # run idle job now
pm compile -m <filter> -f <pkg>            # force a filter
```text
### 📋 Cheat-sheet

(Linked: [Appx A Q-110 → Q-118](./appendix-a-interview-bank.md))

5. **[Staff]** Trade-offs of `speed` vs `speed-profile` for system_server's classpath. *Fully AOT'd boot classpath cuts ~200 ms but inflates `/system` and harms dirty pages; profile-guided is the modern choice with `dirty-image-objects`.*
4. **[Staff]** A device shows 35% slower cold start after an OTA. Where do you look first in dexopt? *Check `dumpsys package dexopt` for filter regression, then `bg-dexopt-job` failures in `logcat -b system`, then `installd` denials in audit.*
3. **[Senior]** How does Cloud Profiles (Play) interact with `speed-profile`? *Play seeds a `cur.prof` from aggregate data; ART merges with local profile on background dexopt.*
2. **[Senior]** What is in a `.vdex` file and why does it accelerate verification? *Pre-verified DEX + quickening hints; lets the runtime skip re-verification when class hierarchy hasn't changed.*
1. **[Mid]** Why does Android default to `verify` then upgrade to `speed-profile`? *Answer summary: balances install-time CPU/storage against startup; profiles are cheap to gather and target only hot methods, giving ~80% of `speed`'s benefit at ~25% of the size.*
### 🎓 Interview Questions

- Running `pm compile` as `shell` and wondering why nothing happened — needs root or `installd`.
- Forgetting that A/B updates re-compile in the **inactive slot**; storage spike during OTA.
- Disabling `bg-dexopt-job` to "save battery" — first-launch latency regresses by 30–60% as profiles never compile.
- Choosing `speed` for every app on a low-storage device — `/data` exhaustion, OTA balloons.
### ⚠️ Pitfalls

Full lab tree: `curriculum/labs/art-pgo/` (drives the same flow with a sample app and a `profman --create-profile` from a method list).

🧪 **Verifying:** With `speed-profile`, `oatdump --header-only` shows `Compiler filter: speed-profile`; profile dump lists hot methods. With `verify`, the OAT file is tiny (no compiled code) and methods are `kInterpreter` at runtime.

```
cf:# dumpsys package dexopt | grep -A3 calculator2
cf:# am start -n com.android.calculator2/.Calculator
cf:# am force-stop com.android.calculator2
cf:# cmd package compile -m verify -f com.android.calculator2
```bash
Force a JIT-only run:

```
        --apk=/data/app/*/com.android.calculator2*/base.apk
        --reference-profile-file=/data/misc/profiles/ref/com.android.calculator2/primary.prof \
cf:# profman --dump-only \
        --header-only
cf:# oatdump --oat-file=/data/app/*/com.android.calculator2*/oat/x86_64/base.odex \
cf:# ls -lh /data/app/*/com.android.calculator2*/oat/x86_64/
cf:# pm compile -m speed-profile -f com.android.calculator2
```bash
Force compile with speed-profile and read the result:

```
$ adb shell pm list packages -f | grep com.android.calculator2
$ adb root && adb shell setenforce 0    # for inspection only; don't ship
```bash
Setup:

### 🛠️ Code Lab — Force `speed-profile` and inspect (Android 15)

- `*.art` — image of pre-resolved classes/strings (used at startup to avoid resolution)
- `*.vdex` — verified dex + quickening info
- `*.odex` — compiled native ELF (`.text` is `.oat` data section)
**Files emitted to `/data/dalvik-cache/<isa>/`:**

| `everything` | dev only | AOT every method, including unused ones |
| `speed` | Boot-critical apps, OEM choice | Full AOT |
| `speed-profile` | **Default after first runs** | AOT only hot methods (from `profile.prof`) |
| `space` | OTA, low-storage | Minimal AOT |
| `quicken` | Pre-Android 12 | Quickening of opcodes; deprecated |
| `verify` | Default for many apps (Android 7+) | DEX verification only, no native code; smallest |
|---|---|---|
| Filter | When chosen | What you get |

**Compiler filters** (set in `PackageDexOptimizer`, ultimately a flag to `dex2oat`):

*Figure 2B.1 — Compilation pipeline. The same `.dex` may be reached through three filters during a device's lifetime.*
```
    Zygote --> AppProc
    OAT -->|mmap| Zygote
    PROFD -->|bg-dexopt| dex2oat
    ART_JIT -->|profile.prof| PROFD[profman]
    APK -->|JIT path| ART_JIT[ART JIT]
    dex2oat -->|writes| OAT["/data/dalvik-cache/<isa>/<pkg>.odex<br/>+.vdex +.art"]
    PMS -->|installd| dex2oat
    APK[apk: classes.dex] -->|install| PMS
flowchart LR
```mermaid

### 📐 Concept

A wrong dex2oat compiler filter on a system app costs ~80 MB of `/data` and ~250 ms of cold start. The PMS-ART contract — *what filter, when, and why* — is one of the most-asked Staff-level questions because it touches OTA size, battery (idle compilation), and launch latency simultaneously.
### 🟦 Why it matters

## §2B.1 dex2oat, JIT, and AOT

---

ART is not "the JVM with a different name." It is a runtime co-designed with the framework, the package manager, and the kernel. This chapter is what you point an interviewer to when they ask *"walk me through what happens between `pm install foo.apk` and the first frame of the app's launch activity."*

> **Primary target:** Android 15 on Cuttlefish · **Audience:** Senior → Staff
> **Curriculum days:** 37–39 · **Prereq:** [L2 Framework Internals](./level-02-framework-internals.md), [L2A Binder Native](./level-02a-deep-dive-binder-native.md)

