# BlackBerry KEY2 Research

Reverse-engineering, unlock, and LineageOS research for the **BlackBerry KEY2
(BBF100-6, India dual-SIM)** — the UEFI/ABL sibling of the KEYone, and the
device where the unpatched **CVE-2021-1931** fastboot overflow still yields a
bootloader unlock.

> **Disclaimer.** This repo is a *research aid*, not a flashing guide.
> Everything here is for educational / defensive research on devices owned by
> the author. Unlocking a bootloader and flashing firmware can **permanently
> brick** a device with no recovery short of JTAG/ISP chip-out, and unlocks
> wipe userdata. Proceed at your own risk. Third-party tool binaries are
> **not redistributed here** — hashes and provenance are recorded instead.

---

## Device

| Field | Value |
|---|---|
| Model | **BBF100-6** (India/APAC, dual-SIM) |
| Codename | `bbf100` / athena |
| Serial | `5000116887` |
| SoC | **SDM660 / Snapdragon 660** (`sdm660`) |
| Bootloader | **UEFI ABL** (`abl_a` / `abl_b`) — CVE-2021-1931 target |
| Stock build | **ABN088** (Android 8.1.0, kernel 4.4.78-perf+) |
| Security patch | 2018-12-01 |
| Fingerprint | `blackberry/bbf100dsglobalindia/bbf100:8.1.0/OPM1.171019.026/ABN088:user/release-keys` |
| OEM unlock | `ro.oem_unlock_supported = true` |
| Verified boot | green / `flash.locked = 1` (stock) |
| Status | **Unlocked + running LineageOS 22.2 (Android 15), dual-SIM working** |

Full live recon: [`notes/04-key2-lineageos-research.md`](notes/04-key2-lineageos-research.md).

### A/B partition layout

Unlike the KEYone (A-only), the KEY2 is A/B — and the bootloader is slotted:

```
abl_a  -> mmcblk0p37     abl_b  -> mmcblk0p49
xbl_a  -> p27            xbl_b  -> p39
tz_a   -> p28            tz_b   -> p40
boot   -> p23 (not slotted)   recovery -> p57   recoverysig -> p58
vbmeta -> p19   system -> p73   vendor -> p74   oem -> p75
userdata -> p77   misc -> p54   frp -> p56   nvuser -> p67   perm -> p66
```

BlackBerry's A/B implementation is inconsistent enough that the unlock
procedure requires flashing the stock autoloader **twice** (both slots) before
unlocking. Raw capture: [`recon/key2_byname.txt`](recon/key2_byname.txt).

---

## The unlock — CVE-2021-1931

BlackBerry/TCL's `authboot` normally refuses bootloader operations (as on the
KEYone), **but TCL never patched Qualcomm CVE-2021-1931 on the KEY2 series**:
a buffer overflow in the UEFI ABL fastboot parser lets a host overwrite the
RAM-resident bootloader with a pre-patched copy, flipping the device from
`PRODUCT` to `FACTORY` mode.

Tools: **kibo** (BotchedRPR, Linux, GPL) or the Windows
`BlackBerryBootUnlock.exe` (krab-ubica).

### Tool reverse engineering

The Windows tool is a .NET/C++-CLI assembly over `libusb-1.0.dll`; its entire
fastboot surface is `getvar:bb_bc_version`, `getvar:all`, `getvar:bootmode`,
`reboot-bootloader`. It selects a payload by bootloader version
(`ACQ160` → `160.bin`, `ACT575` → `575.bin`), patches a hardcoded table of byte
offsets (the signature-check patch), then sends the 429,108-byte buffer over a
bulk transfer — the overflow. Success = `bootmode` changes `PRODUCT → FACTORY`.

Full analysis: [`notes/07-key2-unlock-tool-re.md`](notes/07-key2-unlock-tool-re.md).
Decompiled IL: [`exploit/key2-unlock/il.txt`](exploit/key2-unlock/il.txt).

### Tool binary provenance (not redistributed)

