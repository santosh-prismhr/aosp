# Lab — `sepolicy-vendor` (Day 10 / Day 82–83 capstone)

Author a complete vendor sepolicy module: `.te` + `file_contexts` + `service_contexts` + neverallow-clean.

## Scenario
A vendor daemon `vendor.diaglog` runs as `system` user, writes rotating logs under `/data/vendor/diaglog`, exposes an AIDL service `vendor.example.diag.IDiag/default`, and reads `persist.vendor.diaglog.*` properties.

## Files
```
sepolicy-vendor/
├── README.md
├── diaglog.te
├── file_contexts
├── service_contexts
├── property_contexts
└── genfs_contexts            (if reading sysfs)
```

## `diaglog.te`
```te
# Domain & exec
type diaglog, domain;
type diaglog_exec, exec_type, vendor_file_type, file_type;
init_daemon_domain(diaglog)

# Data dir
type diaglog_data_file, file_type, data_file_type;
allow diaglog diaglog_data_file:dir create_dir_perms;
allow diaglog diaglog_data_file:file create_file_perms;
userdebug_or_eng(`
  allow diaglog self:capability { dac_override };
')

# Properties
type diaglog_prop, property_type, vendor_property_type;
set_prop(diaglog, diaglog_prop)
get_prop(diaglog, diaglog_prop)

# Binder service
binder_use(diaglog)
add_service(diaglog, diag_service)

# Logd
allow diaglog logd_socket:sock_file write;
allow diaglog logdr_socket:sock_file write;

# Neverallow clean: don't grant net access
neverallow diaglog domain:tcp_socket *;
```

## `file_contexts`
```text
/vendor/bin/hw/vendor\.example\.diag-service     u:object_r:diaglog_exec:s0
/data/vendor/diaglog(/.*)?                       u:object_r:diaglog_data_file:s0
```

## `service_contexts`
```text
vendor.example.diag.IDiag/default  u:object_r:diag_service:s0
```

## `property_contexts`
```text
persist.vendor.diaglog.   u:object_r:diaglog_prop:s0
```

## Build hooks (`vendor/example/sepolicy/Android.mk`)
```mk
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := sepolicy_vendor_diaglog
LOCAL_MODULE_CLASS := ETC
LOCAL_MODULE_RELATIVE_PATH := selinux
include $(BUILD_PHONY_PACKAGE)

BOARD_VENDOR_SEPOLICY_DIRS += $(LOCAL_PATH)
```

## Verify
```bash
$ m selinux_policy && m -j$(nproc) && launch_cvd
cf:# stop diaglog && start diaglog
cf:# dmesg | grep avc
cf:# audit2allow -i /data/misc/audit/audit.log    # should be empty
cf:# sesearch --allow -s diaglog $OUT/system/etc/selinux/plat_sepolicy.cil 2>/dev/null \
       || sesearch --allow -s diaglog $OUT/vendor/etc/selinux/vendor_sepolicy.cil
```

## Pitfalls
- Putting types in `system/sepolicy/` instead of `BOARD_VENDOR_SEPOLICY_DIRS` → fails `system_ext` partition split rules in `checkneverallows`.
- Forgetting `vendor_property_type` attribute → `set_prop` denied at boot.
- Using `dontaudit` to silence rather than fix — never ship.

