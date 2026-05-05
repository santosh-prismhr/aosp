# Level 0 — Deep Dive: Linux Kernel Internals for Android Engineers

> *Supplement to Level 0. This chapter provides the kernel-level depth that "Embedded Android" (Yaghmour) covers — process model, virtual memory, filesystems, the device model, and Android-specific kernel patches — at the level needed to debug platform issues from first principles.*

---

## 0A.1 — The Linux Process Model on Android

### 0A.1.1 task_struct — The Kernel's View of a Process

Every process (and every thread) in Linux is represented by a `task_struct` (~8 KB on ARM64). Key fields that matter for Android:

```c
struct task_struct {
    volatile long state;           // TASK_RUNNING, TASK_INTERRUPTIBLE, etc.
    void *stack;                   // kernel stack pointer
    struct mm_struct *mm;          // address space (NULL for kernel threads)
    pid_t pid;                     // thread ID (what Android calls "TID")
    pid_t tgid;                    // thread group leader (what Android calls "PID")
    const struct cred *cred;       // UID, GID, capabilities, SELinux context
    struct signal_struct *signal;  // shared among threads in a group
    struct files_struct *files;    // file descriptor table
    struct fs_struct *fs;          // root dir, cwd
    struct nsproxy *nsproxy;       // namespaces (mnt, pid, net, ipc, uts, user)
    struct css_set *cgroups;       // cgroup membership
    // ... hundreds more fields
};
```

**Why this matters for Android:**

- `tgid` is what `getpid()` returns and what `dumpsys` shows. `pid` is the per-thread ID visible in `/proc/<tgid>/task/<tid>/`.
- `cred->uid` is the app's UID (e.g., `u0_a150` = UID 10150). The Binder driver reads this directly to implement `getCallingUid()`.
- `cred->security` points to the SELinux security context (`u:r:untrusted_app_32:s0:c150,c256,c512,c768`).
- `nsproxy->mnt_ns` controls which filesystem tree the process sees — this is how `/storage/emulated/0` appears differently per-user.

### 0A.1.2 Process Creation: fork() vs clone() vs vfork()

On Android:

| Mechanism | Used by | What happens |
|-----------|---------|-------------|
| `fork()` | Zygote → app process | Full address-space copy (CoW). Child inherits all mapped pages as read-only; first write triggers page fault and copy. |
| `clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_THREAD | ...)` | `pthread_create` | New thread shares address space with parent; only gets its own stack and `task_struct`. |
| `posix_spawn` / `fork+exec` | init → native services | Fork then immediately `execve`, replacing address space. |

Zygote uses `fork()` *without* `exec()` — this is the key Android optimization. The child already has a complete JVM, loaded classes, and warm page cache. The price: fork-safety. Any library holding a lock at fork time leaves the child in undefined state. This is why Zygote is extremely careful about what runs before `fork()` and why `atFork` handlers exist in bionic.

### 0A.1.3 Process States and Scheduling

```
                  ┌─────────────┐
     fork()       │   CREATED   │
                  └──────┬──────┘
                         │ wake_up_new_task()
                         ▼
┌──────────┐      ┌─────────────┐      ┌──────────────────┐
│ SLEEPING │◄─────│   RUNNABLE  │─────►│     RUNNING      │
│(INTER-   │ wait │  (on rq)    │ pick │ (on CPU)         │
│ RUPTIBLE)│      └─────────────┘      └────────┬─────────┘
└──────────┘             ▲                      │
      │                  │ wakeup               │ schedule()
      │                  │                      ▼
      │           ┌──────┴──────┐      ┌──────────────────┐
      └──────────►│   STOPPED   │      │ UNINTERRUPTIBLE  │
                  │  (SIGSTOP)  │      │ (D state — I/O)  │
                  └─────────────┘      └──────────────────┘
```

**D-state (TASK_UNINTERRUPTIBLE)** is the most important for Android debugging. A process in D-state is waiting for I/O (typically disk or binder) and **cannot be killed** — not even by SIGKILL. When you see a process stuck in D-state:

```bash
adb shell cat /proc/<pid>/wchan     # shows kernel function where it's sleeping
adb shell cat /proc/<pid>/stack     # full kernel stack trace
```

Common causes on Android:
- **dm-verity hash computation** on first access to a system partition block.
- **f2fs GC (garbage collection)** holding a page lock while compacting.
- **UFS command queue full** — storage controller saturated.
- **Binder driver** waiting for a free thread in the target process.

