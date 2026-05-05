# Level 4A — Deep Dive: Bootloaders, Kernel Bring-Up, and Partition Architecture

> *Supplement to Level 4. This chapter covers what "Embedded Android" and SoC vendor BSP guides teach — the complete boot chain from power-on to userspace, bootloader internals, partition table design, and the kernel/userspace handoff at production depth.*

---

## 4A.1 — The Complete Boot Chain

### 4A.1.1 Power-On to First Instruction

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. POWER ON                                                       │
│    SoC hard-wired: CPU0 starts executing from ROM address         │
│    (eFuse-selected boot device: eMMC, UFS, SPI NOR, USB)         │
├──────────────────────────────────────────────────────────────────┤
│ 2. BootROM (SoC-specific, burned in silicon, ~64-256 KB)         │
│    • Initializes minimal SRAM (no DRAM yet)                       │
│    • Reads first-stage bootloader from boot device                │
│    • Verifies signature against eFuse public key hash             │
│    • Jumps to first-stage bootloader in SRAM                      │
├──────────────────────────────────────────────────────────────────┤
│ 3. First-Stage Bootloader (SBL1 / PBL / SPL)                     │
│    • Initializes DRAM controller (DDR training)                   │
│    • Loads second-stage bootloader into DRAM                      │
│    • Verifies chain-of-trust signature                            │
│    • Configures basic clocks, power rails                         │
│    • Jumps to second-stage bootloader                             │
├──────────────────────────────────────────────────────────────────┤
│ 4. Second-Stage Bootloader (U-Boot / LK / ABL / crosvm firmware) │
│    • Full hardware init (display for splash, USB for fastboot)    │
│    • Reads A/B slot metadata (misc partition)                     │
│    • Loads and verifies vbmeta (AVB 2.0)                          │
│    • Loads kernel + ramdisk from boot_a/b                         │
│    • Loads DTB from vendor_boot / dtbo                            │
│    • Sets kernel cmdline (androidboot.* properties)               │
│    • Jumps to kernel entry point                                  │
├──────────────────────────────────────────────────────────────────┤
│ 5. Linux Kernel                                                   │
│    • Decompresses itself (if compressed)                          │
│    • Initializes virtual memory, interrupts, timers               │
│    • Parses device tree → creates platform devices                │
│    • Probes drivers (display, storage, serial)                    │
│    • Mounts initramfs (from boot.img ramdisk)                     │
│    • Executes /init (PID 1)                                       │
├──────────────────────────────────────────────────────────────────┤
│ 6. Android init (first-stage)                                     │
│    • Mounts /system, /vendor, /product via dm-verity+dm-linear    │
│    • Loads SELinux policy                                         │
│    • Re-execs self with SELinux context                           │
│    • Continues to second-stage init (trigger processing)          │
└──────────────────────────────────────────────────────────────────┘
```

### 4A.1.2 SoC Vendor Differences

| SoC Vendor | BootROM | 1st Stage | 2nd Stage | Notes |
|-----------|---------|-----------|-----------|-------|
| Qualcomm | PBL (Primary Boot Loader) | SBL1/XBL | ABL (Android Boot Loader) | Heavily proprietary; binary-only |
| MediaTek | BROM | Preloader | LK (Little Kernel) | Some source available |
| Samsung Exynos | iROM | BL1 | BL2/U-Boot | Proprietary chain |
| NXP i.MX | ROM | SPL | U-Boot | Mostly open source |
| NVIDIA Tegra | Boot ROM | MB1/MB2 | CBoot/UEFI | Complex multi-stage |
| Google (Cuttlefish) | crosvm firmware | — | Minimal | UEFI-like in virtual |

### 4A.1.3 Kernel Command Line — The Bridge

The bootloader passes critical information to the kernel via cmdline:

```
console=ttyMSM0,115200n8
androidboot.hardware=lahaina
androidboot.serialno=ABC123
androidboot.bootdevice=1d84000.ufshc
androidboot.slot_suffix=_a
androidboot.verifiedbootstate=green
androidboot.veritymode=enforcing
androidboot.keymaster=1
androidboot.selinux=enforcing
androidboot.dtbo_idx=0
androidboot.boot_devices=soc/1d84000.ufshc
lpm_levels.sleep_disabled=1
firmware_class.path=/vendor/firmware
```

Android's `init` parses `androidboot.*` properties and sets them as system properties (removing the prefix):
- `androidboot.hardware=lahaina` → `ro.hardware=lahaina`
- `androidboot.serialno=ABC123` → `ro.serialno=ABC123`

```bash
adb shell cat /proc/cmdline
```

---

## 4A.2 — Partition Table Architecture

### 4A.2.1 GPT Layout (Modern Android)

Modern Android devices use GPT (GUID Partition Table). A typical layout:

```
LBA 0:     Protective MBR
LBA 1:     GPT Header
LBA 2-33:  GPT Entries (128 entries × 128 bytes)
...
Partitions:
  xbl_a, xbl_b           (Qualcomm bootloader)
  abl_a, abl_b           (Android bootloader)
  tz_a, tz_b             (TrustZone firmware)
  hyp_a, hyp_b           (Hypervisor)
  modem_a, modem_b       (Modem firmware)
  bluetooth_a, bluetooth_b
  boot_a, boot_b         (kernel + ramdisk)
  init_boot_a, init_boot_b  (generic ramdisk, Android 13+)
  vendor_boot_a, vendor_boot_b  (vendor ramdisk + DTB)
  dtbo_a, dtbo_b         (device tree overlays)
  vbmeta_a, vbmeta_b     (AVB metadata)
  vbmeta_system_a/b      (system AVB chain)
  super                  (dynamic partitions container)
  userdata               (user data, encrypted)
  metadata               (encryption metadata)
  misc                   (bootloader messages, A/B metadata)
  persist                (factory calibration data, survives reset)
  frp                    (Factory Reset Protection)
