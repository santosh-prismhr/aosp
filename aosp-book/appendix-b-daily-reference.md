# Appendix B — AOSP Developer Daily Reference

> *"A reference you can reach for mid-debug at 2 a.m. — no preamble, just the answer."*

This appendix is a **cheatsheet and cookbook** for daily AOSP platform work. Bookmark it. Print it. Keep it next to your terminal.

---

## B.1 — Environment Setup (One-Time)

### Ubuntu 22.04 / 24.04 LTS

```bash
# Build dependencies
sudo apt install -y git-core gnupg flex bison build-essential zip curl \
  zlib1g-dev libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev \
  libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig \
  ccache rsync python3 python-is-python3 openjdk-17-jdk \
  libssl-dev bc cpio lz4 device-tree-compiler

# repo tool
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod +x ~/bin/repo
echo 'export PATH=~/bin:$PATH' >> ~/.bashrc

# ccache (100 GB recommended)
echo 'export USE_CCACHE=1' >> ~/.bashrc
echo 'export CCACHE_EXEC=/usr/bin/ccache' >> ~/.bashrc
echo 'export CCACHE_DIR=$HOME/.ccache' >> ~/.bashrc
source ~/.bashrc && ccache -M 100G

# KVM (for Cuttlefish)
sudo apt install -y qemu-kvm libvirt-daemon-system cpu-checker
sudo usermod -aG kvm,render,video $USER
# logout/login

# Git identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global color.ui auto
```

### macOS Differences

```bash
# macOS requires a case-sensitive APFS volume for AOSP
hdiutil create -type SPARSE -fs 'Case-sensitive APFS' -size 300g aosp.sparseimage
hdiutil attach aosp.sparseimage -mountpoint /Volumes/aosp

# Install deps via Homebrew
brew install git curl python@3.11 openjdk@17 ccache rsync
```

---

## B.2 — Repo Cheatsheet

```bash
# Init
repo init -u https://android.googlesource.com/platform/manifest \
          -b android-14.0.0_r30 --depth=1 --partial-clone --clone-filter=blob:limit=10M

# Sync (initial)
repo sync -c -j$(nproc) --no-tags --no-clone-bundle --optimized-fetch

# Daily re-sync
repo sync -c -j8 --optimized-fetch

# Status across all projects
repo status

# Create a topic branch everywhere
repo start my-feature --all

# Upload to Gerrit
repo upload .

# Run a command in every project
repo forall -c 'git log --oneline -1'

# Find which project owns a path
repo info frameworks/base

# Diff manifest (what changed since last sync)
repo diff

# Abandon all local branches
repo abandon my-feature

# Snapshot manifest (for reproducibility)
repo manifest -r -o snapshot.xml
```

---

## B.3 — Build Commands

```bash
source build/envsetup.sh

# Common lunch targets
lunch aosp_cf_x86_64_phone-trunk_staging-userdebug   # Cuttlefish phone
lunch aosp_cf_x86_64_auto-trunk_staging-userdebug    # Cuttlefish AAOS
lunch aosp_cf_arm64_phone-trunk_staging-userdebug    # ARM64 CF
lunch aosp_arm64-trunk_staging-userdebug             # Generic ARM64

# Build
m -j$(nproc)                 # full build
m installclean               # wipe product out (keep host tools)
m clean                      # nuke everything
m <module>                   # single module
mma                          # module + all deps in CWD
mmm <path>                   # module at path

# Useful build targets
m droid                      # default
m selinux_policy             # just policy
m vendorimage                # just vendor.img
m systemimage                # just system.img
m bootimage                  # just boot.img
m dist                       # create distributable (target_files.zip)
m sdk                        # build SDK
m cts                        # build CTS
m vts                        # build VTS

# Build info
m printvars                  # all Make vars
get_build_var TARGET_PRODUCT
get_build_var PLATFORM_VERSION
get_build_var PRODUCT_PACKAGES | tr ' ' '\n' | sort

# Find module owner
grep -rn '"libfoo"' --include=Android.bp .
gomod libfoo                 # if configured
```

---

## B.4 — Cuttlefish Operations

