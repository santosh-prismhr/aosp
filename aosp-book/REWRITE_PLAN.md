# AOSP Book — Rewrite Plan (Single Source of Truth, 100-Day Curriculum)
*End of plan. Subsequent commits implement sections 7.1 → 7.6 in order.*

---

4. **Connectivity scope (L3b):** v1 covers Wi-Fi + BT + IpClient + NetworkStack deeply, treats Telephony/RIL as a pointer chapter; deep Telephony lives in a future v2 supplement.
3. **Lab hosting:** keep short snippets inline + ship full runnable trees under `curriculum/labs/` referenced by chapter.
2. **Android version pinning:** target Android 15 as primary with 14/16 callouts; revisit yearly.
1. **Single big file vs many?** Keep multi-file (better navigation), but generate a concatenated `BOOK.md` artifact via `scripts/build_book.py` for offline/PDF use.

## 8. Further Considerations / Decisions

---

6. **Rewrite `README.md`** with role-based fast paths (Junior / Mid / Senior / Staff / AAOS / Release) plus the 100-day map, and add a CI lint script (`scripts/check_xrefs.py`) that validates cross-references and code-fence languages.
5. **Expand Appendices A–E** (interview bank to ~250 Qs aligned to new chapters; daily reference with per-subsystem tables; glossary to ~400 terms; add Projects 5–8 to Appx C).
4. **Refactor existing levels** in order L0 → L10 to (a) conform to the 6-block template, (b) close the gaps in the inventory table, (c) add cross-refs to the curriculum days and Appendix A questions.
3. **Add the NEW chapter files** (`level-02b`, `level-03a`, `level-03b`, `level-06a`, `level-08a`) as stubs following the 6-block template, then fill them with the labs listed in the table.
2. **Author `appendix-f-curriculum-100-day.md`** with the full Day 1–100 table (topic, anchor chapter §, lab path, deliverable, est. hours) using the phase mapping above.
1. **Create `STYLE_GUIDE.md` and `appendix-g-tooling-and-devcontainer.md`** first; they govern every later edit. Include the `Dockerfile` and `devcontainer.json` skeletons.

## 7. Implementation Steps (for the executing agent / author)

---

- **IDE:** Android Studio for AIDL/Java, VS Code + clangd for native, `aidegen` for module browsing.
- **Caches:** ccache 100 GB; document RBE setup for teams; sccache as fallback.
- **Versions:** primary Android 15 (`android-15.0.0_r*`); call out Android 14 LTS and Android 16 main differences.
- **Targets:** Cuttlefish (`aosp_cf_x86_64_phone`, `aosp_cf_x86_64_auto`), Pixel 6/8 (Tensor) for HW labs, AAOS emulator for Level 5.
- **Containers:** `Dockerfile` + `.devcontainer/devcontainer.json` pinned to Ubuntu 22.04 with `repo`, JDK, Python, ccache, adb, fastboot.
- **Host:** Ubuntu 22.04 / 24.04, 32 GB+ RAM, 16 cores, 1 TB NVMe; WSL2 documented but flagged.

## 6. Tooling & Targets (full file: `appendix-g-tooling-and-devcontainer.md`)

---

- Each chapter ends with a **"Verifying"** section.
- **Cross-refs:** `[L3.4 §3.4.2](./level-03-hal-native.md#342-...)` style; auto-checked by `scripts/check_xrefs.py`.
- **Code blocks:** language fences mandatory; every shell command prefixed `$` (host) or `cf:#` (Cuttlefish root) or `cf:$` (CF shell user).
- **Versioning:** every code lab tagged `(Android 14 | 15 | 16)`; differences in a "Version Note" block.
- **Diagrams:** prefer `mermaid` for flow/sequence; ASCII for memory/partition layouts.
- **Callouts:** 🟦 Concept · 🛠️ Code Lab · 🐞 Production Bug · ⚠️ Pitfall · 🎯 Staff Insight · 📐 Diagram · 🔗 Xref · 🎓 Interview Q · 📋 Cheat-sheet