```

### 4A.2.2 Dynamic Partitions (super)

Since Android 10, logical partitions live inside a single physical `super` partition, managed by `dm-linear`:

```
super (physical, e.g., 8 GB)
├── system_a (logical, ~2.5 GB)
├── system_ext_a (logical, ~500 MB)
├── vendor_a (logical, ~1.5 GB)
├── product_a (logical, ~800 MB)
├── odm_a (logical, ~100 MB)
├── vendor_dlkm_a (logical, ~50 MB, kernel modules)
├── system_b (logical, CoW snapshot for VAB)
├── ... _b variants (Virtual A/B: snapshot, not full copy)
└── free space (used for COW during OTA)
```

Benefits:
- Partition sizes are flexible (no more "vendor partition too small" bricking during OTA).
- Virtual A/B doesn't need double the physical space.
- Can add new partitions without repartitioning.

Tools:
```bash
adb shell lpdump            # dump logical partition metadata
adb shell lptools           # manage logical partitions
# On host:
lpmake --device-size=... --metadata-size=... --partition=system_a:readonly:2684354560 ...
```

### 4A.2.3 The `misc` Partition

The `misc` partition (typically 4 MB) is the **communication channel** between Android userspace and the bootloader:

```c
struct bootloader_message {
    char command[32];    // "boot-recovery", "boot-fastboot", ""
    char status[32];     // status for recovery
    char recovery[768];  // recovery command args
    char stage[32];      // recovery UI stage
    char reserved[1184]; // padding
};
```

When you run `adb reboot recovery`, Android writes `command="boot-recovery"` to `misc`. Bootloader reads it on next boot and jumps to recovery.

A/B metadata also lives here (bootctl):
```c
struct boot_ctrl {
    uint32_t slot_info[2];  // per-slot: priority, tries_remaining, successful_boot
    uint8_t recovery_tries_remaining;
};
```

---

## 4A.3 — Kernel Configuration for Android

### 4A.3.1 Required Android Config Options

The GKI kernel enforces these (and VTS tests them):

```kconfig
# Security
CONFIG_SECURITY_SELINUX=y
CONFIG_AUDIT=y
CONFIG_NET_CLS_BPF=y
CONFIG_BPF_JIT=y
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y

# Binder
CONFIG_ANDROID_BINDERFS=y
CONFIG_ANDROID_BINDER_IPC=y

