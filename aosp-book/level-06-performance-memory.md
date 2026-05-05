# Level 6 — Performance & Memory (Staff Level)

> *"Performance work on Android is not about clever micro-optimizations. It is about understanding what the system is doing every microsecond — boot, scheduling, memory, I/O, binder — and removing the work you did not need to do in the first place."*

By this level you can build, modify, and debug AOSP. Staff-level performance work is what separates engineers who *ship* Android products from those who *struggle* to ship them. This chapter teaches the mental models, the tools, and the production patterns used at OEMs and Tier-1 silicon vendors.

---

## Chapter 6.1 — A Staff Engineer's Mental Model of Android Performance

### 6.1.1 The Four Axes

Every Android performance problem reduces to one (or a combination) of four resources:

```
┌─────────────────────────────────────────────────────────────┐
│  1. CPU         — cycles, scheduling latency, contention    │
│  2. Memory      — RSS, PSS, swap, kernel slab, ION/DMA-BUF  │
│  3. I/O         — flash bandwidth, fsync latency, F2FS GC   │
│  4. Power       — wakelocks, CPU idle residency, modem      │
└─────────────────────────────────────────────────────────────┘
```

Every metric a user perceives — boot time, app launch, scroll jank, ANRs, battery — is a *symptom* manifested across these axes. Staff engineers reason **bottom-up from the axis**, not top-down from the symptom.

### 6.1.2 The Three Time Horizons

| Horizon | Examples | Tooling |
|---------|----------|---------|
| µs–ms | scheduler latency, binder RTT, page fault | `perfetto`, ftrace, `simpleperf` |
| 10–500 ms | app cold start, frame drops, ANR precursors | Perfetto, `am start -W`, `gfxinfo` |
| seconds–minutes | boot, OTA, garage mode, thermal throttling | `bootchart`, `dumpsys batterystats`, thermal HAL logs |

### 6.1.3 The Cost Model You Must Internalize

Approximate, but burn into memory:

```
L1 cache hit                ~1   ns
L2 cache hit                ~4   ns
L3 cache hit                ~15  ns
DRAM access                 ~100 ns
Binder one-way (same SoC)   ~50  µs
Binder sync round-trip      ~150 µs   ← every getter on every API call
JNI call                    ~1   µs
Linux syscall               ~0.5 µs
fsync() on UFS              ~1–10 ms
fsync() on eMMC             ~5–50 ms
Cold app launch (warm cache)~300–800 ms
Cold boot (Cuttlefish)      ~10–20 s
Cold boot (real automotive) ~1.5–3 s to backup-camera, ~12–20 s to full IVI
```

🎯 **Staff-Level Insight:** When a peer says "let's just add a binder call," you should immediately translate that to "150 µs on the critical path, every invocation, forever." Most regressions ship because nobody priced the call.

---

## Chapter 6.2 — Boot Time Optimization

Boot time is the most-watched metric on any Android product. On automotive it is regulated (FMVSS 111, ECE R46). On phones it determines OTA reboot windows.

### 6.2.1 Boot Phases (Detailed)

```
0 ms      Power on / SoC ROM
  │       Bootloader (LK / U-Boot / ABL)
~600 ms   Kernel hand-off
  │       Kernel init, driver probe
~1500 ms  init first stage (mounts /, /vendor, /system, dm-verity)
  │       SELinux load
  │       init second stage
~2500 ms  property service, vold, servicemanager, hwservicemanager
  │       early HALs (gralloc, composer, audio, vehicle)
~3500 ms  zygote (32+64), zygote preloads ~3500 classes & resources
  │       system_server starts → ~110 services in dependency order
~6500 ms  PHASE_BOOT_COMPLETED → BOOT_COMPLETED broadcast
  │       SystemUI, Launcher, packageinstaller dexopt
~9000 ms  user-perceptible "ready"
```

### 6.2.2 Measuring

```bash
# Coarse
adb shell getprop ro.boottime.init
adb shell getprop ro.boottime.zygote
adb shell getprop sys.boot_completed
adb shell dumpsys SurfaceFlinger | grep -i "Boot"

# Fine — bootchart
adb shell 'touch /data/bootchart/enabled'
adb reboot
adb shell /system/bin/bootchart stop
adb pull /data/bootchart .
${ANDROID_BUILD_TOP}/system/core/init/grab-bootchart.sh

# Best — Perfetto boot trace
adb shell perfetto -c /data/misc/perfetto-configs/boottrace.pbtxt --txt -o /data/misc/perfetto-traces/boot.pftrace
```