## 5. Style Guide Summary (full file: `STYLE_GUIDE.md`)

---

| Appx C P5–P8 | kmod, OTA delta, perfetto SQL pack, RIL stub |
| L9 | Virtual A/B snapshot OTA generation + apply lab |
| L8a (NEW) | Tradefed module config + sharded run on 4 CF instances |
| L7a | KeyMint attestation chain validation script |
| L7 | Full sepolicy for a vendor daemon: `.te` + `file_contexts` + `service_contexts` |
| L6a (NEW) | Perfetto SQL queries for jank, dma-buf accounting via `dmabuf_dump` |
| L5 | Custom `VehicleProperty` (CUSTOM_OEM_PROP), Car app reads via CarPropertyManager |
| L4a | Sign a `boot.img` with custom AVB key; flash via `fastboot` |
| L4 | Custom DTS overlay adding a virtual `sensor@1` to CF kernel |
| L3b (NEW) | Wi-Fi `wificond` query, `IpClient` trace, RIL stub modem on CF |
| L3a (NEW) | Camera HAL3 stub (capture 1 frame), Sensors AIDL stub, GNSS AIDL stub |
| L2b (NEW) | Force `dex2oat speed-profile`, inspect `oat` files, profile-guided AOT |
| L1a | `Android.bp` cc_binary + java_library + apex; build with `m`, `b`, `mm` |
| L0a §0A.x | `hello.ko` kernel module with `Makefile` for goldfish/CF kernel |
| L0 §0.2 | `init.myhello.rc` + matching service binary, sepolicy stub |
|---|---|
| Chapter | New lab |

### High-value labs to add (currently missing)

6. **Cheat-sheet** (5–15 commands/paths; cross-ref Appx B)
5. **Interview Qs** (5–10 questions with model answers; cross-ref Appx A)
4. **Pitfalls** (3–6 OEM/Tier-1 mistakes)
3. **Code Lab** (copy-paste, runs on Cuttlefish; specific files listed below per chapter)
2. **Concept** (first-principles explanation + ASCII/mermaid diagram)
1. **Why it matters** (3–6 lines, business + production-bug framing)

Every chapter must be rewritten/authored using these six blocks:

## 4. Per-Chapter Brief Template (6-block form)

---

| 11. Staff mindset | 97–100 | ADRs, system design, interview sprint | L10, Appx A | Mock loop + 3 ADRs written |
| 10. Production | 93–96 | OTA, signing, Virtual A/B, telemetry | L9 | Generate signed delta OTA and apply on CF |
| 9. Testing | 88–92 | CTS/VTS/STS/MTS, tradefed | L8, L8a | Run CTS-on-GSI; write 5 GTest VTS cases for own HAL |
| 8. Security | 81–87 | AVB, SELinux, Keystore, TEE | L7, L7a | Vendor daemon with full sepolicy + key attestation |
| 7. Performance | 74–80 | Boot, memory, perfetto, simpleperf | L6, L6a | Cut Cuttlefish boot time; produce a perfetto report |
| 6. AAOS | 66–73 | VHAL, CarService, EVS, multi-zone audio | L5, L5a | Custom VHAL property + Car app reading it |
| 5. BSP & bring-up | 56–65 | DTS, kernel, AVB, partitions | L4, L4a | Add a virtual sensor to Cuttlefish DTS |
| 4. Native + HAL | 41–55 | Treble, VINTF, AIDL HAL, Camera, Sensors, Connectivity | L3, L3a, L3b | Ship an AIDL HAL + framework consumer end-to-end |
| 3. Framework internals | 26–40 | Boot, Binder, ServiceManager, AMS/PMS/WMS, ART | L2, L2a, L2b | Build a new SystemService visible via `dumpsys` |
| 2. AOSP basics | 15–25 | Tree, repo, Soong, Bazel, lunch | L1, L1a, Appx B | Add a new `system/` module + APK to image |
| 1. Linux foundations | 4–14 | Kernel, init, properties, sandbox | L0, L0a | Custom `init.rc` service running on CF |
| 0. Setup | 1–3 | Host, Docker, repo, first sync | Appx G, L1.1–1.3 | `aosp_cf_x86_64_phone-userdebug` builds |
|---|---|---|---|---|
| Phase | Days | Theme | Anchor chapters | Capstone deliverable |

