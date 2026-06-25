# Cubot KingKong ES 3 — Complete Unlock, Root & NetHunter Guide

> Complete verified guide for FDL2 access, bootloader unlock, Magisk root, and NetHunter installation on the Cubot KingKong ES 3 with Unisoc T615 / UMS9230_6h10.

---

## Status

| Item                       | Status                             |
| -------------------------- | ---------------------------------- |
| Device                     | Cubot KingKong ES 3                |
| SoC                        | Unisoc T615 / UMS9230_6h10         |
| BROM                       | SPRD3                              |
| Storage                    | UFS                                |
| Partition Layout           | A/B slots                          |
| FDL2 Access                | Verified                           |
| Bootloader Unlock          | Verified                           |
| Magisk Root                | Verified                           |
| NetHunter                  | Installable                        |
| Built-in WiFi Monitor Mode | Not supported                      |
| WiFi Injection             | Requires external USB WiFi adapter |

Verified during live testing:

```text
2026-06-23 / 2026-06-24
```

Tested build:

```text
CUBOT_KINGKONG_ES_3_F071_V16_20260309
```

---

## Table of Contents

* [How to Use This Guide](#how-to-use-this-guide)
* [Warnings](#warnings)
* [Verified Device](#verified-device)
* [Compatibility](#compatibility)
* [What You Need](#what-you-need)
* [What This Device Actually Is](#what-this-device-actually-is)
* [Critical Discovery](#critical-discovery)
* [Build spd_dump](#build-spd_dump)
* [Prepare Unlock Folder](#prepare-unlock-folder)
* [Enter BROM Mode](#enter-brom-mode)
* [FDL2 Access](#fdl2-access)
* [Bootloader Unlock](#bootloader-unlock)
* [Magisk Root](#magisk-root)
* [NetHunter Install](#nethunter-install)
* [The WiFi Reality](#the-wifi-reality)
* [Troubleshooting](#troubleshooting)
* [Files Reference](#files-reference)
* [Partition Reference](#partition-reference)
* [Verified Working Commands](#verified-working-commands)
* [Changelog](#changelog)
* [Credits](#credits)
* [Disclaimer](#disclaimer)

---

## How to Use This Guide

This guide is for unlocking the bootloader and gaining root on the Cubot KingKong ES 3 and similar Unisoc UMS9230 devices.

It uses the CVE-2022-38694 exploit through `spd_dump` to reach FDL2 mode. FDL2 mode allows low-level partition read/write access.

Use this guide if:

* You want to unlock the bootloader on a Cubot KingKong ES 3.
* You want to root with Magisk.
* You want to install NetHunter.
* You want to dump boot-critical partitions.
* You want to understand the exact working FDL2 chain for this device.

Do not use this guide if:

* You are not willing to wipe the device.
* You do not have the correct files for your firmware.
* You are not comfortable with raw partition flashing.
* You expect fastboot unlock commands to work.
* You want built-in WiFi monitor mode.

Recommended workflow:

1. Confirm your device is compatible.
2. Gather all required files.
3. Build `spd_dump` on Linux.
4. Enter BROM mode.
5. Use the verified `loadexec` command to reach FDL2.
6. Flash patched `splloader.bin`.
7. Flash `misc-wipe.bin`.
8. Reboot and let the phone wipe.
9. Install Magisk.
10. Install NetHunter if desired.
11. Use an external USB WiFi adapter for monitor mode or injection.

---

## Warnings

This process can wipe your data, break Android boot, or brick the device if done incorrectly.

Important warnings:

* Flashing `misc-wipe.bin` triggers a factory reset.
* Bootloader unlock will erase user data.
* Do not flash random FDL files from another phone.
* Do not flash random bootloader files from another model.
* Do not rename the payload file unless you know exactly what you are doing.
* Do not use `exec_addr` or `exec_addr2` on this device.
* Use Linux for critical BROM/FDL work.
* macOS libusb behavior can be unstable with Unisoc BROM mode.
* Built-in WiFi does not support monitor mode or packet injection.

No warranty is provided.

You are responsible for your own device.

Tiny shocking development: raw bootloader flashing can in fact destroy bootloaders. Who could have guessed.

---

## Verified Device

Verified working device:

```text
Cubot KingKong ES 3
```

Observed device profile:

| Field                 | Observed Value                        |
| --------------------- | ------------------------------------- |
| Device                | Cubot KingKong ES 3                   |
| Build Number          | CUBOT_KINGKONG_ES_3_F071_V16_20260309 |
| SoC                   | Unisoc T615 / UMS9230_6h10            |
| BROM                  | SPRD3                                 |
| Storage               | UFS                                   |
| Partition Layout      | A/B slots                             |
| Boot Key              | Volume Down while plugging USB        |
| Root Method           | Magisk                                |
| NetHunter             | Installable                           |
| Built-in Monitor Mode | Not supported                         |

Important correction:

Early assumptions treated the device as EMMC. Live testing confirmed this device uses **UFS**, not EMMC.

---

## Compatibility

### Verified Working

* Cubot KingKong ES 3
* Unisoc T615 / UMS9230_6h10
* SPRD3 BROM
* UFS storage
* A/B partitioning

### Likely Similar

These devices may use similar patterns, but are not fully verified by this guide:

* Unisoc T615 devices
* Unisoc T606 UMS9230 variants
* Unisoc T616 UMS9230 variants
* Other SPRD3 / UMS9230 devices

### Possibly Similar

These may share some similarities, but require firmware-specific verification:

* Some Doogee UMS9230 phones
* Some Ulefone UMS9230 phones
* Some Oukitel UMS9230 phones
* Some Blackview UMS9230 phones

### Not Covered

This guide does not cover:

* MediaTek devices
* Qualcomm devices
* Samsung Exynos devices
* Unisoc T610 / T618
* Unisoc T700 / T770
* Newer patched Unisoc platforms
* Devices with different FDL addresses
* Devices with different BROM behavior

### Compatibility Check

Power off the phone completely.

Hold:

```text
Volume Down
```

While holding Volume Down, plug in USB.

On Linux, check:

```bash
lsusb | grep "1782:4d00"
```

Possible expected result:

```text
Spreadtrum Communications Inc.
1782:4d00
```

If `spd_dump` returns:

```text
CHECK_BAUD bootrom
BSL_REP_VER: "SPRD3"
CMD_CONNECT bootrom
```

then the device is likely compatible with this method.

---

## What You Need

### Hardware

Required:

* Cubot KingKong ES 3 or compatible Unisoc UMS9230 device
* USB-C cable
* Linux machine
* Kali Linux recommended
* Battery charged to at least 20%

Recommended:

* Raspberry Pi 4 running Kali Linux
* Good USB-C cable
* USB-C OTG adapter
* External USB WiFi adapter for NetHunter monitor mode

### Software

Required:

* `spd_dump`
* `adb`
* `git`
* `make`
* Magisk app
* NetHunter app

Optional:

* `fastboot`
* firmware extraction tools
* PAC firmware extractor
* partition backup storage

### Required Files

Place these files in:

```text
~/cubot_unlock/
```

Required files:

```text
custom_exec_no_verify_65015f08.bin
fdl1-dl.bin
fdl2-dl.bin
splloader.bin
misc-wipe.bin
spd_dump
```

Optional / backup files:

```text
uboot.bin
uboot_bak.bin
splloader_og.bin
boot_a.bin
boot_b.bin
vendor_boot_a.bin
dtb_a.bin
init_boot_a.bin
```

---

## What This Device Actually Is

Observed hardware/software reality:

```text
SoC:        Unisoc T615 / UMS9230_6h10
Storage:    UFS
Partitions: A/B
BROM:       SPRD3
Boot key:   Volume Down
```

Critical notes:

* The device uses UFS storage.
* The device has A/B slots.
* Fastboot unlock commands may not be implemented.
* BROM is accessible with the correct key combo.
* FDL2 access is possible with the correct exploit delivery.
* The working method depends on `loadexec`.

---

## Critical Discovery

The most important discovery:

```text
The only verified working exploit delivery method on this device is loadexec.
```

The working payload filename must be:

```text
custom_exec_no_verify_65015f08.bin
```

The address is part of the filename:

```text
65015f08
```

Working command pattern:

```text
loadexec custom_exec_no_verify_65015f08.bin
```

Failed methods:

```text
exec_addr
exec_addr2
```

During testing, `exec_addr` and `exec_addr2` caused the BROM to crash on the first FDL1 data packet.

The `loadexec` method allowed the chain to continue:

```text
BROM → payload → FDL1 → FDL2 → FDL2>
```

### Why This Matters

The old style approach manually sends a payload to a specific address.

On this device, that path fails.

The working method uses filename-based address parsing through `loadexec`, which appears to deliver the payload in a way this UMS9230_6h10 target accepts.

Practical rule:

```text
Use loadexec only.
```

Do not rename the payload unless you know exactly what you are doing.

Do not change:

```text
custom_exec_no_verify_65015f08.bin
```

to:

```text
custom_exec_no_verify_65015f70.bin
```

or any other guessed address.

Guessing addresses is how phones become paperweights. Very futuristic paperweights, sure, but still paperweights.

---

## Build spd_dump

Clone the source:

```bash
git clone https://github.com/TomKing062/spreadtrum_flash.git
```

Enter the folder:

```bash
cd spreadtrum_flash
```

Build:

```bash
make
```

Create your unlock folder:

```bash
mkdir -p ~/cubot_unlock
```

Copy `spd_dump`:

```bash
cp spd_dump ~/cubot_unlock/
```

Make it executable:

```bash
chmod +x ~/cubot_unlock/spd_dump
```

---

## Prepare Unlock Folder

Create the unlock folder:

```bash
mkdir -p ~/cubot_unlock
```

Enter it:

```bash
cd ~/cubot_unlock
```

Your folder should look similar to:

```text
~/cubot_unlock/
├── custom_exec_no_verify_65015f08.bin
├── fdl1-dl.bin
├── fdl2-dl.bin
├── misc-wipe.bin
├── spd_dump
└── splloader.bin
```

Set executable permission:

```bash
chmod +x spd_dump
```

---

## Get FDL Files

The safest source is your own stock firmware PAC file.

Extract:

```text
fdl1-dl.bin
fdl2-dl.bin
```

These may appear in stock firmware under names such as:

```text
fdl1.bin
fdl2.bin
splloader.bin
uboot.bin
```

Warning:

Do not blindly use FDL files from another phone.

FDL files are device and firmware sensitive.

Using the wrong ones may brick the device.

---

## Enter BROM Mode

Phone steps:

1. Power off the phone completely.
2. Hold **Volume Down**.
3. While holding Volume Down, plug in USB.
4. Keep holding until `spd_dump` connects.

Linux check:

```bash
lsusb | grep "1782:4d00"
```

Expected style of result:

```text
Spreadtrum Communications Inc.
1782:4d00
```

If Volume Down does not work, try:

```text
Volume Up
```

or:

```text
Volume Up + Volume Down
```

Some Unisoc devices vary because apparently buttons are too boring to standardize.

---

## FDL2 Access

From Linux:

```bash
cd ~/cubot_unlock
```

Run the verified command:

```bash
sudo ./spd_dump --verbose 2 --wait 300 loadexec custom_exec_no_verify_65015f08.bin fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
```

Expected output signs:

```text
CHECK_BAUD bootrom
BSL_REP_VER: "SPRD3"
CMD_CONNECT bootrom
SEND fdl1-dl.bin to 0x65000800
EXEC FDL1
SEND fdl2-dl.bin to 0x9efffe00
FDL2 >
```

Success condition:

```text
FDL2 >
```

Once you see `FDL2>`, you have low-level partition read/write access.

Do not freestyle commands here. FDL2 is powerful, and power in computing usually just means “can destroy things faster.”

---

## Bootloader Unlock

At the `FDL2>` prompt:

Flash patched bootloader:

```text
w splloader splloader.bin
```

Trigger factory reset:

```text
w misc misc-wipe.bin
```

Reboot:

```text
reset
```

Expected behavior:

1. Phone reboots.
2. Recovery/wipe process appears.
3. Userdata is erased.
4. Phone boots to Android setup.
5. Bootloader is unlocked.

### Why the Wipe Is Required

`misc-wipe.bin` triggers a factory reset.

This removes old Android security/encryption state that may conflict with the unlocked bootloader state.

Skipping the wipe may cause boot issues.

Recommended:

```text
Flash splloader.bin + misc-wipe.bin
```

Not recommended unless you know what you are doing:

```text
Flash splloader.bin only
```

---

## Magisk Root

After bootloader unlock and initial Android setup, install Magisk.

### Method A: Magisk Direct Install

This is the easiest method if available.

Steps:

1. Download Magisk APK to the phone.
2. Install Magisk.
3. Open Magisk.
4. Select **Install**.
5. Select **Direct Install**.
6. Reboot.
7. Verify root.

Verify root:

```bash
adb shell su -c "id"
```

Expected result:

```text
uid=0(root)
```

### Method B: Patch Boot Image Through FDL2

Use this if Direct Install does not work or if you want manual control.

At `FDL2>`:

```text
r boot_a
```

Copy dumped boot image to phone:

```bash
adb push boot_a.bin /sdcard/Download/
```

In Magisk:

```text
Install → Select and Patch a File → boot_a.bin
```

Pull patched image back to Linux:

```bash
adb pull /sdcard/Download/magisk_patched-*.img
```

Boot back into FDL2 and flash:

```text
w boot_a magisk_patched.img
```

Reboot:

```text
reset
```

### Method C: Fastboot

If fastboot flashing works after unlock:

```bash
adb reboot bootloader
```

Flash patched boot image:

```bash
fastboot flash boot_a magisk_patched.img
```

Reboot:

```bash
fastboot reboot
```

If fastboot commands are not implemented, use FDL2.

---

## NetHunter Install

After Magisk root works, NetHunter can be installed.

### Method A: In-App Install

1. Download NetHunter from:

```text
https://store.nethunter.com/
```

2. Install NetHunter.
3. Open NetHunter.
4. Grant root access in Magisk.
5. Open Kali Chroot Manager.
6. Install Kali Chroot.
7. Choose Minimal or Full.
8. Wait for installation.

Approximate space:

```text
Minimal: 2GB+
Full:    4GB-5GB+
```

### Method B: Manual Chroot Install

On Linux:

```bash
wget https://kali.download/nethunter-images/current/rootfs/kalifs-arm64-minimal.tar.xz
```

Push to phone:

```bash
adb push kalifs-arm64-minimal.tar.xz /sdcard/
```

On phone:

```text
NetHunter → Kali Chroot Manager → Install from SDCard
```

Select the `.tar.xz` file.

### NetHunter Troubleshooting

If download fails:

* Check free space on `/data`.
* Use a stable network.
* Try manual install.
* Clear NetHunter app data and retry.

If NetHunter says no root:

* Open Magisk.
* Go to Superuser.
* Grant NetHunter root permission.

If extraction fails:

* Check available storage.
* Clear app data.
* Try manual chroot install.

---

## The WiFi Reality

Built-in WiFi does not support monitor mode.

This is one of the most important practical findings.

### Observed WiFi Stack

Observed driver:

```text
sprd_wlan_combo
```

Observed behavior:

* `wlan0` supports station/client mode.
* `wlan0` supports AP mode.
* `wlan0` supports P2P mode.
* `wlan0` does not expose monitor mode.
* Packet injection does not work on built-in WiFi.

### Why Built-In Monitor Mode Does Not Work

The issue is not simply a missing kernel option.

The Unisoc WiFi driver is proprietary.

It does not expose monitor mode through the normal Linux wireless stack.

Rebuilding the kernel may help with other features, but it will not automatically make `sprd_wlan_combo` support monitor mode.

The bottleneck is the proprietary driver stack.

### Practical Solution

Use an external USB WiFi adapter.

Recommended examples:

```text
Alfa AWUS036ACH
TP-Link TL-WN722N v1
```

Expected path:

```text
USB WiFi adapter → USB-C OTG → wlan1 or wlan2 → monitor mode
```

Example:

```bash
airmon-ng start wlan1
```

This is the sane path.

The insane path is reverse-engineering the proprietary Unisoc WiFi driver stack. Humanity has enough suffering.

---

## Troubleshooting

### `LIBUSB_ERROR_TIMEOUT`

Possible causes:

* Phone is not in BROM mode.
* Wrong key timing.
* Bad cable.
* Bad USB port.
* macOS libusb instability.

Fix:

* Use Linux.
* Power off completely.
* Hold Volume Down before plugging USB.
* Keep holding until connection starts.
* Try a different USB cable.
* Try a different USB port.

---

### `LIBUSB_ERROR_IO` on FDL1 START_DATA

Likely cause:

```text
You are using exec_addr or exec_addr2.
```

Fix:

```text
Use loadexec only.
```

Correct command:

```bash
sudo ./spd_dump --verbose 2 --wait 300 loadexec custom_exec_no_verify_65015f08.bin fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
```

---

### `wrong command or wrong mode detected`

Likely cause:

```text
BROM crashed.
```

Fix:

1. Unplug USB.
2. Hold Power + Volume Up for around 10 seconds.
3. Wait around 10 seconds.
4. Try BROM entry again.

---

### `FDL2: incompatible partition`

This may be non-fatal.

If FDL2 still loads and you get:

```text
FDL2 >
```

continue carefully.

---

### Phone Will Not Enter BROM

Try:

```text
Volume Down
```

If that fails:

```text
Volume Up
```

If that fails:

```text
Volume Up + Volume Down
```

Also check:

* Battery charge
* USB cable
* USB port
* Whether the phone is fully powered off

---

### Magisk Direct Install Fails

Try the manual patch method.

Basic flow:

```text
Dump boot_a → patch with Magisk → flash patched image through FDL2
```

---

### NetHunter Chroot Download Fails

Check:

* Free space on `/data`
* Root access granted in Magisk
* Network connection
* NetHunter app permissions

Fallback:

```text
Use manual install from tar.xz
```

---

### No Monitor Mode on `wlan0`

Expected.

Use an external USB WiFi adapter.

---

### Fastboot Unlock Fails

Expected on this device.

Commands such as:

```bash
fastboot flashing unlock
```

or:

```bash
fastboot oem unlock
```

may fail or return unsupported command errors.

Use the FDL2 bootloader unlock method instead.

---

## Files Reference

Example unlock folder:

```text
~/cubot_unlock/
├── custom_exec_no_verify_65015f08.bin
├── fdl1-dl.bin
├── fdl2-dl.bin
├── spd_dump
├── splloader.bin
├── misc-wipe.bin
├── uboot.bin
├── uboot_bak.bin
└── splloader_og.bin
```

### File Descriptions

| File                                 | Purpose                       |
| ------------------------------------ | ----------------------------- |
| `custom_exec_no_verify_65015f08.bin` | CVE-2022-38694 payload        |
| `fdl1-dl.bin`                        | FDL1 stage loader             |
| `fdl2-dl.bin`                        | FDL2 stage loader             |
| `spd_dump`                           | Unisoc flashing / FDL utility |
| `splloader.bin`                      | Patched bootloader            |
| `misc-wipe.bin`                      | Factory reset trigger         |
| `uboot.bin`                          | Patched U-Boot                |
| `uboot_bak.bin`                      | U-Boot backup                 |
| `splloader_og.bin`                   | Original splloader backup     |

Approximate file notes from testing:

```text
custom_exec_no_verify_65015f08.bin   ~96 bytes
fdl1-dl.bin                          ~59KB
fdl2-dl.bin                          ~913KB
spd_dump                             ~134KB
splloader.bin                        ~65KB
misc-wipe.bin                        ~2KB
uboot.bin                            ~1.3MB
uboot_bak.bin                        ~1.3MB
splloader_og.bin                     ~65KB
```

---

## Partition Reference

Observed partition notes from live testing:

```text
boot_a / boot_b:                    64MB each
vendor_boot_a / vendor_boot_b:      100MB each
dtb_a / dtb_b:                       8MB each
init_boot_a / init_boot_b:           8MB each
super:                             5600MB
userdata:                         237151MB (~231GB)
storage type:                      UFS
total partitions:                  74
```

Important backup partitions:

```text
boot_a
boot_b
vendor_boot_a
vendor_boot_b
dtb_a
dtb_b
init_boot_a
init_boot_b
splloader
uboot
misc
```

---

## Verified Working Commands

### FDL2 Access

```bash
sudo ./spd_dump --verbose 2 --wait 300 loadexec custom_exec_no_verify_65015f08.bin fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
```

### Bootloader Unlock

```text
w splloader splloader.bin
w misc misc-wipe.bin
reset
```

### Dump Partitions

```text
r boot_a
r boot_b
r dtb_a
r vendor_boot_a
```

### Flash Magisk Patched Boot

```text
w boot_a magisk_patched.img
reset
```

### List Partitions

```text
p
```

### Reboot to Recovery

```text
reboot-recovery
```

### Reboot to Fastboot

```text
reboot-fastboot
```

### Power Off

```text
poweroff
```

### Increase Timeout

```text
timeout 30000
```

### Increase Block Size

```text
blk_size 65535
```

### Enable Raw Data Mode

```text
rawdata 1
```

---

## Recommended Backup Flow

Before flashing anything, dump key partitions when possible.

At `FDL2>`:

```text
r boot_a
r boot_b
r vendor_boot_a
r vendor_boot_b
r dtb_a
r dtb_b
r init_boot_a
r init_boot_b
```

Store backups safely.

Suggested backup folder on Linux:

```bash
mkdir -p ~/cubot_unlock/backups
```

Move dumped files:

```bash
mv *.bin ~/cubot_unlock/backups/
```

Do not overwrite original backups.

---

## Notes on A/B Slots

This device uses A/B partitioning.

Be aware of the active slot before flashing boot-critical partitions.

Boot-related partitions may exist as:

```text
boot_a
boot_b
vendor_boot_a
vendor_boot_b
dtb_a
dtb_b
init_boot_a
init_boot_b
```

If you flash only one slot, make sure you understand which slot the device is booting from.

Flashing the wrong slot may cause confusion, failed boots, or slot mismatch issues.

---

## Notes on macOS

macOS may work for building tools or file preparation.

However, macOS libusb behavior can be unstable with Unisoc BROM mode.

Recommended:

```text
Use Linux for BROM / FDL operations.
```

Kali Linux on Raspberry Pi 4 was used successfully during testing.

---

## Notes on Fastboot

Fastboot may exist but not fully implement unlock or flash behavior.

Observed expectation:

```text
fastboot flashing unlock
fastboot oem unlock
```

may fail or return unknown command style errors.

The verified unlock method is through FDL2, not fastboot.

---

## No Downloads Notice

This repository provides documentation, verified commands, file references, and troubleshooting notes.

Binary unlock files are not hosted here at this time.

Users should extract device-specific FDL files and firmware components from their own stock firmware package whenever possible.

Do not blindly use bootloader or FDL files from another model.

---

## Future Automation

A future helper script may automate parts of this workflow.

Recommended features for any future automation:

* Dry-run mode
* File existence checks
* Exact filename validation
* BROM detection helper
* Explicit wipe warning
* Confirmation prompts before flashing
* Command preview before execution
* No bundled proprietary binaries
* Clear failure handling
* Log output saving

Automation should not hide what is being flashed.

Rooting tools that hide flashing details are how users speedrun regret.

---

## Changelog

| Date       | Change                                                |
| ---------- | ----------------------------------------------------- |
| 2026-06-23 | Initial testing on macOS                              |
| 2026-06-23 | `exec_addr` variants failed                           |
| 2026-06-24 | Moved testing to Kali Linux                           |
| 2026-06-24 | Confirmed `loadexec` as the working method            |
| 2026-06-24 | Verified FDL2 access                                  |
| 2026-06-24 | Dumped partitions                                     |
| 2026-06-24 | Confirmed UFS storage                                 |
| 2026-06-24 | Verified bootloader unlock                            |
| 2026-06-24 | Verified Magisk root                                  |
| 2026-06-24 | Confirmed built-in WiFi does not support monitor mode |
| 2026-06-24 | Guide finalized with verified command chain           |

---

## Credits

* **TomKing062** — Unisoc research tools and `spreadtrum_flash`
* **topjohnwu** — Magisk
* **Kali NetHunter Project** — NetHunter platform
* **Hovatek Forum** — Unisoc bootloader and AVB research
* **Kimi AI** — Troubleshooting, reconstruction, and corrections
* **Original Tester / Maintainer:** Miguel Portugal / KiMiGuel

---

## Disclaimer

This guide is provided for research, repair, recovery, and owner-controlled device modification.

No warranty is provided.

You are responsible for any damage, data loss, boot failure, or device brick.

Use stock firmware from your own device whenever possible.

Do not flash random bootloader files across different models.

Do not use this guide on devices you do not own or control.

---

## Final Verified Status

```text
Device: Cubot KingKong ES 3
SoC: Unisoc T615 / UMS9230_6h10
Build: CUBOT_KINGKONG_ES_3_F071_V16_20260309
Storage: UFS
Partition Layout: A/B
BROM: SPRD3
FDL2: Confirmed
Bootloader: Unlocked
Magisk root: Confirmed
NetHunter: Installable
Built-in WiFi monitor mode: Not supported
WiFi injection: External USB adapter required
```

---

*Last Updated: 2026-06-24*