A boot Perfetto config worth memorizing:

```protobuf
buffers { size_kb: 200000 fill_policy: RING_BUFFER }
data_sources {
  config { name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      ftrace_events: "sched/sched_waking"
      ftrace_events: "power/cpu_frequency"
      ftrace_events: "power/cpu_idle"
      ftrace_events: "ext4/ext4_da_write_begin"
      ftrace_events: "f2fs/f2fs_sync_file_enter"
      atrace_categories: "am" atrace_categories: "wm" atrace_categories: "binder_driver"
      atrace_apps: "*"
    }
  }
}
data_sources { config { name: "linux.process_stats" } }
data_sources { config { name: "android.log" } }
duration_ms: 30000
```

### 6.2.3 The Real Levers

Ranked by typical impact on a real product:

1. **Kernel driver probe parallelization** — async probe (`module_init` → `module_async_probe`). Often 200–600 ms.
2. **Reduce zygote preload** — `frameworks/base/config/preloaded-classes` and `preloaded-resources`. Each unused class is a class init you pay for. Profile-guided trimming (Android 12+ uses `boot-image-profile.txt`).
3. **System server service start order** — `frameworks/base/services/java/com/android/server/SystemServer.java`. `traceBeginAndSlog("StartXxxService")`. Move non-critical services into `PHASE_BOOT_COMPLETED`.
4. **dm-verity / fs-verity hashing** — reduce by sharing precomputed hashes.
5. **HAL `*.rc` `class` and `oneshot`** — start late HALs in `class late_start` rather than `class core`.
6. **dexopt at boot** — `pm.dexopt.first-boot=quicken` instead of `speed-profile`; defer hot apps to idle.
7. **CPU governor at boot** — pin to performance governor with thermal cap until `boot_completed`.

### 6.2.4 Worked Example — Saving 800 ms on Cuttlefish

🛠️ **Hands-On:**

```bash
# Baseline
adb shell getprop ro.boottime.init.selinux  # e.g. 412 ms

# 1. Move non-critical service to late
$EDITOR frameworks/base/services/java/com/android/server/SystemServer.java
# wrap startTextServicesManagerService(), startLockSettingsService() ... inside
# t.traceBegin/traceEnd and move them to startOtherServices() late phase.

# 2. Trim preloaded-classes
m build-art-boot-profile
# regenerate boot image profile from your golden trace
```

Reboot, re-trace, diff in Perfetto's **System View → Process Lifecycle** track.

🐞 **Common Production Bug:** A vendor adds a `class core` HAL that takes 1.2 s to probe (waiting for I²C). Boot regresses by 1.2 s. Fix: move HAL to `class hal` or `class late_start`, and make the underlying driver async-probe.

🎯 **Staff-Level Insight:** Never optimize boot without a flame-graph-style trace. Engineers routinely "optimize" code that wasn't on the critical path — Amdahl punishes them. The Perfetto **Critical Path** plugin (Android 14+) shows the wakeup chain that gates `boot_completed`.

---

## Chapter 6.3 — Memory Management Internals

### 6.3.1 The Android-specific Layers Above Linux MM

```
┌─────────────────────────────────────────────────────────┐
│  Java heap (per process, ART)                           │
│   ├─ image space   (boot.art, mmap'd, shared)           │
│   ├─ zygote space  (CoW from zygote fork)               │
│   ├─ main space    (per-app allocations)                │
│   └─ large object space                                 │
├─────────────────────────────────────────────────────────┤
│  Native heap (jemalloc / scudo)                         │
│  Stack(s) per thread (8 KB guard + 1 MB default)        │
│  Memory-mapped .so / .odex / .vdex / .art               │
│  Anonymous + named ashmem regions                       │
├─────────────────────────────────────────────────────────┤
│  Kernel: page cache, slab, kgsl/ION/DMA-BUF, zram       │
└─────────────────────────────────────────────────────────┘
```

### 6.3.2 RSS vs PSS vs USS — Get This Right

