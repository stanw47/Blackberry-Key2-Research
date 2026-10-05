# KEY2 Bootloader Unlock Tool — Reverse Engineering

Source: `BlackBerryBootUnlock.exe` (v1.1), shipped in `key2-unlock-public.zip`.
Analyzed via `ildasm` + string/IL inspection. Dumps: `exploit/key2-unlock/`.

## What the tool is

- **.NET assembly** (managed, `mscorlib v4.0.0.0`) with a **C++/CLI interop layer** over
  `libusb-1.0.dll`. Not native — the whole thing is readable IL.
- Ships `libusb-1.0.dll`, `plist-2.0.dll`, and `data/160.bin` + `data/575.bin`.
- Built from `C:\Users\a\Downloads\BlackBerryBootUnlockNew\x64\Release\...pdb` (author path leaked).

## Protocol (the entire fastboot surface it uses)

```
getvar:bb_bc_version     <- identify BC (bootloader) version
getvar:all
getvar:bootmode
reboot-bootloader
```
That's it. It talks raw fastboot over **libusb bulk endpoints**, not Google fastboot.

## The mechanism (confirmed from IL)

`UnlockBoot_Click()`:
1. `init_bb_device(ctx, path)` — opens VID:PID **0FCA:8040** (same as KEY2 fastboot).
2. Reads `getvar:bb_bc_version` / `bootmode` to identify the device.
3. Detects firmware: string `ACQ160` -> `data/160.bin`, `ACT575` -> `data/575.bin`.
4. `LoadPEFile()` reads the chosen `.bin` into a buffer.
5. **`malloc(0x68C34)` = 429,108 bytes** — the payload size (matches FakeShell PoC).
6. **Patches a table of hardcoded byte offsets** in the buffer (e.g. `0x201f`, `0x201e`,
   `0x66d6f`, `0x417fb`, `0x1f0c` ... dozens of single-byte writes). This is the
   **signature-check patch** applied to a copy of the ABL image.
7. Sends the buffer over USB bulk transfer (the **CVE-2021-1931 overflow**: the ABL is
   executed from RAM, and the oversized payload overwrites the running bootloader with the
   patched copy — Wade's exact technique).
8. Waits 7 s, `libusb_close`, reconnects up to 5x, re-reads `bootmode`.
9. Success = `bootmode` changed **PRODUCT -> FACTORY**.

## Why this is KEY2/KEY2LE-only

- The payloads are **pre-patched copies of the KEY2 (`ACQ160`) and KEY2 LE (`ACT575`) ABL
  images** — `abl.elf` (UEFI/edk2), SDM660.
- The hardcoded patch offsets are valid **only** for those specific ABL binaries.
- The vulnerability is in the **UEFI ABL fastboot parser** (CVE-2021-1931).

## Applicability to the KEYONE (BBB100-3)

**None directly.** The KeyOne:
- Uses **LittleKernel** (`emmc_appsboot.mbn`), NOT a UEFI ABL.
- Has **no `abl.elf`**, so there is no same-shaped image to patch.
- Is **not** in CVE-2021-1931's affected set.
- Its bootloader has no `getvar:bb_bc_version` "ACQ160/ACT575" model — it uses `ABLxxx` builds.

The tool's *shape* (raw fastboot over libusb + a patched-bootloader-in-RAM overflow) is a
general technique, but the KeyOne would need a **different vulnerability in the LK
`image_verify`/`boot_linux_from_mmc` path** and its own offsets. The candidates there remain
**CVE-2013-2598** and **CVE-2014-0973**.

## Value

- Confirms the exact unlock primitive (RAM-resident ABL overwrite) so it can be
  re-implemented/understood, e.g. with `tools/fastboot_libusb.py` + our own patch table.
- Documents the fastboot surface BlackBerry's ABL exposes (`bb_bc_version`).
- Provides a template to attempt an analogous overflow on the KeyOne LK — if a matching bug
  exists.