Full Day 1–100 schedule lives in `appendix-f-curriculum-100-day.md`.

## 3. 100-Day Curriculum Mapping (summary)

---

```
      ota-delta/ , perfetto-sql/ , ril-stub/ , devcontainer/
      hal-light-aidl/ , kmod-hello/ , sepolicy-vendor/ , vhal-prop/ ,
    labs/                                   (NEW: copy-paste runnable code per project)
    week-01..week-15/README.md              (NEW: per-week pointers into chapters + labs)
  curriculum/
  appendix-g-tooling-and-devcontainer.md    (NEW: Ubuntu host, Docker, ccache, RBE, IDE)
  appendix-f-curriculum-100-day.md          (NEW: full Day 1–100 schedule with deliverables)
  appendix-e-aosp-development-manual.md     (keep; cross-link to curriculum)
  appendix-d-glossary.md                    (expand: ~400 terms + acronyms + source paths)
  appendix-c-project-walkthroughs.md        (add Projects 5–8: kmod, OTA delta, perfetto SQL, RIL stub)
  appendix-b-daily-reference.md             (expand: per-subsystem one-liner tables)
  appendix-a-interview-bank.md              (expand to ~250 Qs aligned to new chapters)
  level-10-staff-mindset.md                 (keep; add cross-links to curriculum)
  level-09-production-release.md            (expand: Virtual A/B, APEX release, fuzzing in CI)
  level-08a-deep-dive-tradefed.md           (NEW: harness, sharding, MTS/CTS-on-GSI)
  level-08-testing-compatibility.md         (expand: tradefed internals, GTS/xTS)
  level-07a-deep-dive-security-tee.md       (keep)
  level-07-security.md                      (keep)
  level-06a-deep-dive-perf-tracing.md       (NEW: perfetto SQL, ftrace, BPF, dma-buf accounting)
  level-06-performance-memory.md            (expand: ANR triage, simpleperf labs)
  level-05a-deep-dive-aaos-internals.md     (keep)
  level-05-aaos.md                          (keep)
  level-04a-deep-dive-bootloader-kernel.md  (expand: U-Boot vs ABL contrast)
  level-04-bsp-bringup.md                   (expand: GKI/KMI, real DTS lab)
  level-03b-deep-dive-connectivity.md       (NEW: Wi-Fi, BT, Telephony/RIL, IpClient, NetworkStack)
  level-03a-deep-dive-hal-subsystems.md     (NEW: Camera, Audio, Sensors, GNSS, NNAPI, Graphics/HWC2)
  level-03-hal-native.md                    (expand: Camera/Sensors/GNSS HAL labs)
  level-02b-deep-dive-art-runtime.md        (NEW: dex2oat, JIT/AOT, GC, profile-guided)
  level-02a-deep-dive-binder-native.md      (expand: libbinder_ndk, FMQ, RPC binder)
  level-02-framework-internals.md           (expand: PMS, AMS, WMS chapters)
  level-01a-deep-dive-build-system.md       (expand: Bazel migration, sdk snapshots)
  level-01-aosp-basics.md                   (expand: Bazel `b`, ccache, Docker)
  level-00a-deep-dive-kernel.md             (add: ashmem, dma-buf, binder driver bridge)
  level-00-foundations.md                   (expand: bionic, logd)
  REWRITE_PLAN.md                           (this file)
  STYLE_GUIDE.md                            (NEW: 6-block chapter template, callouts, mermaid)
  README.md                                 (rewritten: 100-day map + role fast-paths)
aosp-book/
```

