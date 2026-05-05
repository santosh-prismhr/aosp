# Level 10 — Staff / Principal Engineer Mindset

> *"At this level, the code you write matters less than the code you cause to be written — by reviewing, by mentoring, by setting the architectural rails, and by knowing which fights to pick. You are paid to be right about the things that are expensive to change."*

This final Level is **not** more APIs. It is the operating system that runs *on top of* an Android platform engineer's brain. Every Staff/Principal-level engineer has internalized these patterns; they are what separate a senior who ships features from a Staff engineer who shapes platforms.

---

## Chapter 10.1 — How Staff Engineers Think

### 10.1.1 Reasoning From Invariants, Not Examples

A senior engineer asks *"how do I do X?"* A Staff engineer asks *"what invariants must hold so that X, Y, Z, and the things we haven't named yet, all work?"*

Examples in Android:

- **Treble invariant:** vendor partition must boot any compatible system partition. Every design decision at the system/vendor boundary is checked against this.
- **Process model invariant:** any process can crash without bricking the system. Every new daemon must answer "what happens when I die?"
- **Update invariant:** the device must always be able to recover to a working state via OTA or factory reset. Every new partition or key must answer "how do we recover if this is corrupted?"
- **Security invariant:** trust flows only outward from the hardware root. Every feature is checked: does it preserve the chain, or does it allow trust to flow upward?

Once you internalize the invariants, decisions become *derivations* rather than judgments. Disagreements become traceable to which invariant each side is privileging.

### 10.1.2 The Three-Horizon Question

When facing any non-trivial decision, ask:

1. **What is right for this release** (3 months)?
2. **What is right for this product** (3 years)?
3. **What is right for the platform** (10 years)?

These often disagree. A Staff engineer makes the disagreement explicit and recommends a path. A junior chooses one horizon and is surprised when the others bite back.

### 10.1.3 Costs You Must Price (Even If No One Else Does)

| Hidden cost | Manifests as |
|-------------|--------------|
| Cognitive load of an extra abstraction | Slower onboarding, brittle bugfixes for years |
| Compatibility surface | Mainline APEX you can't update, vendor APIs you can't break |
| Test runtime | CI bill, developer feedback loop, eventually skipped tests |
| Build time | "I'll just clean and rebuild" → 1 hr × 50 engineers × 200 days |
| Symbol/debug archive size | Forensics-impossible at year 3 if cut for storage |
| Telemetry pipeline | Privacy review, regulatory exposure, GDPR DSARs |

Staff engineers say "no" to features whose hidden costs exceed visible benefits. Saying no is half the job.

---

## Chapter 10.2 — Architectural Decision Records

A Staff engineer leaves a paper trail. Every non-trivial decision is captured as an **ADR** (Architectural Decision Record). Format:

```
# ADR-0042: Use AIDL (not HIDL) for the new ThermalStats HAL

## Context
We need a HAL to expose per-zone thermal stats to system_server. We could:
  A. Add to existing IThermal HIDL HAL (Android 11 era)
  B. Define a new AIDL HAL (Android 12+)
  C. Bypass HAL with a sysfs node + framework reader

## Decision
We choose B (AIDL HAL).

## Rationale
- HIDL is deprecated (Android 12 freeze; no new HIDL in main).
- AIDL HAL gets versioning, vendor freeze enforcement, fuzz scaffolding.
- Sysfs bypass violates Treble (system reading vendor-controlled sysfs).

## Consequences
- (+) Future-proof; matches Google direction.
- (+) Free fuzzing via libfuzzer integration.
- (-) Existing IThermal clients must add a path for the new interface.
- (-) Vendor partition must ship libbinder_ndk transitively.

## Status
Accepted, 2026-04-12. Owner: santosh.

## Alternatives considered
... (A and C details) ...
```

Store ADRs in-repo, version-controlled, alongside the code. When a new engineer asks "why is it like this?", point them at the ADR. When the answer is "we don't know," that is itself a finding — propose an ADR retroactively.

🎯 **Staff‑Level Insight:** Code rots; ADRs persist. The engineer who maintains the *why* is more valuable than the engineer who wrote the *what*.

---

## Chapter 10.3 — Framework vs HAL vs Kernel — The Eternal Tradeoff

A recurring Staff-level decision: where does this new functionality live?

```
                       Easier to update
                       More portable
                       Slower
                            ▲
   ┌───────────────────────────────────────────────┐
   │ App / SDK   (updatable via Play)              │
   ├───────────────────────────────────────────────┤
   │ Mainline module / APEX                        │
   ├───────────────────────────────────────────────┤
   │ Framework / system_server                     │
   ├───────────────────────────────────────────────┤
   │ Native services / HAL (AIDL)                  │
   ├───────────────────────────────────────────────┤
   │ Kernel driver                                 │
   └───────────────────────────────────────────────┘
                            ▼
                       Faster, more privileged
                       Harder to update, more dangerous
```

Decision principles:

1. **As high as possible, as low as necessary.** Higher layers ship faster, are easier to test, and have a smaller blast radius on bugs.
2. **Hard real-time → kernel.** Anything sub-millisecond, deterministic. Nothing else.
3. **SoC-specific → HAL.** If two OEMs would do it differently, it belongs in vendor.
4. **Policy → framework.** Decisions about *when* and *how much* belong in `system_server`, not the HAL.
5. **User-visible feature → SDK/app.** If a third party could implement it, do not embed it in the platform.

Common anti-pattern: putting **policy in HAL** (e.g., a thermal HAL deciding whether to throttle the GPU). The framework should decide; the HAL reports and applies.

### 10.3.1 Worked Tradeoff — "We Need to Mask Caller IDs from Some Apps"

- Kernel? No — too privileged, no app-level identity.
- HAL? No — telephony stack already in framework.
- Framework? Yes, but tightly integrated with `TelephonyManager`. Policy here.
- App-level? An SMS spam filter app can do most of this without platform changes. **Choose this if possible.**

The Staff move is often "we don't need to do this in the platform at all." Resist the gravitational pull toward platform code.

---

## Chapter 10.4 — Reviewing Code at Scale

You will review **hundreds** of CLs per quarter. A high-leverage review process:

### 10.4.1 The Five-Pass Review

1. **Intent pass.** Read only the CL description and unit tests. Is the intent clear and correct? If not, stop and ask.
2. **Architectural pass.** Skim file structure. Are abstractions in the right place? New module? New process? New SELinux domain?
3. **Correctness pass.** Read the code. Concurrency, error paths, lifecycle, memory.
4. **Compatibility pass.** Public API change? `@SystemApi`? `@hide`? AIDL stability? SELinux neverallow? CTS impact?
5. **Operability pass.** Logging, metrics, debug surfaces, failure modes. Will I be able to debug this at 2am from a tombstone?

