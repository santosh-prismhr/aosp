# Level 1 — AOSP Basics (Junior AOSP Engineer)

> *"The single most expensive lesson in your AOSP career is learning to read the source tree fluently. Do it now. Everything else compounds."*

By the end of this Level you will have:
- A working AOSP checkout.
- A successful Cuttlefish build.
- A booted virtual device, accessible via `adb`.
- The mental map of the source tree that every senior engineer carries in their head.

---

## Chapter 1.1 — AOSP Architecture Overview

### 1.1.1 The Five-Layer Architecture (Treble Era)

```
                       ┌──────────────────────────┐
   Apps (APKs) ───────▶│     Java Framework       │
                       │   (frameworks/base)      │
                       │  system_server processes │
                       └────────────┬─────────────┘
                                    │ Binder (AIDL stable)
                       ┌────────────▼─────────────┐
                       │    Native Services       │
                       │  surfaceflinger, audio,  │
                       │   mediaserver, netd...   │
                       └────────────┬─────────────┘
                                    │ Binder (HIDL/AIDL HAL)
                       ┌────────────▼─────────────┐
                       │      HAL Processes       │
                       │ (hardware/interfaces +   │
                       │   vendor/<oem>/...)      │
                       └────────────┬─────────────┘
                                    │ ioctl/sysfs/netlink
                       ┌────────────▼─────────────┐
                       │      Linux Kernel        │
                       │  (kernel/common +        │
                       │   vendor BSP modules)    │
                       └──────────────────────────┘
```

### 1.1.2 Partition Map (Modern A/B Device)

| Partition | Owner | Purpose | Updateable |
|-----------|-------|---------|------------|
| `boot_a/b` | Google/SoC | kernel + ramdisk | OTA |
| `init_boot_a/b` | Google | generic ramdisk (Android 13+) | OTA |
| `vendor_boot_a/b` | SoC/OEM | vendor ramdisk | OTA |
| `system_a/b` | Google | framework | OTA / Mainline (APEX) |
| `system_ext_a/b` | OEM | system extensions | OTA |
| `product_a/b` | OEM | product-specific apps/configs | OTA |
| `vendor_a/b` | SoC/OEM | HALs, vendor libs | OTA |
| `odm_a/b` | ODM | ODM customizations | OTA |
| `vbmeta_a/b` | Google/OEM | AVB metadata | OTA |
| `userdata` | — | user data | wiped on factory reset |
| `metadata` | — | encryption metadata | persistent |
| `misc` | — | bootloader messages | persistent |

🔗 Deep dive: **Level 4 — BSP & Device Bring-Up**, **Level 9 — Production & Release**.

---

## Chapter 1.2 — Source Tree Layout (Memorize This)

A senior engineer can navigate AOSP blindfolded. Here is the canonical map.

```
aosp/
├── art/                       # Android Runtime (dex2oat, libart)
├── bionic/                    # libc, libm, libdl, dynamic linker
├── bootable/                  # recovery image, libbootloader
├── build/                     # Soong, Make, envsetup
│   ├── soong/
│   ├── make/
│   └── blueprint/
├── cts/                       # Compatibility Test Suite (Java/host)
├── dalvik/                    # legacy; mostly dexdump/dexlist tools now
├── developers/                # samples
├── development/               # SDK & dev tools
├── device/                    # Per-device configs (BoardConfig, AndroidProducts)
│   └── google/
│       └── cuttlefish/        # ★ Cuttlefish device tree
├── external/                  # Third-party (kernel headers, openssl, libcxx, etc.)
├── frameworks/
│   ├── av/                    # audio, camera, media native
│   ├── base/                  # ★ THE Java framework (Activity, Window, etc.)
│   ├── hardware/              # framework-side HAL helpers
│   ├── libs/                  # gui, ui, sensorservice
│   ├── native/                # surfaceflinger, binder, libbinder
│   ├── opt/
│   └── proto_logging/
├── hardware/
│   ├── interfaces/            # ★ HIDL & AIDL HAL definitions
│   ├── google/
│   ├── libhardware/           # legacy HAL headers
│   └── ril/
├── kernel/                    # not present in AOSP source; built separately
├── libcore/                   # Java core libraries (java.*, javax.*)
├── packages/
│   ├── apps/                  # System apps (Settings, Launcher3, etc.)
│   ├── modules/               # Mainline modules (APEX)
│   ├── providers/             # Content providers (MediaProvider, ...)
│   └── services/              # Java system services (Telephony, etc.)
├── pdk/
├── platform_testing/          # Platform tests
├── prebuilts/                 # Toolchains, prebuilt libs, NDK
├── sdk/
├── system/
│   ├── apex/                  # APEX module infra
│   ├── bpf/
│   ├── core/                  # ★ init, libutils, libcutils, adb, toolbox
│   ├── extras/                # perfetto helpers, simpleperf
│   ├── hardware/interfaces/   # AIDL HAL infra
│   ├── libbase/
│   ├── libfmq/                # Fast Message Queues
│   ├── libhidl/
│   ├── libvintf/              # ★ Vendor Interface (VINTF)
│   ├── logging/
│   ├── netd/                  # network daemon
│   ├── sepolicy/              # ★ SELinux policy
│   ├── server_configurable_flags/
│   ├── update_engine/         # ★ A/B OTA engine
│   └── vold/                  # storage daemon
├── test/
├── toolchain/                 # clang, NDK build glue
├── tools/
└── vendor/                    # OEM/SoC closed code (in OEM trees)
```

