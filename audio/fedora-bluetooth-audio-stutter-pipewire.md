# Fedora Bluetooth Audio Stutter Fix (PipeWire + WirePlumber)

**Hardware example:** Audio‑Technica ATH‑M50xBT  
**OS:** Fedora (PipeWire + WirePlumber + BlueZ)  
**Symptoms:** Bluetooth audio stutters/cuts out; wired audio is clean

This guide documents a complete, reproducible fix for Bluetooth audio instability on Fedora. The root cause in this case was the system being locked into **HSP/HFP (headset/phone‑call mode)** with **A2DP hidden by policy**, which guarantees low quality and stutter.

---

## TL;DR (What actually fixed it)

1. Ensure BlueZ is in **dual** controller mode (initial troubleshooting)
2. Explicitly **re‑enable A2DP** at the PipeWire policy layer
3. Switch the headset to **A2DP + AAC**
4. Permanently **disable HFP/HSP** so the system can’t regress
5. **Force BR/EDR only** to eliminate BLE device oscillation on desktops

---

## Background (Why this happens on Fedora)

Fedora uses:
- **PipeWire** – audio engine
- **WirePlumber** – session/policy manager
- **BlueZ** – Bluetooth stack

If any application requests microphone access (browser tab, Discord, WebRTC), WirePlumber may switch the device to **HSP/HFP**. In some cases, **A2DP is never re‑exposed**, leaving the device stuck in phone‑call mode:

- Low bandwidth
- Narrowband audio
- Extremely sensitive to buffer underruns
- Audible stutter under normal desktop activity

Wired audio works because it bypasses all of this.

---

## Step 1 — BlueZ Controller Configuration

Edit the system Bluetooth config:

```bash
sudo nano /etc/bluetooth/main.conf
```

Ensure:

```ini
[General]
ControllerMode = dual
FastConnectable = true
```

Restart Bluetooth:

```bash
sudo systemctl restart bluetooth
```

Power‑cycle the headphones before reconnecting.

---

## Step 2 — Verify the Problem (Before Fix)

Check the active Bluetooth profile:

```bash
pactl list cards | grep -A20 bluez
```

If you see:

```
Active Profile: headset-head-unit
```

You are in **HSP/HFP** — this will stutter.

---

## Step 3 — Explicitly Re‑Enable A2DP (Critical Fix)

On Fedora, WirePlumber may suppress A2DP unless explicitly enabled.

Create the PipeWire override directory:

```bash
mkdir -p ~/.config/pipewire/pipewire.conf.d
```

Create the Bluetooth policy override:

```bash
nano ~/.config/pipewire/pipewire.conf.d/10-bluez-audio.conf
```

Paste:

```ini
context.properties = {
    bluez5.enable-a2dp = true
    bluez5.enable-hfp = true
    bluez5.enable-hsp = true
    bluez5.enable-msbc = true
    bluez5.enable-sbc-xq = true
}
```

Restart services **in this order**:

```bash
systemctl --user restart pipewire pipewire-pulse
systemctl --user restart wireplumber
sudo systemctl restart bluetooth
```

Power‑cycle the headphones and reconnect.

---

## Step 4 — Confirm A2DP Profiles Exist

```bash
pactl list cards | grep -A30 "ATH-M50xBT"
```

Expected profiles:

- `a2dp-sink-aac`
- `a2dp-sink-sbc_xq`
- `a2dp-sink-sbc`

If these exist, the policy fix worked.

---

## Step 5 — Switch to the Correct Codec

Prefer codecs in this order:

1. **AAC** (best for ATH‑M50xBT)
2. SBC‑XQ
3. SBC

Command:

```bash
pactl set-card-profile bluez_card.00_0A_45_0B_43_53 a2dp-sink-aac
```

Verify:

```bash
pactl list cards | grep "Active Profile"
```

Expected:

```
Active Profile: a2dp-sink-aac
```

At this point, audio should be clean and stable.

---

## Step 6 — Lock the Fix (Prevent Regression) ⭐ Recommended

If you **do not use the headset microphone** on Linux, disable HFP/HSP permanently.

Edit the same file:

```bash
nano ~/.config/pipewire/pipewire.conf.d/10-bluez-audio.conf
```

Change to:

```ini
context.properties = {
    bluez5.enable-a2dp = true
    bluez5.enable-hfp = false
    bluez5.enable-hsp = false
    bluez5.enable-msbc = true
    bluez5.enable-sbc-xq = true
}
```

Restart audio:

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

Result:
- System **cannot** switch back to headset mode
- No random stutter
- Bluetooth audio behaves like a proper music device

---

## Verification & Debugging

Check live audio health:

```bash
pw-top
```

Things to watch for:
- Underruns → buffer / RF issue
- Source attached to headset → mic mode re‑enabled

Quick sanity check anytime:

```bash
pactl list cards | grep "Active Profile"
```

---

## Notes & Gotchas

- 2.4 GHz Wi‑Fi can still cause interference; prefer 5 GHz
- USB autosuspend *can* cause rare issues, but wasn’t needed here
- This fix is **policy‑level**, not hardware‑specific

---

## Additional Root Cause: BLE vs BR/EDR Device Oscillation

Some Bluetooth headphones (including the ATH‑M50xBT) advertise **two logical identities**:

- **BLE (Bluetooth Low Energy)** – control plane (battery status, buttons)
- **BR/EDR (Classic Bluetooth)** – audio transport (A2DP, AAC, SBC)

On Fedora GNOME, BlueZ may expose **both identities simultaneously**. GNOME Settings can then oscillate between:

- `BLE_ATH‑M50xBT`
- `ATH‑M50xBT`

This oscillation can occur **before the device is even connected** and leads to:

- profile renegotiation
- headset (HFP) re‑exposure
- intermittent stutter even when A2DP is active

### Symptom

- Bluetooth device name flickers between `BLE_…` and the normal device name
- HFP reappears after reconnects
- Audio stutters despite `a2dp-sink-aac` being selected

### Fix (Desktop‑appropriate)

For **desktop A2DP use**, BLE is not required. Forcing Classic Bluetooth removes the duplicate device entirely.

Edit:

```bash
sudo nano /etc/bluetooth/main.conf
```

Set:

```ini
[General]
ControllerMode = bredr
FastConnectable = true
```

Apply:

```bash
sudo systemctl restart bluetooth
```

Power‑cycle the headphones and re‑pair once.

### Result

- Only **one** Bluetooth device appears
- No `BLE_…` entry
- No profile oscillation
- A2DP remains stable

> **Summary:** Bluetooth headphones may expose both LE and BR/EDR identities; GNOME may oscillate between them, causing profile instability. For desktop A2DP use, forcing BR/EDR eliminates the issue.

---

## Why This Is Worth Documenting

This is a common Fedora/WirePlumber failure mode:
- Hardware supports A2DP
- BlueZ works
- PipeWire works
- **Policy suppresses the correct profile**

The system appears broken until A2DP is explicitly re‑enabled.

This guide provides a clean, minimal, reversible fix.

---

## License

CC0 / Public Domain — feel free to reuse, adapt, or submit upstream.