# Filesystems
CONFIG_F2FS_FS=y
CONFIG_F2FS_FS_ENCRYPTION=y
CONFIG_F2FS_FS_COMPRESSION=y
CONFIG_EROFS_FS=y
CONFIG_FUSE_FS=y
CONFIG_OVERLAY_FS=y
CONFIG_FS_VERITY=y
CONFIG_DM_VERITY=y

# Memory
CONFIG_ZRAM=y
CONFIG_MEMCG=y
CONFIG_PSI=y
CONFIG_TRANSPARENT_HUGEPAGE=y

# Power
CONFIG_PM_WAKELOCKS=y
CONFIG_CPU_FREQ=y
CONFIG_CPU_IDLE=y
CONFIG_ENERGY_MODEL=y

# Input
CONFIG_INPUT_EVDEV=y
CONFIG_INPUT_TOUCHSCREEN=y

# Graphics
CONFIG_DRM=y
CONFIG_SYNC_FILE=y
CONFIG_SW_SYNC=y

# Misc Android
CONFIG_STAGING=y            # for some android drivers
CONFIG_ASHMEM=y             # legacy, being removed
CONFIG_ANDROID_LOW_MEMORY_KILLER=n  # removed; lmkd in userspace
```

### 4A.3.2 GKI Module Policy

In GKI (Android 11+), the kernel binary is **frozen by Google** for the LTS duration. All vendor-specific code must be in **loadable kernel modules** (`.ko`):

```
Allowed in GKI vmlinux:
  - Core kernel, MM, scheduler, filesystems, networking
  - Android-specific patches (binder, dm-verity, etc.)
  - KMI (Kernel Module Interface) — exported symbols vendors may use

Must be in modules (vendor_dlkm):
  - Display drivers (DRM/KMS driver for the SoC)
  - Camera ISP drivers
  - GPU kernel driver
  - Modem/cellular drivers
  - Sensor hub drivers
  - Audio codec drivers
  - Touch controller drivers
  - UFS/eMMC vendor quirks
  - Crypto accelerator drivers
```

The **KMI (Kernel Module Interface)** is a stable set of exported symbols. GKI guarantees these don't change within an LTS cycle. If a vendor module uses a non-KMI symbol, it breaks when Google updates the GKI.

```bash
# Check KMI symbols
cat android/abi_gki_aarch64  # in kernel source tree
# Lists every stable exported symbol
```

### 4A.3.3 Building Kernel Modules for Android

```makefile
# Kbuild file for an out-of-tree driver
obj-m += my_sensor.o
my_sensor-objs := core.o i2c.o calibration.o