🎯 **Staff-Level Insight:** Knowing *which directory owns a class of bugs* is half the job:

| Symptom | First place to look |
|---------|--------------------|
| App crashes on launch | `frameworks/base/services/core/java/.../am/` (ActivityManager) |
| HAL crash loop | `hardware/interfaces/<iface>/` + vendor `Android.bp` |
| Boot loop in early init | `system/core/init/`, device `*.rc` |
| SELinux denial | `system/sepolicy/` + `device/<vendor>/sepolicy/` |
| OTA fails to apply | `system/update_engine/` + `bootable/recovery/` |
| Camera frame drops | `frameworks/av/services/camera/` + vendor Camera HAL |
| Bluetooth pairing fail | `packages/modules/Bluetooth/` (APEX) + Bluetooth HAL |

---

## Chapter 1.3 — `repo`, Manifests, Branches, Tags

### 1.3.1 What `repo` Is

`repo` is a Python wrapper that orchestrates **hundreds of independent Git repositories** as if they were one. AOSP cannot live in a single Git repo (terabytes of history, divergent ownership). The **manifest** XML defines which repos at which revisions form a "build."

### 1.3.2 The Manifest

`.repo/manifests/default.xml` (a snippet):

```xml
<manifest>
  <remote name="aosp" fetch="https://android.googlesource.com/" />
  <default revision="refs/tags/android-14.0.0_r30" remote="aosp" sync-j="8" />

  <project path="frameworks/base"   name="platform/frameworks/base" />
  <project path="system/core"       name="platform/system/core" />
  <project path="device/google/cuttlefish"
           name="device/google/cuttlefish" />
  <!-- ... ~700 more projects ... -->
</manifest>
```

### 1.3.3 Branches and Tags You Care About

- `main` — bleeding edge (formerly `master`). **Do not** ship from here.
- `android-latest-release` — symlink branch to most recent dessert release.
- `android14-release`, `android15-release` — release branches.
- Tags: `android-14.0.0_r30` — exact security patch level / public release.

### 1.3.4 Initial Sync (Cuttlefish target)

🛠️ **Hands-On:**

```bash
# 1. Install repo
sudo apt install -y git curl python3 python-is-python3
mkdir -p ~/bin && curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod +x ~/bin/repo
export PATH=~/bin:$PATH

# 2. Configure git identity
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"

# 3. Init a checkout (Android 14, latest patch)
mkdir ~/aosp && cd ~/aosp
repo init -u https://android.googlesource.com/platform/manifest \
          -b android-14.0.0_r30 --depth=1 --partial-clone --clone-filter=blob:limit=10M

# 4. Sync (this takes 1–4 hours on first run; ~150 GB)
repo sync -c -j$(nproc) --no-tags --no-clone-bundle --optimized-fetch
```

