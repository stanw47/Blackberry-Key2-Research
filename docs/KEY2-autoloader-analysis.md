# BlackBerry KEY2 ACQ160 Autoloader — Structure and Flash Flow

Analysis of the official **KEY2 ACQ160** factory package
(`KEY2_ACQ160_all.7z`): what's inside, how it flashes the device, the
BlackBerry auth/signature stack it uses, and how it relates to the unlock
payload documented in [`KEY2-unlock-mechanism.md`](KEY2-unlock-mechanism.md).

## 1. Provenance

| Field | Value |
|---|---|
| Package | `KEY2_ACQ160_all.7z` (2.6 GiB compressed, ~3.1 GB) |
| Downloaded by | `_dl_acq160.bat` via `megatools` from a MEGA share link |
| Package file dates | 2020-08-07 (`flashall.bat` 2019-04-13; images 2020-05/06) |
| Type | BlackBerry "all" factory image package (host tools + images + scripts) |
| Bootchain version | **ACQ160** (metadata in `abl.elf` trailer) |
| Product | APBC (autoloader) / APBI (HLOS signatures) |

Unlike the KEYone (which shipped a self-contained autoloader `.exe` with a
Lua-driven flash script), the KEY2 "all" package is a plain 7z containing
host binaries and `flashall.bat`. There is no single autoloader executable.

## 2. Package contents

```
flashall.bat / flashallnowipe.bat      flash scripts (see §3)
host/
  windows-x86/bin/  fastboot.exe, authboot.exe, pcauthtool.exe, adb.exe,
                    libusbauth.dll, libxpmux.dll, libcurl.dll, AdbWinApi.dll
  linux-x86/bin/    fastboot, authboot, pcauthtool, adb
  darwin-x86/bin/   fastboot, authboot, adb, libxpmux.dylib
  */lre/            Lua 5.2 runtime + luac (unused by flashall.bat)
img/
  abl.elf            UEFI ABL (LZMA-packed, see §7)
  xbl.elf, tz.mbn, rpm.mbn, hyp.signed.mbn, pmic.elf,
  devcfg.mbn / devcfg_cn.mbn, cmnlib(.64).signed.mbn,
  keymaster64.signed.mbn, mdtpsecapp.signed.mbn
  boot.img, recovery.img, cache.img, userdata.img,
  system.img, vendor.img, oem_{common,china,eea,russia}.img,
  NON-HLOS-{global,americas,cn,dsglobal,dsjapan,japan}.bin,
  dspso.bin, BTFM.bin
  boot.img*.sig, recovery.img*.sig      HLOS signature blobs (see §6)
```

## 3. `flashall.bat` — the official flash flow

```
1.  getvar device / variant / subvariant
    → picks NON-HLOS-<variant>.bin and oem_<subvariant>.img
2.  oem securewipe                       (wipes userdata + resets password)
3.  getvar bootchain-slots               → determines active slot
4.  Flash boot chain to the ACTIVE slot:
      tz_<slot>, devcfg_<slot>, rpm_<slot>, xbl_<slot>, hyp_<slot>,
      pmic_<slot>, abl_<slot>, cmnlib_<slot>, cmnlib64_<slot>,
      keymaster_<slot>, mdtpsecapp_<slot>
5.  oem switch-bootchain:<slot>          (make the flashed slot active)
6.  reboot bootloader
7.  flash bootsig  boot.img<variant>.sig      ← signature first
    flash recoverysig recovery.img<variant>.sig
    flash boot / recovery / cache / userdata / modem / dsp / bluetooth /
          vendor / system / oem
8.  reboot
```

Why the guide says to run it **twice**: the KEY2 is A/B and the script flashes
only the *active* slot, then switches. A second run flashes the other slot,
leaving both slots on the same build — which is exactly what kibo's
`device_supports_kibo` check requires before unlocking.

Notable: `oem securewipe` and `oem switch-bootchain:` are invoked by the
official script directly. On the KEYone we confirmed `oem securewipe` is not
RTAS-gated; here it is part of BlackBerry's own factory flow.

## 4. Custom fastboot surface

The bundled `fastboot.exe` is a BlackBerry fork, not Google's. Its strings
expose custom commands and internal functions:

- Custom variables used by the scripts: `device`, `variant`, `subvariant`,
  `bootchain-slots`, `bb_bc_version`.
- `oem switch-bootchain:<slot>`, `oem securewipe`, `oem debugtokens`,
  `getvarp [bsn|pin|procid|authbootlog]`, `oem enable-usb-reset`,
  `oem enable-usb-shutdown`, `flash partition <filename>` (User GPT).
- Internal helpers: `do_send_signature`, `do_update_signature`,
  **`do_bypass_unlock_command`**, plus standard `flashing unlock*` support.