- **RSS** — pages resident, **double-counted** across processes for shared mappings.
- **PSS** — RSS but shared pages divided by sharer count. The metric Android uses for accounting.
- **USS** — pages unique to a process; how much would be freed by killing it.

```bash
adb shell dumpsys meminfo --proc com.android.systemui
adb shell dumpsys meminfo -a com.android.systemui   # detailed
adb shell procrank
adb shell showmap <pid>
```

`dumpsys meminfo` summary categories:

```
Native Heap, Dalvik Heap, Dalvik Other, Stack, Ashmem, Gfx dev, Other dev,
.so mmap, .jar mmap, .apk mmap, .ttf mmap, .dex mmap, .oat mmap, .art mmap,
Other mmap, EGL mtrack, GL mtrack, Unknown
```

🎯 **Staff-Level Insight:** When debugging "this app uses too much memory," always start at `dumpsys meminfo -a` and look at **Pss Total**, then **Private Dirty** — that's the killable, reclaimable cost.

### 6.3.3 Zygote, App Processes, and Copy-on-Write

The zygote is the secret of Android memory efficiency. Every app process is a `fork()` of zygote, sharing every preloaded class, resource, and `.so` until written to. CoW pages stay shared — that's why the `.art` mapping for `boot.art` is ~80 MB shared and ~kB private dirty per app.

Implication: **anything you preload in zygote is paid once**. Anything you load *post-fork* is paid per-process. This is why `preloaded-classes` is sacred and contested.

### 6.3.4 ION / DMA-BUF and Graphics Memory

Graphics buffers do not appear in process RSS. They live in DMA-BUF (was ION on older kernels) and are accounted via `dmabuf_dump`:

```bash
adb shell dmabuf_dump
adb shell dmabuf_dump --pid <pid>
adb shell cat /sys/kernel/dma_heap/system/total_pools_kb
```

🐞 **Common Production Bug:** SurfaceFlinger leaks a `BufferQueue`; DMA-BUF grows unbounded; `dumpsys meminfo` looks fine because RSS is unaffected; OOM killer slaughters background apps. Always include `dmabuf_dump` in your perf snapshots.

### 6.3.5 Kernel Page Cache and File-backed Memory

`.dex`, `.odex`, `.so`, `.apk` are mmap'd. Under memory pressure, the kernel evicts these clean pages — but the next access page-faults, causing jank. This is why **boot reading patterns matter**: if dexopt prefetches in the right order, page cache stays warm.

### 6.3.6 zram and Swap

Modern Android uses **zram** (compressed RAM swap) aggressively. `lz4` on most devices, `zstd` on newer ones. Configured by `init`:

```
on early-init
    swapon_all /vendor/etc/fstab.${ro.hardware}
```

```bash
adb shell cat /proc/swaps
adb shell cat /sys/block/zram0/mm_stat
```

The columns of `mm_stat` are: orig_size, compr_size, mem_used, mem_limit, mem_used_max, same_pages, pages_compacted, huge_pages.

A healthy compression ratio is 2.5×–3.5×. Below 2× means you're storing already-compressed data (like images) in swap — bug.

---

## Chapter 6.4 — LMKD and OOM Scoring

### 6.4.1 The Killer Hierarchy

```
┌──────────────────────────────────────────────────────┐
│ 1. lmkd  (userspace)  — proactive, PSI-driven        │
│ 2. kernel OOM killer  — last resort, never desirable │
└──────────────────────────────────────────────────────┘
```

`lmkd` (`system/memory/lmkd/`) is the userspace **Low Memory Killer Daemon**. Modern lmkd uses **PSI (Pressure Stall Information)** rather than the legacy "minfree thresholds." Properties:

```
ro.lmk.use_psi=true
ro.lmk.psi_partial_stall_ms=70
ro.lmk.psi_complete_stall_ms=700
ro.lmk.swap_free_low_percentage=20
ro.lmk.thrashing_limit=100
```

### 6.4.2 oom_score_adj — The Currency

ActivityManager assigns each process an **adj** value reflecting its importance:

```
-1000  native / persistent system
  -900  persistent (system_server)
  -800  persistent UI
   0    foreground app  (visible, in focus)
  100   perceptible foreground service (music, nav)
  200   visible
  250   perceptible
  900   cached
  906   cached + empty + LRU
```

