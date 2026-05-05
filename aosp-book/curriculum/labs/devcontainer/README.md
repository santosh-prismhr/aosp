# Lab — `devcontainer` (Day 1)

Containerized AOSP dev environment. Mirrors [Appendix G](../../appendix-g-tooling-and-devcontainer.md). Use this when host install is not allowed.

## Files
```
devcontainer/
├── README.md
├── Dockerfile
└── .devcontainer/devcontainer.json
```

(See Appendix G for the canonical contents.)

## Quick start

```bash
$ cd curriculum/labs/devcontainer
$ docker build -t aosp-dev:latest .
$ docker run -it --rm --privileged --device=/dev/kvm \
    -v aosp-src:/home/aosp/aosp \
    -v aosp-ccache:/home/aosp/.cache/ccache \
    aosp-dev:latest

# inside container:
$ cd ~/aosp
$ repo init -u https://android.googlesource.com/platform/manifest -b android-15.0.0_r10 \
            --partial-clone --clone-filter=blob:limit=10M
$ repo sync -c -j$(nproc)
$ source build/envsetup.sh && lunch aosp_cf_x86_64_phone-userdebug
$ m -j$(nproc)
```

## Notes
- `--device=/dev/kvm` is required for Cuttlefish.
- Bind-mount your existing tree if you have one: `-v $HOME/aosp:/home/aosp/aosp`.
- For VS Code: open the folder, "Reopen in Container."