### 0A.1.4 The Completely Fair Scheduler (CFS) and Android's Tweaks

Linux uses CFS (Completely Fair Scheduler) for normal-priority tasks. Key concepts:

- **Virtual runtime (vruntime):** Each task accumulates vruntime proportional to wall time / weight. The task with the *lowest* vruntime is scheduled next.
- **Nice values (-20 to +19):** Map to weights. Nice 0 = weight 1024; nice -20 = weight 88761; nice 19 = weight 15.
- **Scheduling groups:** CFS can group tasks (cgroups) and allocate CPU shares to groups.

Android's additions:

```
/dev/cpuctl/                          # cgroup v1 cpu controller
  ├── top-app/                        # shares=1024, boosted
  ├── foreground/                     # shares=1024
  ├── background/                     # shares=52 (starved intentionally)
  ├── system-background/              # shares=52
  └── rt/                             # for real-time (RT) tasks like audioflinger

/dev/cpuset/
  ├── top-app/    cpus=0-7            # all cores
  ├── foreground/ cpus=0-7            # all cores
  ├── background/ cpus=0-3            # little cores only (big.LITTLE)
  └── system-background/ cpus=0-3    # little cores only
```

The framework's `OomAdjuster` moves processes between these groups as their importance changes. A backgrounded app drops from `top-app` → `background` within seconds, losing access to big cores.

**EAS (Energy-Aware Scheduling):** Modern ARM SoCs (big.LITTLE, DynamIQ) use EAS to balance performance and power. The scheduler considers per-CPU energy cost when deciding where to place a task. A task that fits on a little core should run there; only tasks exceeding the little core's capacity migrate to big cores.

```bash
adb shell cat /proc/<pid>/sched      # scheduler stats for a task
adb shell cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq
```

### 0A.1.5 Real-Time Scheduling on Android

Some Android services use `SCHED_FIFO` or `SCHED_RR`:

| Service | Policy | Priority | Why |
|---------|--------|----------|-----|
| `audioflinger` (mixer thread) | `SCHED_FIFO` | 2–3 | Sub-10ms latency required |
| `surfaceflinger` (vsync thread) | `SCHED_FIFO` | 2 | Must hit vsync deadline |
| `InputDispatcher` | `SCHED_FIFO` | 1 | Touch latency |
| Binder threads (during transaction) | Temporarily boosted | Inherited from caller | Priority inheritance |

Binder implements **priority inheritance**: when a high-priority thread calls into a low-priority server, the server thread's priority is temporarily raised to match the caller's. This prevents priority inversion. The kernel's `futex` implementation also does this for mutexes — but only if `PTHREAD_PRIO_INHERIT` is set (which bionic does for normal mutexes).

---

## 0A.2 — Virtual Memory on Android

### 0A.2.1 The Address Space Layout

A typical 64-bit Android app process:

```
0x0000'0000'0000'0000 ─────────────────────────
                        ← NULL page (unmapped, traps nullptr deref)
0x0000'0000'0000'1000 ─────────────────────────
                        ← .text, .rodata, .data of app_process64
                        ← linked .so libraries (libandroid_runtime, libart, etc.)
                        ← mmap'd .odex / .vdex / .art files (ART)
                        ← Java heap (managed by ART's GC)
                        ← Native heap (jemalloc/scudo)
                        ← mmap'd APK, resources
                        ← Anonymous memory (ashmem/memfd, Binder buffers)
                        ← Thread stacks (8 MB default per thread)
                        ← ...
                        ← vDSO (kernel-provided fast syscalls: clock_gettime)
0x0000'007F'FFFF'FFFF ─────────────────────────
                        ← kernel space (inaccessible from EL0)
0xFFFF'FFFF'FFFF'FFFF ─────────────────────────
```

### 0A.2.2 Page Tables and TLB

ARM64 uses 4-level page tables (4KB pages, 48-bit VA). Translation: VA → PGD → PUD → PMD → PTE → PA. Each TLB miss costs ~20–50 ns (vs. ~1 ns for a hit). On Android with hundreds of processes, TLB thrashing is a real performance concern — this is why **huge pages** (2 MB THP) and **ASID (Address Space IDs)** matter.