# Build against GKI headers:
KERNEL_SRC ?= /path/to/android-kernel/common
make -C $(KERNEL_SRC) M=$(PWD) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules
```

The resulting `.ko` goes into `vendor_dlkm.img`:
```makefile
# In device/<vendor>/<board>/BoardConfig.mk:
BOARD_VENDOR_KERNEL_MODULES += $(KERNEL_SRC)/my_sensor.ko
```

---

## 4A.4 — First-Stage Init: From Ramdisk to Mounted System

### 4A.4.1 The Generic Ramdisk (`init_boot.img`, Android 13+)

Contents:
```
/
├── init                    (the init binary, statically linked)
├── system/etc/ramdisk/     (first-stage init configs)
├── first_stage_ramdisk/
│   └── fstab.<hardware>    (how to mount /system, /vendor, etc.)
└── ...
```

### 4A.4.2 First-Stage Init Actions

1. Mount essential filesystems: `/dev`, `/proc`, `/sys`.
2. Read `androidboot.*` from `/proc/cmdline` and set as properties.
3. Read `fstab.<hardware>` to learn partition layout.
4. Mount `/system` (via super/dm-linear + dm-verity).
5. Mount `/vendor`, `/product`, `/system_ext`.
6. Load SELinux policy from mounted `/system/etc/selinux/`.
7. Call `selinux_android_load_policy()`.
8. Re-exec self (now runs as `u:r:init:s0` instead of kernel context).
9. Continue to second-stage init.

### 4A.4.3 The fstab

```
# /first_stage_ramdisk/fstab.cutf_cvm  (Cuttlefish example)
#<dev>                      <mnt_point>  <type>  <mnt_flags>        <fs_mgr_flags>
system                      /system      ext4    ro,barrier=1       wait,logical,first_stage_mount,slotselect,avb=vbmeta_system,avb_keys=/avb
vendor                      /vendor      ext4    ro,barrier=1       wait,logical,first_stage_mount,slotselect,avb=vbmeta
userdata                    /data        f2fs    noatime,nosuid     wait,check,formattable,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption,quota,reservedsize=128M
/dev/block/by-name/metadata /metadata    ext4    noatime,nosuid     wait,check,formattable,first_stage_mount
```

Key `fs_mgr_flags`:
- `logical` — partition inside `super` (dm-linear)
- `first_stage_mount` — mount during first-stage init (before SELinux)
- `slotselect` — append `_a` or `_b` based on active slot
- `avb=<vbmeta_partition>` — verify via AVB using specified vbmeta
- `fileencryption=...` — FBE parameters
- `quota` — enable quota for storage management
- `formattable` — init may format this partition if corrupt

### 4A.4.4 dm-verity Setup During Mount

When init mounts a partition with `avb=vbmeta`:

1. Read `vbmeta` partition (already verified by bootloader).
2. Extract hashtree descriptor for the target partition.
3. Set up `dm-verity` device-mapper target:
   - Data device: the raw block partition
   - Hash device: the Merkle tree appended to the partition (or separate)
   - Root hash: from vbmeta descriptor
   - Algorithm: SHA-256
4. Mount the dm-verity device as the filesystem.
5. Any read of a tampered block → EIO → process killed / kernel panic (depending on mode).

---

## 4A.5 — The Vendor Boot Image

`vendor_boot.img` (Android 11+) contains:

```
vendor_boot.img
├── Vendor ramdisk
│   ├── /lib/modules/*.ko  (early kernel modules: display, storage)
│   ├── /first_stage_ramdisk/fstab.*
│   └── /vendor_ramdisk_init.rc  (additional init triggers)
├── DTB (Device Tree Blob — main board DT)
├── Vendor boot header (cmdline extension, page sizes, etc.)
└── Vendor ramdisk fragments (Android 12+: modular ramdisk table)
```

The generic ramdisk (from `init_boot` or `boot`) is overlaid with the vendor ramdisk. This allows Google to update the generic ramdisk independently of vendor-specific early-boot code.

---

## 4A.6 — Board Bring-Up Debugging Techniques

### 4A.6.1 UART Console (The Lifeline)

Before display works, UART is your only visibility:

```
# Typical connection:
minicom -D /dev/ttyUSB0 -b 115200

# First thing you'll see (if bootloader works):
[ABL] Booting slot A
[ABL] Loading boot image...
[ABL] AVB verification: PASS
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x51af8014]
[    0.000000] Linux version 6.1.43-android14-11-g1234567 (build@server) ...
```

If you see nothing: check voltage levels (1.8V vs 3.3V), TX/RX swap, baud rate.

### 4A.6.2 Early Boot Debugging Checklist

| Symptom | Where to look | Likely cause |
|---------|--------------|--------------|
| No UART output at all | BootROM/SBL1 | Wrong boot device, eFuse misconfigured |
| Bootloader logo, then hang | ABL/LK | Kernel image corrupt, wrong DTB |
| Kernel decompresses, then panic | dmesg early | Missing DT node for critical peripheral |
| "init: first stage..." then hang | init log | fstab wrong, partition not found |
| "init: Failed to mount /system" | init + dmesg | dm-verity key mismatch, partition corrupt |
| "init: SELinux denied" flood | dmesg | Policy mismatch with vendor |
| "zygote" starts but dies | logcat | Missing libraries, wrong ABI |
| SurfaceFlinger crash loop | tombstone | Display HAL / HWC configuration |

### 4A.6.3 Using `adb` Before Full Boot

If the device reaches init but can't fully boot:

```bash
# Wait for device in any state:
adb wait-for-device

# If in recovery:
adb shell  # limited shell

# If in sideload:
adb sideload image.zip

# If init is running but system_server hasn't started:
adb shell  # full root shell in userdebug
adb shell dmesg
adb shell cat /proc/mounts
adb shell ls /sys/class/
```

### 4A.6.4 Fastboot Debugging

```bash
fastboot devices                    # confirm device is in fastboot mode
fastboot getvar all 2>&1 | head -50 # dump all bootloader variables
fastboot getvar current-slot
fastboot getvar slot-retry-count:a
fastboot getvar slot-unbootable:a
fastboot oem device-info            # lock state, verified boot state
fastboot flash boot boot.img        # flash individual partition
fastboot boot boot.img              # boot without flashing (temporary)
fastboot reboot                     # reboot device
```

---

## 4A.7 — The Android Build System's Device Integration

### 4A.7.1 How `lunch` Finds Your Device

```
lunch aosp_cf_x86_64_phone-trunk_staging-userdebug
       │                    │              │
       │                    │              └── variant (user/userdebug/eng)
       │                    └── release config (Android 14+ trunk model)
       └── product name

Resolution path:
1. build/envsetup.sh scans device/*/AndroidProducts.mk
2. Each declares: COMMON_LUNCH_CHOICES += aosp_cf_x86_64_phone-trunk_staging-userdebug
3. The matching .mk file defines PRODUCT_NAME, PRODUCT_DEVICE, inherits
4. PRODUCT_DEVICE → looks for device/<vendor>/<PRODUCT_DEVICE>/BoardConfig.mk
```

### 4A.7.2 The Inheritance Chain (Real Example)

```
aosp_cf_x86_64_phone.mk
  └── inherits: device/google/cuttlefish/vsoc_x86_64/phone/device.mk
       └── inherits: device/google/cuttlefish/shared/phone/device_vendor.mk
            └── inherits: device/google/cuttlefish/shared/device.mk
                 └── inherits: build/make/target/product/core_64_bit_only.mk
                      └── inherits: build/make/target/product/generic_system.mk
                           └── inherits: build/make/target/product/mainline_system.mk
