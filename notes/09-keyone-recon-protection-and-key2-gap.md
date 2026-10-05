# KEYone Deep Recon, Protection Model & KEY2-vs-KeyOne Analysis

Date: 2026-10-02  |  Device: BBB100-3 Sprint, ABL766, Android 7.1.1
Live: booted to Android, USB debugging on, adb shell uid=2000(shell), no root.

---

## 1. The protection model (answering "what can/can't we write")

Boot chain: **PBL (ROM) → SBL1 → ABOOT (LK) → boot.img (kernel)**. Each stage verifies
the *next* stage's signature. Partition write-ability is NOT the wall — **signature
verification** is.

| Partition | fastboot `erase`/`flash` | Verified by | Protected because |
|---|---|---|---|
| `sbl1` | **UNGATED** (works) | PBL (fused key) | signature |
| `aboot` | **UNGATED** (works) | SBL1 (`boot_authenticator.c`, SHA256) | signature |
| `boot`/`recovery` | **UNGATED** | aboot (LK `image_verify`) | signature |
| `modem`,`system`,`cache`,`userdata` | **UNGATED** | (system: dm-verity) | — |
| `bootsig`/`recoverysig` | gated | — | authboot |
| `misc`,`config`,`frp`,`devinfo`,`persist`,`abootbak`,`param`,`keystore` | **GATED** (`authboot erase permission denied`) | — | authboot |

**So the bootloader partitions ARE writable — but only with signed images.** The daemon
string in SBL1 `bbry_is_insecure: TRUE` and `bbss_insecure` is the *switch* that tells SBL1
to skip `boot_authenticator.c`. That switch is stored in the **write-protected** boot0
(`bbss_wp_type = permanent`), which is the actual chokepoint.

### What we still cannot do
- **Flip `bbss.insecure`** (boot0 is permanent HW WP — `B_PERM_WP_EN`). This is the *one*
  byte that would make SBL1 skip verification.
- **Get a modified aboot accepted** — SBL1's `boot_authenticator.c` checks SHA256 of ELF
  segments against the fused OEM key (`auth_hash_seg_*`, `elf_segs_hash_verify_entry`).
- **Disable dm-verity** — `/system` is `dm-0` verity, `ro`.
- **Root** — no `su`, `adbd cannot run as root in production builds`, SELinux enforcing.

### The genuine flaw found this session
`aboot`/`sbl1` are **erasable/writable from locked fastboot** (not authboot-gated) while
their backups (`abootbak`) ARE gated. That's a whitelist inconsistency — but it only lets
you *overwrite with signed* images. It does NOT bypass signature verification. (Cost a scare;
fully recovered via the ABL766 autoloader `flashall`.)

---

## 2. Live recon summary

- Kernel `3.18.31-perf`, msm8953, aarch64, SELinux **enforcing**.
- **BIDE active** — dmesg shows live section-hashing (`[BIDE] DBG: New section being hashed`)
  of `/system` libs. BIDE char device `/dev/bide` (229:0).
- `/system` = `dm-0` verity, `ro`. `/data` = `dm-1`, encrypted.
- Shell **cannot read block devices** (`/dev/block/*` permission denied) — SELinux.
- Writable mounts: `cache`, `persist`, `bbpersist`, `oempersist`, plus BB fuse `/nvram/*`.
- Captured: 622 props, 147 services, 191 packages, 1223 dmesg lines, SELinux policy (368 KB).

---

## 3. Why the KEY2 was cracked but not the KeyOne

| | KEY2 (cracked) | KEYone (not) |
|---|---|---|
| SoC | SDM660 | MSM8953 |
| Bootloader | **UEFI ABL** (`abl.elf`, edk2) | **LittleKernel** (`emmc_appsboot.mbn`) |
| Exploit | **CVE-2021-1931** — buffer overflow in ABL fastboot `flash:` parser | n/a — different codebase |
| Fix status | TCL never patched it | LK appears hardened (see notes/08) |
| Tool | `BlackBerryBootUnlock.exe` / kibo — malformed payload overruns the **RAM-resident ABL** | needs an LK equivalent |