Android kernel enables **ASID** (up to 65536 on ARMv8.1+), which means context switches don't flush the TLB — each process's entries are tagged with its ASID. This is critical for the rapid process switching that Android does (foreground app → system_server → HAL → back in microseconds via Binder).

### 0A.2.3 Copy-on-Write (CoW) — The Foundation of Zygote

When Zygote forks:
1. All page table entries in the child are marked **read-only**.
2. Parent's pages also become read-only (both must trigger fault on write).
3. On first write by either process, the CPU faults; the kernel copies that single 4 KB page and remaps it writable for the writing process.
4. The other process retains the original page (still read-only until it too writes).

Measurement:

```bash
# Compare shared vs. private pages for an app
adb shell showmap <pid> | tail -5
# "shared clean" = pages still shared with zygote (CoW not triggered)
# "private dirty" = pages copied (CoW triggered)
```

A fresh app process has ~150 MB shared-clean and ~5 MB private-dirty. After running for a while, private-dirty grows as the app modifies its heap. **Minimizing private-dirty is the single biggest memory optimization for multi-app devices.**

### 0A.2.4 mmap() and Android's Use Cases

| Pattern | Example | Flags |
|---------|---------|-------|
| Code mapping | `.so`, `.odex` | `PROT_READ|PROT_EXEC`, `MAP_PRIVATE` |
| File-backed data | `.apk`, fonts | `PROT_READ`, `MAP_PRIVATE` |
| Anonymous heap | native malloc | `PROT_READ|PROT_WRITE`, `MAP_PRIVATE|MAP_ANONYMOUS` |
| Shared memory (IPC) | `ashmem`/`memfd` | `MAP_SHARED` |
| Binder buffer | `/dev/binder` mmap'd 1MB | `MAP_PRIVATE` (kernel manages) |
| Graphics buffers | DMA-BUF fd from gralloc | `MAP_SHARED` (often not mapped to CPU at all) |

### 0A.2.5 Memory Overcommit and the OOM Killer

Linux **overcommits** memory by default (`/proc/sys/vm/overcommit_memory = 0`). This means `malloc()` almost never fails — the kernel gives you a virtual mapping; actual physical pages are allocated only on first write (demand paging). If total demand exceeds physical RAM + swap, the OOM killer fires.

On Android, LMKD intervenes **before** the kernel OOM killer. It uses PSI (Pressure Stall Information) to detect early signs of pressure and proactively kills cached processes. If LMKD fails, the kernel OOM killer fires — which is a much worse experience (it picks victims less intelligently).