```

Each level adds packages, overlays, properties, and can override values from parents.

### 4A.7.3 Key Variables Reference

| Variable | Where set | Effect |
|----------|-----------|--------|
| `PRODUCT_NAME` | product.mk | Build output directory name |
| `PRODUCT_DEVICE` | product.mk | Maps to BoardConfig.mk path |
| `PRODUCT_PACKAGES` | product.mk | What APKs/binaries to include |
| `PRODUCT_COPY_FILES` | product.mk | Raw file copies into image |
| `PRODUCT_PROPERTY_OVERRIDES` | product.mk | Set build.prop values |
| `PRODUCT_VENDOR_PROPERTIES` | product.mk | Set vendor/build.prop values |
| `PRODUCT_SHIPPING_API_LEVEL` | product.mk | Min API level (affects CTS) |
| `TARGET_ARCH` | BoardConfig.mk | CPU architecture |
| `BOARD_KERNEL_CMDLINE` | BoardConfig.mk | Kernel cmdline additions |
| `BOARD_SUPER_PARTITION_SIZE` | BoardConfig.mk | Super partition size |
| `BOARD_AVB_ENABLE` | BoardConfig.mk | Enable/disable AVB |
| `BOARD_SEPOLICY_DIRS` | BoardConfig.mk | Additional SELinux policy dirs |

---

## 4A.8 — Verifying This Deep Dive

You should be able to:

1. Trace the boot chain from BootROM to init for a Qualcomm and a Cuttlefish device.
2. Explain the role of each partition in the GPT layout.
3. Read an fstab and explain what `logical,first_stage_mount,avb=vbmeta` means.
4. Explain why vendor kernel modules must only use KMI symbols.
5. Debug "init: Failed to mount /system" from first principles (dm-linear, dm-verity, keys).
6. Follow the `lunch` resolution chain from product name to BoardConfig.mk.
7. Explain the difference between `boot.img`, `init_boot.img`, `vendor_boot.img`, and when each was introduced.

---

⬅️ Back to **[Level 4 — BSP & Device Bring-Up](./level-04-bsp-bringup.md)** | ➡️ **[Level 5 — AAOS](./level-05-aaos.md)**

