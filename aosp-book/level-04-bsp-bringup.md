# Level 4 — BSP & Device Bring‑Up

> *"Bring-up is where Android meets reality. The kernel is not yours; the bootloader is not yours; the silicon is not yours; and yet, you must make them all behave like Android."*

---

## Chapter 4.1 — Anatomy of a Device Tree (AOSP `device/`)

Every Android device has a directory under `device/<vendor>/<product>/` (and frequently a shared `device/<vendor>/common/` parent). This is **not** the Linux Device Tree (`.dts`). It is the **AOSP product configuration**.

### 4.1.1 Cuttlefish Device Tree

```
device/google/cuttlefish/
├── shared/                        # shared across all CF variants
│   ├── BoardConfig.mk             # ★ board-level build flags
│   ├── device.mk                  # PRODUCT_PACKAGES, init.rc copies
│   ├── sepolicy/                  # vendor SELinux
│   ├── overlay/                   # resource overlays
│   └── manifest.xml               # vendor VINTF manifest
├── vsoc_x86_64/
│   ├── AndroidProducts.mk         # declares the lunch targets
│   ├── aosp_cf.mk                 # the product
│   └── BoardConfig.mk             # board specifics for x86_64
├── vsoc_arm64/
├── shared/auto/                   # automotive variant
└── ...
```

### 4.1.2 The Three Levels of Configuration

| File | Scope | Examples |
|------|-------|----------|
| **`AndroidProducts.mk`** | Lunch enumeration | declares `aosp_cf_x86_64_phone` |
| **`<product>.mk`** | Per-product | `PRODUCT_PACKAGES`, RROs, properties |
| **`BoardConfig.mk`** | Per-board (SoC) | partition sizes, kernel cmdline, arch |

### 4.1.3 BoardConfig Essentials

```make
# BoardConfig.mk (excerpt)
TARGET_ARCH := x86_64
TARGET_ARCH_VARIANT := x86_64
TARGET_CPU_ABI := x86_64
TARGET_CPU_VARIANT := generic

BOARD_KERNEL_CMDLINE += console=ttyS0 androidboot.console=ttyS0
BOARD_KERNEL_CMDLINE += androidboot.hardware=cutf_cvm

BOARD_USES_GENERIC_AUDIO := false
BOARD_VNDK_VERSION := current

# Dynamic Partitions (Android 10+)
BOARD_USES_METADATA_PARTITION := true
BOARD_SUPER_PARTITION_SIZE := 7516192768
BOARD_SUPER_PARTITION_GROUPS := google_dynamic_partitions
BOARD_GOOGLE_DYNAMIC_PARTITIONS_PARTITION_LIST := \
    system system_ext product vendor vendor_dlkm odm_dlkm

# AVB
BOARD_AVB_ENABLE := true
BOARD_AVB_ALGORITHM := SHA256_RSA4096
BOARD_AVB_KEY_PATH := external/avb/test/data/testkey_rsa4096.pem

# A/B
AB_OTA_UPDATER := true
AB_OTA_PARTITIONS += boot system vendor product system_ext vbmeta
```

### 4.1.4 Product Makefile

```make
# aosp_cf.mk
PRODUCT_USE_DYNAMIC_PARTITIONS := true
PRODUCT_SHIPPING_API_LEVEL := 34
PRODUCT_OTA_ENFORCE_VINTF_KERNEL_REQUIREMENTS := true

$(call inherit-product, $(SRC_TARGET_DIR)/product/core_64_bit.mk)
$(call inherit-product, $(SRC_TARGET_DIR)/product/aosp_base_telephony.mk)
$(call inherit-product, device/google/cuttlefish/shared/device.mk)

PRODUCT_NAME := aosp_cf_x86_64_phone
PRODUCT_DEVICE := vsoc_x86_64
PRODUCT_BRAND := Android
PRODUCT_MODEL := Cuttlefish x86_64 phone

PRODUCT_PACKAGES += \
    android.hardware.health-service.cuttlefish \
    android.hardware.power-service.cuttlefish \
    ...

PRODUCT_COPY_FILES += \
    device/google/cuttlefish/shared/config/init.product.rc:$(TARGET_COPY_OUT_VENDOR)/etc/init/init.product.rc
```

🎯 **Staff‑Level Insight:** `inherit-product` is **order-sensitive and override-prone**. Late inherits override earlier ones for `?=` style assignments. Read the full inheritance chain (`m printvars`) before adding a property; you may be silently shadowed.

---

## Chapter 4.2 — Linux Kernel for Android (High Level)

Android does not ship a kernel from `kernel.org` directly. Instead:

### 4.2.1 GKI — Generic Kernel Image (Android 11+)

GKI splits the kernel into:

- **Generic Kernel Image (`boot.img`)** — Google-built, ABI-stable for the lifetime of the LTS branch.
- **Vendor modules (`vendor_dlkm.img`)** — SoC-specific drivers as loadable kernel modules.