```bash
# Launch (basic)
launch_cvd --daemon --cpus=4 --memory_mb=4096

# Launch (multi-display, AAOS)
launch_cvd --daemon --cpus=4 --memory_mb=8192 \
  --display0=width=1920,height=1080,dpi=160 \
  --display1=width=1280,height=720,dpi=120

# Launch multi-instance
launch_cvd --num_instances=4 --base_instance_num=1

# Stop
stop_cvd

# Logs
tail -f ~/cuttlefish_runtime/logs/launcher.log
tail -f ~/cuttlefish_runtime/logs/kernel.log
tail -f ~/cuttlefish_runtime/logs/logcat

# WebRTC display (open in browser)
# https://localhost:8443

# Custom kernel
launch_cvd --kernel_path=/path/to/bzImage \
           --initramfs_path=/path/to/initramfs.img

# Snapshot/restore (Android 14+)
adb shell cmd incidentd section 3000 > snapshot.pb
```

---

## B.5 — ADB Power Commands

```bash
# Basics
adb devices -l
adb root                               # restart adbd as root (userdebug)
adb unroot
adb remount                            # mount /system, /vendor rw
adb disable-verity && adb reboot       # needed before remount on user builds
adb enable-verity && adb reboot

# Shell
adb shell
adb shell <command>
adb -s <serial> shell                  # specific device

# File transfer
adb push local remote
adb pull remote local
adb sync system                        # sync changed system files (requires remount)

# Reboot variants
adb reboot
adb reboot bootloader                  # to fastboot
adb reboot recovery
adb reboot sideload                    # to sideload mode

# App management
adb install -r -g -t app.apk
adb uninstall com.foo.bar
adb shell pm list packages -f
adb shell pm clear com.foo.bar
adb shell pm grant com.foo.bar android.permission.CAMERA

# Activity management
adb shell am start -n com.android.settings/.Settings
adb shell am force-stop com.foo.bar
adb shell am broadcast -a android.intent.action.BOOT_COMPLETED
adb shell am start-foreground-service -n com.foo/.MyService

# Input simulation
adb shell input tap 500 500
adb shell input text "hello"
adb shell input keyevent KEYCODE_HOME

# Screenshots/video
adb shell screencap /sdcard/screen.png && adb pull /sdcard/screen.png
adb shell screenrecord /sdcard/video.mp4  # Ctrl-C to stop, then pull

# Bug report
adb bugreportz                         # compressed; prefer this
```

---

## B.6 — Debugging & Diagnostics

### Logcat

```bash
adb logcat -v threadtime               # best format
adb logcat -b all                      # all buffers
adb logcat -s TAG1:V TAG2:E            # filter by tag
adb logcat --pid=<pid>                 # filter by process
adb logcat -d > log.txt                # dump and exit
adb logcat -c                          # clear buffers
adb logcat -G 16M                      # increase buffer size
```

### System Dumps

```bash
adb shell dumpsys                      # everything (huge)
adb shell dumpsys activity             # ActivityManager
adb shell dumpsys activity processes   # process list with oom_adj
adb shell dumpsys package com.foo      # package details
adb shell dumpsys window               # WindowManager
adb shell dumpsys SurfaceFlinger       # display, layers
adb shell dumpsys meminfo              # system-wide memory
adb shell dumpsys meminfo <pid>        # per-process
adb shell dumpsys battery              # battery state
adb shell dumpsys power                # wakelocks
adb shell dumpsys alarm                # alarms
adb shell dumpsys jobscheduler         # jobs
adb shell dumpsys connectivity         # network
adb shell dumpsys wifi                 # Wi-Fi state
adb shell dumpsys bluetooth_manager    # BT
adb shell dumpsys car_service          # AAOS CarService
adb shell dumpsys thermalservice       # thermal
adb shell service list                 # all registered services
```

### Native Crashes

```bash
adb shell ls -lt /data/tombstones/
adb pull /data/tombstones/tombstone_00 .
# Symbolize:
development/scripts/stack tombstone_00 --symbols-dir=out/target/product/<dev>/symbols
# Or:
ndk-stack -sym out/target/product/<dev>/symbols < tombstone_00
```

### ANR Analysis