### 10.4.2 Comments That Move Teams

Bad: *"This is wrong."*
Better: *"This will deadlock if onTrimMemory fires during onCreate; see the lifecycle in §2.3."*
Best: *"This will deadlock under condition X. Two options: (a) move to handler thread, (b) switch to one-way binder. I'd pick (b) because — ; see ADR-0027 for similar precedent."*

### 10.4.3 What to Block, What to Suggest

| Block (–1, blocking) | Suggest (comment) |
|----------------------|--------------------|
| Public API break, undocumented | Naming improvements |
| SELinux widening | Test-only style |
| Loss of test coverage | Refactors orthogonal to CL |
| Regression in critical path | Optional perf micro-tuning |
| Security violation | Doc nits |

🎯 **Staff‑Level Insight:** Block rarely, with explanation, on things that are expensive to fix later. Comment generously. Conflate the two and you become a bottleneck or a rubber stamp — both fail the team.

---

## Chapter 10.5 — Mentoring and Multiplying

A Staff engineer's leverage is **the team**, not their fingers. Practical patterns:

- **Office hours.** 2 × 1 hr/week, open calendar, "bring me your design doc, your bug, your weird flake." Non-negotiable on your calendar.
- **Pairing on debugging.** When a senior asks for help, do **not** take the keyboard. Ask "what have you tried?", "what does the trace say?", "what would falsify your hypothesis?". Teach the method.
- **Design doc reviews.** Every new module gets a 2-page design doc *before* code. You comment, ask questions, force the author to articulate invariants.
- **Postmortems.** Run them blameless. The artifact is the timeline + corrective actions, not the apology. Track CA closures with the rigor of bugs.
- **Teach the trace.** Run a "Perfetto reading club" — one trace, one room, one hour per month. The fastest way to level up a team's perf intuition.

### 10.5.1 The Brag Document Discipline (For Yourself)

Keep a running file of: decisions you made, ADRs you wrote, incidents you led, engineers you mentored. Most companies' performance review is sampling-biased toward the last 2 months; your file fights that. This is not vanity; it is the same discipline you apply to telemetry.

---

## Chapter 10.6 — Designing Future-Proof Platforms

### 10.6.1 The Forces

```
Backward compatibility ◄──── Platform change pressure ────► Forward features
       (apps, vendors, OEMs)                              (security, perf, hw)
```

Every platform change must respect both. Patterns that survive:

- **Versioned, frozen interfaces** (AIDL `@VintfStability`, SystemApi @ApiSince).
- **Feature flags** for experimental APIs.
- **Compat framework** (`frameworks/base/core/java/android/compat/`) — opt-in, per-app, per-target-SDK behavior changes.
- **Updatable modules** — Mainline's existence allows aggressive change without bricking the world.
- **Deprecation with migration window** — minimum 2 API levels of overlap, with lint warnings → errors → removal.

### 10.6.2 The Four Questions for Any New API

1. **Who calls it?** Apps, system, vendor, all? This sets stability tier.
2. **What is the failure mode?** What does the caller do if this returns null / throws / never returns?
3. **What is its lifecycle?** Singleton, per-process, per-user, per-display?
4. **How will we deprecate it?** If you cannot answer this, the API is not done.

### 10.6.3 Staff-Level "Smells"

Things a Staff engineer notices a junior misses:

- **Implicit globals masquerading as services.** `static` state in a system service that survives across users — multi-user bug waiting to happen.
- **Synchronous binder on the main thread.** Any blocking call into another process from a UI or system handler is wrong.
- **Leaky abstractions across Treble.** A `system` module reading vendor sysfs by hardcoded path is illegal even if it works today.
- **Properties read at process start.** A property changed at runtime won't take effect for the consumer; design accordingly.
- **String-keyed contracts.** Bundle keys as plain strings (`"foo_param"`) without constants — typos become silent bugs.
- **Boolean parameters.** A method `doThing(true, false, true)` is a future maintenance crime.

---

## Chapter 10.7 — Picking Your Battles

You will see ten things you would do differently. Fight three. Which three?

A heuristic — fight on:

1. **Security and safety.** Always. No politics is worth a CVE or a recall.
2. **Decisions hard to reverse.** Public API, on-disk schema, key management, partition layout.
3. **Force multipliers.** Build infra, test infra, telemetry — fixing these makes every engineer faster.

Let go of:

1. **Style preferences** if a linter could (eventually) automate them.
2. **Naming**, *unless* it's public API.
3. **Local optimizations** in non-hot code paths.
4. **Architecture choices in domains you don't own** — coach the owner, do not seize the wheel.

🎯 **Staff‑Level Insight:** Authority comes from being right *and* being seen to be right. Burn capital on the irreversible; conserve it elsewhere.

---

## Chapter 10.8 — Communicating Up

Engineering directors and VPs don't need your stack trace. They need:

- **One-page summary.** Problem, impact (users, $$, risk), options with tradeoffs, your recommendation, ask.
- **Risk language.** "This OTA path has a 1-in-10,000 brick rate based on field data; staged rollout caps exposure to 5,000 devices per stage."
- **Time and money.** "We can ship in 6 weeks with 3 engineers, or 3 weeks with 6, with the marginal 3 having to come from team X."
- **Decisions, not status.** Every meeting should produce a decision or be canceled.

When uncertainty is high, *say so* with a confidence interval. "I'm 70% sure this is a kernel bug; 4 hours to confirm." This is more credible than false certainty and protects you when the 30% materializes.

---

## Chapter 10.9 — Reviewing AOSP Patches Upstream

Contributing back to AOSP is a Staff-level skill (and a technical-recruiting weapon). The norms:

- Read `Documentation/SubmittingPatches`-style guidance at `https://source.android.com/setup/contribute/submit-patches`.
- One logical change per CL. No "drive-by" formatting.
- Commit message: imperative mood, **why** in the body, bug link, test plan.
- Run the relevant test suite before upload.
- Respond to reviewers within 48 h or the patch decays.
- Expect 3–10 review rounds. AOSP reviewers are exacting; this is a feature.

For a vendor, **upstreaming** SoC enablement to AOSP / mainline kernel is a long-term cost reducer: every release after that point requires *less* downstream patching. Staff engineers fight for upstreaming time even when the next release deadline says no.

---

## Chapter 10.10 — A Day, A Quarter, A Year in the Life

### 10.10.1 A Day