This decouples kernel updates from SoC vendors. As of Android 14, all new launches require GKI (LTS 6.1 / 6.6 / etc.).

```
boot.img (GKI)
├── vmlinux
├── ramdisk (generic)
└── modules (only "in-tree-essential")

vendor_boot.img
├── vendor ramdisk
└── DTBs

vendor_dlkm.img
└── /lib/modules/<ver>/   ← .ko files for SoC peripherals
```

### 4.2.2 Building the Cuttlefish Kernel (Optional)

Cuttlefish ships with a prebuilt GKI in `kernel/prebuilts/`. You build a custom kernel only if you are debugging kernel issues or developing drivers.

```bash
mkdir ~/aosp-kernel && cd ~/aosp-kernel
repo init -u https://android.googlesource.com/kernel/manifest -b common-android14-6.1
repo sync -j$(nproc)
tools/bazel run //common:kernel_x86_64_dist
# output in out/kernel_x86_64/dist/
```

Then point Cuttlefish at it:
```bash
launch_cvd --kernel_path=out/kernel_x86_64/dist/bzImage \
           --initramfs_path=out/kernel_x86_64/dist/initramfs.img
```

### 4.2.3 What You Should Know About the Kernel as a Platform Engineer

You don’t need to write drivers, but you must read kernel logs, understand:

- **dmesg severities** (`<3>` error, `<4>` warning, `<6>` info).
- **kernel cmdline** (`androidboot.*`).
- **dm-verity** failures (`dm-verity device corrupted`).
- **panic stacktraces** (collect via `pstore` / `ramoops`).
- **kASLR** — addresses change per boot; symbolize with the `vmlinux` from your build.

```bash
adb shell dmesg -w | head -50
adb shell cat /proc/cmdline
adb shell ls /sys/fs/pstore/                  # crash dumps from previous boot
```

---

## Chapter 4.3 — Vendor Blobs

Real silicon (Qualcomm, MediaTek, Samsung Exynos, Tesla, NXP, etc.) ships proprietary binaries:

- DSP firmware (audio, modem, ISP)
- GPU userspace drivers (`libGLES_*.so`, `vulkan.<vendor>.so`)
- Modem images, RIL libraries
- Security TA (Trusted Apps for TEE)
- Camera tuning libraries

These live in a **vendor tree** (`vendor/<oem>/<device>/proprietary/`) that is **not** in AOSP. OEMs fetch them via `extract-files.sh` from a stock image or NDA-locked archives.

⚠️ **OEM Pitfall:** Vendor blob updates often break the AVB chain (different signatures), require a `vbmeta` re-sign, and may bump SELinux types. Always do a full `vts_treble_*` run after a blob bump.

For Cuttlefish: there are **no proprietary blobs**. Everything is open-source, which is one of its biggest advantages.

---

## Chapter 4.4 — Project Treble: Why Bring-Up Looks Like It Does

Pre-Treble (Android 7 and earlier):
- Vendor patches across `frameworks/`, `system/`, `kernel/` — anywhere.
- Each Android upgrade required SoC vendor recompilation of the whole stack.
- Net effect: typical OEM lag was 6–18 months on framework version.

Treble's contract:
1. **`/vendor` and `/system` are independently buildable.**
2. **VINTF** versions the boundary.
3. **GSI (Generic System Image)** built from AOSP must boot on any Treble-compliant device.

### 4.4.1 GSI Boot Test (Vendor Test Suite Requirement)

```bash
adb reboot bootloader
fastboot flash system gsi_<arch>.img
fastboot --disable-verification flash vbmeta vbmeta-disabled.img
fastboot reboot
# Device must boot to lockscreen with GSI on top of vendor
```

If your device fails this test, your vendor partition violated VINTF. CTS-on-GSI is mandatory for Android 12+ launches.

### 4.4.2 Generic Ramdisk Split (Android 13+)

`ramdisk` was historically inside `boot.img`. Android 13 split it: `init_boot.img` holds the generic ramdisk; `boot.img` holds only the kernel. This lets Google update the ramdisk independently of the SoC kernel.

---

## Chapter 4.5 — Cuttlefish BSP Walkthrough

The whole stack, end-to-end, on a real BSP. Cuttlefish's "SoC" is `crosvm` + virtio devices.

### 4.5.1 The Boot Path

```
crosvm (host VMM)
   │
   ▼
crosvm bootloader (firmware)
   │
   ▼
GKI kernel + initramfs
   │   cmdline: androidboot.hardware=cutf_cvm
   ▼
init (PID 1)
   │   parses /init.rc
   │   mounts super.img (system, vendor, product, system_ext) via dm-linear
   │   verifies via dm-verity (test keys)
   ▼
Vendor HALs (gralloc=minigbm/gbm, audio, sensors, virtual RIL)
   │
   ▼
SurfaceFlinger ─ HWC HAL (drm_hwcomposer) ─ virtio-gpu ─ host display
```

