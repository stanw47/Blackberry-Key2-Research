# KEY2 Unlock Tool — Reverse Engineering

Complete reverse engineering of **`BlackBerryBootUnlock.exe` V1.1**, the
Windows tool that unlocks the BlackBerry KEY2 (BBF100-6). This documents the
tool itself — UI, USB layer, protocol, payload handling, patch tables, and
quirks. For the exploit theory and instruction-level semantics, see
[`KEY2-unlock-mechanism.md`](KEY2-unlock-mechanism.md).

## 1. Artifact and provenance

| Property | Value |
|---|---|
| File | `BlackBerryBootUnlock.exe` (V1.1) |
| SHA-256 | `374293c54518406b4145a2bc7a1a9c91b3e39109fdf629d059d1ed1b73c455f4` |
| Type | .NET assembly (managed IL) with a **C++/CLI interop layer** over `libusb-1.0.dll` |
| Runtime | `mscorlib v4.0.0.0`, WinForms |
| Analysis method | `ildasm` decompilation → `exploit/key2-unlock/il.txt` (26,718 lines) |
| Leaked build path | `C:\Users\a\Downloads\BlackBerryBootUnlockNew\x64\Release\BlackBerryBootUnlock.pdb` |
| Embedded manifest | `exploit/key2-unlock/il.res` — `requestedExecutionLevel level='asInvoker'` (no elevation) |

The whole tool is readable IL; nothing is native except the P/Invoke calls to
libusb.

### Distribution files

| File | SHA-256 / notes |
|---|---|
| `BlackBerryBootUnlock.exe` | `374293c5…` |
| `data/160.bin` (ACQ160 ABL PE) | `2d270254389b00576d2430468fd79752020c45fc3f5dd0dd16eff9108fd37cce` |
| `data/575.bin` (ACT575 ABL PE) | `ef9b30171df44710c5325cc7d5b5226a6f15cfe334fbde308b23e03616687a18` |
| `libusb-1.0.dll` | USB transport |
| `plist-2.0.dll` | bundle metadata (Apple libplist build) |

`160.bin` and `575.bin` are both exactly `0xCA000` (827,392) bytes and are
**byte-exact copies of the stock ABL PE images** from the official
ACQ160/ACT575 autoloaders (see mechanism doc §2 for the extraction proof).
They differ in 137,305 bytes from each other and contain the version string
`ACQ160` / `ACT575` respectively.

## 2. UI and action surface

`MyForm` (WinForms) contains a device combo box, a progress bar, a log box,
and five buttons:

| Method | Action |
|---|---|
| `ScanPort_Click` | enumerate USB, fill combo box |
| `UnlockBoot_Click` | unlock (boot mode → FACTORY) |
| `RelockBoot_Click` | relock (boot mode → PRODUCT) |
| `checkBootLoader_Click` | query `getvar:bootmode`, show locked/unlocked |
| `Exit_Click` | exit |

Combo box entries are built as **`"fastboot - <serial>"`**; each action takes
the serial as the substring after the last `-`.

Other methods: `LoadPEFile` (payload loader), `InsertText` (log helper),
`UpdateProgressBar`, `InitializeComponent`.

## 3. USB transport layer

`init_bb_device(ctx, serial)` — IL lines 5876–6054:

1. `libusb_get_device_list`.
2. For each device: `libusb_get_device_descriptor`; match
   **VID `0x0FCA`, PID `0x8040`** (fastboot).
3. `libusb_open`.
4. Read the serial string descriptor (index `iSerialNumber`) into a 256-byte
   buffer.
5. If a serial filter was passed, byte-compare it against the descriptor;
   on mismatch close and continue scanning.
6. `libusb_claim_interface(handle, 0)` — interface 0 is claimed.
7. Returns the handle, or NULL.

Endpoints are **hardcoded, not discovered**:

| Helper | Endpoint | Direction | IL |
|---|---|---|---|
| `write_to_adu` | `0x01` | bulk OUT | 5784–5874 |
| `read_bulk` | `0x81` | bulk IN | 5696–5782 |

`write_to_adu` sends raw bytes (`libusb_bulk_transfer`); despite the name there
is no framing — "ADU" is just the author's label. Responses are read as raw
bytes and interpreted as fastboot text (`OKAY…` / `INFO…` / `FAIL…`).

## 4. Scan and status check

### ScanPort_Click (IL 13290–13538)

- Clears the UI, `libusb_init`, `init_bb_device(ctx, NULL)` (no serial filter).
- On success: reads the serial descriptor, adds `"fastboot - <serial>"` to the
  combo box, logs `-Device Found, Fastboot mode!`.
- On failure: logs `-BlackBerry device not found in fastboot mode` and prints
  manual instructions (power off, hold Vol-Down while connecting Type-C, wait
  for the BlackBerry logo, release).
- No device commands are sent during scan.

### checkBootLoader_Click (IL 16507–17036)

- Takes the serial from the selected combo entry, opens the device.
- Sends **`getvar:bootmode`** (15 bytes, timeout 1000 ms).
- Reads up to `0x5000` bytes with **3 attempts**, 500 ms apart.
- Parses the response (`ParseBootMode`) and requires exactly
  `OKAYPRODUCT_MODE` or `OKAYFACTORY_MODE`:
  - `PRODUCT_MODE` → "Device is already Locked"
  - `FACTORY_MODE` → "Device is already Unlocked"
  - anything else → "Unsupported boot mode"
- Displays `-Current boot mode: <mode>`.

## 5. Unlock operation — exact sequence

From `UnlockBoot_Click` (IL 13702–15097). All values are literal from the IL.

