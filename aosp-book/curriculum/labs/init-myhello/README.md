# Lab — `init-myhello` (Day 6 / Day 14 capstone)

A custom `init` service + system property + sepolicy that boots cleanly on `aosp_cf_x86_64_phone-userdebug`.

## Layout
```
init-myhello/
├── README.md
├── Android.bp
├── myhello.cpp                     ← the daemon
├── init.myhello.rc                 ← init language
├── myhello.te                      ← sepolicy
├── file_contexts                   ← path label
└── property_contexts               ← prop label
```

## `myhello.cpp`
```cpp
#include <android-base/logging.h>
#include <android-base/properties.h>
#include <chrono>
#include <thread>

int main(int, char**) {
    LOG(INFO) << "myhello started, pid=" << getpid();
    int n = 0;
    while (true) {
        android::base::SetProperty("persist.myhello.tick", std::to_string(++n));
        std::this_thread::sleep_for(std::chrono::seconds(5));
    }
}
```

## `Android.bp`
```bp
cc_binary {
    name: "myhello",
    srcs: ["myhello.cpp"],
    shared_libs: ["libbase", "liblog"],
    init_rc: ["init.myhello.rc"],
    required: ["myhello.te"],
    system_ext_specific: true,
}
```

## `init.myhello.rc`
```rc
service myhello /system_ext/bin/myhello
    class late_start
    user system
    group system
    oneshot false
    seclabel u:r:myhello:s0

on property:sys.boot_completed=1
    setprop persist.myhello.boot 1
```

## `myhello.te`
```te
type myhello, domain;
type myhello_exec, exec_type, system_file_type, file_type;
init_daemon_domain(myhello)

# Allowed to set persist.myhello.* properties
set_prop(myhello, myhello_prop)
```

## `property_contexts`
```text
persist.myhello.   u:object_r:myhello_prop:s0
```

## `file_contexts`
```text
/system_ext/bin/myhello   u:object_r:myhello_exec:s0
```

## Build & verify
```bash
$ m myhello && m -j$(nproc) && launch_cvd
cf:# getprop | grep myhello
cf:# logcat -s myhello:V init:V | head
cf:# dmesg | grep avc            # should be empty for myhello
```