### 4.5.2 Where Each Piece Lives

| Component | Path |
|-----------|------|
| Cuttlefish host launcher | `device/google/cuttlefish/host/` |
| Guest device tree | `device/google/cuttlefish/shared/`, `vsoc_*/` |
| Virtual HALs (audio, sensors, vibrator) | `device/google/cuttlefish/guest/hals/` |
| Virtual camera HAL | `device/generic/goldfish/camera/` |
| GPU stack | `external/minigbm/`, `external/drm_hwcomposer/`, Mesa for guest |
| Kernel | `kernel/common-android14-6.1` (GKI) |

### 4.5.3 Anatomy of a Cuttlefish Boot

```bash
launch_cvd --report_anonymous_usage_stats=N
# logs in $HOME/cuttlefish_runtime/logs/
tail -f $HOME/cuttlefish_runtime/logs/launcher.log
tail -f $HOME/cuttlefish_runtime/logs/kernel.log
tail -f $HOME/cuttlefish_runtime/logs/logcat
```

Useful artifacts:
- `kernel.log` — virtual kernel’s dmesg (host-visible).
- `logcat` — Android logs streamed from guest.
- `launcher.log` — crosvm + assemble_cvd output.

### 4.5.4 Modifying the Cuttlefish BSP

🛠️ **Hands‑On — Add a Vendor Init Property:**

Edit `device/google/cuttlefish/shared/config/init.vendor.rc`:
```rc
on init
    setprop vendor.bookdemo.enabled 1
```
Rebuild `vendor.img`:
```bash
m vendorimage
launch_cvd
adb shell getprop vendor.bookdemo.enabled
```

🛠️ **Hands‑On — Add a Custom Vendor HAL** (from Level 3) and verify it boots and is in `lshal`.

---

## Chapter 4.6 — Bring-Up on Real Hardware (Conceptual)

You will not get hardware in this book, but the workflow is identical conceptually. The path:

1. **Bootloader** — flash a working bootloader (LK / U-Boot / EDK2). Often pre-flashed by SoC vendor.
2. **Kernel** — boot a serial console; verify `dmesg` reaches userspace.
3. **`init` reaches `late-init`** — printk all the way to "Starting service 'servicemanager'".
4. **Recovery first** — minimal ramdisk that lets you `adb shell` even if `/system` is broken.
5. **`/system` mounts** — usually first dm-verity disaster; disable verity (`avbtool make_vbmeta_image --flag 2`) for debug.
6. **`servicemanager` running** — SELinux policy must allow the service domains.
7. **Display HAL up** — first pixels via HWC + KMS/DRM.
8. **`zygote` forks `system_server`** — Java side starts.
9. **`PHASE_BOOT_COMPLETED`** — UI visible.
10. **Wifi, BT, Camera, Audio HALs** — light up one by one.

Each step has its own bring-up tickets. A typical flagship phone bring-up takes a 10-engineer team **3–6 months**.

### 4.6.1 The Bring-Up Checklist (Used at Tier-1s)

- [ ] Bootloader signs / unlocks correctly.
- [ ] AVB chain validates root keys.
- [ ] Kernel boots to userspace, console available.
- [ ] init reaches `boot` trigger.
- [ ] `/data` mounts and decrypts.
- [ ] All HALs declared in VINTF actually start.
- [ ] `lshal` clean (no missing services).
- [ ] `dumpsys` clean for AMS, PMS, WMS.
- [ ] First UI displayed within 30 s.
- [ ] `vts_treble_vintf_test` passes.
- [ ] `vts -m VtsHalAudioV*` passes.
- [ ] CTS on GSI passes.

### 4.6.2 Differences vs Cuttlefish

| Topic | Cuttlefish | Real device |
|-------|-----------|-------------|
| Bootloader | crosvm firmware | LK / U-Boot, vendor-specific |
| AVB keys | test keys | OEM production keys |
| Display | virtio-gpu | DSI/eDP via DRM/KMS |
| Camera | virtual sensor | Camera HAL3 with ISP+CCI |
| Modem | none | RIL talking to modem firmware over QMI/AT |
| Power | none meaningful | regulators, PMIC, thermal |
| Sensors | virtual | I2C/SPI sensors via vendor HAL |
| Storage | qcow2 | UFS / eMMC, partition table from xBL |

---

## Chapter 4.7 — Verifying Level 4

You should be able to:

1. Read a `BoardConfig.mk` and explain every line.
2. Explain GKI, vendor_dlkm, and why Treble splits the boot/init_boot images.
3. Walk through a Cuttlefish boot from `launch_cvd` to `PHASE_BOOT_COMPLETED`.
4. List the bring-up steps for a real device and the most common failure at each.
5. Modify a Cuttlefish vendor property and verify it on-device.

---

➡️ Continue to **[Level 5 — Android Automotive (AAOS)](./level-05-aaos.md)**

