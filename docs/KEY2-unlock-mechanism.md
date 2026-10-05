# How the BlackBerry KEY2 Bootloader Unlock Actually Works

A complete, evidence-backed explanation of the **CVE-2021-1931** unlock on the
KEY2 (BBF100-6) — what the bug is, what the payload is, every byte that gets
patched, and why the device ends up in `FACTORY` mode.

Everything below was verified against the shipped artifacts:

- Official autoloader: `KEY2_ACQ160_all.7z` → `img/abl.elf`
- kibo payload: `ldr/acq160.exe` (GPL source: `BotchedRPR/kibo`)
- Windows tool: `BlackBerryBootUnlock.exe` (decompiled IL) + `data/160.bin`

## TL;DR

1. The ABL (Android BootLoader, a UEFI application) receives fastboot data
   into a fixed buffer **without validating the length** (CWE-120). Sending
   ~1.5 MB of data over the USB bulk endpoint overflows that buffer.
2. The overflow target is the ABL's own code in RAM. The attacker sends
   1 MB of zeros as padding, then a **byte-exact copy of the stock ABL PE
   image with 9 instructions patched** — so execution continues seamlessly,
   but now running attacker-modified code.
3. The patched code is only reached when the host sends `getvar:all`. The
   `bootmode` info handler then calls `switch_bootmode(1, 0)`, which persists
   boot mode **1 (FACTORY)** to RPMB and reboots to the bootloader.
4. Two supporting patches keep the device alive and unlocked:
   - **BbryWipeLib is defused** so the factory-wipe path does not fire.
   - The **"Boots Remaining" expiry counter is set to `0xffff`** so the
     factory mode never expires.
5. Result: `MODE: PRODUCT → FACTORY`, persisted across reboots. The device is
   unlocked. The official stock OS refuses to boot while unlocked, which is
   why a modified boot image is required afterwards.

## 1. The vulnerability — CVE-2021-1931

Qualcomm's advisory (July 2021 bulletin):

> *"Possible buffer overflow due to improper validation of buffer length while
> processing fast boot commands"* — CWE-120, fixed in SoCVersion 2021-07-05.

Christopher Wade (Pen Test Partners, "Breaking Mobile Bootloaders", QPSS
2022) found and exploited it:

- Target was a 2017 mid-range **SDM660** phone — the BlackBerry KEY2.
- The ABL is stored as an **ELF file in the `abl` partition that contains a
  UEFI firmware filesystem** (the actual executable is a PE/COFF UEFI
  application inside it).
- BlackBerry had **modified the fastboot `flash:` command** to allow flashing
  certain partitions while locked. That custom path is where the length
  validation is missing.
- Fuzzing via a reset command (opcode `0x13`) crashed the phone after ~130
  resets; crash analysis indicated a buffer overflow.
- The minimum size that produced the overflow was **`0x11bae0`** (~1.16 MB).
- Most 64-bit Snapdragon parts tested were vulnerable (SDM765 was not).
- To unlock, they generated a branch to **jump to the code after the RSA
  check** in the bootloader's unlock function.

TCL/BlackBerry never shipped the July 2021 Qualcomm fix to the KEY2 series,
so the bug is still live on ACQ160/ACT575 firmware.

## 2. The payload is the stock ABL — proven

The unlock tools don't ship a custom bootloader. They ship a **byte-exact copy
of the stock ABL PE image** from the official autoloader, extracted through
this container chain:

```
img/abl.elf (ELF32)
└── LOAD segment @ file 0x3000 (vaddr 0x9fa00000, 0x40000 bytes)
    └── UEFI Firmware Volume (FV, header @ 0x3000)
        └── FFS file type 0x0B "FV image" @ 0x3048 (0x37956 bytes)
            └── GUID_DEFINED section @ 0x3060
                GUID = ee4e5898-3914-4259-9d6e-dc7bd79403cf  (EFI LZMA)
                └── LZMA stream @ 0x3078
                    └── decompresses to a nested FV (827,592 bytes)
                        └── file "LinuxLoader" → PE32+ AArch64 image @ +0xB8
                            827,392 bytes (0xCA000)
```

That 827,392-byte PE is exactly:

| Artifact | SHA-256 |
|---|---|
| PE extracted from official `abl.elf` (LZMA → FV) | `2d270254389b00576d2430468fd79752020c45fc3f5dd0dd16eff9108fd37cce` |
| kibo `ldr/acq160.exe` | `2d270254389b00576d2430468fd79752020c45fc3f5dd0dd16eff9108fd37cce` |
| Windows tool `data/160.bin` | `2d270254389b00576d2430468fd79752020c45fc3f5dd0dd16eff9108fd37cce` |