⚠️ **OEM Pitfall:** Skipping `--depth=1` on a CI builder. You will burn a TB of disk for branches you never check out. For developer workstations, `--depth=1` plus `--partial-clone` is the modern default.

### 1.3.5 Daily `repo` Commands

```bash
repo status                 # local changes across all projects
repo branch                 # current topic branches
repo start <topic> --all    # create branch in every project
repo upload                 # push to Gerrit
repo forall -c 'git log --oneline -1'   # run command in each project
repo info                   # manifest details
```

---

## Chapter 1.4 — The Build System (Soong + Make)

### 1.4.1 Soong / Blueprint

Soong replaces Make for new code. Modules are declared in `Android.bp`:

```blueprint
cc_binary {
    name: "myhello",
    srcs: ["main.cpp"],
    shared_libs: ["liblog", "libbase"],
    cflags: ["-Wall", "-Werror"],
    vendor: false,
}

cc_library_shared {
    name: "libfoo",
    srcs: ["foo.cpp"],
    export_include_dirs: ["include"],
    vendor_available: true,
}

android_app {
    name: "MyApp",
    srcs: ["src/**/*.java"],
    sdk_version: "current",
    certificate: "platform",
    privileged: true,
}
```

Module **types** include: `cc_binary`, `cc_library_shared`, `cc_library_static`, `java_library`, `android_app`, `prebuilt_etc`, `apex`, `aidl_interface`, `cc_test`, `filegroup`, `genrule`.

### 1.4.2 Make (legacy)

Some projects still have `Android.mk`. Soong calls Kati to convert these to Ninja. New code: **always Soong**.

### 1.4.3 `envsetup.sh` and `lunch`

```bash
source build/envsetup.sh
lunch aosp_cf_x86_64_phone-trunk_staging-userdebug
```

The `lunch` target is `<product>-<release_config>-<variant>`:

- **product** — `aosp_cf_x86_64_phone` (Cuttlefish 64-bit phone), `aosp_cf_x86_64_auto` (AAOS), `aosp_arm64`, etc.
- **release_config** — `trunk_staging`, `trunk`, `next` (Android 14+ replaced the old 3-tuple).
- **variant** — `eng` / `userdebug` / `user`:
  - `eng` — engineering, all debug, root by default.
  - `userdebug` — release-like but `adb root` works; **what you use daily**.
  - `user` — production; no adb root, locked down.

### 1.4.4 The Build Commands

```bash
m                    # build everything for current lunch
m -j$(nproc)         # parallel
m droid              # default top-level target
m installclean       # wipe out/target/product/<device>
m clean              # nuke entire out/
mma                  # build module(s) in CWD and dependencies
mm                   # build only module in CWD
mmm path/to/dir      # build module at path
m sync               # rebuild + sync to running device (if mounted RW)
```

### 1.4.5 Output Layout

```
out/
├── host/linux-x86/             # host tools (adb, fastboot, aapt2, signapk)
├── soong/                      # soong intermediates
└── target/product/vsoc_x86_64/
    ├── system.img              # ★
    ├── vendor.img              # ★
    ├── boot.img                # ★
    ├── super.img               # combined dynamic partitions
    ├── ramdisk.img
    ├── system/                 # extracted view (useful for grep)
    ├── vendor/
    ├── obj/                    # intermediates
    └── symbols/                # unstripped binaries (for debugging!)
```

🎯 **Staff-Level Insight:** `out/target/product/<device>/symbols/` is gold for crash debugging. Always keep these when triaging tombstones — the addresses in a tombstone match the symbols here.

---

## Chapter 1.5 — Building AOSP for Cuttlefish

### 1.5.1 Why Cuttlefish

- Officially supported by Google.
- Runs on a normal x86_64 Linux host (KVM).
- Boots a real Android image (no emulator-specific shortcuts).
- Used internally at Google for CTS, framework dev, and crashes can be triaged like real devices.
- Supports phone, auto, foldable, wear, tablet form factors.

### 1.5.2 Host Setup

