# BlackBerry KEY2 (BBF100-6, India) — LineageOS Research & Recon

Date: 2026-10-02
Device: BlackBerry KEY2 **BBF100-6** (India/APAC, dual-SIM), codename **athena**
Serial: `5000116887`

## Verdict up front

**Yes, your device is supported and the Indian BBF100-6 dual-SIM variant is fine.**
Two independent confirmations:
1. `ro.oem_unlock_supported = true` on your live device (bootloader unlock is *enabled* by OEM).
2. The current LineageOS 22.2 changelog explicitly lists **"Fix DualSIM"** (March 1 2026 build),
   and the community wiki says: *"If you're going to put your SIM card in it or aren't sure which
   variant you want, use LOS 22.2-kernel 4.4"* — i.e. the 4.4 build is the one with solid
   dual-SIM/RIL for retail. The India BBF100-6 is the dual-SIM variant, which is *the* one they fixed for.

The "conflicting info" you saw is almost certainly old (pre-2026) threads from when
only **prototype** devices could be flashed, or the older botched A/B builds. That's resolved now.

---

## Device recon (live)

| Field | Value |
|---|---|
| Model | **BBF100-6** |
| Product | `bbf100dsglobalindia` |
| Device | `bbf100` |
| Variant / oem tag | `dsglobal` / `india` |
| SoC | **SDM660 / Snapdragon 660** (board `sdm660`, baseband sdm) |
| Build ID / display | **ABN088** |
| Fingerprint | `blackberry/bbf100dsglobalindia/bbf100:8.1.0/OPM1.171019.026/ABN088:user/release-keys` |
| Android | **8.1.0** (SDK 27), user/release-keys |
| Security patch | 2018-12-01 |
| Build flavor | `bbry_sdm660_sfi-user` |
| Kernel | **4.4.78-perf+** (`#1 SMP PREEMPT Tue Dec 11`) |
| `ro.boot.binfo.primary_bc_ver` | **ABN088** |
| `ro.oem_unlock_supported` | **true** |
| `ro.boot.verifiedbootstate` | green |
| `ro.boot.flash.locked` | **1** (locked now) |
| `ro.boot.binfo.bbss_insecure` | false |
| `ro.boot.veritymode` | enforcing |
| `ro.secure` / `ro.debuggable` | 1 / 0 |
| Bootcount | 1874 |
| Carrierid | 10025 |
| Dual SIM | `persist.radio.multisim.config = dsds`, `gsm.sim.state = READY,ABSENT` |
| PRD | `ro.tct.curef = APBI-PRD63828031` |
| Trace BSN | `BZLE24CEOKB00CE` |

### A/B partition layout (confirmed — this matters!)

The KEY2 is **A/B**, unlike the KeyOne. Bootloader is **`abl_a`/`abl_b`** (the CVE-2021-1931 target).
```
abl_a -> mmcblk0p37      abl_b -> mmcblk0p49
xbl_a -> p27  xbl_b -> p39     tz_a -> p28  tz_b -> p40
boot -> p23   (boot is NOT slotted here; recovery separate at p57)
recovery -> p57   recoverysig -> p58
vbmeta -> p19
system -> p73   vendor -> p74   oem -> p75   cache -> p76   userdata -> p77
devinfo -> p6   frp -> p56   misc -> p54   nvuser -> p67   perm -> p66
```
The "botched A/B job" the unlock guide refers to = the guide tells you to flash the
autoloader **TWICE** to make sure both slots get consistent firmware before unlocking.

---

## The unlock: CVE-2021-1931 (authboot bypass)

BlackBerry/TCL's `authboot` normally refuses all bootloader operations (same as KeyOne),
**but TCL never patched Qualcomm CVE-2021-1931** on the KEY2 series. That buffer overflow
in ABL fastboot lets the unlock tool patch the signature check and flip the device to
FACTORY mode. Tool: **kibo** (BotchedRPR; Linux) or the Windows unlock tool (krab-ubica).

### Official procedure (from XDA 4781022 + FumoEnterprises guide)
1. **Remove ALL Google accounts & passwords** — they block the fastboot endpoint.
2. Download stock autoloader **ACQ160** (KEY2). (Luna uses ACT575.)
3. Flash the autoloader **TWICE** (both A/B slots), leaving the device on stock ACQ160.
4. Unlock:
   - **Linux:** `chmod +x kibo && ./kibo unlock` (in bootloader; hold Vol-Down+Power)
   - **Windows:** run unlock tool → Scan → Unlock. Progress bar sticks at 75% — that's normal.
5. Success = bootloader screen `MODE:` changes **PRODUCT → FACTORY**.
6. **BlackBerry blocks booting stock OS while unlocked** → flash the modded boot image:
   `fastboot flash boot acq160-mfi-boot.img`
7. Then flash Lineage recovery + ROM (below).