**All three are byte-identical.** The FFS file is even named `LinuxLoader`
(UTF-16), matching kibo's internal name for the payload.

PE facts (imagebase 0, file offset == RVA, so patch offsets are directly
disassemblable):

```
machine 0xAA64 (AArch64)  entry 0x1000  sizeofimage 0xCA000
.text  VA=0x1000  size 0x95000
.data  VA=0x96000 size 0x33000
.reloc VA=0xc9000 size 0x1000
```

Because only the first `0x68C34` bytes are re-uploaded, and the maximum patch
site is `0x4180B`, every patched instruction is inside the sent region.

## 3. The attack sequence

Both tools use the same primitive over raw fastboot (USB bulk OUT EP `0x01`,
IN EP `0x81`) — no Google fastboot binary involved.

```
1. Identify bootchain
   kibo:          "oem info"  → Primary BC and Backup BC must match
   Windows tool:  "getvar:bb_bc_version" → ACQ160 / ACT575 / ACI448
   (payload selection is version-specific: acq160.exe / act575.exe / aca360.exe)

2. "reboot-bootloader"  → ABL restarts in fastboot

3. Overflow the fastboot buffer:
   write 0x100000 bytes of zeros        (padding to reach the ABL image)
   write 0x68C34 bytes of patched PE    (overwrites the running ABL image)

4. Drain the bulk-IN endpoint

5. "getvar:all"  → the bootmode info handler now executes the injected code:
      mov  w0, #1          ; mode = 1 (FACTORY)
      mov  w1, #0
      bl   switch_bootmode ; persist mode to RPMB
      mov  w0, #2
      b    0x42500         ; reboot to bootloader
```

The device reboots into the bootloader showing `MODE: FACTORY`.

The Windows tool sends a single `"0"` byte and a read before the overflow;
kibo does not. It is a tool quirk, not required by the exploit.

## 4. Every patch, decoded

The Windows tool applies **9 four-byte patches**. kibo applies the same core
set plus optional extras. Original vs. patched (AArch64):

### 4.1 Defuse BbryWipeLib (`defuse_wipelib`)

| Offset | Original | Patched | Meaning |
|---|---|---|---|
| `0x201c` | `bl #0x66d6c` | `mov x0, #0` | caller of FactoryWipe no longer calls it |
| `0x66d6c` | `sub sp, sp, #0x30` | `b #0x1000` | FactoryWipe entry jumps to PE entry (app restart) |
| `0x66d70` | `stp x20, x19, [sp, #0x10]` | `ret` | safety return if entered mid-way |

BbryWipeLib is BlackBerry's wipe path. Without defusing it, the unlock state
transition can trigger a factory wipe of userdata.

### 4.2 Allow more boots (`unlock` / `unlock_le`)

| Offset | Original | Patched | Meaning |
|---|---|---|---|
| `0x1f0c` | `csel x0, x8, xzr, eq` (`0x64`=100 or 0) | `mov w0, #0xffff` | boot/attempt counter maxed — "Boots Remaining: **Expiry Disabled**" |

This site lives inside `switch_bootmode` (`0x1e98`); the value is passed to a
counter helper (`0x21d8`). The ABL's `bootmode` output contains the strings
`Device Mode: %a`, `Boots Remaining: Expiry Disabled`, `bootmode from rpmb`.

### 4.3 The trigger (`unlock_le` block)

This block is inside the **`bootmode` info handler** (string
`bootmode info handler. status=%r`), which runs when `getvar:all` enumerates
`bootmode`:

| Offset | Original | Patched |
|---|---|---|
| `0x417f8` | `adrp x19, #0x90000` | `mov w0, #1` |
| `0x417fc` | `adrp x20, #0x6e000` | `mov w1, #0` |
| `0x41800` | `add x22, x8, #0x18` | `bl #0x1e98` (`switch_bootmode`) |
| `0x41804` | `add x19, x19, #0x43e` | `mov w0, #2` |
| `0x41808` | `add x20, x20, #0x2da` | `b #0x42500` (reboot) |

So the original code that builds an `INFO`-prefixed reply string is replaced
by `switch_bootmode(1,0)` + reboot.

### 4.4 kibo's additional ACQ160-path patches

