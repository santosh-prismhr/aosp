# Level 2A — Deep Dive: Binder Internals & Native Services

> *Supplement to Level 2. This chapter provides the depth of "Embedded Android" and the Android Internals books — kernel driver walkthrough, transaction lifecycle at the byte level, and the architecture of critical native services (SurfaceFlinger, AudioFlinger, InputFlinger).*

---

## 2A.1 — Binder Kernel Driver: A Complete Walkthrough

### 2A.1.1 Source Location and Build

The Binder driver lives in `drivers/android/` in the kernel tree:
- `binder.c` — main driver (~7000 lines)
- `binder_alloc.c` — buffer allocation in target process's mmap'd region
- `binder_internal.h` — internal structures

It registers three character devices:
- `/dev/binder` — framework IPC
- `/dev/hwbinder` — HIDL HALs (deprecated but still present)
- `/dev/vndbinder` — vendor-to-vendor IPC

### 2A.1.2 Process Initialization

When a process first uses Binder:

```cpp
// In ProcessState::init() (frameworks/native/libs/binder/ProcessState.cpp)
mDriverFD = open("/dev/binder", O_RDWR | O_CLOEXEC);
mmap(NULL, BINDER_VM_SIZE /*1MB - 8KB*/, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
ioctl(mDriverFD, BINDER_SET_MAX_THREADS, &maxThreads);
```

In the kernel:
1. `binder_open()` — creates a `binder_proc` for this process, adds to global `binder_procs` list.
2. `binder_mmap()` — allocates the virtual address space (but **no physical pages yet** — pages are allocated on demand when transactions arrive). This is the target's receive buffer.
3. `BINDER_SET_MAX_THREADS` — tells the driver how many threads the process is willing to spawn for incoming transactions.

### 2A.1.3 The Transaction Lifecycle (Byte-Level)

A synchronous two-way transaction:

```
CLIENT                              KERNEL                              SERVER
──────                              ──────                              ──────
1. Build Parcel (data + offsets)
2. Fill binder_transaction_data:
   - target.handle = N
   - code = METHOD_ID
   - flags = 0 (sync)
   - data.ptr.buffer = parcel data
   - data.ptr.offsets = flat_binder_objects
3. Write to bwr.write_buffer:
   BC_TRANSACTION + binder_transaction_data
4. ioctl(BINDER_WRITE_READ, &bwr)
   ─────────────────────────────►
                                 5. binder_thread_write():
                                    - parse BC_TRANSACTION
                                    - translate target.handle → binder_node (in server)
                                    - allocate binder_transaction
                                    - allocate buffer in SERVER's mmap space
                                    - copy data from CLIENT userspace → SERVER buffer
                                    - translate flat_binder_objects:
                                      • BINDER_TYPE_HANDLE → look up ref, increment
                                      • BINDER_TYPE_FD → dup fd into server's fd table
                                    - enqueue transaction on server's todo list
                                    - wake server thread
                                    - put client thread to sleep (TASK_INTERRUPTIBLE)
                                                                     ◄─────────────────
                                                                     6. Server thread wakes from
                                                                        ioctl(BINDER_WRITE_READ)
                                                                     7. binder_thread_read():
                                                                        - dequeue transaction
                                                                        - write BR_TRANSACTION to
                                                                          bwr.read_buffer
                                                                     8. Server reads parcel from
                                                                        its own mmap'd buffer
                                                                     9. Server processes request
                                                                     10. Server writes reply:
                                                                         BC_REPLY + data
                                                                     11. ioctl(BINDER_WRITE_READ)
                                                                     ─────────────────────────►
                                 12. binder_thread_write():
                                     - parse BC_REPLY
                                     - allocate buffer in CLIENT's mmap space
                                     - copy reply data
                                     - enqueue on client's thread todo
                                     - wake client thread
   ◄─────────────────────────────
13. Client wakes, reads BR_REPLY
14. Parse reply Parcel
15. Return to caller
```

### 2A.1.4 The Security Model Inside the Driver

The driver enforces:

1. **Caller identity.** `binder_transaction.sender_pid` and `sender_euid` are set by the **kernel** from `current->tgid` and `current_euid()`. Userspace cannot fake them. This is why `Binder.getCallingUid()` is trustworthy.