```
07:30  Skim overnight CI dashboards. Triage red.
08:00  Office hours / pair-debug session.
09:30  CL reviews (45 min cap).
10:15  Design doc review for new feature; comments back to author.
11:00  Working block — the ADR for the thermal HAL.
12:30  Lunch + walk.
13:30  Architecture review meeting (cross-team).
14:30  Incident retro (last week's OTA hiccup).
15:30  Working block — Perfetto trace from field bug.
16:30  Mentoring 1:1 with a senior engineer.
17:00  Async writeups: weekly platform notes, ADR drafts.
```

### 10.10.2 A Quarter

- 1–2 ADRs landed for non-trivial decisions.
- 1 incident led, 1 postmortem published.
- 1 platform-level improvement (build, test, telemetry) shipped.
- 30+ CLs reviewed, ≥ 5 with substantive architectural input.
- 2–3 engineers visibly leveled up.

### 10.10.3 A Year

- Owned the architecture of a major subsystem through ship.
- Mentored a senior into Staff readiness.
- Reduced a class of bugs by an order of magnitude (boot-time, OTA flake, security regressions, your choice).
- Authored an internal "how we ship Android" document that becomes the onboarding standard.
- Spoke at a conference or published a public technical post that brings recruits.

---

## Chapter 10.11 — The Closing Tenets

Twelve principles to operate by. Print them. Argue with them. Make them yours.

1. **Cuttlefish first.** If it doesn't repro on Cuttlefish, you don't understand it yet.
2. **Source is the source of truth.** Read AOSP before you read the blog.
3. **Trace before guess.** Perfetto, simpleperf, ftrace — always.
4. **Treble is sacred.** System and vendor evolve independently; do not couple them.
5. **SELinux narrowly.** Never widen a type to silence a denial.
6. **Reproducibility or it didn't ship.** Pin manifests, archive symbols, lock toolchains.
7. **Security is end-to-end.** Bootloader to app sandbox; no link skipped.
8. **Compatibility is a contract.** CTS is the floor; field is the ceiling; CDD is the constitution.
9. **Updates are mandatory.** A platform you cannot patch in the field is a platform that fails.
10. **Telemetry is owed to users.** Privacy-respecting, consented, useful — or absent.
11. **Documentation is code.** ADR, runbook, postmortem; treat them as deliverables.
12. **Multiply, don't solo.** Your highest-impact code is in someone else's CL because of your review.

---

## Chapter 10.12 — The Staff/Principal AOSP Interview — How It Actually Works

A Staff-level AOSP/AAOS interview loop at a serious OEM, Tier-1, or platform company is **not** a LeetCode gauntlet. It is a 5–7 hour evaluation of whether you can be trusted with the *platform*. Understanding the rubric is half the preparation.

### 10.12.1 The Loop, Decoded

| Round | Length | What is *actually* being measured |
|-------|--------|-----------------------------------|
| **Phone screen** (hiring manager) | 45 min | Sanity check: have you really shipped Android? Can you talk about it in production terms? |
| **Deep technical 1 — Framework / Binder / Boot** | 60 min | Do you understand the system below the SDK line? Can you reason from first principles? |
| **Deep technical 2 — HAL / VINTF / Treble (or VHAL for AAOS)** | 60 min | Can you architect across the system/vendor boundary? Do you understand stability and versioning? |
| **System design — "Design X for Android"** | 60 min | Can you decompose an ambiguous platform problem? Where do you draw process, partition, and trust boundaries? |
| **Debugging round** | 60 min | Given a tombstone/ANR/Perfetto trace, can you find root cause and propose a fix? |
| **Performance / security deep-dive** (varies) | 60 min | Optional 6th technical; specialism check. |
| **Behavioral / leadership** | 45 min | Have you led, mentored, said no, recovered from incidents, made irreversible calls? |
| **Bar-raiser / cross-team** | 45 min | Do you raise the bar of the org, not just clear it? |

The bar at Staff: **"Would I trust this person, alone, on a Sunday at 2 a.m., with the keys to the OTA pipeline of my fleet?"** Every answer should reinforce *yes*.

### 10.12.2 The Five Frames Interviewers Score You On

After a Staff loop, debriefs converge on five axes. Train each one:

1. **Depth** — Can you go three levels below the API surface without flinching? (e.g., from `Activity.startActivity` → `ActivityTaskManagerService` → `Binder.transact` → `binder_thread_write` in the kernel driver.)
2. **Breadth** — Can you stitch boot, IPC, HAL, SELinux, OTA into one mental model?
3. **Judgment** — When given a choice, do you state tradeoffs, invariants, and a recommendation — or do you pick the first plausible answer?
4. **Ownership** — Do your stories show *you* made the call, *you* held the pager, *you* wrote the postmortem?
5. **Communication** — Can a non-Android director understand the risk you just described? Can a junior on your team learn from how you described it?

If any frame is weak, the rest cannot compensate. A Staff "no-hire" is almost always one frame missing, not all five borderline.

### 10.12.3 The Anti-Patterns That End a Loop

- **API tourism.** Quoting class names without explaining the IPC underneath ("ActivityManager handles activities" — *how*?).
- **Buzzword binding.** Saying "Treble" without being able to draw the manifest/matrix and name the failure mode when versions diverge.
- **Single-horizon thinking.** Solving for the release; ignoring 3-year and 10-year consequences (see §10.1.2).
- **Hero stories without team.** "I fixed it in 4 hours" is junior. "I wrote the runbook so anyone on-call could fix it in 20 min" is Staff.
- **Refusing to commit.** "It depends" without then *making* it not depend by stating assumptions.
- **Diagnostic by guess.** "I'd add a log and see." Staff: "I'd capture a Perfetto trace with `am`, `binder`, `sched`, `power` categories; correlate the binder transaction id with the watchdog stack; if the lock holder is in another process, follow it via the binder peer pid."

### 10.12.4 The Universal Answer Skeleton

For *any* deep technical or design question, force yourself through this skeleton out loud. It signals seniority even when you are unsure:

```
1. Restate the problem & its constraints.
   "So we want X, with constraints A, B, C. Are D, E in scope?"
2. State the invariants that must hold.
   "Treble compliance, no synchronous binder on main, OTA-recoverable, ..."
3. Enumerate ≥ 2 viable approaches.
   "Option 1: in framework. Option 2: in a HAL. Option 3: in an APEX."
4. Score each against the invariants and 3-horizon cost.
5. Recommend one, name the tradeoff you accept, name the risk.
6. State how you'd validate (test, trace, telemetry).
7. State how you'd roll back if you are wrong.
```

This is the **same** skeleton an ADR uses. You are demonstrating that your *thinking* produces ADRs, not just code.

