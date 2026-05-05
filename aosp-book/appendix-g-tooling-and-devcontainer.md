# Appendix G — Tooling & Devcontainer

> **Goal:** A new contributor goes from a blank machine to `aosp_cf_x86_64_phone-userdebug` boot in **under 8 hours of wall time** (most of it sync + first build). This appendix is the canonical setup. Day 1 of the [100-day curriculum](./appendix-f-curriculum-100-day.md) starts here.

---

## G.1 Hardware Recommendations

| Resource | Minimum | Recommended | Notes |
|---|---|---|---|
| CPU | 8 cores | **16+ cores** (AMD 7950X / Xeon W) | `m` is embarrassingly parallel |
| RAM | 32 GB | **64–128 GB** | `dex2oat` and Soong analysis are RAM-hungry |
| Disk | 500 GB SSD | **1–2 TB NVMe** | AOSP source ≈ 300 GB; build out ≈ 200 GB; ccache 100 GB |
| Network | 100 Mbps | 1 Gbps | First sync ≈ 80–120 GB |

> 🎯 **Staff insight:** Builds are I/O bound long before CPU bound. Always put `~/aosp` and `~/.cache/ccache` on NVMe. Spinning rust will turn a 90-minute build into a 4-hour build.

## G.2 Host OS

- **Primary:** Ubuntu 22.04 LTS or Ubuntu 24.04 LTS (bare metal).
- **Acceptable:** Debian 12.
- **WSL2:** documented (§G.6) but **not recommended** for daily use — case-sensitivity quirks, ext4 performance, and `adb` over USB needs `usbipd-win`.
- **macOS:** unsupported for full builds. Use a Linux dev box / VM.

## G.3 One-shot Host Bootstrap

**`scripts/host-bootstrap.sh`**
```bash
#!/usr/bin/env bash
# Ubuntu 22.04/24.04 — run as user with sudo, NOT as root.
set -euo pipefail

sudo apt-get update
sudo apt-get install -y \
  git-core gnupg flex bison build-essential zip curl zlib1g-dev \
  libc6-dev-i386 x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev \
  libxml2-utils xsltproc unzip fontconfig ccache rsync python3 python3-pip \
  openjdk-17-jdk openjdk-21-jdk-headless \
  android-tools-adb android-tools-fastboot \
  bc bzip2 lzop imagemagick lib32ncurses5-dev libssl-dev

# repo
mkdir -p ~/.local/bin
curl -fsSL https://storage.googleapis.com/git-repo-downloads/repo > ~/.local/bin/repo
chmod +x ~/.local/bin/repo
grep -q 'export PATH="$HOME/.local/bin:$PATH"' ~/.bashrc \
  || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc

# ccache
mkdir -p ~/.cache/ccache
ccache -M 100G
ccache -o compression=true
grep -q 'USE_CCACHE=1' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export USE_CCACHE=1
export CCACHE_EXEC=$(command -v ccache)
export CCACHE_DIR=$HOME/.cache/ccache
EOF

# git identity (edit if needed)
git config --global user.name  "${GIT_NAME:-AOSP Dev}"
git config --global user.email "${GIT_EMAIL:-aosp-dev@example.com}"
git config --global color.ui   true

echo "Bootstrap complete. Re-source your shell: source ~/.bashrc"
```

Run:
```bash
$ chmod +x scripts/host-bootstrap.sh
$ scripts/host-bootstrap.sh
$ source ~/.bashrc
$ repo --version   # sanity
$ javac -version   # 17 or 21
$ ccache -s
```

## G.4 Sync the AOSP Tree (Android 15)

```bash
$ mkdir -p ~/aosp && cd ~/aosp
$ repo init -u https://android.googlesource.com/platform/manifest \
            -b android-15.0.0_r10 --partial-clone --clone-filter=blob:limit=10M
$ repo sync -c -j$(nproc) --fail-fast --force-sync --no-tags --no-clone-bundle
```

📦 **Version Note (Android 16, expected):** branch `android-16.0.0_r*` — same flow.

> ⚠️ **Pitfall:** Without `--partial-clone`, your `.git` directories alone exceed 200 GB. Always partial-clone unless you need full history.

## G.5 First Build (Cuttlefish)

```bash
$ cd ~/aosp
$ source build/envsetup.sh
$ lunch aosp_cf_x86_64_phone-userdebug
$ m -j$(nproc)                # ≈ 60–120 min cold; 5–10 min warm
$ launch_cvd                  # boots Cuttlefish
$ adb wait-for-device && adb shell getprop ro.build.fingerprint
```

🧪 **Verifying:** `adb shell` works; `dumpsys SurfaceFlinger | head` shows the CF display; UI visible at `https://localhost:8443` (WebRTC).

## G.6 Devcontainer (Docker / VS Code)

Use this when you cannot install on the host (corp laptops, shared CI builders).