`authboot.exe` documents the "BTAS protected" extended command set:

```
authboot extended commands (BTAS protected):
  oem debugtokens                          Uploads all whitelisted debug tokens to device.
  getvarp [bsn|pin|procid|authbootlog]     Display a RTAS protected bootloader variable.
  oem securewipe                           Secure erase all user data and reset device password.
  oem enable-usb-reset                     Trigger a device reset upon next USB cable removal.
  oem enable-usb-shutdown                  Trigger a device shutdown upon next USB cable removal.
  flash partition <filename>               Flash the User partition GPT.
```

It also contains debug-token server logic (`flash:debug_token`,
`erase:all_debug_tokens`, "Error in retrieving tokens from the server") — the
same token ecosystem documented for the KEYone.

## 5. The auth stack (BTAS / RTAS)

| Component | Role |
|---|---|
| `libusbauth.dll` | WinUSB transport for the BlackBerry USB auth interface (`GUID_CLASS_BB_USBAUTH`); raw pipe read/write, descriptor queries |
| `pcauthtool` | PC-side RTAS client: `DEV_HANDSHAKE_PC`, `DEV_PASSWORD_PC`, RTAS challenge (`BBAuthDecryptRtasBlob2`), `BBAuthVerifyCred`, `calculate_password_hash[_with_entropy]`, `hu_SHA512Begin`, `hu_DESKeySet`; takes `-p <device_pass>` |
| `authboot` | BlackBerry fastboot fork with the BTAS-protected commands above |

This is the **same RTAS2/BTAS system** reverse-engineered on the KEYone
(RTAS challenge/response, protected `getvarp`/`securewipe`, debug tokens).
On the KEY2 the official flash path ships the client end of it, which is a
useful reference implementation for the KEYone RTAS work.

## 6. Image signatures — BBRYBlob

`abl.elf` carries a **293-byte signed trailer**:

```
BBRYBlob v1  type 2
├── "Info" v1 (0x43 bytes)
│     field 7 = "ACQ160"          (bootchain version string)
│     field 4 = 32-byte digest    (f618f879…)
└── "Sig!" v1 (0xB2 = 178 bytes)
      "ec_agent"                  (builder)
      "20200526.064843"           (build timestamp)
      "APBC"                      (product)
      digest + 64-byte signature
```

HLOS signature files (`boot.img.sig`, `recovery.img*.sig`, 208 bytes) share
the same field layout: 4-byte type, builder (`ec_agent`), timestamp,
product (`APBI`), zero padding, two 32-byte digests, and a **64-byte
signature** — consistent with **ECDSA P-256** (`r || s`), matching the
BlackBerry ECDSA P-256 boot chain used across BB10/Android devices.

The flash flow delivers the signature first (`fastboot flash bootsig
boot.img.sig`), then the image; the ABL verifies the image against the
signature before accepting it.

## 7. The ABL container chain

`img/abl.elf` is an ELF32 whose LOAD segment contains a UEFI firmware volume.
The actual ABL application is LZMA-compressed inside a nested volume:

```
abl.elf → FV @0x3000 → FFS type 0x0B (FV image) @0x3048
        → GUID_DEFINED section (GUID ee4e5898… = EFI LZMA) @0x3060
        → LZMA @0x3078 → nested FV (827,592 bytes)
        → "LinuxLoader" PE32+ AArch64 @+0xB8 (827,392 bytes)
```

That PE is the exact image used by both unlock tools (see
[`KEY2-unlock-mechanism.md`](KEY2-unlock-mechanism.md) §2 for the SHA-256
proof). Extracting it required only the standard FV/FFS/GUID-section walk plus
`lzma` decompression — no vendor tooling.

## 8. Observations

- The official autoloader **contains the exact payload** the unlock exploits
  reuse. The unlock is not a custom bootloader; it is the stock ABL patched
  in RAM.
- The package's Lua runtime (`lre/`) is unused by `flashall.bat`; the KEYone
  autoloader used Lua scripts (`aveflash.lua`) to drive its flash. This
  package looks like the "raw" SFI release that a Lua-driven wrapper would
  normally consume.
- `oem securewipe` + `oem switch-bootchain` are factory-script commands; on a
  locked device the bundled fastboot binary performs the auth handshake via
  `libusbauth`.
- Signature verification is per-image (`bootsig`/`recoverysig`), separate from
  the `flash` operation — the same split used by the KEYone HLOS token flow.
- All signature metadata is plaintext (builder, timestamp, product); only the
  final 64 bytes are cryptographic. The format is a useful reference for
  analyzing other BlackBerry firmware packages (Priv, KEYone, KEY2 LE).