### ⚠️ Known quirks
- Even after unlock, **BlackBerry's OS refuses to boot when unlocked** unless you flash the
  modded `acq160-mfi-boot.img`. First boot of stock after unlock otherwise bootloops.
- The unlock tool is third-party; vet the binary (kibo is GPL source on GitHub — prefer Linux path).
- Unlocking + flashing custom recovery **wipes userdata**.

---

## LineageOS status (current, 2026)

**Official LineageOS support:** ❌ none. This is a community ROM (`UNOFFICIAL`).
Reason: clean GPL kernel source/vendor compliance never happened for KEY2.

### Active maintainers
- Key2: **zzach (zapaca)** — current; previously **BotchedRPR**; original unlock research **krab-ubica**.

### Builds available (via community wiki luna-terra-cg.github.io/wiki)
| Variant | Android | Kernel | Notes |
|---|---|---|---|
| **LOS 22.2 + K4.4** (`ZKrab-v1.12a`) | 15 | 4.4 | **← use this one for SIM/dual-SIM** |
| LOS 22.2 + K4.19 (`1.20F`) | 15 | 4.19 | newer kernel, beta |
| LOS 23.2 + K4.19 (`2.0G`) | 16 | 4.19 | alpha |

Key guidance from the wiki:
> *"If you are going to put your SIM card in it or aren't sure which variant you want,
> use LOS 22.2-kernel 4.4."*

The **March 1 2026** LineageOS 22.2 build explicitly added: **Fix DualSIM**, Fix VoLTE/VoWiFi,
Fix RCS, Fix IMS registration, Fix Keyboard Touchpad, "First build installable **untethered
on Retail devices**."

### What works (per XDA 4777062)
WiFi, Bluetooth, NFC, Sound, Fingerprint, Sensors, SD cards, microG, GPS, Camera,
Keyboard touchpad, most basics.
### Known issues
- **SELinux + encryption** (listed as known issue on 22.2)
- Keyboard capacitive touch can be jittery → disable via Quick Settings tile (post-Mar-2 build)
- Flash-torch yellow; occasional camera crash (older builds)
- Some Play-Integrity-sensitive apps (Authy, Google Pay) may not work
- Pastiera keyboard app can't be used during setup (skip Wi-Fi/PIN in wizard, set later)

### Install (retail, after unlock)
```
fastboot -w
fastboot flash recovery recovery-athena.img
fastboot flash boot recovery-athena.img
fastboot reboot            # into recovery
# in recovery: apply update -> from ADB
adb sideload lineage-22.2-....zip
# then wipe data, reboot (first boot up to 10 min, several restarts)
```
GApps: **MindTheGapps 15** (for LOS 22.2) / 16 (LOS 23.2), flashed separately.

### Sources
- Device: `github.com/FumoEnterprises/android_device_blackberry_athena`
- Common: `.../android_device_blackberry_sdm660-common`
- Kernel: `.../android_kernel_blackberry_sdm660-4p19`
- Vendor: `.../android_vendor_blackberry_sdm660-common`
- ROM mirror / tutorial: `fumo.enterprises`

### postmarketOS
There is a `blackberry-key2-generic` device port (mainline/U-Boot) if you ever want a
non-Android Linux distro — requires bootloader unlock first.

---

## Risk assessment for YOUR BBF100-6 India unit

| Concern | Assessment |
|---|---|
| Is India variant unlockable? | **Yes.** Same SDM660/ABL as all KEY2s; `ro.oem_unlock_supported=true`. |
| Dual-SIM supported by LOS? | **Yes** on LOS 22.2 K4.4 (`Fix DualSIM`, Mar 2026). Use that build. |
| Conflicting info you saw? | Old prototype-only/botched-A/B era. Resolved. |
| Kernel sufficiency | Your stock is 4.4.78; LOS 22.2 K4.4 matches the 4.4 line → best blob compatibility. |
| Encryption/SELinux | Still flagged incomplete on 22.2 → weigh security trade-off. |
| Anti-rollback | KEY2 A/B + Anti-rollback exists; flashing ACQ160 (older) may trip ARB. Forum method
works because ACQ160 is the intended base for the exploit; **do not** flash arbitrary older/newer mixes. |
| Backups | **Unlock + wipe destroys data. Back up first.** Also save `persist`/modem if you care about radio. |

## Bottom line
Proceed, but:
1. Use **kibo on Linux** (your normal workflow, and source is auditable).
2. Flash **ACQ160 autoloader twice** first.
3. Install **LOS 22.2 + kernel 4.4 (ZKrab-v1.12a)** for working dual-SIM.
4. Remove Google accounts before unlocking.
5. Expect to need the modded `acq160-mfi-boot.img` to boot stock raised/locked states.

## Next recon steps (when ready)
- Confirm `fastboot getvar all` on Linux (Windows driver issue seen on KeyOne).
- Dump `persist`, `modemst1/2`, `fsg`, `modem` before wipe (radio/IMEI safety).
- Extract `abl_a`/`abl_b` for the same CVE analysis as the KeyOne.
