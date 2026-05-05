# Level 1A — Deep Dive: Soong, Kati, Ninja, and the AOSP Build Graph

> *Supplement to Level 1. This chapter provides the architectural depth of the AOSP build system, explaining how Blueprint, Soong, Kati, and Ninja interact, the ongoing Bazel migration, and advanced module configurations required for production BSPs.*

---

## 1A.1 — The Build System Architecture

The modern AOSP build system is a multi-stage generator that ultimately produces a massive `build.ninja` file.

### 1A.1.1 The Generation Pipeline

```
Android.bp (Declarative)        Android.mk (Legacy/Imperative)
       │                                     │
       ▼                                     ▼
    Soong                                  Kati
(Go-based parser)                 (C++ Make-to-Ninja compiler)
       │                                     │
       ▼                                     ▼
out/soong/build.ninja          out/build-<product>.ninja
       │                                     │
       └─────────────────┬───────────────────┘
                         │
                         ▼
                       Ninja (Executes the graph)
                         │
                         ▼
        out/target/product/<device>/... (Images)
```

1. **Soong** parses `Android.bp` files using the `Blueprint` framework. Blueprint handles the generic graph traversal, while Soong defines Android-specific module types (`cc_binary`, `java_library`). Soong outputs `out/soong/build.ninja`.
2. **Kati** is a custom GNU Make clone. It evaluates `Android.mk` files and translates them into Ninja rules (`out/build-<product>.ninja`). It avoids executing shell commands directly, replacing them with Ninja rules.
3. **Ninja** merges these files and executes the minimal set of commands needed to update the targets.

### 1A.1.2 The Bazel Migration (Android 13+)

Google is migrating AOSP to **Bazel**. As of Android 14, this is a hybrid process (`bzlmod`).
- **bp2build:** An automatic converter that turns `Android.bp` into `BUILD.bazel` files in `out/soong/workspace/`.
- During the transition, Ninja still drives the overall build, but it shells out to Bazel (`b run`) for modules that have been fully migrated.

---

## 1A.2 — Advanced Soong Mechanics

### 1A.2.1 Namespaces in Soong

By default, module names in `Android.bp` must be globally unique across the tree. To support multiple OEM variants of the same HAL, Soong uses **`soong_namespace`**.

```blueprint
// vendor/oem/hardware/Android.bp
soong_namespace {
    imports: ["hardware/google/interfaces"],
}

cc_binary {
    name: "android.hardware.camera.provider@2.4-service",
    // This name doesn't conflict with AOSP's default implementation
    // because it is scoped to this namespace.
}
```

To tell the build system which namespace to use for a product:
```make
# BoardConfig.mk
PRODUCT_SOONG_NAMESPACES += vendor/oem/hardware
```

### 1A.2.2 Defaults and VINTF Fragments

```blueprint
cc_defaults {
    name: "my_hal_defaults",
    cflags: ["-Wall", "-Werror"],
    shared_libs: ["libbase", "libprocessgroup"],
}

cc_binary {
    name: "my_hal",
    defaults: ["my_hal_defaults"],
    srcs: ["main.cpp"],
    vintf_fragments: ["my_hal_manifest.xml"],
    init_rc: ["my_hal.rc"],
}
```
`vintf_fragments` automatically checks the XML against the VINTF schema and assembles it into the vendor's `/vendor/etc/vintf/manifest/` directory.

---

## 1A.3 — Handling Prebuilts at Scale

BSPs contain hundreds of binary blobs. Handling them correctly ensures they are stripped, signed, and placed in the right partition.

### 1A.3.1 Prebuilt Binaries and Shared Libraries

```blueprint
cc_prebuilt_library_shared {
    name: "libgl_oem",
    vendor: true,
    target: {
        android_arm64: { srcs: ["arm64/libgl_oem.so"] },
        android_arm: { srcs: ["arm/libgl_oem.so"] },
    },
    strip: { none: true },   // Do not let Soong strip a pre-stripped blob
    check_elf_files: false,  // If OEM blob has missing dependencies
}
```

### 1A.3.2 Prebuilt APKs and Signatures

```blueprint
android_app_import {
    name: "OemCamera",
    apk: "OemCamera.apk",
    presigned: true,         // Use the signature already on the APK
    privileged: true,        // Install to priv-app
    dex_preopt: {
        enabled: false,      // Vendor app, don't preopt in system image build
    },
}
```

If it needs to be signed by the platform key:
```blueprint
    certificate: "platform", // Re-sign with AOSP platform key
```

---

## 1A.4 — Build Optimization Techniques

### 1A.4.1 Remote Build Execution (RBE)

Google uses RBE (derived from Goma/Bazel) to compile C++ across hundreds of cloud workers. You can set it up locally if your company has an RBE farm:
```bash
export USE_RBE=true
export RBE_DIR=prebuilts/remoteexecution-client
export RBE_cxx_exec_strategy=remote_local_fallback
```

### 1A.4.2 Kati Stamps and build.ninja Regeneration

If a build repeatedly says `ninja: rebuilding 'build.ninja'`, Kati is re-evaluating the Makefiles. This happens if an `Android.mk` executes a `$(shell ...)` command whose output changes every run (e.g., `date`).

**Staff Fix:** Run `m kati` with tracing to find the non-deterministic shell command.
```bash
export KATI_DUMP_SHELL=1
m kati 2>&1 | grep "shell"
```

---
⬅️ Back to **[Level 1 — AOSP Basics](./level-01-aosp-basics.md)** | ➡️ **[Level 2 — Framework Internals](./level-02-framework-internals.md)**