Source: `frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java`.

```bash
adb shell cat /proc/<pid>/oom_score_adj
adb shell dumpsys activity oom    # full ranked list
```

When lmkd needs to free memory, it walks processes from highest adj down and kills until pressure recedes.

🎯 **Staff-Level Insight:** A common "my app is killed too aggressively" complaint is really "my service is not promoting the process to foreground." Check `dumpsys activity processes <pkg>` and look at the `adj=` line. If you're in `cached`, you're killable — that's correct behavior.

### 6.4.3 Reading lmkd Decisions

```bash
adb logcat -s lowmemorykiller lmkd
# example:
# lmkd : Kill 'com.foo.bar' (pid 12345) tasksize 245312kB rss 280MB,
#        adj 906, reason: LMK_KILL_REASON_PSI
```

Add to your bug reports always.

🐞 **Common Production Bug:** A vendor sets `ro.lmk.kill_heaviest_task=false` to "protect" their app. lmkd starts killing *random* mid-priority apps; users see Dialer disappear during calls. Lesson: never tune lmkd without a 72-hour soak test on real workloads.

---

## Chapter 6.5 — Tracing Tools You Must Master

### 6.5.1 Perfetto — The One Tool to Rule Them All

Perfetto (`external/perfetto/`) is the unified tracing system. It supersedes systrace and atrace as a frontend.

```bash
# Quick interactive trace (10s)
adb shell perfetto -o /data/misc/perfetto-traces/t.pftrace -t 10s \
  sched freq idle am wm gfx view binder_driver hal dalvik input res memory

adb pull /data/misc/perfetto-traces/t.pftrace
# open at https://ui.perfetto.dev
```

What to look for in the UI:

- **Top bar timeline** — CPU frequency, idle states.
- **Process tracks** — runnable, running, sleeping, uninterruptible.
- **Flow events** — connect "binder transaction" arrows between processes.
- **SQL view** — `Tools → Run SQL`. Yes, Perfetto traces are queryable as SQLite.

A killer query — *what was the longest binder transaction during boot?*

```sql
SELECT ts, dur, name FROM slice
WHERE name LIKE 'binder transaction%'
ORDER BY dur DESC LIMIT 20;
```

### 6.5.2 atrace — When You Just Need a Quick Look

```bash
adb shell atrace --async_start -b 32768 sched gfx view wm am
# ... reproduce issue ...
adb shell atrace --async_stop -o /data/local/tmp/trace.html
```

Deprecated in favor of Perfetto but still ubiquitous in older kernels and CI.

### 6.5.3 simpleperf — CPU Profiling

`development/scripts/simpleperf` for CPU sampling, including JIT/AOT symbols.

```bash
adb shell simpleperf record -e cpu-cycles -f 1000 -p $(pidof system_server) --duration 10
adb pull /data/local/tmp/perf.data
simpleperf report -g | head
```

Use case: a system_server thread eating CPU during scroll → flamegraph it.

### 6.5.4 gfxinfo and SurfaceFlinger

```bash
adb shell dumpsys gfxinfo <pkg> framestats   # last 120 frames, ns precision
adb shell dumpsys SurfaceFlinger --latency <layer>
adb shell service call SurfaceFlinger 1025 i32 1   # enable frame log
```

For jank: 16.67 ms is the budget at 60 Hz; 8.33 ms at 120 Hz. Anything over budget on **any of the four phases** (record, sync, draw, composition) drops a frame.

### 6.5.5 strace, ltrace, bpftrace

For native services, `strace -p <pid> -f -tt -T` reveals syscall costs. On modern kernels, `bpftrace` is the surgical tool:

```bash
adb shell bpftrace -e 'tracepoint:binder:binder_transaction { @[comm] = count(); }'
```

(Requires a kernel with BPF and a `bpftrace` binary in your tree — Android 14+ GKI has both.)

---

## Chapter 6.6 — Real-World Optimization Walkthroughs

### 6.6.1 Walkthrough — App Cold Launch Is 1.4 s

**Symptom:** `am start -W com.foo/.MainActivity` reports `TotalTime=1432`.

**Process:**

