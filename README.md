# BlackBerry KEY2 (BBF100-6) — Research

> Reverse-engineering, unlock, and LineageOS research for the **BlackBerry KEY2**
> (BBF100-6, India/APAC dual-SIM) — the UEFI/ABL sibling of the KEYone, and the
> device where the unpatched **CVE-2021-1931** fastboot overflow still yields a
> bootloader unlock.
>
> Part of the **[Blackberry-Research](https://github.com/stanw47/Blackberry-Research)**
> collection · [Williamson Security Solutions](https://williamsonsecuritysolutions.com)

---

## Disclaimer

> **Research aid, not a flashing guide.** Unlocking a bootloader and flashing
> firmware can **permanently brick** a device, and unlocks wipe userdata.
> Third-party tool binaries are **not redistributed** — hashes/provenance are
> recorded instead. For educational / defensive research on a device the author
> owns. **At your own risk.**

---

## Device Details

| Field | Value |
|---|---|
| Model | BlackBerry KEY2 **BBF100-6** |
| Codename | `bbf100` / athena |
| SoC | Qualcomm **SDM660** (Snapdragon 660) |
| OS / software | Android 8.1 (stock) to **LineageOS 22.2** |
| Current build | LineageOS 22.2 (Android 15), kernel 4.4.302 |
| Previous builds | stock **ABN088** (Android 8.1.0, kernel 4.4.78), patch 2018-12-01, **PRD-63828-031** |
| Carrier / unlock | **India/APAC variant; dual-SIM**; `ro.oem_unlock_supported=true`; **UNLOCKED** |
| SIM | **dual-SIM** |

### A/B partition layout

Unlike the KEYone (A-only), the KEY2 is A/B and the bootloader is slotted:

```
abl_a -> p37   abl_b -> p49   xbl_a -> p27   xbl_b -> p39   tz_a -> p28   tz_b -> p40
boot -> p23 (not slotted)   recovery -> p57   recoverysig -> p58
vbmeta -> p19   system -> p73   vendor -> p74   oem -> p75
userdata -> p77   misc -> p54   frp -> p56   nvuser -> p67   perm -> p66
```

BlackBerry's A/B implementation is inconsistent enough that the unlock requires
flashing the stock autoloader **twice** (both slots). Raw capture:
[`recon/key2_byname.txt`](recon/key2_byname.txt).

---

## Current Status

The KEY2 is **unlocked and running LineageOS 22.2 (Android 15)** with dual-SIM
working. Unlock was achieved via the unpatched Qualcomm **CVE-2021-1931** in the
UEFI ABL fastboot parser (the `kibo` payload), which flips the device from
`PRODUCT` to `FACTORY` mode. The remaining work is ROM polish/stability.

---

## Completed

- **Bootloader unlock** via CVE-2021-1931 (`kibo` / `BlackBerryBootUnlock.exe`).
- **Full unlock-tool RE** — .NET CLI tool over libusb; exact command sequence,
  patch table, and payload provenance.
- **Autoloader analysis** — ACQ160 package, A/B slot flow, BBRYBlob signatures.
- **LineageOS port** — community build (kernel 4.4), dual-SIM working.

## Achieved

- **Boot mode `PRODUCT FACTORY` persisted to RPMB** — a permanent,
  reboot-surviving unlock.
- **Running LineageOS 22.2 (Android 15)** on retail hardware.
- **Payload provenance proven by SHA-256** — autoloader extraction == kibo
  `acq160.exe` == Windows tool `160.bin`.

## In Progress

- **ROM stability:** SELinux + encryption completeness, keyboard-touchpad jitter
  (disable via Quick Settings), some Play-Integrity-sensitive apps.

## Failed

- **Stock OS while unlocked** — BlackBerry's stock build refuses to boot in
  FACTORY mode, so a **modified boot image is required** after unlock.
- **Official LineageOS** — never happened (no clean GPL kernel source for the
  KEY2); only unofficial/community builds exist.

## Future Plans

1. Polish the LineageOS 22.2 port (stability/encryption).
2. Track newer community Android builds.

---

## Community Activity

**The most successful device in this collection.**

- **Christopher Wade (Pen Test Partners)** found and exploited **CVE-2021-1931**.
- **BotchedRPR (Igor)** authored **kibo** and maintains the LineageOS 22.2 builds
  for KEY2 (athena) and KEY2 LE (luna); **krab-ubica** led the bootloader-unlock
  research; **npjohnson** contributed Nokia SDM660 blobs.
- **FumoEnterprises** hosts recovery/ROM images; **tim-ecoder** maintains
  `lineageos_blackberry_athena`; **ronnz98** publishes /e/OS builds.
- **FakeShell/CVE-2021-1931-BBRY-KEY2** is a public proof-of-concept.
- **postmarketOS** has a `blackberry-key2-generic` port (mainline/U-Boot).
- Known issues on 22.2: SELinux and encryption disabled, keyboard-touchpad
  jitter, some camera/flash quirks.
- Hubs: XDA, Reddit r/blackberry, the BlackBerry Android Hideout Discord, and the
  community wiki.

## Unlock deep-dive (CVE-2021-1931)

### The vulnerability

Qualcomm advisory (July 2021): *"Possible buffer overflow due to improper
validation of buffer length while processing fast boot commands"* — CWE-120,
fixed in SoCVersion 2021-07-05. Christopher Wade's *"Breaking Mobile Bootloaders"*
(QPSS 2022) found and exploited it on an **SDM660** BlackBerry (the KEY2):

- The ABL (Android BootLoader, a UEFI application) receives fastboot data into a
  fixed buffer **without validating the length**. Sending ~1.5 MB over the USB
  bulk endpoint overflows it.
- BlackBerry had **modified the `flash:` command** to allow flashing certain
  partitions while locked — that custom path is where validation is missing.
- The overflow target is the ABL's own code in RAM.

### The payload

1. Send ~1 MB of zeros as padding, then a **byte-exact copy of the stock ACQ160
   ABL PE** (extracted from the official autoloader, LZMA-inside-`abl.elf`) with
   **9 instructions patched** — execution continues seamlessly but now runs
   attacker-modified code.
2. The patches: **defuse BbryWipeLib**, set the **"Boots Remaining" expiry
   counter to `0xffff`** so factory mode never expires, and replace the
   `bootmode` getvar handler with a call to `switch_bootmode(1, 0)` + reboot.
3. Sending **`getvar:all`** triggers the patched handler: boot mode **1
   (FACTORY)** is persisted to **RPMB**, then the device reboots — unlocked,
   across reboots.
4. Result: `MODE: PRODUCT FACTORY`. Stock OS refuses to boot while unlocked ->
   flash the modded `acq160-mfi-boot.img`, then recovery + ROM.

### Tool reverse engineering

The Windows tool is a .NET/C++-CLI assembly over `libusb-1.0.dll`; its fastboot
surface is `getvar:bb_bc_version`, `getvar:all`, `getvar:bootmode`,
`reboot-bootloader`. It selects a payload by bootloader version (`ACQ160` ->
`160.bin`, `ACT575` `575.bin`), patches a hardcoded table of byte offsets (the
signature-check patch), then sends the 429,108-byte buffer — the overflow.
Success = `bootmode` changes `PRODUCT FACTORY`. Progress sticks at 75% (normal).

Full analysis: [`notes/07-key2-unlock-tool-re.md`](notes/07-key2-unlock-tool-re.md);
decompiled IL: [`exploit/key2-unlock/il.txt`](exploit/key2-unlock/il.txt).

### Tool binary provenance (not redistributed)

| File | SHA-256 |
|---|---|
| `BlackBerryBootUnlock.exe` (v1.1) | `374293c54518406b4145a2bc7a1a9c91b3e39109fdf629d059d1ed1b73c455f4` |
| `data/160.bin` (ACQ160 ABL payload) | `2d270254389b00576d2430468fd79752020c45fc3f5dd0dd16eff9108fd37cce` |
| `data/575.bin` (ACT575 ABL payload) | `ef9b30171df44710c5325cc7d5b5226a6f15cfe334fbde308b23e03616687a18` |

### Procedure (summary)

1. Remove all Google accounts (they block the fastboot endpoint).
2. Flash stock **ACQ160** autoloader **twice** (both A/B slots).
3. Unlock: `kibo unlock` (Linux) or the Windows tool (progress sticks at 75%).
4. `MODE:` on the bootloader screen changes **PRODUCT FACTORY**.
5. Flash the modded `acq160-mfi-boot.img`, then recovery + ROM.
6. Unlock + custom recovery **wipes userdata** — back up first.

### LineageOS

Unofficial/community (no official support). Recommended dual-SIM build:
**LineageOS 22.2 + kernel 4.4** (`ZKrab-v1.10a`). Step-by-step:
[`docs/KEY2-LineageOS-Guide.md`](docs/KEY2-LineageOS-Guide.md). Known issues on
22.2: SELinux/encryption completeness, keyboard-touchpad jitter, some
Play-Integrity apps.

### Why the KEY2 fell and the KEYone did not

| | KEY2 (cracked) | KEYone (not) |
|---|---|---|
| SoC | SDM660 | MSM8953 |
| Bootloader | **UEFI ABL** (`abl.elf`, edk2) | **LittleKernel** (`emmc_appsboot.mbn`) |
| Exploit | **CVE-2021-1931** — ABL fastboot parser overflow | n/a — different codebase |
| Fix status | TCL never patched it | LK hardened against the classic LK CVEs |

The KEY2 win is a bug in the UEFI ABL fastboot parser; the KEYone's LK shows the
hardened length/size checks that patched CVE-2013-2598 / CVE-2014-0973. Even on
the KEY2 the unlock is shallow — stock OS won't boot while unlocked without a
patched boot image.

---

## Repository layout

- `docs/KEY2-unlock-mechanism.md` — full unlock explanation (CVE, payload
  provenance, patched instructions, why FACTORY persists).
- `docs/KEY2-unlock-tool-re.md` — complete RE of the Windows unlock tool.
- `docs/KEY2-autoloader-analysis.md` — official ACQ160 package analysis.
- `docs/KEY2-LineageOS-Guide.md` — complete install guide.
- `notes/04-key2-lineageos-research.md`, `notes/07-key2-unlock-tool-re.md` — live recon + tool RE.
- `recon/` — raw live captures (`getprop`, `by-name`, unlock state, kernel parts).
- `exploit/key2-unlock/` — decompiled IL + resource dump (binaries not redistributed).
- `devmaps/key2-bbf100-6.json` — live device map (L0–L2).

---

## Related repos

- **Hub:** [Blackberry-Research](https://github.com/stanw47/Blackberry-Research) —
  cross-device mechanisms + the `devmap` framework (`toolchain/devmap.py`).
- **KEYone:** [Blackberry-KeyOne-Research](https://github.com/stanw47/Blackberry-KeyOne-Research) —
  the locked sibling (LK bootloader + KGSL/IOMMU kernel research).

---

## Citations & Acknowledgements

| Source | URL | Relevance |
|---|---|---|
| CVE-2021-1931 | Qualcomm | fastboot/ABL buffer overflow |
| Christopher Wade — Pen Test Partners | QPSS 2022 talk | found/exploited CVE-2021-1931 |
| BotchedRPR / kibo | https://github.com/BotchedRPR/kibo | unlock payload/tool |
| FumoEnterprises | LineageOS device trees | athena / sdm660-common / kernel |
| krab-ubica, npjohnson | community | unlock + ROM maintenance |
| postmarketOS | https://postmarketos.org | `blackberry-key2-generic` port |

Thanks to the XDA / Reddit / postmarketOS communities.

---

## License

Research notes and original scripts are provided for educational purposes;
third-party code retains its own license.