kibo's non-LE table also patches (not present in the Windows tool, which uses
the LE-style trigger above):

| Offset | Original | Patched | kibo comment |
|---|---|---|---|
| `0x41be0` | `cbnz w8, #0x41bf8` | `b #0x41bf8` | disable "Flashing unlock is not allowed" |
| `0x1bc6c` | `cbz x19, #0x1bcfc` | `cbnz x19, #0x1bcfc` | disable BbryKeys (invert check) |
| `0x4d084` | `cmp w0, #9` | `mov w0, #1` | trigger at the flash-unlock handler |
| `0x4d088` | `b.hi #0x4d0a4` | `mov w1, #0` | |
| `0x4d08c` | `mov w8, #1` | `bl #0x1e98` (`switch_bootmode`) | |
| `0x4d090` | `mov w9, #0x3a0` | `mov w0, #2` | |
| `0x4d094` | `lsl w8, w8, w0` | `bl #0x42500` (reboot) | |

### 4.5 kibo's RTAS skip (`rtas` action)

| Offset | Original | Patched | Meaning |
|---|---|---|---|
| `0x21bfc` | `mov w21, w0` | `mov w21, #0` | zeroes the argument — bypasses an RTAS check |

This is the same RTAS/BTAS authorization ecosystem mapped on the KEYone
(`getvarp`, `oem securewipe`, `debugtokens` are BTAS-protected). kibo can
defuse the RTAS gate in RAM without touching flash.

## 5. Why this yields a persistent FACTORY unlock

- `switch_bootmode` validates the mode (`Invalid mode requested`) and writes
  the mode state via RPMB (`bootmode from rpmb`). **RPMB is persistent**, so
  the unlock survives reboot and reflash of the OS.
- Mode `1` is the factory/unlocked mode (`Device Mode` in `getvar bootmode`);
  the guide's observable result is `MODE: PRODUCT → FACTORY`.
- `0x42500` with `w0=2` reboots to the bootloader so the new mode is displayed
  immediately.
- The boot counter patch (`0xffff`) disables the expiry of factory mode.
- BbryWipeLib defusal prevents the transition from wiping userdata.

The patched ABL only lives in RAM — nothing is written to the `abl` partition.
The persistence comes from the RPMB boot-mode state the patched code writes.

## 6. Tool comparison

| | kibo (Linux, GPL) | BlackBerryBootUnlock.exe (Windows) |
|---|---|---|
| Version detection | `oem info` (primary + backup must match) | `getvar:bb_bc_version` |
| Overflow pad | 0x100000 zeros | 0x100000 zeros (after a 1-byte `"0"` write) |
| Payload sent | 0x68C38 bytes | 0x68C34 bytes |
| Unlock patches | defuse + 0x1f0c + full non-LE set, or LE set | defuse + 0x1f0c + LE-style trigger |
| Trigger | `getvar:all` | `getvar:all` |
| Extra actions | `lock`, `defuse`, `rtas`, `devinfo` | unlock only |

Both send the **same byte-identical stock payload** and the same core patches;
the Windows tool is essentially a subset of kibo's `unlock_le` path.

## 7. Reproduction constraints (why ACQ160 ×2)

- The patch offsets are valid **only** for the exact ABL build. A different
  bootchain version needs its own offsets — hence payload selection by
  version (ACQ160/ACT575/ACA360).
- The KEY2 is A/B: `abl_a` and `abl_b` are separate partitions. kibo refuses
  to run unless **primary and backup bootchain versions match**
  (`device_supports_kibo`). That is why the official procedure flashes the
  ACQ160 autoloader **twice** before unlocking — it guarantees both slots
  contain the same build.
- Do not run this on a device whose slots mismatch: the exploit may run
  against one slot and reboot into the other, with unpredictable results.

## 8. Evidence files

- `exploit/key2-unlock/il.txt` — full decompiled IL of the Windows tool
  (`UnlockBoot_Click`, patch table at IL_072e–IL_08ad, send sequence at
  IL_08d3–IL_0946, trigger at IL_09de).
- `exploit/key2-unlock/160.bin` / kibo `ldr/acq160.exe` — stock ACQ160 ABL PE.
- Official autoloader `img/abl.elf` — LZMA container holding the same PE.
- kibo source: `src/exploit.c` (`defuse_wipelib`, `unlock`, `unlock_le`,
  `lock`, `rtas`), `src/device.c` (`device_supports_kibo`).