1. Take a Perfetto trace covering 2 s before to 2 s after launch.
2. In the UI, find the "launching: com.foo" slice on `system_server`.
3. Below it, the app's main thread track shows `bindApplication`, `activityStart`, `Choreographer#doFrame`.
4. Look for gaps **between** slices — those are scheduler delays or I/O.
5. Hover slices > 50 ms; common culprits:
   - `Application.onCreate` doing synchronous network or disk I/O.
   - First content provider initialization (Firebase, etc.).
   - Resource inflation of overly nested layouts.
6. Check the binder track — synchronous calls to `package` or `activity_task` services that block the main thread.

**Resolution pattern:**

- Move non-critical work off `Application.onCreate` to a background `WorkManager` job.
- Mark expensive content providers `android:initOrder` and consider Jetpack `App Startup` library.
- Inflate complex views asynchronously with `AsyncLayoutInflater`.

### 6.6.2 Walkthrough — Sustained Scroll Jank

**Symptom:** RecyclerView drops 6/120 frames after 10 s of scrolling.

**Process:**

1. `dumpsys gfxinfo <pkg> framestats` — identify which phase is blowing budget.
2. If **draw** is the culprit: GPU bound. Check overdraw with developer options "Debug GPU overdraw."
3. If **sync** is the culprit: too-large textures uploading. Look at `eglMakeCurrent` and `glTexImage2D` slices.
4. If **record** is the culprit: too many views in the canvas. Layout depth, custom `onDraw`.
5. If **app** time is the culprit: business logic on the UI thread. Find with simpleperf.

🐞 **Common Production Bug:** A vendor enables "high-quality" image filtering on the camera preview surface. Every frame triggers a shader recompile on the first 10 frames after orientation change. Fix: pre-warm shaders at app start.

### 6.6.3 Walkthrough — System-Wide Memory Pressure

**Symptom:** After 4 h of mixed use, lmkd kills foreground app on every backgrounding.

**Process:**

1. `dumpsys meminfo` totals — note `Free RAM`, `Used RAM`, `Lost RAM`.
2. **Lost RAM** > 200 MB means kernel-side leak (DMA-BUF, slab, kgsl). Run `dmabuf_dump`, `cat /proc/slabinfo`.
3. If user-space — `procrank` for top PSS hogs over time.
4. Per-process — `dumpsys meminfo -a <pkg>` snapshot every 15 min, diff.
5. Java heap leaks — capture HPROF: `am dumpheap <pid> /data/local/tmp/x.hprof`, analyze with Android Studio Memory Profiler or `perfetto/tools/heap_profile`.
6. Native leaks — enable `libc.debug.malloc.options=backtrace` (debuggable builds), then `am dumpheap -n <pid> ...`.

🎯 **Staff-Level Insight:** If you see `Lost RAM` growing on a soak test, stop debugging the framework and start debugging the **vendor kernel**. 9 times out of 10 it's a graphics or modem driver leak.

---

## Chapter 6.7 — Power Optimization (Brief Intro)

Power deserves its own book, but every Staff engineer must know:

- **Wakelocks** — `dumpsys power` and `dumpsys batterystats --charged`.
- **Doze and App Standby Buckets** — `frameworks/base/services/core/java/com/android/server/AppStateTracker.java`.
- **Modem state** — RRC connected vs idle dominates cellular battery.
- **CPU idle residency** — `cat /sys/devices/system/cpu/cpu*/cpuidle/state*/usage`. You want >90% time in deepest idle when screen is off.
- **Kernel wakeup attribution** — `cat /sys/kernel/debug/wakeup_sources`.

The single most common production power bug: **a system app or vendor service holding a partial wakelock during deep sleep**. Always run a 4-hour standby test with `dumpsys batterystats` before signing off a release.

---

## Chapter 6.8 — Verifying Level 6

You should be able to:

1. Capture a Perfetto boot trace and identify the critical path.
2. Explain RSS vs PSS vs USS, and produce all three for a given process.
3. Read an lmkd kill log and explain why a particular adj was chosen.
4. Diagnose scroll jank to a specific frame phase using `gfxinfo`.
5. Distinguish a userspace memory leak from a kernel-side DMA-BUF leak.
6. List five things that increase boot time and how to measure each.

---

➡️ Continue to **[Level 7 — Security](./level-07-security.md)**