2. **SELinux checks.** On every `BC_TRANSACTION`, the driver calls into SELinux:
   ```c
   security_binder_transaction(proc->cred, target_proc->cred);
   security_binder_transfer_binder(proc->cred, target_proc->cred);
   security_binder_transfer_file(proc->cred, target_proc->cred, file);
   ```
   These map to `binder_call(source, target)` and `binder_transfer(source, target)` policy rules.

3. **Death notification.** When a `binder_node`'s owning process dies, the driver sends `BR_DEAD_BINDER` to all processes that registered a `BC_REQUEST_DEATH_NOTIFICATION` on that node.

4. **Frozen process handling (Android 12+).** When a process is frozen (cgroup freezer), the driver detects it and returns `BR_FROZEN_REPLY` to senders (for oneway; for sync, it waits or times out).

### 2A.1.5 Buffer Management

Each process has a 1 MB (minus 8 KB) mmap'd region. The driver manages it as a free-list of `binder_buffer` structures:

```c
struct binder_buffer {
    struct rb_node rb_node;       // in allocated tree (by address)
    unsigned free:1;
    unsigned clear_on_free:1;     // security: zero sensitive data
    struct binder_transaction *transaction;
    size_t data_size;
    size_t offsets_size;
    size_t extra_buffers_size;
};
```

When a transaction arrives, the driver allocates from the *target's* buffer. If the buffer is full → `BR_FAILED_REPLY` with `-ENOSPC` → client sees `TransactionTooLargeException`.

**Why 1 MB?** It's a per-process limit shared among ALL concurrent incoming transactions. If a process has 15 binder threads each receiving a 64 KB payload simultaneously, that's 960 KB — nearly full. This is the real-world failure mode: not one large transaction, but many concurrent medium ones.

### 2A.1.6 Scatter-Gather and the Offsets Array

A Parcel can contain mixed data: primitive types, strings, and **binder objects** (references to other services) or **file descriptors**. These special objects are described in the "offsets" array:

```
Parcel data buffer:
[int32][string][IBinder handle][int64][FileDescriptor fd][...]
                     ▲                       ▲
Offsets array:       │                       │
[offset_of_binder, offset_of_fd]

Each offset points to a flat_binder_object:
struct flat_binder_object {
    __u32 hdr.type;    // BINDER_TYPE_HANDLE, BINDER_TYPE_FD, etc.
    __u32 flags;
    union { binder_uintptr_t binder; __u32 handle; };
    binder_uintptr_t cookie;
};
```

The driver processes these objects during copy:
- **BINDER_TYPE_HANDLE:** Translates the sender's handle to a new handle in the receiver's process (or creates one if first time).
- **BINDER_TYPE_FD:** Calls `dup()` in kernel space to create a new fd in the receiver's file descriptor table.

This is how Android passes Binder objects and file descriptors across processes without exposing raw kernel pointers.

---

## 2A.2 — SurfaceFlinger Architecture

### 2A.2.1 What SurfaceFlinger Does

SurfaceFlinger is the **display compositor** — it takes surfaces (BufferQueues) from all visible windows and composites them into the final frame displayed on screen.

```
App1 ────► BufferQueue1 ─┐
App2 ────► BufferQueue2 ─┼──► SurfaceFlinger ──► HWC HAL ──► Display
SystemUI ─► BufferQueue3 ─┘         │
                                     ▼
                              (fallback) GPU composition
```

### 2A.2.2 The Frame Lifecycle

```
VSYNC signal (from display hardware, 60/90/120 Hz)
  │
  ├─► Choreographer in apps (triggers onDraw)
  │     App renders into its Surface's BufferQueue (dequeueBuffer → draw → queueBuffer)
  │
  ├─► SurfaceFlinger wakes on VSYNC-sf
  │     1. Latch: acquire buffers from all active BufferQueues
  │     2. Compute visible region, z-order, transforms
  │     3. Submit layer list to HWC HAL
  │     4. HWC decides: which layers can be composed by dedicated hardware (overlay planes)
  │        vs. which need GPU composition (fallback)
  │     5. If GPU needed: SurfaceFlinger draws fallback layers into a framebuffer via OpenGL/Vulkan
  │     6. HWC presents the final frame (flips to display)
  │
  └─► Display shows frame at next VSYNC
```