```bash
# Verify KVM
sudo apt install -y cpu-checker
kvm-ok                                  # must say "KVM acceleration can be used"
sudo usermod -aG kvm,render,video $USER
# log out / log in

# Build dependencies (Ubuntu 22.04+)
sudo apt install -y git-core gnupg flex bison build-essential zip curl \
  zlib1g-dev libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev \
  libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig \
  ccache rsync python3 python-is-python3 openjdk-17-jdk
```

### 1.5.3 The Build

🛠️ **Hands-On — Your First Build:**

```bash
cd ~/aosp
source build/envsetup.sh
lunch aosp_cf_x86_64_phone-trunk_staging-userdebug
m -j$(nproc)
```

Expected time on a 16-core / 32 GB box with `ccache`: **45–90 minutes** clean, **5–15 minutes** incremental.

### 1.5.4 ccache (Strongly Recommended)

```bash
export USE_CCACHE=1
export CCACHE_EXEC=/usr/bin/ccache
export CCACHE_DIR=$HOME/.ccache
ccache -M 100G
```

Add these to `~/.bashrc`. A second clean build will drop to ~25% of the original time.

### 1.5.5 Common First-Build Failures

| Error | Cause | Fix |
|-------|-------|-----|
| `OpenJDK 11 detected` | Wrong JDK | `sudo update-alternatives --config java` → JDK 17 |
| `Out of memory: Killed` during `r8` / `dex2oat` | <16 GB RAM, no swap | Add 16 GB swap; reduce `-j` |
| `repo sync` fails on `device/google/cuttlefish-vmm` | Manifest depth too shallow | `repo sync -j8` without `--depth=1` for that project |
| `error: unrecognized command-line option '-Wno-...'` | Host clang too new/old | Use the prebuilt clang in `prebuilts/clang/host/linux-x86/` |
| `ninja: error: '... .img', needed by 'droid'` missing | Broken intermediate | `m installclean` then rebuild |
| `unable to find package android.hardware.foo@1.0` | HIDL/AIDL not registered | Check `compatibility_matrix.xml` and `manifest.xml` |

🐞 **Common Production Bug (Real story):** Build was green for everyone except one engineer hitting `ld.lld: error: undefined symbol`. Cause: their `~/.ccache` was corrupted from a previous interrupted build. `ccache -C` (clear) + `m` solved it. **Lesson:** when "it builds for everyone but me," suspect host environment first.

---

## Chapter 1.6 — Running Cuttlefish

### 1.6.1 Launching

```bash
# Still in the AOSP env (after lunch)
cd ~/aosp
launch_cvd \
  --daemon \
  --memory_mb=4096 \
  --cpus=4 \
  --gpu_mode=auto
```

Stop with: `stop_cvd`.

### 1.6.2 Connecting

```bash
adb devices
# 0.0.0.0:6520    device

adb shell
# vsoc_x86_64:/ $
```

WebRTC viewer: `https://localhost:8443/` in a browser (after `launch_cvd`).

### 1.6.3 Useful Cuttlefish Flags

```bash
launch_cvd --num_instances=2          # multi-device
launch_cvd --setupwizard_mode=REQUIRED
launch_cvd --enable_sandbox=true
launch_cvd --extra_kernel_cmdline="androidboot.selinux=permissive"  # debug only
launch_cvd --report_anonymous_usage_stats=N
```

### 1.6.4 Flashing a Real Device (preview)

🔗 Real device flashing is covered in **Level 4 — BSP & Device Bring-Up**. The two-line teaser:

```bash
adb reboot bootloader
fastboot flashall -w        # flashes all partitions, wipes /data
```

---

## Chapter 1.7 — `adb` Fluency

### 1.7.1 Daily Commands

```bash
adb devices -l                  # list with details
adb root                        # restart adbd as root (userdebug only)
adb remount                     # mount /system rw (after disable-verity + reboot)
adb shell                       # interactive shell
adb logcat -v threadtime        # the most useful logcat format
adb logcat -b all -d > log.txt  # dump all buffers
adb bugreport bug.zip           # full system snapshot
adb pull /data/anr/ ./anr/      # ANR traces
adb pull /data/tombstones/ .    # native crashes
adb shell dumpsys activity      # AM state
adb shell dumpsys SurfaceFlinger
adb shell dumpsys meminfo
adb install -r -g app.apk       # reinstall, grant runtime perms
adb push file /sdcard/Download/
adb forward tcp:8080 tcp:8080
adb shell am start -n com.android.settings/.Settings
adb shell pm list packages -f
adb shell setprop log.tag.MyTag VERBOSE
adb shell stop && adb shell start   # zygote restart
```

