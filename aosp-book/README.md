# The AOSP Engineer's Handbook — Single Source of Truth

> **From Foundations to Staff-Level Platform Engineering, in 100 days.**
> A complete, production-grade reference for engineers building, debugging, and shipping Android-based systems — phones, tablets, automotive (AAOS), and embedded devices — using **Cuttlefish** as the canonical learning target.
>
> **Primary target:** Android 15 (`android-15.0.0_r*`) · **Audience:** Junior → Staff · **License:** Author's reuse OK with attribution.

---

## What this book is

1. **A 100-day curriculum** ([Appendix F](./appendix-f-curriculum-100-day.md)) — Day 1 = blank Ubuntu box; Day 100 = mock Staff loop with 10 capstone deliverables.
2. **A reference manual** — every chapter follows the same 6-block layout (Why it matters / Concept / Code Lab / Pitfalls / Interview Qs / Cheat-sheet) so you can dip in mid-investigation.
3. **An interview prep kit** ([Appendix A](./appendix-a-interview-bank.md)) — every chapter ends with model-answered questions that aggregate into a 250-question bank.
4. **A daily cheat-sheet** ([Appendix B](./appendix-b-daily-reference.md)) — when you need the right `dumpsys`, `cmd`, or `lshal` invocation right now.

📖 Read the [Style Guide](./STYLE_GUIDE.md) before contributing. The architecture rationale is captured in [REWRITE_PLAN.md](./REWRITE_PLAN.md).

---

## Start here

| If you are... | Start with | Then |
|---|---|---|
| **Brand new to AOSP** | [Appx G — Tooling](./appendix-g-tooling-and-devcontainer.md) → [L0 Foundations](./level-00-foundations.md) | Follow [Appx F Days 1–25](./appendix-f-curriculum-100-day.md) in order |
| **Junior — already build AOSP** | [L1 AOSP Basics](./level-01-aosp-basics.md) → [L2 Framework Internals](./level-02-framework-internals.md) | Days 15–40 |
| **Mid — writing apps + framework** | [L2 Framework](./level-02-framework-internals.md) → [L2A Binder](./level-02a-deep-dive-binder-native.md) → [L2B ART](./level-02b-deep-dive-art-runtime.md) | Days 26–55 |
| **Senior HAL / device** | [L3](./level-03-hal-native.md) + [L3A](./level-03a-deep-dive-hal-subsystems.md) + [L3B](./level-03b-deep-dive-connectivity.md) → [L4](./level-04-bsp-bringup.md) | Days 41–65 |
| **AAOS engineer** | [L5](./level-05-aaos.md) + [L5A](./level-05a-deep-dive-aaos-internals.md) | Days 66–73 |
| **Performance / Power** | [L6](./level-06-performance-memory.md) + [L6A](./level-06a-deep-dive-perf-tracing.md) | Days 74–80 |
| **Security / TEE** | [L7](./level-07-security.md) + [L7A](./level-07a-deep-dive-security-tee.md) | Days 81–87 |
| **Test / Release** | [L8](./level-08-testing-compatibility.md) + [L8A](./level-08a-deep-dive-tradefed.md) + [L9](./level-09-production-release.md) | Days 88–96 |
| **Staff / Principal** | [L10](./level-10-staff-mindset.md) + [Appx A](./appendix-a-interview-bank.md) | Days 97–100 |

---

## Career Ladder Map

| Level | Title | Audience | Has deep-dive companion |
|---|---|---|---|
| [0](./level-00-foundations.md) | Foundations (Pre-AOSP) | Linux engineers entering Android | [0A — Kernel internals](./level-00a-deep-dive-kernel.md) |
| [1](./level-01-aosp-basics.md) | AOSP Basics & Build | Junior AOSP Engineer | [1A — Soong/Bazel build system](./level-01a-deep-dive-build-system.md) |
| [2](./level-02-framework-internals.md) | Framework Internals | Mid-Level Engineer | [2A — Binder native](./level-02a-deep-dive-binder-native.md), [2B — ART runtime](./level-02b-deep-dive-art-runtime.md) |
| [3](./level-03-hal-native.md) | HAL & Native | Senior Engineer | [3A — HAL subsystems](./level-03a-deep-dive-hal-subsystems.md), [3B — Connectivity](./level-03b-deep-dive-connectivity.md) |
| [4](./level-04-bsp-bringup.md) | BSP & Device Bring-up | Senior Platform Engineer | [4A — Bootloader & kernel](./level-04a-deep-dive-bootloader-kernel.md) |
| [5](./level-05-aaos.md) | Android Automotive | Automotive Platform Engineer | [5A — AAOS internals](./level-05a-deep-dive-aaos-internals.md) |
| [6](./level-06-performance-memory.md) | Performance & Memory | Staff Engineer | [6A — Perfetto / ftrace / dma-buf](./level-06a-deep-dive-perf-tracing.md) |
| [7](./level-07-security.md) | Security | Staff Engineer | [7A — TEE / Keystore / KeyMint](./level-07a-deep-dive-security-tee.md) |
| [8](./level-08-testing-compatibility.md) | Testing & Compatibility | Compatibility Lead | [8A — Tradefed](./level-08a-deep-dive-tradefed.md) |
| [9](./level-09-production-release.md) | Production & Release | Release Engineer / TPM | — |
| [10](./level-10-staff-mindset.md) | Staff / Principal Mindset | Staff / Principal Engineer | — |