```bash
adb shell cat /proc/pressure/memory
# some avg10=2.50 avg60=1.20 avg300=0.80 total=12345678
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

- `some` = at least one task stalled on memory.
- `full` = all tasks stalled. If `full` > 0 for sustained periods, the device is thrashing.

---

## 0A.3 — Filesystems on Android

### 0A.3.1 The Filesystem Hierarchy

```
/                     (rootfs, tmpfs — initramfs stage 1)
├── system/           (ext4 or EROFS, dm-verity, read-only)
├── system_ext/       (ext4 or EROFS, dm-verity, read-only)
├── vendor/           (ext4 or EROFS, dm-verity, read-only)
├── product/          (ext4 or EROFS, dm-verity, read-only)
├── odm/              (ext4 or EROFS, read-only)
├── data/             (f2fs or ext4, encrypted, read-write)
├── cache/            (ext4 or f2fs, read-write, wiped on recovery)
├── metadata/         (ext4, stores FBE key material)
├── misc/             (raw partition — bootloader messages)
├── apex/             (bind-mounts from /data/apex/active/*.apex)
├── dev/              (devtmpfs — device nodes)
├── proc/             (procfs — process and kernel info)
├── sys/              (sysfs — kernel objects, drivers, devices)
├── mnt/              (tmpfs — mount points for storage, media)
├── storage/          (bind-mounts for FUSE/sdcardfs per-user storage views)
└── linkerconfig/     (generated at boot — linker namespace config)
```

### 0A.3.2 ext4 vs F2FS vs EROFS

| Property | ext4 | F2FS | EROFS |
|----------|------|------|-------|
| Type | journaling | log-structured | read-only compressed |
| Used for | `/data` (legacy), `/cache` | `/data` (modern) | `/system`, `/vendor` (Android 13+) |
| Write amplification | low | low (append-only segments) | N/A (read-only) |
| GC | N/A | Yes — can cause latency spikes | N/A |
| Compression | no | lz4/zstd inline (Android 12+) | lz4/lzma at build time |
| Encryption | fscrypt v2 | fscrypt v2 | fscrypt v2 (for metadata) |
| Mount time | fast | fast | very fast (no journal replay) |

**EROFS** (Enhanced Read-Only File System) is the modern choice for read-only partitions. It compresses data at build time (lz4 or lzma), reducing partition size by 20–40% while keeping random-read performance high. This directly reduces OTA download sizes and boot I/O.

**F2FS** (Flash-Friendly File System) for `/data`:
- Log-structured: writes append to a log, reducing random writes (extends flash lifetime).
- **GC (Garbage Collection):** When free segments run low, F2FS copies valid blocks from partially-empty segments to new segments, then erases the old ones. GC can cause 50–200 ms stalls if triggered on the critical path.
- **Atomic writes:** F2FS supports atomic multi-block writes (`F2FS_IOC_START_ATOMIC_WRITE`), used by SQLite's WAL and Android's `AtomicFile` to prevent torn writes on power loss.

```bash
adb shell cat /sys/fs/f2fs/*/stat    # GC stats
adb shell cat /sys/fs/f2fs/*/segment_info
```

🐞 **Common Production Bug:** First boot after OTA takes 3 minutes on a 128 GB device. Cause: F2FS GC triggered by dexopt filling segments; GC competes with dexopt for I/O bandwidth. Fix: run dexopt with `ionice -c3` (idle I/O class) and pre-allocate dex output space.

### 0A.3.3 fscrypt — File-Based Encryption

Every file on `/data` is encrypted with a per-file key derived from the user's credential + hardware-bound key. Directory structure:

```
/data/user/0/com.foo/           ← CE (Credential Encrypted)
/data/user_de/0/com.foo/        ← DE (Device Encrypted)
```

CE keys are unavailable until the user unlocks (first PIN entry after boot). DE keys are available at boot — used for things that must work pre-unlock (alarms, Direct Boot receivers).

The encryption is transparent to apps: the kernel's `fscrypt` layer encrypts on write and decrypts on read. But the **key management** is Android-specific: `vold` talks to `keystore2` which talks to the TEE (KeyMint) to unwrap the keys.

### 0A.3.4 FUSE vs sdcardfs vs Scoped Storage

The path `/storage/emulated/0/` (the user's "sdcard") has gone through multiple implementations:

| Era | Mechanism | Notes |
|-----|-----------|-------|
| Android 4.x | real FAT32 SD card | No permissions |
| Android 5–7 | `sdcardfs` (kernel module) | Per-app permission filtering in kernel |
| Android 8–10 | `sdcardfs` + FUSE fallback | |
| Android 11+ | **FUSE** (MediaProvider as FUSE daemon) | Scoped Storage enforced |
| Android 13+ | FUSE with passthrough for perf | |

The FUSE daemon is `MediaProvider` (`com.android.providers.media`). Every file access in `/storage/emulated/0` results in a message to userspace `MediaProvider` which checks permissions, then issues the real I/O. This has a performance cost (~10–30% overhead for sequential reads) but enables scoped storage's per-app access control.

**Passthrough (Android 13+):** For apps with broad storage access (e.g., file managers), the FUSE layer opens the lower file and passes the fd directly to the requesting process — eliminating the read/write relay. This brings FUSE overhead near zero for granted apps.

---

## 0A.4 — The Linux Device Model and Android

### 0A.4.1 sysfs, udev, and ueventd

Linux exposes hardware through `/sys/` (sysfs). Each device is a kernel object with attributes:

```
/sys/
├── bus/
│   ├── platform/devices/       ← SoC peripherals (SPI, I2C, UART controllers)
│   ├── pci/                    ← PCI devices
│   └── usb/                    ← USB devices
├── class/
│   ├── thermal/thermal_zone*/  ← thermal sensors
│   ├── power_supply/           ← battery, charger
│   ├── leds/                   ← LED control
│   └── input/                  ← input devices (touch, buttons)
├── devices/
│   └── platform/               ← device tree-instantiated devices
└── fs/
    ├── selinux/                ← SELinux interface
    ├── f2fs/                   ← F2FS stats
    └── cgroup/                 ← cgroup controllers
```

Android does **not** use `udev`. Instead, `ueventd` (part of `init`, source: `system/core/init/ueventd.cpp`) handles device node creation:

- Kernel sends `uevent` (kobject add/remove/change) via netlink.
- `ueventd` creates `/dev/` nodes with permissions from `ueventd.rc` files.
- SELinux labels are applied based on `file_contexts`.

```
# /vendor/etc/ueventd.rc (example)
/dev/kgsl-3d0     0666   system   system
/dev/ion          0664   system   system
/dev/video*       0660   media    camera
/dev/binder       0666   root     root
/dev/hwbinder     0666   root     root
/dev/vndbinder    0666   root     root
```

### 0A.4.2 Device Tree (DT) and Android

The Device Tree (`.dts` → compiled `.dtb`) describes non-discoverable hardware to the kernel. On ARM platforms, the bootloader passes the DTB to the kernel at boot.

```dts
// Example: a simple I2C sensor
&i2c3 {
    status = "okay";
    clock-frequency = <400000>;

    ambient_light@29 {
        compatible = "vishay,veml6030";
        reg = <0x29>;
        interrupt-parent = <&tlmm>;
        interrupts = <45 IRQ_TYPE_LEVEL_LOW>;
    };
};
```

Android uses **Device Tree Overlays (DTOs)**: the main DTB comes from the SoC vendor; the OEM applies overlays for board-specific peripherals. DTOs live in a separate `dtbo` partition:

```
boot.img: kernel + generic ramdisk
dtbo.img: Device Tree Blob Overlays (per-board)
vendor_boot.img: vendor ramdisk + main DTB(s)
```

At boot: bootloader loads base DTB from `vendor_boot`, applies overlays from `dtbo`, passes final merged DT to kernel.

```bash
# On device — dump the live device tree
adb shell ls /proc/device-tree/
adb shell cat /proc/device-tree/model
adb shell cat /proc/device-tree/compatible

# On host — decompile a DTB
dtc -I dtb -O dts out/target/product/<dev>/dtbo.img
```

### 0A.4.3 Platform Drivers and Probe

Most Android SoC peripherals are **platform devices** — they don't sit on a discoverable bus. The kernel instantiates them from the device tree and calls the driver's `probe()` function:

```c
static int my_sensor_probe(struct platform_device *pdev) {
    struct device *dev = &pdev->dev;
    // Get resources from DT: regs, irqs, clocks, GPIOs
    void __iomem *base = devm_platform_ioremap_resource(pdev, 0);
    int irq = platform_get_irq(pdev, 0);
    // Initialize hardware
    // Register with framework (e.g., IIO, input, misc)
    return 0;
}

static const struct of_device_id my_sensor_of_match[] = {
    { .compatible = "vendor,my-sensor" },
    { }
};

static struct platform_driver my_sensor_driver = {
    .probe = my_sensor_probe,
    .driver = { .name = "my-sensor", .of_match_table = my_sensor_of_match },
};
module_platform_driver(my_sensor_driver);
```

**Android implication:** `probe()` runs during kernel boot. A slow probe (>100 ms) directly extends boot time. Android's GKI pushes drivers to **modules** (`vendor_dlkm`) loaded after essential init, with `async_probe` where possible:

```c
module_platform_driver_probe(my_sensor_driver, my_sensor_probe);
// Or: add "async" to the module parameter
MODULE_SOFTDEP("pre: my_dependency_driver");
```

---

## 0A.5 — Android-Specific Kernel Patches

The "Android Common Kernel" (ACK) carries patches not in mainline Linux (though many have been upstreamed over the years):

### 0A.5.1 Binder Driver (`drivers/android/binder.c`)

The heart of Android IPC. Key data structures:

```c
struct binder_proc {
    struct hlist_node proc_node;       // global process list
    struct rb_root threads;            // binder_thread per calling thread
    struct rb_root nodes;              // binder_node — objects this process owns
    struct rb_root refs_by_desc;       // binder_ref — handles to other processes' nodes
    struct list_head waiting_threads;  // threads blocked in BINDER_WRITE_READ
    struct mm_struct *vma_vm_mm;       // for the mmap'd buffer region
    // ...
};

struct binder_transaction {
    struct binder_work work;
    struct binder_thread *from;        // caller thread
    struct binder_proc *to_proc;       // target process
    struct binder_thread *to_thread;   // target thread (if reply)
    struct binder_buffer *buffer;      // data payload
    unsigned int code;                 // method number
    unsigned int flags;                // TF_ONE_WAY, TF_ACCEPT_FDS, etc.
    // ...
};
```

Transaction flow in the kernel:
1. Client thread does `ioctl(fd, BINDER_WRITE_READ, &bwr)` with `BC_TRANSACTION`.
2. Driver allocates `binder_transaction`, copies data into target's mmap'd buffer (single-copy — kernel writes directly into target's address space).
3. Driver wakes a thread in the target process (or allocates from thread pool).
4. Target thread returns from its `ioctl()` with `BR_TRANSACTION` containing the data.
5. Target processes, replies with `BC_REPLY`.
6. Driver wakes the original caller with `BR_REPLY`.

**Single-copy optimization:** The target process's `/dev/binder` is mmap'd into both kernel and userspace. When the driver copies data, it copies from the sender's userspace directly into the target's kernel-mapped buffer — which is the *same physical pages* as the target's userspace mmap. Result: data is copied exactly once (sender → target buffer), not twice.

### 0A.5.2 ashmem → memfd

**ashmem** (Anonymous Shared Memory) was Android's pre-standard shared memory mechanism. It allowed creating named shared memory regions that could be pinned/unpinned (kernel could reclaim unpinned pages under pressure).

As of Android 12+, ashmem is **deprecated** in favor of standard Linux `memfd_create()` + `ftruncate()` + `mmap()`. The kernel still supports ashmem for compatibility but new code should use `memfd`.

```c
// Modern pattern:
int fd = memfd_create("my_buffer", MFD_CLOEXEC | MFD_ALLOW_SEALING);
ftruncate(fd, size);
void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
// Pass fd to another process via Binder (ParcelFileDescriptor)
```

### 0A.5.3 ION → DMA-BUF Heaps

ION was Android's memory allocator for buffers shared between CPU, GPU, display, camera, video decoder. It's been replaced by standard **DMA-BUF heaps** (mainlined in Linux 5.6):

```bash
adb shell ls /dev/dma_heap/
# system  (cacheable CPU-accessible)
# system-uncached  (non-cacheable, for hardware-only access)
```

Allocation from userspace:
```c
#include <linux/dma-heap.h>
int heap_fd = open("/dev/dma_heap/system", O_RDONLY);
struct dma_heap_allocation_data alloc = { .len = size, .fd_flags = O_RDWR | O_CLOEXEC };
ioctl(heap_fd, DMA_HEAP_IOCTL_ALLOC, &alloc);
int buf_fd = alloc.fd;  // DMA-BUF fd — shareable across processes
```

### 0A.5.4 Incremental FS

Incremental FS (`fs/incfs/`) allows APKs to be installed and launched before they're fully downloaded. The kernel serves blocks on-demand from a userspace data loader. Used by Play for large games.

### 0A.5.5 dm-verity, dm-bow, dm-default-key

| Target | Purpose |
|--------|---------|
| `dm-verity` | Read-time block integrity for read-only partitions |
| `dm-bow` (backup-on-write) | Used during A/B merge (Virtual A/B) |
| `dm-default-key` | Metadata encryption for `/data` |
| `dm-crypt` | Full-block encryption (legacy FDE, now removed) |
| `dm-linear` | Maps logical partitions in `super.img` to physical blocks |

These are stacked: a typical `/system` mount is: physical → dm-linear (from super partition) → dm-verity → filesystem.

---

## 0A.6 — Kernel Debugging on Android

### 0A.6.1 Reading dmesg Effectively

```bash
adb shell dmesg -T | head -100          # with timestamps
adb shell dmesg -w                       # follow
adb shell dmesg | grep -E "panic|BUG|RIP|Call trace"  # crashers
adb shell dmesg | grep -i "binder"       # binder driver events
```

Severity levels in `<N>` prefix:
- `<0>` EMERG, `<1>` ALERT, `<2>` CRIT, `<3>` ERR, `<4>` WARN, `<5>` NOTICE, `<6>` INFO, `<7>` DEBUG

### 0A.6.2 /proc Files You Must Know

| Path | What it tells you |
|------|-------------------|
| `/proc/meminfo` | System-wide memory stats |
| `/proc/pressure/{cpu,memory,io}` | PSI stall metrics |
| `/proc/<pid>/maps` | Virtual address space layout |
| `/proc/<pid>/smaps_rollup` | PSS/RSS/USS summary |
| `/proc/<pid>/status` | UID, VmRSS, threads, SELinux context |
| `/proc/<pid>/oom_score_adj` | LMKD priority |
| `/proc/<pid>/wchan` | Where thread is sleeping in kernel |
| `/proc/<pid>/stack` | Kernel stack trace (requires root) |
| `/proc/<pid>/fd/` | Open file descriptors |
| `/proc/<pid>/task/` | Per-thread info |
| `/proc/interrupts` | IRQ counts per CPU |
| `/proc/softirqs` | Softirq counts |
| `/proc/cmdline` | Kernel boot command line |
| `/proc/version` | Kernel version + GKI hash |

### 0A.6.3 ftrace — The Kernel's Built-in Tracer

Perfetto's kernel data comes from ftrace. You can use it directly:

```bash
adb shell "echo 1 > /sys/kernel/tracing/tracing_on"
adb shell "echo 'sched:sched_switch' > /sys/kernel/tracing/set_event"
adb shell "cat /sys/kernel/tracing/trace_pipe" | head -50
adb shell "echo 0 > /sys/kernel/tracing/tracing_on"
```

For function tracing:
```bash
adb shell "echo function_graph > /sys/kernel/tracing/current_tracer"
adb shell "echo binder_ioctl > /sys/kernel/tracing/set_ftrace_filter"
adb shell "cat /sys/kernel/tracing/trace" | head -100
```

### 0A.6.4 pstore / ramoops — Surviving a Kernel Panic

When the kernel panics, it writes the last dmesg to a persistent RAM region (ramoops) or to a pstore-backed partition. On next boot:

```bash
adb shell ls /sys/fs/pstore/
# console-ramoops-0  ← last dmesg before crash
# dmesg-ramoops-0    ← panic stack trace
adb shell cat /sys/fs/pstore/console-ramoops-0
```

This is the **only** way to get kernel crash info on production devices without JTAG. Ensure your BSP allocates ramoops memory (typically 2 MB in the device tree):

```dts
reserved-memory {
    ramoops@b0000000 {
        compatible = "ramoops";
        reg = <0x0 0xb0000000 0x0 0x200000>;
        console-size = <0x100000>;
        pmsg-size = <0x80000>;
        record-size = <0x10000>;
    };
};
```

---

## 0A.7 — Bionic: Android's C Library

### 0A.7.1 Why Not glibc?

- **Size:** Bionic is ~300 KB vs. glibc's ~2 MB.
- **License:** BSD (bionic) vs. LGPL (glibc) — avoids viral linking concerns for vendor code.
- **Thread model:** Simpler, with Android-specific TLS (Thread-Local Storage) optimizations.
- **No locale support** (beyond basic C/POSIX): saves ~10 MB of locale data.
- **Android-specific functions:** `__system_property_get()`, `android_log_*()`, Binder-aware `fork()`.

### 0A.7.2 What's Different (Porting Gotchas)

| Feature | glibc | Bionic |
|---------|-------|--------|
| `pthread_cancel()` | Yes | **No** — use cooperative cancellation |
| `dlopen()` from anywhere | Yes | Linker namespace restricted |
| `locale` beyond "C" | Yes | **No** |
| `getifaddrs()` | Yes | Yes (since Android 7) |
| `signalfd()` | Yes | Yes (since Android 4.4) |
| `__cxa_thread_atexit_impl` | Yes | Yes |
| `pthread_mutex_t` size | 40 bytes | 4 bytes (futex-based) |
| `FILE*` buffering | 4 KB | 1 KB |
| `strerror_r` | POSIX or GNU variant | POSIX only |
| Stack protector | `-fstack-protector` | `-fstack-protector-strong` |
| `mallopt()` | Yes | Scudo allocator interface |

### 0A.7.3 Scudo Allocator

Since Android 11, bionic uses **Scudo** (instead of jemalloc for 64-bit, dlmalloc for 32-bit). Scudo is security-hardened:
- Checksummed headers on every allocation.
- Quarantine for freed blocks (use-after-free detection).
- Randomized allocation patterns.
- Hardware support via MTE (Memory Tagging Extension) on ARMv8.5+ (Pixel 8+).

```bash
adb shell "echo 'DeallocationTypeMismatch:true' > /data/local/tmp/scudo_options"
adb shell "SCUDO_OPTIONS=help /system/bin/ls"   # prints available options
```

### 0A.7.4 The Dynamic Linker (`/system/bin/linker64`)

Bionic's linker supports **namespaces** (Level 0, §0.4.1). It also handles:
- `.note.android.ident` — identifies the ABI level of a binary.
- `DT_ANDROID_REL` / `DT_ANDROID_RELA` — Android-specific relocation encoding (saves ~5% binary size).
- `DF_1_GLOBAL` — marks libraries that should be visible across namespace boundaries.
- **Prelink** (deprecated) — was used to speed up library loading.
- **RELR** relocations (Android 9+) — compact relative relocation encoding.

---

## 0A.8 — Android's IPC Mechanisms (Beyond Binder)

While Binder dominates, Android also uses:

| Mechanism | Use case | Example |
|-----------|----------|---------|
| **Unix sockets** | init ↔ property service, logd, zygote | `/dev/socket/zygote`, `/dev/socket/property_service` |
| **Netlink** | Kernel ↔ userspace events | ueventd, connectivity (NETLINK_KOBJECT_UEVENT) |
| **Shared memory (memfd)** | Large buffer transfer | Gralloc buffers, audio track buffers, VHAL high-rate properties |
| **Pipes** | Parent-child stdio | Zygote → child setup |
| **Signals** | Process control | SIGKILL (lmkd), SIGSTOP (freezer debug) |
| **Filesystem** | Implicit IPC | Properties (`/dev/__properties__`), procfs |
| **vsock** | Hypervisor ↔ guest (automotive) | Cuttlefish host↔guest, QNX↔Android IPC |

Each has different SELinux enforcement. Binder has the richest policy model (`binder_call`, `binder_transfer`, `add_service`, `find`). Sockets use `connectto`, `sendto`, `recvfrom`.

---

## 0A.9 — Power Management Kernel Interfaces

### 0A.9.1 Wakelocks and Wakeup Sources

The concept: Android devices should be in suspend (S2RAM or deeper) whenever the screen is off and no app needs the CPU. **Wakeup sources** keep the kernel awake:

```bash
adb shell cat /sys/kernel/debug/wakeup_sources
# name        active_count  event_count  wakeup_count  expire_count  ...
# PowerManagerService  3542  3542  0  3542  ...
# bluetooth_timer      128   128   0  128   ...
```

**PowerManagerService** in `system_server` is the main user — it acquires a kernel wakelock whenever any app holds a `PowerManager.WakeLock` and releases when all user wakelocks are gone.

### 0A.9.2 Runtime PM and Autosuspend

Individual devices (GPU, modem, USB) can suspend independently via Linux Runtime PM:

```bash
adb shell cat /sys/devices/.../power/runtime_status   # active / suspended / suspending
adb shell cat /sys/devices/.../power/autosuspend_delay_ms
```

A GPU that autosuspends after 100 ms of inactivity saves significant power in bursty workloads.

### 0A.9.3 CPU Idle and cpuidle

ARM cores have multiple idle states (WFI, power collapse, cluster collapse):

```bash
adb shell cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
# WFI, C1, C2, C3...
adb shell cat /sys/devices/system/cpu/cpu0/cpuidle/state*/time
# Total time spent in each state (µs)
adb shell cat /sys/devices/system/cpu/cpu0/cpuidle/state*/usage
# Number of times entered
```

For battery analysis: you want >90% time in the deepest idle state when screen is off. If not, something is waking the CPUs — find it in `wakeup_sources`.

---

## 0A.10 — Verifying This Deep Dive

After reading this supplement, you should be able to:

1. Explain why a process stuck in D-state cannot be killed and how to find what it's waiting for.
2. Draw the memory layout of an Android app process and identify which regions are CoW-shared with Zygote.
3. Explain the single-copy optimization in the Binder driver.
4. Read a device tree node and explain what kernel driver will be instantiated.
5. Explain why EROFS on `/system` improves boot time and OTA size simultaneously.
6. Use `/proc/pressure/memory` to diagnose system-wide memory pressure.
7. Explain the difference between `ashmem`, `memfd`, `DMA-BUF`, and Binder buffers.
8. Read a ramoops log to diagnose a kernel panic from a previous boot.

---

⬅️ Back to **[Level 0 — Foundations](./level-00-foundations.md)** | ➡️ **[Level 1 — AOSP Basics](./level-01-aosp-basics.md)**