**`Dockerfile`**
```dockerfile
# syntax=docker/dockerfile:1.6
FROM ubuntu:22.04
ENV DEBIAN_FRONTEND=noninteractive TZ=Etc/UTC
RUN apt-get update && apt-get install -y --no-install-recommends \
      git-core gnupg flex bison build-essential zip curl zlib1g-dev \
      libc6-dev-i386 x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev \
      libxml2-utils xsltproc unzip fontconfig ccache rsync python3 python3-pip \
      openjdk-17-jdk openjdk-21-jdk-headless \
      android-tools-adb android-tools-fastboot \
      bc bzip2 lzop imagemagick lib32ncurses5-dev libssl-dev \
      sudo locales ca-certificates vim less jq \
 && rm -rf /var/lib/apt/lists/*

RUN locale-gen en_US.UTF-8
ENV LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8

ARG USERNAME=aosp
ARG UID=1000
RUN useradd -m -u ${UID} -s /bin/bash ${USERNAME} \
 && echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME}

USER ${USERNAME}
WORKDIR /home/${USERNAME}
RUN mkdir -p ~/.local/bin && \
    curl -fsSL https://storage.googleapis.com/git-repo-downloads/repo > ~/.local/bin/repo && \
    chmod +x ~/.local/bin/repo
ENV PATH="/home/${USERNAME}/.local/bin:${PATH}" \
    USE_CCACHE=1 CCACHE_DIR=/home/${USERNAME}/.cache/ccache
RUN mkdir -p $CCACHE_DIR && ccache -M 100G

CMD ["/bin/bash"]
```

**`.devcontainer/devcontainer.json`**
```jsonc
{
  "name": "aosp-dev",
  "build": { "dockerfile": "../Dockerfile" },
  "runArgs": ["--privileged", "--device=/dev/kvm"],
  "mounts": [
    "source=aosp-src,target=/home/aosp/aosp,type=volume",
    "source=aosp-ccache,target=/home/aosp/.cache/ccache,type=volume"
  ],
  "remoteUser": "aosp",
  "customizations": {
    "vscode": {
      "extensions": [
        "llvm-vs-code-extensions.vscode-clangd",
        "redhat.java",
        "ms-python.python",
        "google.aidl",
        "fwcd.kotlin"
      ]
    }
  }
}
```

Build & run:
```bash
$ docker build -t aosp-dev:latest .
$ docker run -it --rm --privileged --device=/dev/kvm \
      -v aosp-src:/home/aosp/aosp -v aosp-ccache:/home/aosp/.cache/ccache \
      aosp-dev:latest
```

> 🐞 **Production Bug:** Cuttlefish requires `/dev/kvm`. On corporate laptops with HyperV / VBS enabled, KVM is unavailable; use a Linux build host or a cloud VM (GCP `n2-standard-32` works well).

## G.7 WSL2 (only if host Ubuntu is unavailable)

```powershell
PS> wsl --install -d Ubuntu-22.04
PS> wsl --set-version Ubuntu-22.04 2
```
- Place `~/aosp` **inside WSL** (`/home/<you>/aosp`), **never** under `/mnt/c/...`.
- For `adb`/`fastboot` over USB, install [usbipd-win](https://github.com/dorssel/usbipd-win) and `wsl usbipd attach`.
- Cuttlefish under WSL2 needs `wsl --update` for nested KVM (Windows 11 24H2+).

## G.8 IDE Setup

| IDE | Best for | Setup |
|---|---|---|
| Android Studio | Java/Kotlin frameworks, AIDL | `idegen.sh && mm -j idegen && development/tools/idegen/idegen.sh` then open `android.ipr` |
| VS Code + clangd | C++ HAL, native | `compdb` from `out/soong/.intermediates/...`; install `clangd`, `CodeLLDB` |
| Cursor / VS Code | Markdown writing | Use this book's `STYLE_GUIDE.md` linter |

Generate compile_commands.json for clangd:
```bash
$ source build/envsetup.sh && lunch aosp_cf_x86_64_phone-userdebug
$ SOONG_GEN_COMPDB=1 m nothing
$ ln -sf out/soong/development/ide/compdb/compile_commands.json compile_commands.json
```

## G.9 Remote Build (RBE / sccache)

For teams. Set in `~/.aosp_env`:
```bash
export USE_RBE=true
export RBE_DIR=$HOME/.cache/rbe
export RBE_CXX_EXEC_STRATEGY=remote_local_fallback
```
Authenticate with your team's RBE proxy (varies by org). Solo developers should stick with **ccache 100 GB**, which on a warm tree gets `m` to 5–10 minutes.

## G.10 Daily Aliases (`~/.aosp_env`)

```bash
alias ll='ls -laF'
alias src='cd ~/aosp && source build/envsetup.sh'
alias lcf='lunch aosp_cf_x86_64_phone-userdebug'
alias lauto='lunch aosp_cf_x86_64_auto-userdebug'
alias mfast='m -j$(nproc) showcommands'
alias dlogs='adb logcat -v threadtime'
alias dsf='adb shell dumpsys SurfaceFlinger'
alias dwm='adb shell dumpsys window'
alias dam='adb shell dumpsys activity'
alias hal='adb shell lshal'
alias svc='adb shell service list'
alias trace='perfetto -c - --txt -o /tmp/t.perfetto'
```
Source from `~/.bashrc`: `[ -f ~/.aosp_env ] && source ~/.aosp_env`.

## G.11 Verifying the Setup (Day 1 gate)

You're ready to start the curriculum when **all** of these succeed:

```bash
$ repo --version              # >= 2.45
$ javac -version              # 17 or 21
$ ccache -s | head            # cache_size_max >= 100G
$ ls /dev/kvm                 # exists, group writable
$ cd ~/aosp && source build/envsetup.sh && lunch aosp_cf_x86_64_phone-userdebug && m -j$(nproc) nothing
$ launch_cvd && adb wait-for-device && adb shell getprop ro.build.fingerprint
```

If every line is green, jump to [Appendix F — Day 4](./appendix-f-curriculum-100-day.md#phase-1--linux--android-foundations-days-414-).