```bash
adb pull /data/anr/ ./anr/
# Read traces.txt — find main thread, look for:
#   - binder call blocked
#   - lock held by another thread
#   - I/O on main thread
```

### SELinux

```bash
adb shell getenforce                            # Enforcing
adb shell dmesg | grep "avc:"                   # kernel denials
adb logcat -b all -d | grep "avc:"              # all denials
# Generate allow rule (for analysis only, not production):
adb logcat -b all -d | audit2allow -p out/target/product/<dev>/root/sepolicy
# Check contexts:
adb shell ls -Z /vendor/bin/my_hal
adb shell ps -Z | grep my_hal
adb shell getprop -Z vendor.foo
```

### Properties

```bash
adb shell getprop                               # all properties
adb shell getprop ro.build.fingerprint
adb shell setprop debug.foo bar
adb shell watchprops                            # watch changes
```

### Binder

```bash
adb shell dumpsys -l                            # all binder services
adb shell cat /sys/kernel/debug/binder/stats    # driver stats
adb shell cat /sys/kernel/debug/binder/transactions  # in-flight
adb shell cat /proc/<pid>/fd | wc -l            # fd count
```

### HAL

```bash
adb shell lshal                                 # all registered HALs
adb shell lshal --init-vintf                    # generate manifest from live
adb shell lshal debug <fqname>                  # HAL-specific debug dump
adb shell getprop | grep hal                    # HAL-related properties
```

---

## B.7 — Performance Tracing

### Perfetto

```bash
# Quick trace (10 seconds, common categories)
adb shell perfetto -o /data/misc/perfetto-traces/trace.pftrace -t 10s \
  sched freq am wm view binder gfx

# Pull and open in ui.perfetto.dev
adb pull /data/misc/perfetto-traces/trace.pftrace .

# Boot trace (must set before reboot)
cat > /tmp/boot_trace.cfg << 'EOF'
buffers { size_kb: 65536 fill_policy: RING_BUFFER }
data_sources { config { name: "linux.ftrace"
  ftrace_config {
    ftrace_events: "sched/sched_switch"
    ftrace_events: "power/cpu_frequency"
    ftrace_events: "binder/binder_transaction"
    ftrace_events: "f2fs/f2fs_sync_file_enter"
    atrace_categories: "am" atrace_categories: "wm"
    atrace_categories: "view" atrace_categories: "dalvik"
  }
}}
data_sources { config { name: "linux.process_stats"
  process_stats_config { scan_all_processes_on_start: true proc_stats_poll_ms: 1000 }
}}
duration_ms: 60000
EOF
adb push /tmp/boot_trace.cfg /data/misc/perfetto-configs/boottrace.pbtxt
adb shell setprop persist.traced.enable 1
adb reboot
# After boot: adb pull /data/misc/perfetto-traces/...
```

### simpleperf

```bash
# CPU profiling a process
adb shell simpleperf record -p <pid> -g --duration 5 -o /data/local/tmp/perf.data
adb pull /data/local/tmp/perf.data .
simpleperf report -i perf.data --sort comm,dso,symbol

# System-wide for 10 s
adb shell simpleperf stat -a --duration 10

# Flamegraph
simpleperf report-sample --symfs out/target/product/<dev>/symbols -i perf.data \
  | inferno/inferno.sh > flame.html
```

### systrace (legacy, still works)

```bash
python3 external/chromium-trace/systrace.py -t 5 -o trace.html \
  sched gfx view wm am dalvik binder_driver
```

---

## B.8 — OTA & Flashing

```bash
# Build OTA
m dist
# target_files at: out/dist/<product>-target_files-*.zip

# Full OTA
ota_from_target_files out/dist/<product>-target_files.zip ota_full.zip

# Incremental OTA
ota_from_target_files --incremental_from=PREV-target_files.zip \
  CURR-target_files.zip ota_incr.zip

# Sideload
adb reboot sideload
adb sideload ota_full.zip

# Fastboot flash
adb reboot bootloader
fastboot flashall -w                   # flash all + wipe data
fastboot flash system system.img
fastboot flash vendor vendor.img
fastboot flash boot boot.img
fastboot flash vbmeta vbmeta.img --disable-verity --disable-verification  # debug only
fastboot reboot

# Check AVB state
fastboot getvar all 2>&1 | grep -i vbmeta
adb shell avbctl get-verity
adb shell avbctl get-verification

# A/B slot info
adb shell bootctl get-current-slot
adb shell bootctl get-suffix 0
adb shell bootctl mark-boot-successful
```

