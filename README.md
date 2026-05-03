# SmartBlindsDIY
DIY Smart Blinds, no sensors, pull direclty on chain will work on any blind. 

An open-loop stepper motor controller for chain-driven roller blinds using ESPHome, a NodeMCU ESP8266, and a TMC2209 motor driver. Supports up to (or more than) ten blinds working individually or together via Home Assistant and HomeKit.

> **No position sensors required.** Position is tracked entirely in software using step counts, with dual travel calibration to account for the difference in motor effort going up vs down under gravity.

![Hardware photo placeholder](docs/images/hardware.jpg)
*Photo coming soon*

---

## Features

- **Dual travel calibration** — separate step counts for up and down moves to account for gravity, giving accurate positioning in both directions
- **Persistent position** — current position and calibration saved to flash so the blind knows where it is after a reboot
- **Safety lockout** — if power is lost mid-move the blind marks itself as unsafe on boot and refuses to move until manually re-topped, preventing the motor from driving past the physical limits of the blind
- **Gravity overdrive** — configurable per-blind upward overdrive percentage to compensate for the blind dropping slightly when the motor stops, adjustable live from the HA dashboard without reflashing
- **Full Home Assistant and HomeKit integration** — once calibrated the blind exposes a native cover entity in HA with a position slider, open, close and stop buttons. Set the blind anywhere from fully open to fully closed or any point in between — just drag the slider to something like 66% and it goes there. Works in HA dashboards, scenes, automations and HomeKit
- **Jog buttons** — always available regardless of safety state for manual recovery
- **Quick Re-Top** — one button reset to re-establish known position without a full recalibration

---

## Prerequisites

- **ESPHome** installed and working — either via the ESPHome add-on in Home Assistant (the web UI approach) or the command line. If you're new to ESPHome the HA add-on is the easiest starting point as it handles compiling and flashing through a browser interface
- **Home Assistant** with the ESPHome integration
- Basic familiarity with ESPHome yaml config — you don't need to understand all the code but you should be comfortable creating device configs, editing substitutions, and flashing firmware
- A `secrets.yaml` file in your ESPHome config directory for credentials

> **Board compatibility:** This project is built around the NodeMCU ESP8266 but can be adapted for other ESPHome supported boards such as the Seeed XIAO ESP32-C6 with pin mapping changes in `blind_common.yaml` and updating the board platform from `esp8266` to `esp32`.

---

## Hardware

| Component | Details |
|---|---|
| Microcontroller | NodeMCU ESP8266 (v2) |
| Motor driver | TMC2209 (configured for 8x microstepping) |
| Motor | NEMA 17 stepper |
| Power supply | 12V DC (shared across all blinds) |
| Drive | Existing roller blind pull chain via 3D printed gear/adapter |

![Wiring diagram placeholder](docs/images/wiring.jpg)
*Wiring diagram coming soon*

### Pin Mapping

| NodeMCU Pin | Function |
|---|---|
| D1 | Stepper STEP |
| D2 | Stepper DIR (inverted per blind via substitution) |
| D3 | MS1 (microstepping) |
| D4 | MS2 (microstepping) |
| D5 | MS3 (microstepping) |
| D6 | Stepper SLEEP (inverted) |

---

## File Structure

```
├── blind_common.yaml       # Shared template included by all blind configs
├── blindliving01.yaml      # Per-blind config (one file per blind)
├── blindliving02.yaml
├── ...
└── secrets.yaml            # WiFi credentials, API keys, OTA passwords
```

---

## Setup

### 1. secrets.yaml

Create a `secrets.yaml` in your ESPHome project directory:

```yaml
wifi_ssid: "YourNetworkName"
wifi_password: "YourWiFiPassword"
ota_password: "your_ota_password"
api_encryption_key01: "your_base64_key_here"   # generate with: esphome generate-api-key
api_encryption_key02: "your_base64_key_here"
# one key per blind
```

### 2. Per-blind config

Each blind gets its own yaml file. Copy and adjust:

```yaml
substitutions:
  name: blindliving01
  friendly_name: Living Room Blind 1
  dir_inverted: "true"          # flip if your blind moves the wrong way
  api_encryption_key: !secret api_encryption_key01
  ota_password: !secret ota_password

packages:
  common: !include blind_common.yaml
```

`dir_inverted` flips the stepper direction — set to `"true"` or `"false"` depending on how your motor is mounted relative to the chain.

### 3. Flash

```bash
esphome run blindliving01.yaml
```

---

## Calibration

Calibration only needs to be done once per blind, or after any mechanical changes. The blind must be calibrated to run normally — it will refuse to move and report as unavailable in HA until calibration is complete.

> **Tip:** Blinds need a little buffer each side. Allow about two inches from the top and half an inch from the bottom depending on your setup. This gives room for drift and imprecision over time.

**Steps:**

1. Use the **Jog Up / Jog Down** buttons to drive the blind to the **fully open (top)** position
2. Press **Stop** when at the top
3. Press **Mark as Fully Open (Top)** — this records the top position
4. Use **Jog Down** to drive the blind to the **fully closed (bottom)** position
5. Press **Stop** when at the bottom
6. Press **Mark as Fully Closed (Bottom)** — this records the bottom position
7. Use **Jog Up** to drive back to the **fully open (top)** position
8. Press **Stop** when back at the top
9. Press **Save Calibration** — this calculates both travel step counts and saves everything to flash

The blind will set itself to 100% open after saving and is ready to use.

> The reason for going back to the top after marking the bottom is to measure the up travel separately from the down travel. These differ due to gravity and the dual step counts are what give accurate positioning in both directions.

---

## Recovery — Unsafe / Unknown Position

If power is lost while a blind is moving, it will boot in **unsafe mode** and refuse to move via HA or HomeKit. It will report as **unknown** in HA. The jog buttons always work regardless of safety state.

**To recover:**

1. Use **Jog Up** to drive the blind to the fully open (top) position
2. Press **Stop**
3. Press **Quick Re-Top** — this re-establishes the known position without a full recalibration

The blind will immediately return to normal operation.

---

## Overdrive Tuning

Due to gravity, roller blinds drop slightly when the motor stops on an upward move. The overdrive setting adds a small percentage of extra steps to every upward move to compensate.

- Default is **0.5%** (roughly 8mm on a 1557mm travel blind)
- Adjustable per blind via the **Upward Overdrive %** number entity in HA — no reflashing required
- If blinds drift upward over time reduce the value; if they consistently land low increase it
- The value is saved to flash per blind so each one can be tuned independently

---

## Multi-blind Setup Tips

- Give each blind its own API encryption key in secrets.yaml
- Use HA scenes to group blinds and move them together
- A **Stop All** script in HA is highly recommended as an emergency stop — create a script that calls `cover.stop` on all blind entities and put it on your home screen
- If using Apple HomeKit via HA bridge, scenes work well for grouped open/close but per-blind position control is best done through HA directly

---

## Known Limitations

- **Open loop only** — without sensors there is no way to auto-recover a lost position. The safety lockout is intentional: better to be unresponsive than to drive the motor past the physical limits of the blind
- **Slow drift** — step counts drift slightly downward over time as the chain stretches and slips. The Quick Re-Top button and Nudge Up & Re-Top button exist to correct this without a full recalibration
- **ESP8266 WiFi stability** — the NodeMCU can drop its WiFi connection after long idle periods. The code handles HA reconnect re-sends safely so this does not affect the safety state, but you may notice brief unavailability in HA

---

## 3D Printed Case

The enclosure used in this project is the **DIY SmartBlinds v3 for NEMA 17** by Chodyra & Co. The STL files are available for purchase on their site for a small gratuity and include several body variants with different mounting options to suit your wall setup.

👉 [candco.com.au — DIY SmartBlinds v3 for NEMA 17](https://candco.com.au/product/diy-smartblinds-v3-for-nema-17/)

> **Note:** Only the STL files from that project were used here. The ESPHome firmware in this repo was written from scratch to provide native ESPHome integration rather than the approach used in the original project.

---

## License

MIT
