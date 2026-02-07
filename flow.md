
free Flow 2 68 — VIA, HID & Custom Configurator Guide

## Table of Contents

- [Overview](#overview)
- [VIA Setup (Getting It Working)](#via-setup-getting-it-working)
- [Keyboard Specs Relevant to Software](#keyboard-specs-relevant-to-software)
- [Risk Assessment](#risk-assessment)
- [The Escape Hatch (Factory Reset)](#the-escape-hatch-factory-reset)
- [HID Communication Protocol](#hid-communication-protocol)
- [VIA Protocol v3 — Command Reference](#via-protocol-v3--command-reference)
- [Lighting Control](#lighting-control)
- [Keymap Reading & Writing](#keymap-reading--writing)
- [QMK Keycode Encoding](#qmk-keycode-encoding)
- [Building a Custom TUI Configurator — Shopping List](#building-a-custom-tui-configurator--shopping-list)
- [Useful Links & Resources](#useful-links--resources)

---

## Overview

The Lofree Flow 2 68 is a 65%, low-profile mechanical keyboard with VIA support. It has **no proprietary software** — all configuration is done through VIA (web-based or desktop), which communicates with the keyboard over Raw HID. This document captures everything needed to understand, configure, and build custom tooling for this keyboard.

---

## VIA Setup (Getting It Working)

VIA will **not** auto-detect the Flow 2. You must sideload a JSON definition file.

### Prerequisites

- **Chromium-based browser** (Chrome, Edge — Firefox doesn't support WebHID)
- **USB-C wired connection** — VIA works over wired and 2.4GHz dongle, but **not Bluetooth**
- The VIA JSON definition file for the Flow 2

### Getting the JSON File

The JSON files are hosted on Dropbox, linked from the Kickstarter page under "03 Functions" → "Flow2 VIA Json-download":

```
https://www.dropbox.com/scl/fo/gfzjhsawxjxqys27rxawd/AObvtrBR_LB3Z8VlhFjQzcw?rlkey=rb8d8euaqg1moxq0y7fzwv3ut&e=1&st=w0wr8aq5&dl=0
```

> **⚠️ CRITICAL for 68-key model**: The 68-key JSON filename is **mislabeled**. Use the file named `FLOW2-100key-OE928-Json-20250905.json` even though you have the 68. Yes, Lofree mislabeled it.

### Loading into VIA

1. Go to [usevia.app](https://usevia.app) in Chrome
2. Click the **gear icon** (Settings) → toggle **"Show Design tab"** on
3. Click the **paint brush icon** (Design tab)
4. Click **"Load"** and select the JSON you downloaded
5. The keyboard definition should now say **"Flow2@Lofree"**
6. Go back to the **keyboard icon** (first tab) — you should now see your full keymap

---

## Keyboard Specs Relevant to Software

| Property | Value |
|---|---|
| Keys | 68 (65% layout) |
| Layers | 6 total (3 macOS + 3 Windows) |
| Layer 0 | macOS default |
| Layer 1 | macOS Fn |
| Layer 2 | macOS reserved (Clear EEPROM) |
| Layer 3 | Windows default |
| Layer 4 | Windows Fn |
| Layer 5 | Windows reserved (Clear EEPROM) |
| Backlight | **White only — no RGB** |
| Connectivity | USB-C, Bluetooth 5.3, 2.4GHz dongle |
| VIA connectivity | USB-C wired or 2.4GHz only (not Bluetooth) |
| Touch bar | Right side — swipe for volume/brightness, disable with Fn + Space (hold 3s) |
| Firmware | QMK-based (Lofree fork with proprietary BT/2.4GHz/touch bar drivers) |
| OS switching | Fn + M (macOS), Fn + N (Windows) |

The macOS/Windows switch just changes the default layer from 0 to 3. You can repurpose all 6 layers for your own use if you don't need the OS toggle.

---

## Risk Assessment

### Zero Risk

| Action | Details |
|---|---|
| VIA remapping via JSON sideload | Writes to EEPROM (separate from firmware). Factory reset always recovers. JSON file is read-only UI metadata — doesn't touch the keyboard. |
| Reading HID reports / sniffing protocol | Pure observation. Can't break anything. |

### Low Risk

| Action | Details |
|---|---|
| Sending raw HID lighting commands | Same EEPROM VIA writes to. Worst case: bad lighting values. Factory reset clears it. HID devices drop unknown/malformed commands. |
| Bad Fn layer remap in VIA | You can accidentally overwrite BT/2.4GHz switching keycodes. Annoying but recoverable via factory reset. **Always backup your default layers first.** |

### Medium Risk (NOT relevant if you never flash firmware)

| Action | Details |
|---|---|
| Flashing custom QMK firmware | Replaces actual firmware on MCU. Risk of bricking if interrupted. Wrong image = keyboard won't enumerate on USB. |
| Flashing without original firmware backup | If you flash something broken and don't have the original `.bin`, you're stuck begging Lofree support. |
| Flashing QMK | You lose Lofree's proprietary bits: touch bar, Bluetooth, 2.4GHz, possibly their lighting engine. Stock QMK doesn't have any of that. **May not be reversible.** |

### Impossible (via software)

| Action | Details |
|---|---|
| Overwriting hardware bootloader | ROM bootloader is read-only on most MCUs. Cannot be overwritten by software. |

**Bottom line**: If you never flash firmware, the worst that can happen is a bad keymap that's one factory reset away from being fixed.

---

## The Escape Hatch (Factory Reset)

You have **three independent** ways to reset. They are layered — even if one fails, the others work.

### 1. Hardware Key Combo (hardcoded in firmware, cannot be remapped)

```
Fn + Left Shift + Backspace (hold)
```

This clears EEPROM and restores all layers to firmware defaults. Since you're never flashing firmware, this combo is **permanently baked in** and will always work.

### 2. VIA Protocol Command (software-side)

Send command `0x06` (`dynamic_keymap_reset`) over raw HID:

```python
device.write([0x00, 0x06] + [0x00] * 30)
```

This does the same thing — wipes EEPROM keymap back to defaults. Wire this to an "OH SHIT" button in your TUI.

### 3. Layer 2/5 Clear EEPROM Key

Layers 2 and 5 (the "reserved" layers for macOS and Windows respectively) have a dedicated Clear EEPROM key. Accessible via the normal layer switching mechanism.

### Best Practice: Backup Before Touching Anything

Before any modifications, dump the full EEPROM keymap buffer:

```python
def dump_keymap(device):
    """Read entire keymap via bulk read (command 0x0C)"""
    keymap = bytearray()
    offset = 0
    chunk_size = 28  # 32-byte packet minus header
    total_size = 6 * 5 * 15 * 2  # layers × rows × cols × 2 bytes (estimate)
    
    while offset < total_size:
        size = min(chunk_size, total_size - offset)
        packet = [0x00, 0x0C, (offset >> 8) & 0xFF, offset & 0xFF, size] + [0x00] * 27
        device.write(packet)
        response = device.read(32)
        keymap.extend(response[4:4+size])
        offset += size
    
    return keymap
```

Save to a timestamped JSON file. Restore with command `0x0D` (bulk write). This is version control for your keymap.

---

## HID Communication Protocol

### Finding the Device

The keyboard exposes multiple HID interfaces. You want the **Raw HID** interface:

| Property | Value |
|---|---|
| Usage Page | `0xFF60` (QMK/VIA raw HID) |
| Usage | `0x61` |
| Report Size | 32 bytes (zero-padded) |

```python
import hid

def find_flow2():
    for dev in hid.enumerate():
        if dev['usage_page'] == 0xFF60 and dev['usage'] == 0x61:
            return dev
    return None
```

### Packet Format

Every message is a **fixed 32-byte array**. First byte is the command ID. The keyboard responds with the same command ID echoed back plus response data.

```
Send:    [command_id, param1, param2, ..., 0x00 padding to 32 bytes]
Receive: [command_id, response_data..., 0x00 padding to 32 bytes]
```

**Note**: When using `hidapi`, you typically prepend a `0x00` report ID byte, making the actual write 33 bytes, but the logical payload is 32.

### Connection Requirements

- **USB-C wired** or **2.4GHz dongle** — VIA/HID does NOT work over Bluetooth
- No authentication, no encryption, no handshake beyond checking protocol version
- Send 32 bytes, get 32 bytes back

---

## VIA Protocol v3 — Command Reference

### Core Commands

| ID | Name | Description |
|---|---|---|
| `0x01` | `get_protocol_version` | Returns VIA protocol version (sanity check) |
| `0x02` | `get_keyboard_value` | Read keyboard state (uptime, layout options, etc.) |
| `0x03` | `set_keyboard_value` | Write keyboard state |
| `0x04` | `get_keycode` | Read a single key's assigned keycode at `(layer, row, col)` |
| `0x05` | `set_keycode` | Write a keycode to `(layer, row, col)` |
| `0x06` | `dynamic_keymap_reset` | Reset all layers to defaults (EEPROM clear) |
| `0x07` | `lighting_set_value` | Set a lighting parameter |
| `0x08` | `lighting_get_value` | Read a lighting parameter |
| `0x09` | `lighting_save` | Persist lighting changes to EEPROM |
| `0x0C` | `dynamic_keymap_get_buffer` | Bulk-read keymap data |
| `0x0D` | `dynamic_keymap_set_buffer` | Bulk-write keymap data |

### Command Details

**Get Protocol Version (`0x01`)**:
```
Send:    [0x01, 0x00, 0x00, ...]
Receive: [0x01, version_hi, version_lo, ...]
```

**Get Keycode (`0x04`)**:
```
Send:    [0x04, layer, row, col, ...]
Receive: [0x04, layer, row, col, keycode_hi, keycode_lo, ...]
```

**Set Keycode (`0x05`)**:
```
Send:    [0x05, layer, row, col, keycode_hi, keycode_lo, ...]
Receive: [0x05, ...]
```

**Bulk Read Keymap (`0x0C`)**:
```
Send:    [0x0C, offset_hi, offset_lo, size, ...]
Receive: [0x0C, offset_hi, offset_lo, size, data_byte_0, data_byte_1, ...]
```

The keymap is stored as a flat buffer: `layers × rows × cols × 2 bytes` (each keycode is uint16). For the Flow 2 68 with 6 layers, this is approximately `6 × 5 × 15 × 2 = 900 bytes`, chunked into 28-byte reads.

**Bulk Write Keymap (`0x0D`)**:
```
Send:    [0x0D, offset_hi, offset_lo, size, data_byte_0, data_byte_1, ...]
Receive: [0x0D, ...]
```

---

## Lighting Control

### The Flow 2 68 is white-only backlight (no RGB)

This simplifies things significantly. You're working with QMK's simple **backlight** subsystem:

- **Brightness** (0–255)
- **Effect mode** (static, breathing, possibly a few more)
- **On/off toggle**

### VIA Lighting Commands

```
Get value: [0x08, param_id, 0x00, ...]  → response includes current value
Set value: [0x07, param_id, value, ...]  → sets the parameter
Save:      [0x09, ...]                   → persist to EEPROM
```

### Common Lighting Parameter IDs

These are QMK-dependent and may vary. Probe by iterating:

| Param ID | Typical Meaning |
|---|---|
| `0x80` | Backlight brightness (0-255) |
| `0x81` | Backlight effect/mode |
| `0x82` | Backlight on/off |

Since there's no RGB, you likely only have 2-3 valid param IDs. Probe them:

```python
def probe_lighting_params(device):
    """Iterate through param IDs and see which ones return valid data"""
    valid = {}
    for param_id in range(0x00, 0xFF):
        packet = [0x00, 0x08, param_id] + [0x00] * 29
        device.write(packet)
        response = device.read(32)
        if response[0] == 0x08 and response[1] == param_id:
            # Check if it returned meaningful data (not all 0xFF / error)
            value = response[2]
            if value != 0xFF:  # 0xFF often means "unhandled"
                valid[param_id] = value
    return valid
```

### QMK Backlight Keycodes

If you want to assign physical keys for backlight control:

| Keycode | Name | Function |
|---|---|---|
| `0x5DA0` | `BL_TOGG` | Toggle backlight on/off |
| `0x5DA1` | `BL_STEP` | Cycle through brightness steps |
| `0x5DA2` | `BL_ON` | Turn backlight on |
| `0x5DA3` | `BL_OFF` | Turn backlight off |
| `0x5DA4` | `BL_INC` | Increase brightness |
| `0x5DA5` | `BL_DEC` | Decrease brightness |

### Physical Touch Bar

The touch bar on the right side also controls brightness (hold Fn + swipe). Disable it with **Fn + Space (hold 3 seconds)**.

---

## QMK Keycode Encoding

QMK keycodes are **16-bit values**. The encoding is layered:

| Range | Meaning |
|---|---|
| `0x0004`–`0x0067` | Basic HID keycodes (A=`0x04`, B=`0x05`, etc.) |
| `0x00E0`–`0x00E7` | Modifiers (LCtrl, LShift, LAlt, LGui, and right variants) |
| `0x0100`–`0x1FFF` | QMK internal (layer toggles, one-shot, etc.) |
| `0x5100`–`0x51FF` | Momentary layer — `MO(n)` |
| `0x5200`–`0x52FF` | Toggle layer — `TG(n)` |
| `0x7000`–`0x7FFF` | Mod-tap (tap = keycode, hold = modifier) |
| `0x5DA0`–`0x5DA5` | Backlight controls (`BL_TOGG`, `BL_STEP`, etc.) |

The full keycode mapping is in QMK's `quantum/keycodes.h`. For your TUI, you need a lookup table from uint16 → human-readable label. You can rip this from the VIA source: `the-via/app` on GitHub.

---

## Building a Custom TUI Configurator — Shopping List

### What You Need (keyboard-related only)

1. **The VIA JSON definition file** — Parse it for:
   - Physical layout coordinates (`x, y, w, h` for each key) → rendering in TUI
   - `(row, col)` → physical key position mapping
   - Layer count and structure

2. **A HID library** — Pick one for your language:
   - Python: `hid` (hidapi wrapper) — `pip install hidapi`
   - Rust: `hidapi` crate
   - Node: `node-hid`
   - Go: `karalabe/hid`

3. **VIA protocol implementation** — The command table above, packet framing, send/receive with 32-byte fixed packets

4. **QMK keycode lookup table** — Map uint16 → human label. Source: `the-via/app/src/utils/key.ts` or QMK's `quantum/keycodes.h`

5. **Lighting parameter discovery** — Probe the board to figure out which param IDs it supports (script above)

6. **Keymap backup/restore** — Bulk read (`0x0C`) to dump, bulk write (`0x0D`) to restore, save to timestamped files

### What You Don't Need

- QMK source code (you're not flashing)
- Any Lofree proprietary software
- Bluetooth anything (VIA is USB/2.4GHz only)
- RGB knowledge (it's white-only)

---

## Useful Links & Resources

| Resource | URL |
|---|---|
| VIA Web App | https://usevia.app |
| VIA Source Code (protocol reference) | https://github.com/the-via/app |
| QMK Keycodes Reference | https://github.com/qmk/qmk_firmware/blob/master/quantum/keycodes.h |
| VIA JSON for Flow 2 (Dropbox) | [Dropbox link](https://www.dropbox.com/scl/fo/gfzjhsawxjxqys27rxawd/AObvtrBR_LB3Z8VlhFjQzcw?rlkey=rb8d8euaqg1moxq0y7fzwv3ut&e=1&st=w0wr8aq5&dl=0) |
| Lofree Flow 2 Kickstarter | https://www.kickstarter.com/projects/lofree/flow-2the-smoothest-keyboard-evolved-redefined-unleashed |
| Gadgetoid Flow 2 68 Review (VIA details) | https://gadgetoid.com/2025/09/22/lofree-flow-2-68-65-low-profile-mechanical-keyboard-reviewed/ |
| Flow 2 VIA Keycodes (lost keycodes recovery) | https://note.com/fuss/n/ncbf52c506f80 |
| Lofree Download Center | https://www.lofree.co/pages/download-center |
| Python hidapi | https://pypi.org/project/hidapi/ |
| Vial (open-source VIA alternative) | https://get.vial.today |

---

*Last updated: February 2026*
