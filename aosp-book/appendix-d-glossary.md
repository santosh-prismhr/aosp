# Appendix D — Glossary of AOSP Terms

A reference of every acronym and term used in this book and in daily AOSP work.

---

| Term | Meaning |
|------|---------|
| **AAOS** | Android Automotive OS — Android as the native head-unit OS |
| **ABL** | Android Boot Loader (Qualcomm) |
| **ABI** | Application Binary Interface |
| **AIDL** | Android Interface Definition Language |
| **AMS** | ActivityManagerService |
| **AOSP** | Android Open Source Project |
| **APEX** | Android Pony EXpress — updatable module container |
| **APK** | Android Package |
| **ART** | Android Runtime (AOT/JIT compiler + runtime) |
| **ASan/HWASan** | Address Sanitizer / Hardware Address Sanitizer |
| **ASIL** | Automotive Safety Integrity Level (ISO 26262) |
| **ATMS** | ActivityTaskManagerService (split from AMS, Android 10+) |
| **AVB** | Android Verified Boot |
| **A/B** | Dual-partition seamless update scheme |
| **BSP** | Board Support Package |
| **CAN** | Controller Area Network (vehicle bus) |
| **CDD** | Compatibility Definition Document |
| **CE** | Credential Encrypted (FBE key class) |
| **CFI** | Control-Flow Integrity |
| **CoW** | Copy-on-Write |
| **CTS** | Compatibility Test Suite |
| **DAC** | Discretionary Access Control (Unix permissions) |
| **DE** | Device Encrypted (FBE key class) |
| **DMA-BUF** | Direct Memory Access Buffer (kernel graphics memory) |
| **dm-verity** | Device-mapper verity — block integrity verification |
| **DVFS** | Dynamic Voltage and Frequency Scaling |
| **ECU** | Electronic Control Unit (automotive) |
| **EVS** | Exterior View System (AAOS backup camera) |
| **F2FS** | Flash-Friendly File System |
| **FBE** | File-Based Encryption |
| **FCM** | Framework Compatibility Matrix |
| **GKI** | Generic Kernel Image (Android 11+) |
| **GMS** | Google Mobile Services |
| **GSI** | Generic System Image |
| **GTS** | Google Test Suite (closed-source) |
| **HAL** | Hardware Abstraction Layer |
| **HIDL** | HAL Interface Definition Language (deprecated) |
| **HSM** | Hardware Security Module |
| **HSUM** | Headless System User Mode (AAOS Android 13+) |
| **HWC** | Hardware Composer (display HAL) |
| **ION** | Legacy kernel memory allocator (replaced by DMA-BUF heaps) |
| **IVI** | In-Vehicle Infotainment |
| **JNI** | Java Native Interface |
| **LMKD** | Low Memory Killer Daemon |
| **LL-NDK** | Low-Level NDK (stable native APIs for vendor) |
| **LTS** | Long-Term Support (kernel) |
| **MAC** | Mandatory Access Control (SELinux) |
| **Mainline** | Google's modular updatable component framework |
| **MTE** | Memory Tagging Extension (ARM) |
| **MTS** | Mainline Test Suite |
| **MUMD** | Multi-User Multi-Display (AAOS Android 14+) |
| **NDK** | Native Development Kit |
| **OEM** | Original Equipment Manufacturer |
| **OTA** | Over-The-Air update |
| **PMS** | PackageManagerService |
| **PSI** | Pressure Stall Information (kernel) |
| **PSS** | Proportional Set Size (memory metric) |
| **RBE** | Remote Build Execution |
| **RIL** | Radio Interface Layer (telephony) |
| **RRO** | Runtime Resource Overlay |
| **RSS** | Resident Set Size |
| **S2RAM** | Suspend to RAM |
| **SELinux** | Security-Enhanced Linux |
| **SoC** | System on Chip |
| **SP** | Synthetic Password (FBE) |
| **SPL** | Security Patch Level |
| **STS** | Security Test Suite |
| **TEE** | Trusted Execution Environment |
| **Tombstone** | Native crash dump (`/data/tombstones/`) |
| **Tradefed** | Trade Federation — AOSP test harness |
| **Treble** | Project Treble — system/vendor modularization (Android 8+) |
| **USAP** | Unspecialized App Process (Zygote pool) |
| **USS** | Unique Set Size (memory private to process) |
| **VAB** | Virtual A/B (storage-efficient OTA) |
| **VHAL** | Vehicle HAL (AAOS) |
| **VINTF** | Vendor Interface (manifests + matrices) |
| **VNDK** | Vendor Native Development Kit (deprecated Android 15+) |
| **VTS** | Vendor Test Suite |
| **WMS** | WindowManagerService |
| **zram** | Compressed RAM swap (kernel) |
| **Zygote** | Process from which all app processes are forked |

---

## Key Source Paths

| What | Path |
|------|------|
| Build system | `build/soong/`, `build/make/` |
| init + properties | `system/core/init/` |
| Binder (kernel) | `drivers/android/binder.c` |
| Binder (userspace) | `frameworks/native/libs/binder/` |
| ServiceManager | `frameworks/native/cmds/servicemanager/` |
| System services | `frameworks/base/services/` |
| Framework APIs | `frameworks/base/core/java/android/` |
| SELinux policy | `system/sepolicy/` |
| HAL interfaces | `hardware/interfaces/` |
| VHAL (automotive) | `hardware/interfaces/automotive/vehicle/` |
| CarService | `packages/services/Car/` |
| OTA engine | `system/update_engine/` |
| AVB tools | `external/avb/` |
| Perfetto | `external/perfetto/` |
| LMKD | `system/memory/lmkd/` |
| ART | `art/` |
| Bionic | `bionic/` |
| Cuttlefish | `device/google/cuttlefish/` |
| CTS | `cts/` |
| Tradefed | `tools/tradefederation/` |

---

⬅️ Back to **[Table of Contents](./README.md)**