### 1.7.2 logcat Buffers

| Buffer | Contents |
|--------|----------|
| `main` | Default app logs |
| `system` | `system_server` and Java framework |
| `radio` | Telephony |
| `events` | Binary structured events (`EventLog`) |
| `crash` | Crashes only |
| `kernel` | dmesg (via `-b kernel`) |
| `all` | Everything |

🎯 **Staff-Level Insight:** Always capture `adb bugreport` before a device reboots after a crash. It bundles logcat (all buffers), dumpsys, dmesg, ANRs, tombstones, traces, and battery stats. It is the single most powerful artifact for cross-team triage.

---

## Chapter 1.8 — Debugging Common Build Failures (Deep)

### 1.8.1 Reading Soong Errors

Soong errors are noisy. The signal is at the **end** of the error block. Look for:

```
FAILED: out/soong/.intermediates/.../foo.o
clang++: error: ... in expansion of macro ...
```

Search the source for the failing `.cpp` line — it is the truth.

### 1.8.2 The Module Graph

```bash
m nothing                    # parse all Android.bp without building
out/soong/build.ninja        # generated ninja (huge)
# To find what defines a module:
grep -rn '"libfoo"' --include=Android.bp .
```

### 1.8.3 Investigating a Disappearing Module

```bash
m libfoo
# error: unknown target 'libfoo'
m PRODUCT-aosp_cf_x86_64_phone-userdebug   # product analysis
m printvars                  # show PRODUCT_PACKAGES, BOARD_*, etc.
get_build_var PRODUCT_PACKAGES | tr ' ' '\n' | grep libfoo
```

If the module isn't in `PRODUCT_PACKAGES`, it isn't included — even if it builds.

### 1.8.4 `ninja: rebuilding 'build.ninja'`

If you see Soong re-running on every `m`, something is touching a `.bp` or env var. Check:

```bash
strace -f -e trace=stat,openat -p $(pgrep -f soong_build) 2>&1 | head
# or
touch -d "1 hour ago" $(find . -name 'Android.bp' -newer out/soong/build.ninja)
```

🐞 **Common Production Bug:** A custom `BUILD_NUMBER` derived from `date +%s` invalidated Soong's cache every build. Lesson: never embed wall-clock time as a Soong input.

---

## Chapter 1.9 — Your First Code Change

🛠️ **Hands-On:** Change the boot animation log message.

```bash
cd ~/aosp/frameworks/base
git checkout -b learn-aosp-1
# Edit any file, e.g., frameworks/base/services/core/java/com/android/server/SystemServer.java
# Find: Slog.i(TAG, "Entered the Android system server!");
# Change it to: Slog.i(TAG, "Entered the Android system server! [my-build]");
m -j$(nproc)
adb sync                        # if /system mounted rw, otherwise rebuild image
adb reboot
adb logcat -s SystemServer
# You should see your message after reboot.
```

### 1.9.1 Updating the System Image Live

```bash
adb root && adb remount         # requires disable-verity once
adb sync system
adb shell stop && adb shell start
```

⚠️ **OEM Pitfall:** `adb sync` only syncs files with the same path that already exist. Newly added files won't appear. For new files, rebuild the image and re-flash.

---

## Chapter 1.10 — Verifying Level 1

You should now be able to:

1. Sync AOSP at a specific tag.
2. Build for `aosp_cf_x86_64_phone-trunk_staging-userdebug` cleanly.
3. Boot Cuttlefish and `adb shell` into it.
4. Find any class in `frameworks/base` within 10 seconds.
5. Diagnose 3+ classes of common build failure.
6. Modify framework source, rebuild, and verify on device.

If yes, you are now a **Junior AOSP Engineer**. Onward.

---

➡️ Continue to **[Level 2 — Framework Internals](./level-02-framework-internals.md)**