| File | SHA-256 |
|---|---|
| `BlackBerryBootUnlock.exe` (v1.1) | `374293c54518406b4145a2bc7a1a9c91b3e39109fdf629d059d1ed1b73c455f4` |
| `data/160.bin` (ACQ160 ABL payload) | `2d270254389b00576d2430468fd79752020c45fc3f5dd0dd16eff9108fd37cce` |
| `data/575.bin` (ACT575 ABL payload) | `ef9b30171df44710c5325cc7d5b5226a6f15cfe334fbde308b23e03616687a18` |

### Procedure (summary)

1. Remove all Google accounts (they block the fastboot endpoint).
2. Flash stock **ACQ160** autoloader **twice** (both A/B slots).
3. Unlock: `kibo unlock` (Linux) or the Windows tool (progress sticks at 75% — normal).
4. `MODE:` on the bootloader screen changes **PRODUCT → FACTORY**.
5. BlackBerry's stock OS refuses to boot while unlocked → flash the modded
   `acq160-mfi-boot.img`, then recovery + ROM.
6. Unlock + custom recovery wipes userdata; back up first.

---

## LineageOS

**Unofficial/community** (no official LineageOS support — clean GPL kernel
source never happened for the KEY2). Recommended build for dual-SIM:
**LineageOS 22.2 + kernel 4.4** (`ZKrab-v1.10a`). Full step-by-step guide:
[`docs/KEY2-LineageOS-Guide.md`](docs/KEY2-LineageOS-Guide.md).

Known issues on 22.2 include SELinux + encryption completeness, keyboard
touchpad jitter (disable via Quick Settings), and some Play-Integrity
sensitive apps.

---

## Why the KEY2 fell and the KEYone did not

| | KEY2 (cracked) | KEYone (not) |
|---|---|---|
| SoC | SDM660 | MSM8953 |
| Bootloader | **UEFI ABL** (`abl.elf`, edk2) | **LittleKernel** (`emmc_appsboot.mbn`) |
| Exploit | **CVE-2021-1931** — overflow in ABL fastboot parser | n/a — different codebase |
| Fix status | TCL never patched it | LK hardened against the classic LK CVEs |

The KEY2 win is a bug in the UEFI ABL fastboot parser; the KEYone's LK shows
the hardened length/size checks that patched CVE-2013-2598 / CVE-2014-0973.
Even on the KEY2 the unlock is shallow — stock OS will not boot while unlocked
without a patched boot image.

Context: [`notes/09-keyone-recon-protection-and-key2-gap.md`](notes/09-keyone-recon-protection-and-key2-gap.md).

---

## Repository layout

- `docs/KEY2-LineageOS-Guide.md` — complete unlock → LineageOS 22.2 install guide.
- `notes/04-key2-lineageos-research.md` — live device recon, LineageOS status, risk assessment.
- `notes/07-key2-unlock-tool-re.md` — unlock tool reverse engineering (protocol, patch mechanism).
- `notes/09-keyone-recon-protection-and-key2-gap.md` — protection model and KEY2-vs-KEYone architectural gap.
- `recon/` — raw live captures: `getprop`, `by-name` partition map, unlock state, kernel parts, key flags.
- `exploit/key2-unlock/` — decompiled IL of the unlock tool + resource dump (binaries not redistributed).

## References

- **CVE-2021-1931** — Qualcomm fastboot/ABL buffer overflow (KEY2/KEY2 LE unlock) — Christopher Wade / Pen Test Partners.
- **kibo** — https://github.com/BotchedRPR/kibo
- **KEY2 LineageOS device trees** — FumoEnterprises (`android_device_blackberry_athena`, `android_device_blackberry_sdm660-common`, `android_kernel_blackberry_sdm660-4p19`).
- **Community wiki / ROM mirror** — luna-terra-cg.github.io/wiki / fumo.enterprises.
- **postmarketOS** — `blackberry-key2-generic` port (mainline/U-Boot).
