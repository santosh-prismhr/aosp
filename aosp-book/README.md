# The AOSP Engineer's Handbook
### From Foundations to Staff‑Level Platform Engineering

> A complete, production‑grade reference for engineers building, debugging, and shipping Android-based systems — phones, tablets, automotive (AAOS), and embedded devices — using the **Cuttlefish** virtual device as the canonical learning target.

---

## How to Read This Book

This book is structured as a **career ladder**, not a topic dump. Each Level builds strictly on the previous one. Skipping levels will leave gaps that show up later as production bugs you cannot debug.

| Level | Title | Audience |
|------|------|---------|
| 0 | Foundations (Pre‑AOSP) | Linux engineers entering Android |
| 1 | AOSP Basics | Junior AOSP Engineer |
| 2 | Framework Internals | Mid‑Level Engineer |
| 3 | HAL & Native Layer | Senior Engineer |
| 4 | BSP & Device Bring‑Up | Senior Platform Engineer |
| 5 | Android Automotive (AAOS) | Automotive Platform Engineer |
| 6 | Performance & Memory | Staff Engineer |
| 7 | Security | Staff Engineer |
| 8 | Testing & Compatibility | Compatibility Lead |
| 9 | Production & Release Engineering | Release Engineer / TPM |
| 10 | Staff / Principal Mindset | Staff / Principal Engineer |

## Conventions Used

- 🟦 **Concept** — a fundamental idea, explained from first principles
- 🛠️ **Hands‑On** — runnable on Cuttlefish, copy‑paste ready
- 🐞 **Common Production Bug** — real bugs from shipping devices
- ⚠️ **OEM Pitfall** — mistakes Tier‑1s and OEMs repeat
- 🎯 **Staff‑Level Insight** — architectural reasoning, tradeoffs
- 📐 **ASCII Diagram** — visualization
- 🔗 **Cross‑Reference** — pointer to another chapter

## Target Versions

Primary: **Android 13, 14, 15, 16** (latest stable). Where behavior differs across versions, it is called out explicitly with a **(Version Note)** block.

## Host Environment

Primary: **Ubuntu 22.04 LTS / 24.04 LTS** with 32+ GB RAM, 16+ cores, 500+ GB SSD.
macOS notes are inline where relevant. Windows is **not** supported as a build host (use WSL2 with caveats; not recommended for production work).

## Table of Contents

- [Level 0 — Foundations](./level-00-foundations.md)
- [Level 1 — AOSP Basics](./level-01-aosp-basics.md)
- [Level 2 — Framework Internals](./level-02-framework-internals.md)
- [Level 3 — HAL & Native Layer](./level-03-hal-native.md)
- [Level 4 — BSP & Device Bring‑Up](./level-04-bsp-bringup.md)
- [Level 5 — Android Automotive (AAOS)](./level-05-aaos.md)
- [Level 6 — Performance & Memory](./level-06-performance-memory.md)
- [Level 7 — Security](./level-07-security.md)
- [Level 8 — Testing & Compatibility](./level-08-testing-compatibility.md)
- [Level 9 — Production & Release Engineering](./level-09-production-release.md)
- [Level 10 — Staff / Principal Mindset](./level-10-staff-mindset.md)
- [Appendix A — Staff‑Level AOSP/AAOS Interview Question Bank](./appendix-a-interview-bank.md)

## Preparing for a Staff‑Level Interview

If you are reading this book to prepare for a Staff/Principal AOSP or AAOS interview loop, the recommended path is:

1. Read **Level 10** end-to-end — especially §10.12 (loop decoded), §10.13 (system design walkthroughs), §10.14 (deep technical Q&A with model answers), §10.15 (behavioral story matrix), §10.16 (AAOS-specific topics), and §10.17 (the 14-day prep sprint).
2. Drill **Appendix A** — a question bank with model answers spanning every level. Use it for flashcards, mock loops, and gap-finding.
3. Cross-reference back into Levels 0–9 wherever Appendix A or §10.14 exposes a weak area.

The combination of Level 10 + Appendix A is **the** Staff-interview core of this book. Everything in Levels 0–9 supports it, but those two are where you train the reflexes.

---

*Author's note: Every command in this book is verified against AOSP `main` and the latest `android-1X.0.0_rXX` release branches. When in doubt, the source tree is the source of truth — this book teaches you how to **read** that source tree fluently.*