## Appendices

- [A — Interview Bank (≈250 Qs)](./appendix-a-interview-bank.md)
- [B — Daily Reference](./appendix-b-daily-reference.md)
- [C — Project Walkthroughs](./appendix-c-project-walkthroughs.md)
- [D — Glossary](./appendix-d-glossary.md)
- [E — AOSP Development Manual](./appendix-e-aosp-development-manual.md)
- [F — 100-Day Curriculum](./appendix-f-curriculum-100-day.md) ⭐ **Start here for learners**
- [G — Tooling & Devcontainer](./appendix-g-tooling-and-devcontainer.md) ⭐ **Day 1 setup**

## Hands-on Labs (runnable trees)

Under [`curriculum/labs/`](./curriculum/labs/):

| Lab | What you'll build | Days |
|---|---|---|
| [`devcontainer/`](./curriculum/labs/devcontainer/) | Docker-based AOSP build env | 1 |
| [`init-myhello/`](./curriculum/labs/init-myhello/) | init service + property + sepolicy | 6, 14 |
| [`sepolicy-vendor/`](./curriculum/labs/sepolicy-vendor/) | Full vendor sepolicy module | 10, 82–83 |
| [`hal-light-aidl/`](./curriculum/labs/hal-light-aidl/) | AIDL HAL → SystemService → app | 43–55 |
| [`kmod-hello/`](./curriculum/labs/kmod-hello/) | Out-of-tree kernel module | 61, 76 |
| [`vhal-prop/`](./curriculum/labs/vhal-prop/) | Custom OEM VHAL property | 67, 73 |
| [`perfetto-sql/`](./curriculum/labs/perfetto-sql/) | Jank/binder/dma-buf SQL pack | 75–80 |
| [`ota-delta/`](./curriculum/labs/ota-delta/) | Signed delta OTA on Cuttlefish | 94–95 |

See [`curriculum/README.md`](./curriculum/README.md) for the full lab manifest.

---

## How chapters are structured (the 6-block template)

Every `##` topic in every chapter follows this fixed shape:

1. 🟦 **Why it matters** — production-bug or business framing (3–6 lines)
2. 📐 **Concept** — first-principles explanation + diagram (mermaid or ASCII)
3. 🛠️ **Code Lab** — runnable on Cuttlefish, links to `curriculum/labs/<name>/`
4. ⚠️ **Pitfalls** — 3–6 OEM/Tier-1 mistakes
5. 🎓 **Interview Questions** — 5–10 with model answers, cross-linked to Appx A
6. 📋 **Cheat-sheet** — 5–15 one-liner commands/paths, cross-linked to Appx B

Each chapter ends with `## ✅ Verifying this chapter`. See [STYLE_GUIDE.md](./STYLE_GUIDE.md) for the canonical authoring guide.

## Conventions

- 🟦 Concept · 🛠️ Code Lab · 🐞 Production Bug · ⚠️ Pitfall · 🎯 Staff Insight · 📐 Diagram · 🔗 Xref · 🎓 Interview Q · 📋 Cheat-sheet · 🧪 Verifying · 📦 Version Note

## Target Versions

Primary: **Android 15** (`android-15.0.0_r10` and later). Calls out **Android 14 LTS** and **Android 16** deltas explicitly with 📦 Version Note blocks.

## Host Environment

Ubuntu 22.04 / 24.04, ≥ 16 cores, ≥ 32 GB RAM, ≥ 1 TB NVMe, KVM. Full bootstrap in [Appendix G](./appendix-g-tooling-and-devcontainer.md). WSL2 documented; macOS unsupported for full builds.

## Capstone portfolio (what you can demo at Day 100)

1. Custom `init` service + sepolicy + property *(Day 14)*
2. `system/` daemon + APK + APEX bundle *(Day 25)*
3. `IMyService` exposed via `dumpsys` *(Day 40)*
4. AIDL HAL + framework consumer + UI slider end-to-end *(Day 55)*
5. Virtual sensor through DT → driver → HAL → app *(Day 65)*
6. Custom VHAL property + Car app *(Day 73)*
7. Boot-time reduction with perfetto report *(Day 80)*
8. Vendor daemon with full sepolicy + KeyMint attestation *(Day 87)*
9. CTS-on-GSI + VTS GTests for own HAL *(Day 92)*
10. Signed delta OTA + apply/rollback *(Day 95)*

This portfolio maps directly to the rubric in [L10 Staff Mindset](./level-10-staff-mindset.md) and is sufficient to walk into Senior/Staff AOSP loops at any Tier-1 OEM.

---

## Contributing / extending

- Use the 6-block template from [STYLE_GUIDE.md](./STYLE_GUIDE.md).
- New chapters slot in as `level-NNx-deep-dive-<slug>.md` next to their parent level.
- New labs live under `curriculum/labs/<slug>/` with a `README.md` referenced from the chapter.
- File issues / TODOs in chapter prose only with an explicit owner & date.

> 🎯 **Staff insight:** This handbook treats AOSP as a *system*, not a list of APIs. If you finish Day 100 you will not just know "how to build" — you will know how the build serves the framework, how the framework serves the HAL, how the HAL serves the kernel, and how all of them together serve a feature on a user's device. That is the Staff bar.

