# Akamatis LD2410 Presence Sensor: ESPHome package

ESPHome firmware for Akamatis smart presence sensors. Each sensor is an
ESP32-C3 with an HLK-LD2410 mmWave radar. The firmware exposes:

- presence, moving and still targets, with distances and energies
- per-gate move and still thresholds you can tune from Home Assistant
- an **on-device auto calibration** that sets the thresholds from the room's
  background noise
- self-healing watchdogs for the radar

| File | Purpose |
|---|---|
| [`presence-sensor.yaml`](presence-sensor.yaml) | Shared body, pulled by every sensor as a remote ESPHome package. Never flashed directly |
| [`example-device.yaml`](example-device.yaml) | Device file template: substitutions + `packages:`. Copy one per sensor |
| [`secrets.yaml.example`](secrets.yaml.example) | Secrets the body expects |

The `ld2410` component comes from Akamatis'
[esphome-custom](https://github.com/akamatis-smart-devices/esphome-custom)
repository, which extends ESPHome's stock component. This config builds on the
Akamatis factory firmware.

---

## How it's structured

Each sensor's config lives in your ESPHome config directory (not in this repo)
and is only a few lines long:

```yaml
substitutions:
  name: "akamatis-smart-device-5"
  friendly_name: "Akamatis Bedroom"

packages:
  presence:
    url: https://github.com/adman234/presense-sensor
    ref: main
    files: [presence-sensor.yaml]
    refresh: 0s
```

At compile time ESPHome clones this repo, loads `presence-sensor.yaml`, and
applies the device's substitutions to it.

### Substitutions

| Key | Required | Default | What to put |
|---|---|---|---|
| `name` | yes | | Unique node name = hostname = HA device. Lowercase, `a-z 0-9 -`, at most **31** chars |
| `friendly_name` | yes | | Shown in HA and used as the prefix for entity names |
| `log_level` | | `INFO` | |
| `ap_ssid` | | `${friendly_name} Hotspot` | Fallback access point SSID, at most 32 chars |
| `esp32_tx_pin` / `esp32_rx_pin` | | `GPIO21` / `GPIO20` | Radar UART pins |
| `device_address` | | `${name}.local` | Where to reach the sensor for OTA and logs. Only override during a rename |

`!secret` values come from the `secrets.yaml` in **your** ESPHome config
directory.

---

## Adding a sensor

1. **Secrets:** copy `secrets.yaml.example` to `secrets.yaml` in your ESPHome
   config directory and fill it in. Skip this if you've done it before.
2. **Device file:** copy `example-device.yaml` to `<name>.yaml` and fill in the
   placeholders.
3. **Flash:**
   - A factory Akamatis unit answers as `<factory-name>-<mac6>.local`. Set
     `device_address` to that address for the first OTA flash, then remove it.
   - Otherwise, flash over USB: `esphome run <name>.yaml`.
4. **Adopt in HA** with the API key from `secrets.yaml`.
5. **Calibrate:**
   1. Leave the room empty.
   2. Turn on **010: Radar - Auto calibration**.
   3. Wait for **01: Calibration status** to read `Complete`.

Keep your own list of which sensor is in which room. Don't commit per-device
files to this repo.

---

## Updating sensors

1. Edit `presence-sensor.yaml`, validate it (see [Validating](#validating)), and
   push to `main`.
2. Recompile and flash each sensor over the air: `esphome run <name>.yaml`, or
   **Update All** in the HA ESPHome dashboard.

- **A sensor only changes when it's reflashed.**
- **To freeze a sensor on a known-good version**, set `ref:` to a tag or commit
  SHA.
- **To test a change on one sensor first**, point only that sensor at a branch.
- **Compiling needs internet access.** Both this repo and Akamatis'
  `esphome-custom` are fetched from GitHub on every build.

---

## Node names and entity IDs

`name_add_mac_suffix` is **`false`**, so the node name you give each sensor is
its identity:

- its hostname, `<name>.local`
- its HA device
- the prefix of every entity ID it owns

Every sensor needs a unique `name`. Renaming a deployed sensor changes all of
its entity IDs.

### Renaming a sensor

1. Set `device_address` to the address the sensor answers on **right now**.
2. Flash it OTA.
3. Remove the override.
4. Update any HA dashboards or automations that used the old entity IDs.

---

## Auto calibration

The calibration adjusts the per-gate thresholds to the room's background noise:

1. **Turning on 010: Radar - Auto calibration** puts the radar in engineering
   mode. The sensor then records the peak move and still energy seen at each of
   the 9 gates.
2. **This runs for 011: Duration minutes** (1–10). The status shows the
   countdown.
3. **Then each threshold is set to** `min(100, peak + 012: Margin)`. Thresholds
   are only ever **raised**, never lowered, and the new values are written to
   the radar.

To lower thresholds again, set them by hand in HA, or run **03: Radar - Factory
reset** before calibrating.

### Watchdogs

- **Every 5 min:** if `Detection Distance` is unknown (the radar has stopped
  talking), the radar is factory-reset and the ESP32 restarts.
- **Every 60 min:** engineering mode is switched off, unless a calibration is
  running. Left on, it can flood the UART and stall the radar.

---

## Known traps

1. **A push to `main` reaches every sensor on its next flash.** Validate before
   pushing, or pin sensors to a tag.
2. **The radar watchdog calls `factory_reset()`,** which wipes the radar's gate
   thresholds, not just the ESP32 state. If a sensor keeps losing its tuning,
   check the logs for `Radar sensor is not responding`. Re-run calibration after
   a watchdog reset.
3. **`esphome-custom` is not pinned.** A change upstream in Akamatis' component
   is picked up on the next compile. Add a `ref:` under `external_components`
   if that becomes a problem.
4. **Validation warnings** (ESPHome 2026.9):
   - `web_server` auth defaults to `basic`, which changes to `digest` in 2027.1.
   - The `web_server` OTA endpoint is not covered by OTA encryption.
   - The Akamatis component's `bluetooth_password.set` action lacks
     `synchronous=`.

   All three are harmless for now.
5. **Compiling on Windows:**
   - Run `esphome` from PowerShell or cmd, not Git Bash. ESP-IDF refuses to
     install under MSYS/MinGW.
   - Build from a short path such as `C:\ev`. The git clones can hit Windows'
     path-length limit.

### Validating

Point a scratch device file's package at your local working copy:

```yaml
substitutions:
  name: "akamatis-test"
  friendly_name: "Akamatis Test"
packages:
  presence: !include C:/path/to/presense-sensor/presence-sensor.yaml
```

Then run `esphome config scratch.yaml`, next to a dummy `secrets.yaml`. Also run
`esphome compile scratch.yaml` when a lambda changes.

Last verified with `esphome config` and a full `esphome compile` on ESPHome
2026.9.0, 2026-09-25.

---

## Hardware reference

| | |
|---|---|
| MCU | ESP32-C3 (`esp32-c3-devkitm-1`), framework `esp-idf` |
| Radar | HLK-LD2410, UART 256000 8N1 |
| Radar TX → ESP | ESP `GPIO20` (`rx_pin`) |
| ESP → Radar RX | ESP `GPIO21` (`tx_pin`) |

## License

The files in this repository are MIT licensed, see [LICENSE](LICENSE). The `ld2410`
component is not copied here: ESPHome fetches it from Akamatis'
[esphome-custom](https://github.com/akamatis-smart-devices/esphome-custom) at compile
time, and that repository has no license file, so it stays under its author's terms.