```
 1. getvar:bb_bc_version          write 100 ms → read 0x2800 @ 100 ms
    response must start with "OKAY"; version = Substring(4).Trim()
      ACT575  → payload "575"
      ACQ160  → payload "160"
      ACI448  → payload "160"
      else    → "Unsupported firmware version" (abort)

 2. reboot-bootloader             write 1000 ms → read 0x100 @ 1000 ms

 3. reconnect loop: up to 5 attempts, 2000 ms sleep between;
    re-init via init_bb_device(ctx, serial)
    success → "-Reconnected: Successfully"

 4. LoadPEFile("<payload>"):  data\<payload>.bin
      - 'data' folder must exist
      - file must be EXACTLY 0xCA000 bytes ("Invalid PE size!" otherwise)
      - reads the full 0xCA000

 5. malloc(0x68C34) + zero; copy first 0x68C34 bytes of the PE
 6. apply 9 dword patches (see §6)
 7. malloc(0x100000) + zero   (overflow padding)
 8. sleep 500 ms

 9. write "0"                    1 byte, 100 ms      ← tool quirk
10. sleep 100 ms
11. read 0x2800 @ 300 ms          (response ignored)
12. write 0x100000 zeros         @ 3000 ms           (fill/overflow)
13. write 0x68C34 patched PE     @ 3000 ms           (overwrite ABL image)
14. sleep 500 ms; free both buffers
15. read 0x2800 @ 300 ms          (response ignored)
16. "-Preparation: Sucess"

17. getvar:all                   write 3000 ms       ← TRIGGER
18. "-Bootloader Unlock: Successfully!"
```

### Notes on the sequence

- The 1 MB zero write + 429,108-byte payload is the CVE-2021-1931 overflow;
  see the mechanism doc §3 for why the payload lands on the ABL image.
- The `"0"` write (step 9) is **not present in kibo** and is not required by
  the exploit; the tool also ignores both post-overflow responses.
- The tool only checks the **primary** `bb_bc_version` — it never verifies the
  backup bootchain like kibo does (`device_supports_kibo`). On a mismatched
  A/B device this is a real risk.
- Error paths are user-friendly strings: `-Failed to send command`,
  `1.Please restart your device in fastboot manually`, `2.And try again`,
  `-Invalid device response format`, `-Failed to reconnect to device after
  reboot`, etc.

## 6. Patch tables

The tool patches the payload with single-byte writes in this order (IL
072e–08ad). Both operations patch the same nine dwords; only `0x417f8`
differs.

| Offset | Original instruction | Patched word | Patched instruction | Purpose |
|---|---|---|---|---|
| `0x201c` | `bl #0x66d6c` | `0xd2800000` | `mov x0, #0` | don't call FactoryWipe |
| `0x1f0c` | `csel x0, x8, xzr, eq` | `0x529fffe0` | `mov w0, #0xffff` | max "Boots Remaining" (expiry disabled) |
| `0x66d6c` | `sub sp, sp, #0x30` | `0x17fe68a5` | `b #0x1000` | FactoryWipe entry → PE entry |
| `0x66d70` | `stp x20, x19, [sp, #0x10]` | `0xd65f03c0` | `ret` | safe return if called |
| `0x417f8` | `adrp x19, #0x90000` | unlock: `0x52800020` / relock: `0x52800040` | `mov w0, #1` / `mov w0, #2` | boot mode: FACTORY / PRODUCT |
| `0x417fc` | `adrp x20, #0x6e000` | `0x52800001` | `mov w1, #0` | second arg |
| `0x41800` | `add x22, x8, #0x18` | `0x97ff01a6` | `bl #0x1e98` | `switch_bootmode` |
| `0x41804` | `add x19, x19, #0x43e` | `0x52800040` | `mov w0, #2` | reboot mode arg |
| `0x41808` | `add x20, x20, #0x2da` | `0x1400033e` | `b #0x42500` | reboot to bootloader |

This is exactly kibo's `defuse_wipelib` + `unlock`/`lock` (LE-style trigger).
kibo additionally has `lock`, `rtas`, and the non-LE ACQ160 trigger set
(`0x41be0`, `0x1bc6c`, `0x4d084…`), which this tool does not implement.

## 7. Relock operation

`RelockBoot_Click` (IL 15100–16495) is a copy of the unlock flow with a single
difference: `0x417f8` is patched to `mov w0, #2`, so `switch_bootmode(2, 0)`
sets boot mode **2 (PRODUCT / locked)**. It still logs the copy-pasted string
`-Sending unlock command...` and triggers with `getvar:all`.

## 8. Windows tool vs kibo

| | Windows tool | kibo |
|---|---|---|
| Transport | libusb-1.0, EP 0x01/0x81, interface 0 | same |
| ID | VID 0x0FCA PID 0x8040, serial match | same |
| Version check | `getvar:bb_bc_version` (primary only) | `oem info` (primary **and** backup must match) |
| Payload | stock PE, exact size enforced | stock PE |
| Overflow | 1 MB zeros + 0x68C34 | 1 MB zeros + 0x68C38 |
| Extra `"0"` write | yes | no |
| Patches | defuse + boots + LE trigger | full set incl. non-LE, RTAS, lock/defuse actions |
| Trigger | `getvar:all` | `getvar:all` |
| Actions | Unlock, Relock, Check status | unlock, lock, defuse, rtas, devinfo |

## 9. Artifacts

- `exploit/key2-unlock/il.txt` — full IL decompilation.
- `exploit/key2-unlock/il.res` — embedded application manifest.
- `exploit/key2-unlock/160.bin`, `575.bin` — stock ABL PE payloads.
- Official autoloader `img/abl.elf` — source of the same PE (see
  [`KEY2-autoloader-analysis.md`](KEY2-autoloader-analysis.md)).
- kibo source — https://github.com/BotchedRPR/kibo (GPL-2.0).