## 2. Target File Structure (post-rewrite)

---

**Cross-cutting gaps** (no current home): Connectivity stack (Wi-Fi/BT/Telephony/RIL/IpClient), ART/Dalvik runtime, Camera HAL pipeline, APEX/Mainline modules, dma-buf/gralloc/ION, logd/logger/tombstoned, fuzzing & sanitizers (HWASan/MTE), GKI/KMI versioning, WSL2/Docker dev container.

| Appendix A–E | Solid scaffolding | Glossary thin; need a new **Appendix F (curriculum)** and **Appendix G (tooling)** |
| `level-10-staff-mindset.md` | Strong staff interview content | Keep mostly as-is; cross-link to new curriculum |
| `level-09-production-release.md` | Branching, OTA, signing | Virtual A/B internals, APEX/Mainline release flow, fuzzing in CI |
| `level-08-testing-compatibility.md` | CTS/VTS/STS/MTS | No deep-dive; tradefed harness internals, GTS, xTS thin |
| `level-07-security.md` + `07a` | AVB, SELinux, Keystore2, KeyMint | OK — strong |
| `level-06-performance-memory.md` | Boot opt, LMKD, perfetto | No deep-dive companion; simpleperf, ANR triage thin |
| `level-05-aaos.md` + `05a` | VHAL, multi-display, EVS, audio zones, SOME/IP | OK — minor: rotary HMI lab |
| `level-04a-deep-dive-bootloader-kernel.md` | Boot chain, partitions, AVB | U-Boot vs ABL contrast example |
| `level-04-bsp-bringup.md` | Device tree, kernel, blobs, CF | Real DTS examples, GKI/KMI versioning depth |
| `level-03-hal-native.md` | Treble, VINTF, AIDL HAL | Camera/Sensors/GNSS HAL examples missing; no HAL deep-dive |
| `level-02a-deep-dive-binder-native.md` | Driver walk, SF, AF | libbinder_ndk specifics, RPC binder, FMQ, fast-path |
| `level-02-framework-internals.md` | Boot, Binder, ServiceManager, core services | PMS/AMS/WMS deeper; **ART/Dalvik missing** |
| `level-01a-deep-dive-build-system.md` | Soong/Kati/Ninja pipeline, prebuilts | Bazel migration depth, `m` vs `mm` vs `mma` perf, sdk_snapshot |
| `level-01-aosp-basics.md` | Tree layout, repo, lunch, Soong intro | Bazel/`b` workflow, ccache/RBE day-1 setup |
| `level-00a-deep-dive-kernel.md` | task_struct, CFS, mm, fs | binder/ashmem/dma-buf as kernel subsystems missing here |
| `level-00-foundations.md` | init, properties, namespaces | Bionic libc internals, logd/logger thin |
| `README.md` | Career ladder, conventions | No 100-day map, no role-based fast paths |
|---|---|---|
| Existing file | Strengths | Gaps to fix in rewrite |

## 1. Inventory & Gap Analysis

---

The existing 21-file handbook is restructured into a single source of truth that simultaneously serves: (a) a **100-day basic→advanced curriculum**, (b) **staff-level interview prep**, and (c) a **daily developer cheat-sheet**. Most existing levels remain as the spine; we add new deep-dives, expand thin areas, introduce a `curriculum/` 100-day track, and standardize every chapter on a fixed **6-block style** ("Why it matters / Concept / Code Lab / Pitfalls / Interview Qs / Cheat-sheet").

> **Primary Android target:** Android 15 (`android-15.0.0_r*`), with Android 14 LTS callouts and Android 16 deltas.
> **Owner:** Principal AOSP Architect track.  
> **Status:** Approved. This document drives every subsequent edit to the book.  