**The core reason:** the KEY2 win is a bug in the *UEFI ABL fastboot parser*. The KeyOne
doesn't use UEFI ABL at all — it uses LK. The KeyOne's LK shows the **hardened length/size
checks** that patched the classic LK CVEs (CVE-2013-2598, CVE-2014-0973). TCL modernized the
KEY2's platform (SDM660/UEFI) and *there* left a fastboot bug; the older LK-based KeyOne
inherited BlackBerry's comparatively careful LK.

Also relevant (loaded but likely N/A): the 2026 fastboot CVEs (CVE-2026-24087/24091/24085) —
affected chips are modern (SA8xxx/SM4xxx); MSM8953 is EOL (CVE feed ends 2022).

**KEY2 "unlock" is also shallow** — even there, the unlock is fragile (needs modified boot
image to run stock OS). BlackBerry's model is fundamentally different from standard fastboot.

---

## 4. Reference material discovered (key for next steps)

- **`Aarqw12/lk_msm8953`** — the actual **Qualcomm LK source for MSM8953**
  (`app/aboot/aboot.c`). This lets us **diff BlackBerry's `emmc_appsboot.mbn` against
  reference LK** to see exactly which checks BlackBerry added/changed.
- `android.googlesource.com/kernel/lk` — AOSP LK tree.
- BootStomp (USENIX 2017) — aboot/LK vulnerability methodology.
- OnePlus fastboot CVEs (5624 `disable_dm_verity`, 5626 hidden unlock) — same bug class.

### SBL1 verify internals (from strings + disasm)
```
boot_authenticator.c, boot_elf_loader.c, boot_config_data.c
auth_hash_seg_entry / auth_hash_seg_exit / elf_segs_hash_verify_entry
"SHA256 auth failure!"
FUSED_FLOOR                 <- fuse anti-rollback
bbss_insecure / "bbry_is_insecure: TRUE"   <- verification-skip switch
bbry_load_file_info at offset 0x%llx
```

---

## 5. How to proceed (no hardware)

Since hardware is off the table, the software paths in priority order:

1. **Diff `emmc_appsboot.mbn` vs `Aarqw12/lk_msm8953` reference LK.** BlackBerry's patches
   are the map to the weakness. Look for: any *added* bounds check we might bypass, any
   *new* BlackBerry function (`bbauthtool`, `rtas`, `nvverify`) that is less mature than the
   core.
2. **Fuzz the LK fastboot `download:`/`flash:` parser** via `tools/keyone_unlock.py`
   (malformed length fields, oversized payloads) — the same bug class that cracked the KEY2,
   tested against LK. Runtime memory corruption is *not* stopped by signature checks; only
   the *flash image* is verified, not the fastboot parser's stack.
3. **Attack the RTAS/bbauthtool layer** — the parsers (`parse_hlos_signature_token`,
   `bbauthtool_cond_unlock`, `password_challenge_verify`) are BlackBerry-specific and less
   battle-tested than core LK.
4. **`hlos_unsigned.tkn` path** — the bootloader exposes `hlos_unsigned.tkn:disabled` and
   `hlos_signature.tkn:NONE`; `fastboot_publish_hlos_unsigned_tkn` exists. If an unsigned
   HLOS token can be injected (via `nvuser`, which is writable and not boot0-WP'd), the OS
   could be made to boot unsigned — a potential path to a custom OS **without** unlocking
   the bootloader.

**The single most valuable next artifact:** obtain `Aarqw12/lk_msm8953` `aboot.c` and
function-diff it against our `emmc_appsboot.mbn` symbols. BlackBerry's *deltas* from stock
LK are where a bug is most likely to live.

---

## Files
- `recon/keyone_*` — full live captures (props, services, packages, dmesg, mounts, selinux policy)
- `recon/keyone_selinux_policy.bin` — 368 KB policy
- `notes/08-keyone-lk-cve-assessment.md` — LK CVE static assessment
- `tools/keyone_unlock.py` — fastboot-over-libusb harness (patch table + overflow sender)
- `firmware/keyone-abl766/...` — SBL1 (`sbl1_signed.mbn`), aboot (`emmc_appsboot.mbn`), firehose