---

## B.9 — SELinux Quick Reference

```bash
# File to type mapping
# /system/etc/selinux/plat_file_contexts
# /vendor/etc/selinux/vendor_file_contexts

# Check a file's context
ls -Z /path/to/file

# Change context temporarily (debug only)
chcon u:object_r:my_type:s0 /path/to/file

# Restore contexts
restorecon -R /vendor/etc/

# Compile policy
m selinux_policy

# Check neverallow violations at build time
m checkpolicy   # implicit in m

# Common macros (system/sepolicy/public/te_macros):
# init_daemon_domain(D)
# binder_use(D)
# hwbinder_use(D)
# binder_call(C, S)
# add_service(D, svc)
# add_hwservice(D, hwsvc)
# r_file_perms  = { getattr open read ioctl lock map }
# rw_file_perms = r_file_perms + { write append }
# r_dir_perms   = { open getattr read search ioctl lock }
```

---

## B.10 — AAOS Quick Reference

```bash
# Build AAOS Cuttlefish
lunch aosp_cf_x86_64_auto-trunk_staging-userdebug && m -j$(nproc)

# Launch with cluster display
launch_cvd --display0=width=1920,height=1080,dpi=160 \
           --display1=width=1280,height=480,dpi=120

# VHAL commands
adb shell cmd car_service inject-vhal-event <propId> <value>
adb shell cmd car_service inject-vhal-event <propId> -a <areaId> <value>
adb shell cmd car_service get-vhal-property <propId>
adb shell dumpsys car_service --hal

# Car Service
adb shell dumpsys car_service
adb shell dumpsys car_service --services CarPropertyService
adb shell dumpsys car_service --services CarAudioService
adb shell dumpsys car_service --services CarPowerManagementService
adb shell dumpsys car_service --services CarUserService
adb shell dumpsys car_service --user

# User management
adb shell pm list users
adb shell cmd car_service create-user "Passenger"
adb shell cmd car_service switch-user <userId>
adb shell cmd car_service get-initial-user-info

# Power
adb shell cmd car_service power-off
adb shell cmd car_service suspend
adb shell cmd car_service garage-mode on
adb shell cmd car_service garage-mode off

# UX Restrictions
adb shell cmd car_service day-night-mode day
adb shell cmd car_service day-night-mode night
```

---

## B.11 — CTS/VTS Quick Reference

```bash
# Run CTS module
cd android-cts/tools && ./cts-tradefed
> run cts -m CtsPermissionTestCases
> run cts -m CtsPermissionTestCases -t android.permission.cts.PermissionTest#testFoo
> run cts --shard-count 4
> retry --retry 0

# Run VTS module
cd android-vts/tools && ./vts-tradefed
> run vts -m VtsHalLightTargetTest

# Run via atest (in-tree, faster iteration)
atest CtsPermissionTestCases
atest VtsHalLightTargetTest
atest --host MyHostTest
atest -v MyTest -- --log-level verbose

# Build CTS
m cts -j$(nproc)
# Output: out/host/linux-x86/cts/android-cts.zip
```

---

## B.12 — Git Patterns for AOSP

```bash
# Find which commit introduced a bug (binary search)
git bisect start
git bisect bad HEAD
git bisect good android-14.0.0_r5
git bisect run <test_script.sh>

# Cherry-pick across projects
cd frameworks/base
git cherry-pick <sha1>

# Generate a patch for upstream
git format-patch -1 HEAD
# Apply:
git am < 0001-*.patch

# Find who last touched a file
git log --follow -p -- path/to/file
git blame path/to/file

# Find all changes between two tags
git log --oneline android-14.0.0_r5..android-14.0.0_r30

# Search commit messages
git log --all --grep="binder deadlock"

# Revert a commit
git revert <sha1>
```