---

## Chapter 10.13 — Staff Interview System Design — Three Worked Examples

These are full walkthroughs at the depth a Staff candidate is expected to deliver in 45–60 minutes on a whiteboard. Read them as templates, not memorization targets.

### 10.13.1 "Design a system-wide screenshot service for a foldable + automotive device"

**Restate.** A platform-provided screenshot service available to: (a) third-party apps with permission, (b) system UI shortcut, (c) automotive driver-distraction-aware caller, (d) per-display on a foldable (inner/outer) and per-display on an IVI (cluster, center, passenger). Image must land in MediaStore for phones and in a per-user vault for automotive.

**Invariants identified out loud.**

- Multi-user (esp. AAOS): screenshots belong to the *requesting user*, not user 0.
- Multi-display: must capture the **specific** display, not the default.
- Privacy: cannot capture content from another user's profile, FLAG_SECURE windows, or DRM surfaces.
- Process model: must not run synchronously on the caller's main thread.
- Treble: must not require any vendor-side code (it's a platform feature).
- AAOS safety: when vehicle is in motion, only system-initiated captures allowed (driver distraction).

**Decomposition.**

```
   App / SystemUI                 (caller, any UID)
        │ Manager API (ScreenshotManager.captureDisplay(displayId, …))
        ▼
   system_server: ScreenshotService (new)        ◄── policy + permissions
        │ AIDL one-way request
        ▼
   SurfaceFlinger: existing captureLayers / captureDisplay
        │ HW composer reads framebuffer
        ▼
   GraphicBuffer → in-process JPEG/HEIF encode (libui + libimage_io)
        │
        ├─► MediaStore insert (per-user URI)               [phone path]
        └─► CarStorageManager vault, per-user encrypted    [AAOS path]
```

**Where each piece lives — with rationale.**

- **`ScreenshotManager` SDK class** (`@SystemApi` for system UI; `@RequiresPermission` gated for 3P).
- **`ScreenshotService` in `system_server`** — owns policy: per-user, FLAG_SECURE, AAOS driving-state. Why here? Policy belongs in framework (10.3 principle 4). Why not `SystemUI`? `SystemUI` is per-user; we need a cross-user singleton.
- **No HAL.** SurfaceFlinger already abstracts the GPU/display path. Adding a HAL would re-do work and break Treble for no gain.
- **APEX?** Tempting (updatable), but the API surface is `@SystemApi` and dependencies on `SurfaceFlinger` are tight. Defer APEX-ification to a follow-up release once API stabilizes.

**Permissions and SELinux.**

- New runtime permission: `android.permission.CAPTURE_DISPLAY_CONTENT` (dangerous; user prompt).
- New signature permission: `CAPTURE_DISPLAY_CONTENT_SYSTEM` for SystemUI/Settings.
- SELinux: new domain `screenshot_service` (subprocess, *not* in `system_server`? — discuss): allow `binder_call` to `surfaceflinger`, write to `mediaprovider_data_file`, read AAOS `car_service` for driving state. **Tradeoff to state aloud:** isolating into a separate process gives blast-radius containment but costs ~6 MB and adds a binder hop; for a feature this rarely used, keep it in `system_server` initially.

**AAOS-specific.**

- Hook into `CarUxRestrictionsManager`: when `UX_RESTRICTIONS_NO_VIDEO` is active, deny 3P callers; allow only system. Decision is *policy* in `ScreenshotService`, queried from `CarService`.
- Per-display: the IVI may have driver/center/passenger displays with different occupants. Resolve target user via `CarOccupantZoneManager.getUserForOccupant(displayId)`.

**Failure modes & rollback.**

- SurfaceFlinger capture timeout (>500 ms) → return `STATUS_HW_BUSY`, telemetry atom logged.
- Encode OOM → fall back to PNG; if still OOM, fail cleanly without affecting caller.
- Feature flag: `persist.sys.screenshot_service.enabled` (gated by `DeviceConfig`); off by default in first release for AAOS.

**Test plan.**

- CTS: new module `CtsScreenshotServiceTestCases` — permission gating, FLAG_SECURE blocking, multi-user isolation.
- VTS: not applicable (no HAL).
- Per-display tests run on Cuttlefish multi-display config (`launch_cvd --num_display=3`).
- AAOS: driving-state simulation via `cmd car_service inject-vhal-event`.

**Telemetry.**

- New atoms: `SCREENSHOT_REQUESTED`, `SCREENSHOT_FAILED` (with reason enum). Privacy review: no image content, just success/failure and caller package hash.

**What I would *not* do, and why** (this is a Staff move — explicit non-decisions):

- I would not invent a new HAL. SurfaceFlinger suffices.
- I would not couple this to `MediaProjection`. That is screen *recording* with different security implications.
- I would not store images encrypted at the service layer; FBE on `/data/user/N` already gives that.

### 10.13.2 "Design a per-vehicle property logging pipeline for AAOS that survives a 12-hour drive without filling /data"

**Restate.** Vehicle generates ~200 VHAL property updates/sec across speed, RPM, HVAC, ADAS, etc. We need a logger usable by service engineers, with rotation, privacy, on-vehicle query, and OTA-uploadable summaries.

**Invariants.**

- Bounded disk usage on `/data` (e.g., 500 MB cap).
- No PII (no GPS unless explicitly opted in; no driver identity).
- Power-safe: must survive abrupt power loss (key-off/12V drop) without `/data` corruption.
- Continues working in Garage Mode (AAOS post-shutdown task window).
- Queryable on-device by an authorized diagnostic tool, by SELinux-protected interface.

**Architecture.**

```
VHAL  ──one-way binder──►  CarPropertyService (existing)
                               │
                               ├──► existing app subscribers
                               │
                               └──► CarPropertyLogger (NEW, in CarService)
                                      │
                                      ▼
                               LoggerWriter thread
                                      │ ring of mmap'd append-only files
                                      ▼
                               /data/vendor/car/log/segments/{ts}.bin
                                      │
                                      ▼
                            Garage Mode uploader → OEM telemetry
```

**Key design decisions, with reasoning.**

1. **In CarService, not VHAL.** VHAL is the *interface*, not the *policy*. Logging policy varies by region (GDPR, CCPA, China PIPL); CarService is the right layer.
2. **Append-only segmented files, fsync per segment.** Per-event fsync would destroy SSD endurance; per-segment (e.g., 4 MB or 30 s) is the standard tradeoff. State this aloud.
3. **Ring buffer at file level, not byte level.** Keep N segments, drop oldest. Avoids torn writes on power loss.
4. **Schema = stable proto with explicit field numbers.** Forward/backward compatibility across OTA. Never use Java serialization here.
5. **CarOccupantZone & user awareness.** Multi-user AAOS: a passenger's HVAC adjustment is logged under their user context, not driver's.
6. **Privacy filter at write time, not read time.** GPS, VIN, phone IDs filtered *before* hitting disk. Reading filtered data is a CVE waiting to happen.
7. **SELinux domain `carpropertylogger`** with read-only fd handed to `dumpstate` for bug reports; write access nowhere else.
8. **Garage Mode integration**, not foreground upload: `CarPowerManager.STATE_SHUTDOWN_PREPARE` triggers compress + upload. If interrupted, segment is preserved for next cycle.

**Resource budget (state numbers; Staff candidates use math).**

- 200 events/sec × 64 B/event × 12 h = ~5.5 GB/day raw.
- Delta encoding (only changed properties) → ~10× → 550 MB/day.
- Snappy compression → ~2× → 275 MB/day.
- Cap at 500 MB → ~36 h continuous. Acceptable.
- Per-segment write: 4 MB × 0.7 compression / 200 ev/s × 64 B = ~3.5 min per segment fsync. SSD-friendly.

**Failure modes.**

- `/data` full from another offender → logger pauses, never blocks VHAL thread.
- VHAL flood (misbehaving sensor) → rate-limit per-property at 100 Hz; emit `LOGGER_DROPPED` atom.
- Power loss mid-write → mmap region recovered on next boot via segment header CRC; partial tail discarded.

**What this question is *really* probing.** Multi-user AAOS, power management, privacy law, on-disk schema evolution, SSD reality, and your willingness to do back-of-envelope math. Five frames in one answer.

### 10.13.3 "Design an OTA system for a 200,000-vehicle fleet with regulatory rollback obligations"

**Restate.** Automotive OTA, regulated (EU Cyber Resilience Act, UNECE WP.29 R155/R156), 200k devices, must support rollback within 7 days for safety recalls, must be auditable by regulators.

**Invariants.**

- A/B (Virtual A/B) is mandatory; no recovery-mode path is acceptable in a vehicle.
- All updates signed with HSM-backed prod key; key rotation possible via lineage.
- Rollback cap: monotonic AVB rollback index *frozen* below the recall floor (so we *can* roll back the last release if needed).
- Per-vehicle telemetry of update success/failure, with VIN-pseudonym.
- Geo and vehicle-mode aware: no install while ignition is on or vehicle is moving; respect customer scheduled-install window.
- Audit log immutable for 10 years.

**Components.**

```
 OEM HSM  ─► Signing Service ─► OTA Build Pipeline ─► Object Store (CDN)
                                                          │
                                                          ▼
   Fleet Management Service  ──► Cohort Engine ──► Rollout Controller
                │                       │
                │ (REST + push)         │
                ▼                       ▼
          In-vehicle update_engine    Telemetry sink (Kafka → BQ)
                │
        ┌───────┴────────┐
        Slot A (active)  Slot B (target)
                │
        Garage Mode apply + reboot
                │
        post_install verification → mark successful
                │
        7-day rollback window: keep slot A intact and bootable
```

**Cohorting.**

Tiers: dev (100), QA fleet (1k), early-access opt-in (5k), region wave 1 (20k), wave 2 (60k), full (rest). Each gate is an automated rule:

- crash rate Δ < +5%
- boot success rate > 99.5%
- VHAL error rate not regressed
- no STS-class CVE detected post-install

If any gate trips, **automatic halt** + page on-call.

**Rollback design — the regulator-facing core.**

- Slot A is **not wiped** until day 7 post-success (configurable per-region).
- A signed "recall command" from FMS → vehicle → `update_engine` → `markBootSuccessful(false)` on slot B → reboot to slot A. No user interaction.
- Audit: every recall command is logged at FMS with operator identity, reason code, and approval chain (m-of-n). Immutable WORM storage for 10 years.

**Security.**

- Update package signature **and** manifest signature both verified on device (defense in depth).
- TLS pinning to FMS with rotation.
- Anti-replay: each device's update token bound to a per-device server-issued nonce.

**What this question probes.** Distributed systems instinct (cohorting, gates), regulatory awareness (WP.29, CRA), security depth (key custody, anti-replay), automotive realism (Garage Mode, ignition gating), and operational maturity (kill switches, audit). Saying *"we'd do staged rollout"* is junior; designing the gate predicates and naming the regulators is Staff.

---

## Chapter 10.14 — Deep Technical Q&A — Staff Bar

The questions below recur across OEM, Tier-1, and FAANG-platform AOSP/AAOS loops. Each has a model answer at the depth interviewers actually want. Treat the answers as scaffolding; rephrase in your own voice.

### 10.14.1 "Walk me through what happens between `startActivity()` and the first frame on screen."

A Staff answer hits: caller-side `Instrumentation.execStartActivity` → Binder transaction to `ActivityTaskManagerService` (`ATMS`) → permission/lifecycle resolution → if process exists, `IApplicationThread.scheduleTransaction`; if not, `ActivityManagerService.startProcessLocked` → Zygote fork via socket → new process attaches binder → `ActivityThread.handleBindApplication` → `LoadedApk` → `Activity.onCreate/onStart/onResume` → `ViewRootImpl.setView` → first `Choreographer` callback → `relayoutWindow` to WindowManager → SurfaceFlinger allocates BufferQueue → app draws → `queueBuffer` → SurfaceFlinger composes → display. Mention: cold-start cost dominated by Zygote fork + class loading + first-frame inflate; warm-start skips fork; hot-start skips `onCreate`. Mention Perfetto markers (`Choreographer#doFrame`, `bindApplication`, `activityStart`). Mention common bug: synchronous binder in `onCreate` extending TTI by hundreds of ms.

### 10.14.2 "What exactly is in a Binder transaction, and why is it size-limited?"

A `binder_transaction_data` carries: target handle/ptr, code, flags (one-way? accept-fds?), sender PID/UID (kernel-stamped, *not* trustable from userspace's value — driver overrides), buffer pointer, offsets pointer, size. Buffer max **1 MB minus 8 KB** per process — actually a *per-process kernel buffer* shared among **all** in-flight transactions in that process. Hence the symptom: many small concurrent transactions can exhaust the buffer (`TransactionTooLargeException` even for "small" payloads). Mention: `oneway` transactions go through a separate, smaller queue (asynchronous, ordered per-target). Mention: file descriptors are passed by *kernel translation* — sender's fd → kernel `struct file*` → receiver's new fd. This is why you can't fake `getCallingUid()`: the driver writes it from `task_tgid`.

### 10.14.3 "How does SELinux enforcement coexist with the Linux DAC and capabilities?"

Order of checks at every syscall: **DAC first** (UID/GID/mode bits) → **capabilities** (e.g., `CAP_NET_ADMIN`) → **MAC (SELinux)** last. All three must pass. Therefore SELinux can only *restrict*, never *grant*. On Android, app UIDs have nearly no Linux capabilities (zygote drops them); the platform leans on SELinux to constrain even root-equivalent vendor processes (`vendor_init`, `vold`). Mention: `neverallow` is a *compile-time* check on the policy itself, not a runtime rule — it cannot be defeated by `setenforce 0`. Mention: `permissive` domains exist per-domain (`permissive D;`), useful for new HALs in dev but rejected by VTS for `user` builds.

### 10.14.4 "Explain Project Treble's compatibility guarantee in concrete terms."

Concrete guarantee: a vendor implementation built against `PLATFORM_SEPOLICY_VERSION = N` and HAL interfaces frozen at version V must continue to boot when paired with a system image using `PLATFORM_SEPOLICY_VERSION = N+k` for k ≤ ~5 years, *provided* the vendor manifest declares only frozen interfaces. Mechanisms: (1) `compatibility_matrix.<ver>.xml` on system declares minimums; `vendor_manifest.xml` declares what vendor offers; mismatch fails boot. (2) `prebuilts/api/<ver>/` freezes prior public sepolicy so new platform changes can't break old vendor labels. (3) AIDL `@VintfStability` interfaces are frozen via `aidl_api/` snapshot files; CI fails on unfrozen change. (4) GSI testing validates the contract on every release. Failure mode if a vendor cheats: GSI boot fails or VTS fingerprint test fails — both block GMS certification.

### 10.14.5 "A device boot-loops after an OTA. Walk me through your investigation."

`adb logcat -L` (last kmsg from previous boot) and `/cache/recovery/last_log` first. Triage axes: (1) bootloader stage — does AVB report green? `adb shell avbctl get-verification` if you can stay up; `fastboot getvar` otherwise. (2) Kernel — does dmesg complete? Look for panics, init failures, dm-verity corruption. (3) `init` — `init` exits → Linux kernel panics with "Attempted to kill init"; check for missing `.rc` import or mount failure. (4) `system_server` watchdog — if userspace boots but UI never comes up, `dumpsys activity` from previous boot in `/data/anr/`. (5) A/B slot fight — `bootctl` shows slot retry counter; if both slots have failed, vehicle/device bricks. Distinguish between *cannot complete dexopt* (disk full) vs *cannot mount /data* (FBE key derivation failure post-key-rotation) vs *vendor HAL crash loop* (servicemanager logs). For an **automotive** boot loop, also check VHAL — a stuck `INITIAL_USER_INFO` callback will hang `CarService` and look like a system_server hang.

### 10.14.6 "How does FBE survive a factory reset, and how does it not survive a forgotten password?"

FBE keys are wrapped by a *synthetic password* (SP) derived from the user credential combined with a Keymint hardware-bound key. The SP, not the credential, ultimately unwraps file keys. On factory reset, `vold` triggers `wipeUser` which destroys the wrapped SP blob; the underlying ciphertext on disk is now unrecoverable even with TEE access — that's the "crypto-erase" property used to make factory reset fast (no need to overwrite the disk). On forgotten password without a backup credential (no work profile escrow, no Smart Lock), the SP cannot be reconstructed; CE storage is unrecoverable. DE storage (system stuff like alarms) survives because it's wrapped with a different, credential-independent key. Mention `vold`'s `keystore2` integration and the fact that pre-Android 11 used a different "FDE" model that's now removed.

### 10.14.7 "Why does AAOS need multi-user differently from a phone?"

A phone has *one* primary user with optional secondary/work profiles, time-multiplexed (one foreground at a time). An AAOS vehicle has *concurrent* users — driver on cluster + center display, passenger on rear display, all logged in at the same time. This forces:
- `UserManager` to support **visible background users** (a user is `current` on a display without being THE current user).
- `CarOccupantZoneManager` to map (display, audio zone, input device) → user.
- Per-zone audio routing in `CarAudioService`.
- Activity launch by display (`ActivityOptions.setLaunchDisplayId` becomes a daily API).
- `CarUserService` orchestrating with VHAL `INITIAL_USER_INFO` so the head unit knows which user to put on the driver display before HMI shows.

The phone abstraction was never designed for this; AAOS adds it without breaking phone behavior — a Treble-of-users discipline.

### 10.14.8 "A field crash report shows a tombstone in `/system/bin/audioserver` with `SIGSEGV` at a `libstagefright` symbol. You don't have a repro. What do you do?"

(1) Get the *exact* `ro.build.fingerprint` and pull symbols for it from your archive (Chapter 9.6.2). Symbolize the tombstone — confirm function, file, line. (2) Check the Mainline module versions: `adb shell pm list packages --apex-only --show-versioncode`. If `com.android.media` was updated, the bug may live in the APEX; check Google's Mainline release notes for that version. (3) Cross-reference STS / Android Security Bulletin for known CVEs at that symbol. (4) Inspect `dropbox` for related events on the same device 60 s before crash — usually a media codec error precedes it. (5) Build a hypothesis: e.g., a malformed mp4 from a specific app. (6) Confirm via stat-by-package of crash reports. (7) Mitigation path tree: hotfix in Mainline (fast) → carrier config to disable affected codec for app (faster) → OTA (slow). Pick based on severity and affected install base. (8) Add a regression test using the malformed sample under fuzzing in `external/oss-fuzz/projects/`.

### 10.14.9 "VHAL property X works on the bench but is rate-limited on a moving vehicle. Why?"

Three plausible causes, in order of likelihood at Staff level:
1. **CAN bus saturation under driving load.** The same VHAL is mapped to a CAN signal whose ECU prioritizes safety frames over diagnostic frames at speed. Solution: vendor HAL must coalesce or accept that update rate is bounded by the bus.
2. **Driver-distraction CarUxRestrictions** silently filtering at `CarPropertyService` layer. Check `dumpsys car_service` → restrictions → see if reads/writes for that property are gated.
3. **VHAL's own rate-limit.** AOSP `default` VHAL applies `maxSampleRate` from `VehiclePropConfig`. Check `dumpsys car_service hal --properties`. Vendor HAL may also set `changeMode = ON_CHANGE` which only fires on delta — if the property is dithering below a threshold, the update appears throttled.

The Staff move: state all three, then say "I'd capture a Perfetto trace with the `vehicle` data source plus a CAN logger and overlay them; whichever shows the gap explains it."

### 10.14.10 "Tell me about a time you had to say 'no' to a senior leader on a platform decision."

Behavioral, not technical, but it carries weight. Use **STAR-with-tradeoff**:

- **Situation.** A VP wanted to expose a HAL directly to a 3P app to ship a partner feature in 4 weeks.
- **Task.** I owned the platform API surface; the proposal violated Treble (3P → vendor binder).
- **Action.** Wrote a one-pager: three options (A: VP's plan, B: framework intermediary, C: defer feature). For each: schedule, security blast radius, certification risk, 3-year cost. Pre-aligned with the security lead before the meeting. In the meeting, presented option B, recommended it, owned the 2-week schedule slip.
- **Result.** B shipped on a 6-week timeline, passed certification. Six months later, a separate 3P abused a similar pattern at another OEM and triggered a CVE; we were not affected because of B.
- **Tradeoff I accept telling this story:** I lost goodwill in week 1 of the conversation. I rebuilt it by being right about the certification risk, in writing, before it materialized. Saying no without a written alternative is not a plan; it's an obstacle.

This is the answer interviewers remember. Write yours now, before the loop.

---

## Chapter 10.15 — Behavioral Stories — The Ones You Must Have Ready

Prepare **eight** stories, each ≤ 4 minutes, each pointing at a different Staff competency. Interviewers will probe the seams; rehearse so you can drop into any one of them on demand.

| # | Theme | The competency it proves |
|---|-------|--------------------------|
| 1 | A production incident you led from page to postmortem | Ownership, calm under fire, blameless culture |
| 2 | Saying no to a senior leader | Judgment, courage, written alternatives |
| 3 | A decision that was right at 3 months but wrong at 3 years (or vice versa) | Three-horizon thinking, learning |
| 4 | Mentoring a senior to Staff readiness | Multiplier effect |
| 5 | A migration / deprecation you ran across multiple teams | Influence without authority |
| 6 | A feature you killed (yours or someone else's), with rationale | Cost discipline |
| 7 | A cross-org disagreement you resolved technically (not politically) | Communication, ADR culture |
| 8 | A bug whose fix you found by reading source nobody else would touch (kernel, init, libbinder) | Depth, fearless investigation |

For each, write a **300-word page**. In the loop you'll deliver a 90-second version; the page exists so the 90-second version is dense, not vague.

### 10.15.1 The Anti-Story Filter

Drop any story where:

- You can't name the metric that improved.
- You can't name the team or person who pushed back.
- The arc is "I worked harder" rather than "I changed how the system worked."
- The lesson is "I should have done X" without you actually then doing X later.

Staff stories carry **measurable second-order effects** (the runbook you wrote that prevented the *next* incident; the mentee who later led their own incident).

---

## Chapter 10.16 — AAOS-Specific Staff Interview Topics

If you are interviewing for an AAOS Staff role at an OEM (Volvo, GM, Ford, Stellantis, Renault, Hyundai, Honda, BMW), Tier-1 (Bosch, Continental, Aptiv, Harman, LG VS), or platform vendor (Qualcomm, NVIDIA, Google Automotive Services), these are the **must-know** topics not adequately covered in phone-Android prep.

### 10.16.1 The AAOS Mental Model You Must Defend

```
   Apps (Car-aware)
      │ Car* SDK (CarPropertyManager, CarPowerManager, CarUxRestrictionsManager)
      ▼
   CarService (system_server-adjacent process)
      │ CarPropertyService, CarUserService, CarPowerManagementService,
      │ CarAudioService, CarOccupantZoneService, CarUxRestrictionsService
      ▼
   Vehicle HAL (AIDL, vendor)
      │
      ▼
   Vehicle network adapters (CAN/CAN-FD, FlexRay, LIN, Ethernet/SOME-IP)
      │
      ▼
   ECUs: Engine, BCM, BMS, ADAS, Telematics, ...
```

Staff candidates can draw this *and* answer:

- Why is `CarService` not part of `system_server`? (Crash isolation; an OEM-shipped car module shouldn't reboot the whole UI.)
- How does `CarService` survive a `CarService` crash? (`com.android.car` process; restarted by `init` with `restart_period`.)
- Where do safety-critical loops live? (**Not in Android.** They live in dedicated ECUs or a separate safety MCU; AAOS is QM/ASIL-A at best.)
- What is the role of the **Android Automotive vs Android Auto** distinction? (AA = phone projection; AAOS = the head unit *is* Android. Different APIs, different security models.)

### 10.16.2 VHAL Deep Cuts

- **Property areas.** A property is `(propId, areaId)`. Areas = bitmask of seats / wheels / doors. Window position is per-door area; HVAC fan speed is per-zone area. Subscribe to areas individually.
- **Change modes.** `STATIC` (read once, e.g., VIN), `ON_CHANGE` (delta-driven), `CONTINUOUS` (sampled at `sampleRate`). Audio cabin pressure is CONTINUOUS; gear selection is ON_CHANGE.
- **Access modes.** `READ`, `WRITE`, `READ_WRITE`. Set on every property *and* every area.
- **System vs vendor properties.** System: `0x1xxxxxxx`–`0x6xxxxxxx`, defined by Google in `VehicleProperty.aidl`. Vendor: `0x2xxxxxxx | VEHICLE_PROPERTY_GROUP_VENDOR`. Vendor properties don't appear to non-system apps unless the OEM whitelists.
- **Zoned vs global.** A global property has area `0`. Mistaking a zoned for global is the #1 vendor VHAL bug.
- **Status.** Every value carries `VehiclePropertyStatus`: `AVAILABLE`, `UNAVAILABLE` (e.g., reverse camera off), `ERROR`. Apps **must** handle non-AVAILABLE; vendors **must** emit it.

### 10.16.3 Power Management States

```
ON ─► SHUTDOWN_PREPARE ─► (Garage Mode work) ─► (a) WAIT_FOR_VHAL → SHUTDOWN
                                                (b) CANCELLED → ON
                                                (c) → SUSPEND_TO_RAM
                                                (d) → SUSPEND_TO_DISK (Android 13+, "hibernation")
```

Staff-level questions:
- What runs in Garage Mode? OTA download/apply, dexopt, log upload, telemetry batching. Bounded by VHAL granted budget (typically 5–30 min).
- Difference between S2RAM and S2D. S2RAM keeps DRAM powered; ~100 ms wake. S2D (hibernation) writes RAM image to flash; ~5–15 s wake but ~0 mW standby. Affects 12V battery drain budget.
- Why a `CarPowerPolicy`? Per-state component enable/disable map (e.g., "in S2RAM, disable Wi-Fi but keep BT"). OEM-tunable in `power_policy.xml`.

### 10.16.4 Driver Distraction

- `CarUxRestrictionsManager` exposes a bitmask: `UX_RESTRICTIONS_NO_VIDEO`, `_NO_KEYBOARD`, `_NO_TEXT_MESSAGE`, `_LIMIT_STRING_LENGTH`, etc.
- Sourced from VHAL property `DRIVING_STATUS` + region-specific config.
- Apps must implement *both* a restricted and unrestricted UI; CTS-Verifier checks it.
- **Distraction-optimized activities** declare `<meta-data android:name="distractionOptimized" android:value="true">`; the launcher hides the rest while driving.

### 10.16.5 Functional Safety Boundary

AAOS is **not** functional-safety certified for safety-critical functions. Staff candidates must know:

- ISO 26262 ASIL ratings: QM (none) → A → B → C → D (highest). AAOS is QM.
- Anything ASIL-rated lives outside Android (separate MCU running AUTOSAR or QNX).
- Hypervisor split (e.g., QNX hypervisor with Android guest + AUTOSAR cluster guest) is the common pattern. Communication via shared memory / vsock.
- "Why can Android show speedometer then?" — Either the cluster is *not* Android (separate guest), or it's a "secondary" speedometer with a fallback in the safety domain. Be precise about which one your past product did.

### 10.16.6 Common AAOS Interview Questions With Pointers

- *"Add a new vendor property end-to-end."* Define in vendor AIDL extension or via `VEHICLE_PROPERTY_GROUP_VENDOR`; implement in vendor VHAL service; expose in `CarPropertyService` allowlist; SELinux for vendor service; CTS-on-AAOS test using `MockedVehicleHal`. Walk through versioning if the property changes shape next year (you can't — define a new propId).
- *"How do you debug a 'gear stuck in P' field complaint?"* It is *never* a gear-stuck-in-park bug; that's a safety system in another ECU. The Android symptom is VHAL reporting stale `GEAR_SELECTION`. Trace VHAL → CAN; check CarPropertyService cache; check that the subscriber app (typically cluster) is actually unsubscribed-and-resubscribed across power cycles.
- *"Multi-user: how do you give the rear-seat passenger Spotify without the driver hearing it?"* `CarOccupantZoneManager` to identify display→user; launch via `ActivityOptions.setLaunchDisplayId`; route audio via `CarAudioManager.setZoneIdForUid` (or static OEM audio zone config in `car_audio_configuration.xml`). Multi-zone audio HAL must be implemented; without it, passenger audio bleeds into driver zone.
- *"How do OTAs avoid bricking a vehicle in the customer's garage at 3 a.m.?"* Garage Mode with VHAL budget; A/B with rollback index frozen below recall floor; ignition-on detection halts apply mid-flight; 12V voltage gate (>11.8 V typical) before commit; service-tool override path requiring physical OBD-II handshake.

### 10.16.7 Where Candidates Crater on AAOS Interviews

- Confusing **Android Auto** (phone projection) with **AAOS**.
- Treating VHAL as just another HAL — it has unique change-mode and area semantics.
- Not knowing **functional safety** is *outside* Android.
- Not knowing **multi-user concurrent** semantics (everyone defaults to phone single-foreground-user mental model).
- Forgetting **regulatory**: WP.29 R155/R156 for cyber/OTA, GDPR for EU, CCPA for California, China data localization (PIPL).

---

## Chapter 10.17 — Two-Week Interview Preparation Sprint

If you have 14 days to a Staff loop, follow this sprint. Each day is ~2 hours. It assumes you have already read this book.

| Day | Focus | Output |
|-----|-------|--------|
| 1 | Boot, init, Zygote — re-read Level 2.1, draw it from memory | A clean whiteboard sketch you can do in 4 min |
| 2 | Binder driver — read `drivers/android/binder.c` headers + key paths | Be able to explain transaction lifecycle including fd translation |
| 3 | SELinux + Treble — write a `.te` file and a vendor manifest from scratch | Reproducible exercise on Cuttlefish |
| 4 | HAL design — design a fake `IThermalStats` AIDL HAL end-to-end | AIDL + impl + service + sepolicy + VTS test outline |
| 5 | OTA — read `system/update_engine` README and one CL deeply | One-page explanation of A/B + Virtual A/B differences |
| 6 | Performance — capture three Perfetto traces of your own cold starts; explain TTI | Annotated trace screenshots |
| 7 | (AAOS only) VHAL + CarService — re-read Level 5; design a vendor property | End-to-end walkthrough notes |
| 8 | System design — do §10.13.1 from scratch on a whiteboard, time yourself at 50 min | Recorded explanation, self-graded against the rubric |
| 9 | System design — invent a problem ("design global captioning") and solve it | Same |
| 10 | Behavioral — write all 8 stories from §10.15 | One page each |
| 11 | Mock with a peer — one technical, one design, one behavioral | Written feedback |
| 12 | Read Android Security Bulletins of last 6 months | Be able to discuss two interesting CVEs |
| 13 | Read the diff `android-14.0.0_r5..main` for `frameworks/base/core/java/android/app/ActivityManager.java` | Spot 3 design decisions; have an opinion on each |
| 14 | Rest. Light review. Sleep ≥ 8 h before the loop. | Calm |

The single most undervalued day is **14**. Tired Staff candidates lose loops they would otherwise win.

---

## Chapter 10.18 — Where to Go From Here

You have read what is, in effect, a curriculum. To become it requires reps:

- **Build something Staff-shaped.** Pick a real subsystem (an OTA pipeline, a HAL with attestation, an AAOS feature) and own it from ADR to ship.
- **Run a postmortem in public.** Within your org, on an incident you led. Publish the ADR and CA list.
- **Upstream a patch.** A real one to AOSP. The review will sharpen you.
- **Mentor publicly.** Office hours, a brown bag, a write-up. Teaching exposes the gaps in your model.
- **Read the diff.** Each Android release diff (`android-14.0.0_r5..android-15.0.0_r1`) is a Staff-level masterclass in what Google's platform team chose to change and why. Study the commit messages.

The book ends here. The work begins again on Monday.

---

🔗 **Cross-references:**
- Performance reasoning: [Level 6](./level-06-performance-memory.md)
- Security model: [Level 7](./level-07-security.md)
- Testing discipline: [Level 8](./level-08-testing-compatibility.md)
- Release machinery: [Level 9](./level-09-production-release.md)

⬅️ Back to **[Table of Contents](./README.md)**

