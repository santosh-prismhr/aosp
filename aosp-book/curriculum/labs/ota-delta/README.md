# Lab — `ota-delta` (Day 94–95)

Generate a signed delta OTA between two builds of `aosp_cf_x86_64_phone-userdebug` and apply it on Cuttlefish.

## Steps

### 1. Build target_files for two snapshots

```bash
$ source build/envsetup.sh && lunch aosp_cf_x86_64_phone-userdebug
$ m -j$(nproc) target-files-package
$ cp $OUT/obj/PACKAGING/target_files_intermediates/*.zip /tmp/source.zip

# make a small change (e.g., bump a string), rebuild
$ m -j$(nproc) target-files-package
$ cp $OUT/obj/PACKAGING/target_files_intermediates/*.zip /tmp/target.zip
```

### 2. Sign target_files with test keys

```bash
$ ./build/make/tools/releasetools/sign_target_files_apks \
    -o -d build/make/target/product/security \
    /tmp/source.zip /tmp/source-signed.zip
$ ./build/make/tools/releasetools/sign_target_files_apks \
    -o -d build/make/target/product/security \
    /tmp/target.zip /tmp/target-signed.zip
```

### 3. Generate delta OTA

```bash
$ ./build/make/tools/releasetools/ota_from_target_files \
    --incremental_from /tmp/source-signed.zip \
    /tmp/target-signed.zip /tmp/delta-ota.zip
```

### 4. Side-load on Cuttlefish

```bash
cf:# adb root && adb push /tmp/delta-ota.zip /data/ota_package/
cf:# update_engine_client --update --payload=file:///data/ota_package/delta-ota.zip \
       --offset=0 --size=0 --headers=""
# (Use the values printed by ota_from_target_files for offset/size/headers)
cf:# logcat -s update_engine:V
cf:# reboot
```

### 5. Verify slot flip

```bash
cf:# bootctl get-current-slot
cf:# bootctl get-suffix 0
cf:# getprop ro.boot.slot_suffix
```

## Notes
- Cuttlefish supports A/B and Virtual A/B (snapshot) — same flow.
- For Virtual A/B on real device, `update_engine` will write to `cow_*` files and merge after reboot.