### 2A.2.3 Key Source Paths

```
frameworks/native/services/surfaceflinger/
├── SurfaceFlinger.cpp            # Main compositor logic
├── CompositionEngine/            # Layer composition, output handling
├── DisplayHardware/
│   ├── HWComposer.cpp           # HWC HAL integration
│   └── PowerAdvisor.cpp         # GPU frequency hints
├── Layer.cpp                    # Base class for all layers
├── BufferLayer.cpp              # Layers backed by BufferQueues
├── EffectLayer.cpp              # Color/dim layers
├── Scheduler/                   # VSYNC handling, phase offsets
│   ├── VsyncController.cpp
│   └── Scheduler.cpp
└── main_surfaceflinger.cpp      # entry point
```

### 2A.2.4 BufferQueue — The Core Abstraction

Every Surface is backed by a `BufferQueue` — a producer-consumer queue of GPU buffers:

```
Producer (app)                Consumer (SurfaceFlinger)
───────────────               ───────────────────────
dequeueBuffer()  ────►  [FREE] ──► [DEQUEUED] (app renders into it)
                                        │
                         queueBuffer()  │
                                        ▼
                              [QUEUED] ──► acquireBuffer() ──► [ACQUIRED] (SF composites)
                                                                    │
                                                          releaseBuffer()
                                                                    ▼
                                                              [FREE] (back to pool)
```

Buffer states: FREE → DEQUEUED → QUEUED → ACQUIRED → FREE.

Default queue depth: **3 buffers** (triple buffering). This allows:
- Buffer 0: being displayed
- Buffer 1: being composited by SF  
- Buffer 2: being rendered by app

### 2A.2.5 HWC (Hardware Composer) HAL

The HWC HAL (`hardware/interfaces/graphics/composer/`) tells SurfaceFlinger which layers can be composed by dedicated hardware overlay planes (zero GPU cost) vs. which must fall back to GPU composition.

```
Layer types HWC can handle natively:
- Solid color rectangles (DIM layers)
- Scaled/rotated buffers in supported formats (usually RGBA8888, NV12)
- Limited number (typically 4-8 overlay planes per display)

GPU fallback triggers:
- Too many layers (>N overlay planes)
- Unsupported pixel format
- Complex blend modes
- Rounded corners (before Android 14)
- Layer requiring rotation not supported by hardware
```

Performance impact: GPU composition adds ~2-4 ms per frame and increases power consumption by 30-50% vs. pure HWC overlay composition.

```bash
adb shell dumpsys SurfaceFlinger --latency
adb shell dumpsys SurfaceFlinger | grep -A 5 "Composition"
# Shows which layers are HWC vs GPU composed
```

### 2A.2.6 VSYNC and Scheduling

SurfaceFlinger receives hardware VSYNC from the display and software VSYNC from `DispSync`:

- **VSYNC-app:** Phase-offset VSYNC delivered to apps via `Choreographer`. Apps start rendering here.
- **VSYNC-sf:** Phase-offset VSYNC for SurfaceFlinger. SF starts compositing here.

The phase offset between VSYNC-app and VSYNC-sf is typically ~2–6 ms — giving apps time to finish rendering before SF needs their buffer.

```bash
adb shell dumpsys SurfaceFlinger --vsync
# Shows VSYNC period, phase offsets, missed frames
```

---

## 2A.3 — AudioFlinger Architecture

### 2A.3.1 The Audio Stack

```
App (AudioTrack / MediaPlayer)
   │ shared memory + binder control
   ▼
audioserver process
├── AudioFlinger (native service)
│    ├── MixerThread (per output device)
│    │     Mixes multiple AudioTrack streams into one PCM stream
│    ├── DirectOutputThread (passthrough for compressed/HiRes)
│    └── RecordThread (per input device)
├── AudioPolicyService
│    Decides routing: which device, which stream type, volume
└── audio HAL (dlopen'd .so or AIDL HAL process)
       │
       ▼
    ALSA (tinyalsa) / vendor DSP driver
       │
       ▼
    DAC / Codec / Speaker / Headphone
```

### 2A.3.2 Shared Memory Fast Path

For low-latency audio, Android uses a **shared memory ring buffer** between the app and `audioserver`:

1. App creates `AudioTrack` with `AUDIO_OUTPUT_FLAG_FAST`.
2. AudioFlinger allocates a shared memory buffer (typically 2-4 x frame count).
3. App writes PCM directly into shared memory (no Binder per-frame!).
4. AudioFlinger's `FastMixer` thread (SCHED_FIFO priority 3) reads from shared memory and writes to HAL.
5. Synchronization via atomic counters (read/write pointers) — no kernel involvement per frame.

Result: **~5 ms round-trip latency** on well-tuned devices (vs. ~100 ms without fast path).

### 2A.3.3 Audio HAL Interface

```aidl
// hardware/interfaces/audio/aidl/android/hardware/audio/core/IModule.aidl
@VintfStability
interface IModule {
    IStreamOut openOutputStream(in AudioPortConfigExt config, ...);
    IStreamIn openInputStream(in AudioPortConfigExt config, ...);
    AudioPort[] getAudioPorts();
    AudioRoute[] getAudioRoutes();
    void setAudioPortConfig(in AudioPortConfig config);
}
```

The HAL implementation typically wraps `tinyalsa` (a minimal ALSA userspace library) or talks to a vendor DSP over proprietary transport.

### 2A.3.4 Audio on AAOS (Multi-Zone)

AAOS extends the audio stack with `CarAudioService`:

```
CarAudioService
├── Zone 0 (driver): speakers front-left, front-right, center
├── Zone 1 (rear-left): headphone jack, rear-left speaker
└── Zone 2 (rear-right): headphone jack, rear-right speaker

Each zone → separate HAL device address ("bus0_media_out", "bus1_media_out", ...)
AudioFlinger routes streams to the correct bus based on zone + usage
```

Configuration: `car_audio_configuration.xml` maps zones → HAL devices → usages.

---

## 2A.4 — InputFlinger and the Input Pipeline

### 2A.4.1 From Hardware to App

```
Touchscreen controller (I2C/SPI interrupt)
   │ IRQ → kernel input driver → /dev/input/eventN
   ▼
EventHub (in InputFlinger, inputflinger process)
   │ reads raw events via epoll
   ▼
InputReader
   │ Calibrates, transforms (rotation, pressure)
   │ Classifies: touch, key, trackball, switch
   ▼
InputDispatcher
   │ Finds focused window (via WMS)
   │ Sends InputEvent to app via input channel (unix socket pair)
   ▼
App (ViewRootImpl → View.onTouchEvent)
```

### 2A.4.2 ANR and Input

If the app doesn't consume the input event within **5 seconds**, `InputDispatcher` raises an ANR (Application Not Responding). The timeout is checked per-dispatch:

```
InputDispatcher:
  - Sends event via socket
  - Starts 5s timer
  - If no finish signal from app → ANR
  - ANR callback → system_server → show dialog / kill
```

Common causes of input ANR:
- App's main thread blocked in synchronous Binder call
- Main thread holding a lock that another thread (doing I/O) holds
- GC pause (rare on modern ART, but possible with large heaps)

```bash
adb shell dumpsys input               # full input state
adb shell dumpsys input | grep -A 5 "ANR"
```

---

## 2A.5 — The ART Runtime (Deep)

### 2A.5.1 Execution Modes

ART supports three execution modes for dex bytecode:

| Mode | When | Speed | Storage |
|------|------|-------|---------|
| **Interpreter** | First run, fallback | ~100x slower than native | Zero |
| **JIT (Just-in-Time)** | Hot methods at runtime | ~2-5x slower than AOT | In-memory code cache |
| **AOT (Ahead-of-Time)** | After `dex2oat` compilation | Native speed | `.odex` / `.oat` files |

The default profile-guided compilation flow:

```
Install APK
  │ (no compilation; runs interpreted + JIT)
  ▼
First few runs: JIT compiles hot methods; records profile (.prof)
  │
  ▼ (idle maintenance / Garage Mode / first boot)
dex2oat --compiler-filter=speed-profile
  │ Compiles only methods in the profile
  ▼
.odex file created; future launches use AOT code for hot paths
```

### 2A.5.2 The Heap and GC

ART's heap is divided into spaces:

