# Fedora 43 Bluetooth Audio Fix (PipeWire + BlueZ)

This document records a **working, verified fix** for Bluetooth audio not appearing in Fedora 43 when using PipeWire + WirePlumber.

It is written so future-me (or anyone else) can:
- Understand *why* Bluetooth audio failed
- Reproduce the fix cleanly
- Verify the system is configured correctly

---

## Symptoms

- Bluetooth headphones show **Connected** in GNOME
- Audio works over **HDMI / DisplayPort**
- Audio does **not** work over Bluetooth
- `wpctl status` shows **no `bluez_output.*` sink**
- `pavucontrol` shows no Bluetooth device

GNOME UI is misleading here — Bluetooth can be connected at the control layer while audio is unavailable at the PipeWire layer.

---

## Root Cause

Fedora 43 ships with:
- **BlueZ 5.85** (Bluetooth stack)
- **PipeWire 1.4.x** (audio graph)

BlueZ **prefers LE Audio (Bluetooth Low Energy / BAP)** by default.

PipeWire 1.4 **does not yet expose LE Audio sinks**.

Result:
- Bluetooth connects successfully
- No A2DP profile is negotiated
- PipeWire never creates a Bluetooth audio sink
- Audio silently fails

This is not a user error — it is a stack mismatch.

---

## Correct Fix: Force Classic Bluetooth (A2DP)

We force BlueZ to use **BR/EDR (Classic Bluetooth)** instead of LE Audio.

### 1. Edit BlueZ configuration

```bash
sudo nano /etc/bluetooth/main.conf
```

Under `[General]`, add or modify **uncommented** lines:

```ini
[General]
ControllerMode = bredr
Experimental = false
```

Important:
- No `#` in front of the lines
- `bredr` = Classic Bluetooth (A2DP)

---

### 2. Restart Bluetooth

```bash
sudo systemctl restart bluetooth
```

---

### 3. Reboot (required)

```bash
sudo reboot
```

This ensures:
- LE Audio is fully disabled
- PipeWire enumerates Classic Bluetooth at startup

---

### 4. Re-pair the headphones (once)

```bash
bluetoothctl
```

Inside the prompt:

```text
remove XX:XX:XX:XX:XX:XX
scan on
pair XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX:XX
exit
```

(Replace with the actual MAC address.)

---

## Verification (Critical)

Run:

```bash
wpctl status
```

### Correct output should include:

```text
Devices:
  ATH-M50xBT [bluez5]

Sinks:
  bluez_output.XX_XX_XX_XX_XX.a2dp-sink
```

And active streams:

```text
Brave → ATH-M50xBT:playback_FL/FR [active]
```

If `bluez_output.*` exists, Bluetooth audio is working correctly.

---

## Confirm Default Sink

Ensure Bluetooth is the default audio sink:

```bash
wpctl set-default <sink-id>
```

Example:

```bash
wpctl set-default 70
```

Check:

```bash
wpctl status
```

You should see:

```text
Default Configured Devices:
Audio/Sink bluez_output.…
```

---

## What NOT to Do

- Do **not** chase missing PipeWire packages (they are already present)
- Do **not** copy random WirePlumber Lua configs
- Do **not** rely on GNOME Bluetooth UI for audio state

The issue is **LE Audio vs A2DP**, not PipeWire installation.

---

## Mental Model (Quick)

- **BlueZ**: Bluetooth protocol stack
- **PipeWire**: Audio graph engine
- **WirePlumber**: Policy / routing manager

Bluetooth audio only works when:

```
BlueZ (A2DP) → PipeWire (bluez_output) → Application
```

LE Audio breaks this chain in Fedora 43.

---

## Final Status (Verified Working)

- Bluetooth headphones appear in PipeWire
- A2DP profile active
- Brave audio routed correctly
- Bluetooth is default sink
- Survives reboot

---

## Date

Resolved and verified on Fedora 43 using PipeWire 1.4.9.

