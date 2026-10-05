# Installing LineageOS on the BlackBerry KEY2 (BBF100-6) — Complete Guide

A step-by-step guide to unlocking the bootloader and installing LineageOS 22.2 (Android 15)
on the BlackBerry KEY2. Tested on the **BBF100-6** (India/APAC dual-SIM) variant running stock ACQ160.

> **WARNING:** This process **erases all data** on the phone and voids any warranty.
> It relies on an undocumented bootloader exploit (CVE-2021-1931). Proceed only on a device
> you own and can afford to lose. Back up everything first.

---

## What You Need

**Hardware**
- BlackBerry KEY2 (this guide: BBF100-6)
- Windows PC
- USB-A to USB-C cable (avoid USB-C-to-C cables — they cause flashing issues)
- A charged battery (50%+)

**Files** (all linked in the Resources section)
- BlackBerry USB Drivers
- ACQ160 stock autoloader (KEY2)
- KEY2 bootloader unlock tool
- LineageOS 22.2 ROM (ZKrab v1.12a, kernel 4.4)
- MindTheGapps 15.0.0 arm64
- Android platform-tools (adb + fastboot)

---

## Before You Start

1. **Back up** anything important. Everything will be wiped.
2. On the phone, **remove all Google accounts** and **remove your screen lock**
   (Settings → Accounts / Security). Leaving an account or PIN can block the fastboot endpoint.
3. Install the **BlackBerry USB Drivers** on your PC.

---

## Step 1 — Flash Stock Firmware (ACQ160)

This returns the phone to the exact stock build the unlock exploit expects.

1. Power the phone **fully off**.
2. Hold **Volume Down + Power** for ~30 seconds until the **green bootloader menu** appears.
3. Connect the phone to the PC.
4. Open the extracted autoloader folder and run **`flashall.bat`**.
5. Type **`y`** and press Enter to confirm the wipe.
6. Wait ~10 minutes. The phone will reboot into Android 8.1 setup, an error screen, or back to the
   bootloader — **any of these is normal**.
7. Boot back into the bootloader (Volume Down + Power) and **run `flashall.bat` again**.

> **Why twice?** The KEY2 has a dual-slot (A/B) bootchain that BlackBerry shipped inconsistently.
> Flashing twice ensures both slots are consistent before unlocking.

You can ignore any `error: cannot load` messages during this step.

---

## Step 2 — Unlock the Bootloader

BlackBerry uses a custom `authboot` fastboot that normally refuses all unlock operations.
The unlock tool works around this using the **CVE-2021-1931** bootloader bug.

1. Power the phone off, then boot into the **bootloader** (Volume Down + Power). Stay connected by USB.
2. Run **`BlackBerryBootUnlock.exe`**.
3. Click **Scan** in the top-right corner.
4. Once your device appears, click **Unlock BootLoader**.
5. **The progress bar will stop at ~75%. This is normal — it means it succeeded.**
6. Confirm success: on the phone's bootloader screen, `MODE:` changes from **PRODUCT** to **FACTORY**.

If the tool won't start, install the Microsoft Visual C++ Redistributable.

---

## Step 3 — Flash the LineageOS Recovery

Open a terminal/Command Prompt **in the folder where you extracted the ROM zip**
(the folder containing `boot.img` and `recovery.img`).

With the phone in **bootloader mode**, run:

```
fastboot flash recovery recovery.img
fastboot flash boot recovery.img
fastboot reboot
```

> We flash the recovery to **both** the recovery and boot partitions, because
> `fastboot reboot recovery` doesn't work on this device.

The phone will boot into **Lineage Recovery**. Navigate with **Volume keys**, select with **Power**.

---

## Step 4 — Wipe and Install LineageOS

In Lineage Recovery:

1. Go to **Advanced → Enable ADB**.
2. On the PC, clear Factory Reset Protection:
   ```
   adb shell wipe-frp
   ```
3. On the phone: **Factory reset → Format system partition** (confirm).
4. Then: **Format data / factory reset** (confirm).
5. Return to the main menu (top-right arrow).
6. Select **Apply update → Apply update from ADB**.
7. On the PC, sideload the ROM (the file *inside* the downloaded ROM zip):
   ```
   adb sideload lineage-22.2-20260915-UNOFFICIAL-zkrab-v1.12a-athena.zip
   ```
8. Wait for `Total xfer: 1.00x` — this means the full ROM transferred.

### Install Google Apps (Optional)

9. Back on the phone, select **Apply update → Apply update from ADB** again.
10. On the PC:
    ```
    adb sideload MindTheGapps-15.0.0-arm64-20260915_032013.zip
    ```
11. If recovery shows **"Signature verification failed"**, choose **Yes** to install anyway.
12. Wait for `Total xfer: 1.00x`.

---

## Step 5 — First Boot

1. On the phone, select **Reboot system**.
2. **First boot takes 10–15 minutes** and may restart a few times. Be patient — do not interrupt it.

> During setup, the physical keyboard may not type. **Skip the Wi-Fi and PIN steps** during the
> wizard, then set them up afterward from the home screen.

---

## Step 6 — Enable the Physical Keyboard (Important!)

The ROM ships **K12KB**, a keyboard app built for the KEY2's physical keyboard, but it is
**disabled by default** — so the phone falls back to a generic on-screen keyboard and the
hardware keys behave incorrectly.

Enable it via ADB:

```
adb shell ime enable com.ai10.k12kb/.K12KbIME
adb shell ime set com.ai10.k12kb/.K12KbIME
```

Or manually: **Settings → System → Languages & input → On-screen keyboard**, enable **K12KB**,
then set it as the default.

### Keyboard tips
- **Shift + Space** switches between keyboards.
- Keyboard options live under **Settings → System → Blackberry settings**:
  layout (QWERTZ/AZERTY), backlight timeout, keyboard touchpad.
- Keyboard scrolling too sensitive? Toggle the **"Keyboard scrolling"** tile in Quick Settings.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `error: cannot load` during autoloader | Ignore it; the flash still succeeds. |
| Unlock tool doesn't detect the phone | Remove Google account + PIN first; ensure USB drivers installed. |
| Flash tool stuck at 75% | That's success — the bar doesn't reach 100%. |
| Phone boots to recovery repeatedly | Boot to bootloader and run `fastboot flash recovery boot.img`. |
| Second SIM not detected | Update to ZKrab v1.20d (fixes dual-SIM detection; updates without wiping). |
| Notification/keyboard issues | Ensure K12KB is set as the default IME (Step 6). |
| Play Protect / banking apps fail | Install a Play Integrity spoofing module (APatch 11142 — newer versions bootloop on kernel 4.4). |

---

## Notes

- **LineageOS support for the KEY2 is unofficial** — this is a community build. Official LineageOS
  does not support the device because BlackBerry never released fully GPL-compliant kernel source.
- **Kernel 4.4** is the most stable build and the one recommended for daily use.
- **Keep the ACQ160 autoloader** — it is your recovery lifeline if anything goes wrong.
- Bootloader unlock + flashing custom software **breaks the device's security chain of trust** and
  disables BlackBerry security features (DTEK, hardware-backed attestation).

---

## Resources

- LineageOS build: `LOS22_Key2_ZKrab-v1.12a.zip`
- Google apps: MindTheGapps 15.0.0 (arm64)
- Stock firmware: KEY2 ACQ160 autoloader
- Bootloader unlock tool: `key2-unlock-public.zip`
- USB drivers: BlackBerry USB Drivers 5.0.0.3
- Platform-tools: Android SDK Platform Tools

*Guide compiled by Williamson Security Solutions — williamsonsecuritysolutions.com*