```
┌─────────────────────────────────────────────┐
│ Image Space (boot.art, shared, read-only)   │  ← pre-compiled framework classes
├─────────────────────────────────────────────┤
│ Zygote Space (CoW shared with children)     │  ← allocations during zygote preload
├─────────────────────────────────────────────┤
│ Allocation Space (RegionSpace, Android 8+)  │  ← per-app allocations
│   ├── Thread-Local Allocation Buffers (TLABs)│
│   └── Large Object Space (LOS)              │
├─────────────────────────────────────────────┤
│ Non-Moving Space                            │  ← objects that can't be moved (JNI refs)
└─────────────────────────────────────────────┘
```

GC algorithms:
- **Concurrent Copying (CC) GC** (default since Android 8): moving, concurrent, generational. Pauses <1 ms typically.
- **Young-gen collection:** Frequent, fast, collects recently-allocated objects.
- **Full-heap collection:** Rare, compacts the entire heap.

```bash
adb shell kill -10 <pid>   # force GC
adb logcat -s art          # GC logs with timing
# Example: art: Concurrent copying GC freed 12345(456KB) AllocSpace objects, 0(0B) LOS objects, 25% free, 3MB/4MB, paused 115us total 12.5ms
```

### 2A.5.3 JNI and the Art of Crossing Boundaries

JNI (Java Native Interface) is the bridge between Java/Kotlin and C/C++. Critical performance notes:

- **JNI call overhead:** ~1 µs (vs. ~5 ns for a Java method call). 1000 JNI calls in a tight loop costs 1 ms.
- **Critical JNI:** Methods annotated `@CriticalNative` skip the JNI transition overhead (no `JNIEnv`, no thread state check). Used for hot-path methods like `Binder.getCallingUid()`.
- **Local references:** JNI `jobject` refs are scoped to the native method call. Storing them requires `NewGlobalRef()` — forgetting this causes use-after-free.
- **String handling:** `GetStringUTFChars()` copies; `GetStringCritical()` pins (no copy, but blocks GC). Always release.

### 2A.5.4 Class Loading on Android

Unlike standard Java (classpath-based), Android uses **PathClassLoader** and **DelegateLastClassLoader**:

```
BootClassLoader
  └── PathClassLoader (system_server, loading /system/framework/*.jar)
       └── PathClassLoader (per-app, loading APK's classes.dex)
            └── InMemoryDexClassLoader (for dynamic dex loading — risky, blocked in some contexts)
```

Each class is identified by `(name, classloader)` — two classes with the same name in different loaders are different types. This causes the infamous `ClassCastException` when vendor code uses a different classloader than the framework expects.

---

## 2A.6 — system_server Thread Model

`system_server` has ~180 threads at steady state. Key ones:

| Thread | Role | Priority |
|--------|------|----------|
| `main` | Starts services, then becomes Looper for system-wide messages | Normal |
| `android.display` | Display/Window events, animations | High |
| `android.ui` | UI-related processing | High |
| `android.fg` | Foreground handlers (broadcasts, etc.) | Normal |
| `android.bg` | Background handlers (app switches, recents) | Background |
| `android.io` | I/O operations (package scanning, etc.) | Background |
| `Binder:XXX_Y` | Binder thread pool (serves incoming IPCs) | Inherited |
| `watchdog` | Deadlock detector — pings all handler threads | Normal |
| `FinalizerDaemon` | Runs object finalizers | Normal |
| `HeapTaskDaemon` | GC background work | Normal |

The Watchdog thread pings each registered handler every 30s. If any doesn't respond in 60s → system_server killed → soft reboot.

---

## 2A.7 — Verifying This Deep Dive

You should be able to:

1. Trace a Binder transaction through the kernel driver, identifying where the single-copy occurs.
2. Explain why `TransactionTooLargeException` can occur with "small" payloads (concurrent transactions).
3. Describe the triple-buffering lifecycle in SurfaceFlinger's BufferQueue.
4. Explain why GPU composition is more expensive than HWC overlay and how to check which is active.
5. Describe the audio fast path and why shared memory eliminates per-frame Binder calls.
6. Explain ART's profile-guided compilation flow from install to optimized execution.
7. Name five threads in `system_server` and their roles.

---

⬅️ Back to **[Level 2 — Framework Internals](./level-02-framework-internals.md)** | ➡️ **[Level 3 — HAL & Native Layer](./level-03-hal-native.md)**