---

## B.13 — Key File Locations (Memorize)

| What | Path |
|------|------|
| init main script | `system/core/rootdir/init.rc` |
| init source | `system/core/init/` |
| Property contexts (system) | `system/sepolicy/private/property_contexts` |
| Property contexts (vendor) | `device/<v>/<p>/sepolicy/vendor/property_contexts` |
| File contexts (system) | `system/sepolicy/private/file_contexts` |
| SELinux macros | `system/sepolicy/public/te_macros` |
| ServiceManager | `frameworks/native/cmds/servicemanager/` |
| Binder driver | `drivers/android/binder.c` (kernel) |
| SystemServer startup | `frameworks/base/services/java/com/android/server/SystemServer.java` |
| Zygote | `frameworks/base/cmds/app_process/app_main.cpp` |
| ActivityManagerService | `frameworks/base/services/core/java/com/android/server/am/` |
| WindowManagerService | `frameworks/base/services/core/java/com/android/server/wm/` |
| PackageManagerService | `frameworks/base/services/core/java/com/android/server/pm/` |
| SurfaceFlinger | `frameworks/native/services/surfaceflinger/` |
| VHAL AIDL | `hardware/interfaces/automotive/vehicle/aidl/` |
| CarService | `packages/services/Car/service/` |
| OTA update_engine | `system/update_engine/` |
| AVB tools | `external/avb/` |
| Soong build system | `build/soong/` |
| Envsetup | `build/envsetup.sh` |
| Prebuilt clang | `prebuilts/clang/host/linux-x86/` |
| CTS | `cts/` |
| VTS | `test/vts/` |

---

## B.14 — Common Error → Fix Lookup

| Error / Symptom | Root Cause | Fix |
|----------------|-----------|-----|
| `avc: denied { read }` | Missing SELinux allow rule | Label file narrowly, add allow to specific type |
| `dlopen failed: library not found` | Linker namespace violation | Move lib to correct partition or add to LL-NDK/VNDK |
| `TransactionTooLargeException` | Binder buffer exhausted | Reduce payload; paginate; use shared memory |
| `Watchdog: *** WATCHDOG KILLING SYSTEM PROCESS` | system_server thread blocked >60s | Find lock holder in ANR traces |
| `init: Service ... restart ...` | HAL crash loop | Check tombstone + SELinux denials |
| `vintf: ... not compatible` | Vendor manifest doesn't satisfy system matrix | Update manifest or add missing HAL |
| `java.lang.UnsatisfiedLinkError` | JNI lib not in app's linker namespace | Check `uses-native-library` in manifest |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | Signature mismatch | Uninstall first, or sign with correct key |
| `ninja: error: ... missing` | Stale intermediates | `m installclean && m` |
| `BOOT_PROGRESS stuck at zygote` | dexopt hung or OOM | Increase swap, reduce `-j`, check disk space |
| `device offline` after `adb root` | adbd not compiled in `userdebug` | Use `userdebug` variant |
| `Timed out waiting for service` | HAL not registered in VINTF | Add to manifest.xml + check service_contexts |

---

## B.15 — Useful One-Liners

```bash
# Which process owns a binder service
adb shell service check <service_name>

# Watch property changes in real-time
adb shell watchprops

# Trigger GC in a process
adb shell kill -10 <pid>

# Force stop and cold-start an app (for benchmarking)
adb shell am force-stop com.foo && adb shell am start -W com.foo/.MainActivity

# Capture heap dump
adb shell am dumpheap <pid> /data/local/tmp/heap.hprof && adb pull /data/local/tmp/heap.hprof

# List open files for a process
adb shell ls -la /proc/<pid>/fd

# Check if a process is frozen (Cached App Freezer)
adb shell cat /proc/<pid>/cgroup | grep freezer

# Get system_server PID
adb shell pidof system_server

# Restart system_server without full reboot
adb shell stop && adb shell start

# Measure cold boot time
adb shell cat /proc/stat | head -1  # before reboot, note uptime
adb reboot && adb wait-for-device && adb shell getprop sys.boot_completed
```

---

⬅️ Back to **[Table of Contents](./README.md)**

