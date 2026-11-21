# Ubuntu Touch for the Xiaomi 14T (`degas`)

![Status: Experimental](https://img.shields.io/badge/Status-Experimental-orange.svg)
![Halium: 14](https://img.shields.io/badge/Halium-14-blue.svg)
![Platform: MT6897](https://img.shields.io/badge/Platform-MT6897-green.svg)

Unofficial Ubuntu Touch 24.04 (Halium 14) adaptation for the Xiaomi 14T
(`degas`, MediaTek Dimensity 8300-Ultra `mt6897`). It runs on the stock
HyperOS vendor (Android 14, API 34) with the
[Halium kernel](https://github.com/nyxianglyph/android_kernel_xiaomi_degas)
(branch `halium-6.1.157`), which keeps the stock module ABI, so all stock
`vendor_boot` / `vendor_dlkm` modules are used as they are.

Based on the [CMF Phone 1 `tetris`](https://gitlab.com/ubports/porting/community-ports/android14/nothing-cmf-phone-1/nothing-tetris)
adaptation (MT6878) and the
[halium-generic-adaptation-build-tools](https://gitlab.com/ubports/porting/community-ports/halium-generic-adaptation-build-tools).

> **Work in progress.** Not usable as a daily phone: there is no cellular
> network yet (see [Known issues](#known-issues)).

## Status

Tested on a Xiaomi 14T, HyperOS `OS3.0.305.0.WNEIDXM` vendor, both slots on
that firmware.

| | |
|---|---|
| Boot, Lomiri UI | ✅ |
| Touchscreen | ✅ |
| Battery status | ✅ |
| Bluetooth | ✅ AIDL HAL via bluebinder |
| Wi-Fi | ✅ scan. Connecting not tested |
| SIM cards, IMEI | ✅ both slots detected |
| Cellular network | ❌ no registration, see below |
| Audio | ⚠️ HAL starts, playback not tested |
| ADB | ✅ through Developer Mode (needs a lock PIN) |
| MTP | ❌ USB shows mass storage only |
| Sensors, camera, GPS, vibration, brightness, calls/SMS | ❔ not tested |

## Known issues

### No cellular network (open)

The SIMs and IMEI are visible and oFono shows both modems online, but they
never register. The modem reports `SYS_NV_SIG_ERR_IND` (`+ENVSIGERR`) every
second; MTK RIL logs `handleNvErr: 1` and sets `vendor.ril.imei.check=fail`
(IMEI signature check failed). The modem then rejects every
`NW_SET_RADIO_STATE` request (MIPC result 2) and the RIL retries every 50 ms.

What is known:

- It is **not the kernel**: HyperOS on the same Halium kernel registers both
  SIMs on LTE with no NV error.
- It is not the NV data: Ubuntu Touch mounts copies of the same partitions.
- The RIL sends the same MIPC/AT sequence to the modem as on HyperOS before the
  first radio request (types, order and lengths match).
- So the difference is outside the RIL: some vendor, odm or system service that
  runs on HyperOS but not in the Ubuntu Touch Android container.

On HyperOS, `init.rc` reacts to the same RIL property
(`persist.radio.write.cache=1`) with `rebootrecovery --bad_nv`
(Xiaomi "IMEI protection").

### Never write the real NV partitions without SELinux labels

Ubuntu Touch runs without SELinux. Files the modem creates on the real
`nvdata`, `nvcfg`, `persist`, `protect1` or `protect2` partitions get no label,
and HyperOS later **refuses them**: `ccci_mdinit` is denied, the modem crashes
at boot (`Start MD fail, md status = 11`) and HyperOS shows no SIM cards.

This adaptation therefore mounts copies (`/userdata/<partition>.img`, made on
the first boot) instead of the real partitions
(`overlay/.../mount-android-partitions.service.d/avoid-selinux-unlabeled.conf`).
If you booted an older build, check the real partitions from TWRP with
`ls -RZ` and relabel any `unlabeled` file with the context of its neighbours
(`protect_s_data_file`, `nvdata_file`, ...) before going back to HyperOS. Back
up `nvram`, `nvdata`, `nvcfg`, `protect1`, `protect2`, `proinfo` and `persist`
before trying any of this.

### Other

- With the Ubuntu Touch kernel in `boot`, TWRP and HyperOS do not boot: Android
  init needs SELinux as the first major LSM, and `halium.config` puts AppArmor
  first. To get back, flash the stock `boot` (and `init_boot`) from bootloader
  fastboot (Vol− + Power).
- `apparmor.service` reports failed: some desktop profiles need kernel features
  this kernel lacks. App profiles do load.
- The A/B slot suffix is only in `/proc/bootconfig`, not in `/proc/cmdline`;
  `ramdisk-overlay/scripts/halium` reads both.

## Layout

| Partition | Content |
|---|---|
| `boot_a` | Halium kernel (`Image.lz4`), no ramdisk |
| `init_boot_a` | Halium initramfs (`dynparts`) + this `ramdisk-overlay` |
| `vendor_boot_a` | unchanged (stock or TWRP) |
| `super` | unchanged, HyperOS `vendor`, `odm`, ... are used read-only |
| `userdata` | ext4, unencrypted, with `ubuntu.img` (rootfs) and the NV copies |

## Build

1. Build the kernel from
   [android_kernel_xiaomi_degas](https://github.com/nyxianglyph/android_kernel_xiaomi_degas)
   (`common/degas/build.sh ut`, see its `degas/README.md`).
2. Boot images with the halium-generic-adaptation-build-tools, using the
   kernel build directory as `KERNEL_OBJ`:

   ```bash
   bash make-bootimage.sh <downloads> <KERNEL_OBJ> halium-boot-ramdisk.img out/boot.img out/system
   ```

3. Both images need the stock AVB footer props, or MiTEE KeyMint fails with
   `-33 INVALID_KEY_BLOB`:

   ```bash
   for p in boot:67108864 init_boot:8388608; do n=${p%%:*}; s=${p##*:}
     avbtool erase_footer --image $n.img
     avbtool add_hash_footer --image $n.img --partition_name $n --partition_size $s \
       --prop com.android.build.$n.os_version:14 \
       --prop com.android.build.$n.security_patch:2026-08-01 \
       --prop com.android.build.$n.fingerprint:Xiaomi/hal_mgvi_64_armv82_mt6897_global/mgvi_64_armv82:14/UP1A.231005.007/OS3.0.305.0.WNEIDXM:user/release-keys
   done
   ```

4. Root filesystem: `build-tarball-mainline.sh`, `prepare-fake-ota.sh` and
   `system-image-from-ota.sh` from the build tools give `rootfs.img`.

## Install

Wipes `userdata`. Have backups of the partitions listed above and the stock
`boot`, `init_boot`, `vendor_boot`, `dtbo` and `vbmeta`.

1. Boot TWRP (stock kernel in `boot`), unmount `/data` and remove the
   `userdata` dm device (`dmctl delete userdata`).
2. `mke2fs -t ext4 -m 0 -F -L userdata /dev/block/by-name/userdata`, mount it
   and push `rootfs.img` as `/data/ubuntu.img`.
3. From bootloader fastboot:

   ```bash
   fastboot flash boot_a boot.img
   fastboot flash init_boot_a init_boot.img
   fastboot reboot
   ```

## Related

- [TWRP for degas](https://github.com/nyxianglyph/twrp_device_xiaomi_degas)
- [Halium kernel for degas](https://github.com/nyxianglyph/android_kernel_xiaomi_degas)
