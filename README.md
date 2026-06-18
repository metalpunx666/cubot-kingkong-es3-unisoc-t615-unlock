# cubot-kingkong-es3-unisoc-t615-unlock
Bootloader unlock &amp; Magisk root for Cubot KingKong ES 3 (Unisoc T615 / UMS9230_6h10). Experimental lab report — EMMC, A/B slots, BROM/FDL method. Not a guaranteed beginner guide.
# Cubot KingKong ES 3 — Unisoc T615 Bootloader Unlock & Magisk Root

> **Experimental Lab Report** | Unisoc T615 / UMS9230_6h10 | EMMC | A/B Partitioning  
> **Status:** Successfully Unlocked & Rooted | **Confidence:** High (see §10)  
> **Build:** `CUBOT_KINGKONG_ES_3_F071_V16_20260309`  
> **Host Environment:** macOS Monterey + Kali Linux (Raspberry Pi 4)

---

## ⚠️ Warnings & Disclaimer

- **This is an experimental lab report, not a guaranteed reproducible guide.**
- The exact minimal unlock sequence is **not fully proven**; the documented path represents a successful experimental run.
- **EMMC device:** Do not flash UFS-specific firmware files without verification. Filename mismatches exist in stock firmware packages.
- Erasing `splloader` carries real brick risk. Ensure you have a working BROM recovery path and stock firmware backups before attempting.
- Fastboot unlock commands (`oem unlock` / `flashing unlock`) are **not implemented** on this device. Do not attempt standard Android unlock workflows.
- **A/B slots:** All boot-critical partitions must be flashed to both slots or you risk slot-mismatch bootloops.

---

## 1. Device Profile

| Field | Observed Value |
|-------|----------------|
| Device | Cubot KingKong ES 3 |
| SoC | Unisoc T615 / UMS9230_6h10 |
| Build Number | `CUBOT_KINGKONG_ES_3_F071_V16_20260309` |
| Storage | **EMMC** *(correction: stock firmware contains UFS-named files, but the device uses EMMC)* |
| Partition Layout | A/B |
| Magisk Target | `init_boot.img` *(not `boot-gki.img`)* |
| OEM Unlocking | Must be manually enabled in Developer Options first |

---

## 2. Tools & Files Required

### 2.1 Tools
- `spd_dump` — [TomKing062/spreadtrum_flash](https://github.com/TomKing062/spreadtrum_flash)
- `gen_spl-unlock` — [TomKing062/CVE-2022-38694_unlock_bootloader](https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader)
- `chsize` — from the same unlock tooling set
- `pacextractor` — for extracting stock `.pac` firmware
- Magisk app (v30.x+) — for patching `init_boot.img`

### 2.2 Unisoc / UMS9230 Specific Files
- `fdl1-dl.bin`
- `fdl2-dl.bin`
- `fdl2-cboot.bin`
- `spl-unlock.bin`
- `misc-wipe.bin`
- `custom_exec_no_verify_65015f08.bin`
- `splloader.bin` *(patched — see §6.4)*
- `uboot_bak.bin` *(patched)*

### 2.3 Stock Firmware Files (from `.pac`)
- `lk-sign.bin`
- `vbmeta-sign.img`
- `init_boot.img`
- `boot-gki.img`
- `u-boot-spl-16k-ufs-sign.bin` *(filename implies UFS, but EMMC equivalent must be verified)*

---

## 3. What Does **Not** Work

Do not waste time on these methods on this device/build:

| Method | Result |
|--------|--------|
| `fastboot flashing unlock` / `fastboot oem unlock` | `unknown cmd` / `not implemented` |
| `gen_spl-unlock` against `lk-sign.bin` | Detects pattern but output size mismatch; `lk-sign.bin` is not the correct patch target for this SoC |
| Patching `vbmeta` flags while bootloader is **locked** | Bootloader still enforces verification chain; modified `init_boot` rejected |
| Patching `boot-gki.img` with Magisk | Wrong target on this device; use `init_boot.img` instead |

---

## 4. Unlock Process (Experimental)

**Critical honesty:** The exact clean minimal sequence is not fully proven. The likely unlock event occurred when `spl-unlock.bin` executed, even though the device immediately disconnected and appeared to brick.

### 4.1 Enter BROM & Load FDL Chain

```bash
./spd_dump --wait 300   exec_addr 0x65015f08   fdl fdl1-dl.bin 0x65000800   fdl fdl2-dl.bin 0x9efffe00   exec
```

> **Note:** Early attempts showed `exec_addr` failures. This was resolved by using a **patched `splloader.bin`** and patched `uboot_bak.bin` to enable the execution path. If `exec_addr` fails on your attempt, verify your `splloader` is patched or try the `custom_exec_no_verify_65015f08.bin` path.

### 4.2 Backup Bootloader Partitions

```bash
r splloader
r uboot
```

Create backups before any destructive operations. These are your recovery path.

### 4.3 Erase SPL Loaders

```bash
e splloader
e splloader_bak
reset
```

**Dangerous step.** After erase, the device will not boot normally until the stock chain is restored. BROM access remains available for recovery.

### 4.4 Patch splloader

```bash
./gen_spl-unlock -mac splloader.bin
# mv [output] to appropriate filename
```

This prepares the early bootloader stage for the unlock payload.

### 4.5 Process uboot with chsize

```bash
./chsize -mac uboot.bin
# mv [output] to appropriate filename
```

Bootloader files require exact size/alignment. Incorrect size causes no-boot even if content is otherwise valid.

### 4.6 Flash Temporary cboot

```bash
./spd_dump --wait 300   exec_addr 0x65015f08   fdl fdl1-dl.bin 0x65000800   fdl fdl2-dl.bin 0x9efffe00   exec   w uboot fdl2-cboot.bin   reset
```

The temporary `cboot` image appears to be part of the Unisoc unlock path, allowing the unlock payload to run where the stock boot chain would block it.

### 4.7 Run Unlock Payload

```bash
./spd_dump --wait 300   exec_addr 0x65015f08   fdl spl-unlock.bin 0x65000800
```

**Observed behavior:** After this command, the device disconnected and displayed a black screen. This is likely the **success behavior** — the unlock flag was written before the device dropped.

### 4.8 Apparent Brick → Recovery

If the phone shows a black screen and does not boot, this is a **recoverable soft-brick** (BROM access remains available). Restore the stock boot chain:

```bash
./spd_dump --wait 300   exec_addr 0x65015f08   fdl fdl1-dl.bin 0x65000800   fdl fdl2-dl.bin 0x9efffe00   exec   w uboot_a lk-sign.bin   w uboot_b lk-sign.bin   w splloader u-boot-spl-16k-ufs-sign.bin   w vbmeta_a vbmeta-sign.img   w vbmeta_b vbmeta-sign.img   w init_boot_a init_boot.img   w init_boot_b init_boot.img   w boot_a boot-gki.img   w boot_b boot-gki.img   reset
```

> **Correction:** The restored `splloader` file is named `u-boot-spl-16k-ufs-sign.bin` in the stock package, but this device uses **EMMC**. Verify whether an EMMC-equivalent `splloader` should be substituted. In this experimental run, the UFS-named file restored successfully, but this is a documented uncertainty.

After restore, the phone booted and displayed the **unlocked bootloader warning**.

**Likely explanation:** `spl-unlock.bin` wrote the unlock flag successfully, but the temporary boot chain was broken. Restoring stock boot files allowed the stock bootloader to read the already-written unlock flag.

---

## 5. Magisk Root Process

### 5.1 Patch vbmeta (Disable Verified Boot)

Set bytes at offset `0x78–0x7C` to `0x02000000` to disable Android Verified Boot (AVB) checks.

You can use a hex editor or:
```bash
# Example with dd / xxd — adjust to your preferred method
```

### 5.2 Flash Patched vbmeta to Both Slots

```bash
w vbmeta_a /path/to/vbmeta-sign-patched.img
w vbmeta_b /path/to/vbmeta-sign-patched.img
```

### 5.3 Patch init_boot.img with Magisk

1. Copy stock `init_boot.img` to the phone or use Magisk app on-device
2. Magisk app → **Install** → **Select and Patch a File**
3. Select `init_boot.img`
4. Output will be similar to: `magisk_patched-30700_6PawC.img`

> **Critical:** On this device, `init_boot.img` is the correct Magisk patch target. Patching `boot-gki.img` will not produce root and may cause boot failure.

### 5.4 Flash Patched init_boot to Both Slots

```bash
w init_boot_a /path/to/magisk_patched.img
w init_boot_b /path/to/magisk_patched.img
reset
```

### 5.5 Verify Root

```bash
adb shell su -c 'id'
```

**Expected result:**
```
id=0(root) gid=0(root) groups=0(root) context=u:r:magisk:s0
```

---

## 6. Evidence & Findings

| Evidence | Observation |
|----------|-------------|
| Boot Warning | `LOCK FLAG IS :UNLOCK!!!` displayed during boot |
| Skip Verify | `SKIP VERIFY` visible on boot screen |
| Developer Options | `Bootloader is already unlocked` reported |
| Build Number | `CUBOT_KINGKONG_ES_3_F071_V16_20260309` |
| Magisk Root | `adb shell su -c 'id'` returns `uid=0(root)` |
| NetHunter/Kali | Kali NetHunter Lite environment installed and booting post-root |

---

## 7. Troubleshooting Matrix

| Symptom | Likely Cause | Action |
|---------|--------------|--------|
| `LIBUSB_ERROR_TIMEOUT` | Device resetting or changing modes | Wait, reconnect, retry BROM entry |
| `CHECK_BAUD FAIL` | Timing or connection issue | Retry BROM connection, check cable |
| `FDL2: file does not exist` | Path issue | Use full quoted paths to all `.bin` files |
| Fastboot unlock fails | Command not implemented on this device | Use BROM/FDL path exclusively |
| Black screen after unlock payload | Temporary boot chain broken, not hard-brick | Restore stock boot chain via FDL (§4.8) |
| "No valid OS found" after patched image | AVB still active or boot chain invalid | Unlock first, then patch `vbmeta`, then flash `init_boot` |
| `exec_addr` fails | SPL/FDL mismatch or unpatched loader | Use patched `splloader.bin` or `custom_exec_no_verify` file |

---

## 8. Confidence Assessment

| Claim | Confidence | Notes |
|-------|------------|-------|
| Device can be bootloader unlocked | **High** | Successfully achieved and verified |
| Device can be rooted with Magisk | **High** | `init_boot` patch + vbmeta disable confirmed working |
| `spl-unlock.bin` was the actual unlock trigger | **Medium-High** | Most likely, but not 100% isolated from other variables |
| Exact minimal unlock sequence is known | **Low** | Requires repeat testing from stock on identical build |
| Ready as a guaranteed public beginner guide | **No** | Repeat testing and minimal-step refinement needed first |

---

## 9. Recommended Next Steps

Before treating this as a reproducible public guide:

1. **Repeat the unlock** from clean stock firmware on the same build or an identical device.
2. **Identify the minimal required steps** — remove unnecessary recovery panic and redundant flashes.
3. **Verify the correct EMMC `splloader` filename** vs. the UFS-named stock file used during restoration.
4. **Document the exact `exec_addr` fix** — the relationship between patched `splloader` and successful `exec_addr` execution needs clarification.
5. **Test slot consistency** — confirm A/B slot behavior after OTA updates when rooted.

---

## 10. Credits

- **TomKing062** — Unisoc research tools (`spd_dump`, `gen_spl-unlock`, `chsize`) and CVE-2022-38694 unlock research
- **topjohnwu** — Magisk
- **Hovatek Forum** — Previous Unisoc bootloader and AVB research
- **Cubot** — For shipping a device that appears to be unlockable (intentionally or not)
- **Kimi AI** — Troubleshooting, reconstruction, and corrections
- **Original Tester:** m3t4l|>unx


---

## 11. Session Corrections (vs. Original Draft)

This report incorporates the following factual corrections identified during review:

| Correction | Original Error | Fix Applied |
|------------|----------------|-------------|
| Storage type | Listed as UFS | Confirmed **EMMC** |
| Host environment | Not specified | Documented as **macOS Monterey + Kali Pi4** |
| `exec_addr` behavior | Implied immediate success | Noted that `exec_addr` **failed initially** and required patched `splloader`/`uboot_bak` |
| `splloader` source | Unclear which file was patched | Clarified that **patched `splloader.bin`** from prior session was used with `gen_spl-unlock` |
| `lk-sign.bin` target | Suggested as viable | Confirmed **output size mismatch** — not correct target for this SoC |
| `boot-gki.img` | Suggested as Magisk target | Confirmed **wrong target** — use `init_boot.img` instead |

---

*Document Status: Corrected Edition — June 2026*  
*Original Report: March 2026*  
*License: Use at your own risk. No warranty expressed or implied.*
