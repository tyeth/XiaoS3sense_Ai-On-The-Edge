# XIAO ESP32-S3 Sense — On-Device Meter OCR → Home Assistant

Read a mechanical/utility meter (electricity, gas, water) with a **Seeed Studio
XIAO ESP32-S3 Sense + OV5640 camera**, run the digit OCR **entirely on the
device**, and publish the reading to **Home Assistant via MQTT auto-discovery**.

- 🧠 **On-device inference** — a quantised digit CNN (TensorFlow Lite Micro)
  runs on the ESP32-S3. The model is **committed to this repo and embedded into
  the firmware** at build time. No SD card, no external OCR server.
- 💾 **PSRAM everything** — the JPEG framebuffer and the TFLite tensor arena
  live in the XIAO's 8 MB PSRAM; images are held only transiently while being
  processed.
- 🏠 **Home Assistant native** — integrates over the ESPHome native API
  (auto-discovered), exposing all entities, the camera image, and runtime
  services. MQTT discovery is a ready-to-enable alternative. Feeds straight
  into the Energy dashboard.
- 📷 **OV5640** — the 120° wide-angle autofocus module (same socket as the
  stock OV2640).

Built as an ESPHome config layer on top of
[**nliaudat/esphome_ai_component**](https://github.com/nliaudat/esphome_ai_component)
(the on-device inference engine) — see [ATTRIBUTION.md](ATTRIBUTION.md).

---

## How it works

```
 ┌──────────────────────── XIAO ESP32-S3 Sense ────────────────────────┐
 │  OV5640 ──► JPEG frame (PSRAM) ──► crop digit ROIs ──► grayscale     │
 │                                          │                           │
 │                                          ▼                           │
 │                       TFLite Micro digit CNN  (arena in PSRAM)       │
 │                       model embedded in firmware (models/*.tflite)   │
 │                                          │                           │
 │                        per-digit class + confidence                 │
 │                                          │                           │
 │                        value_validator (sanity / rate checks)       │
 └──────────────────────────────────────────┼──────────────────────────┘
                                             ▼
                          ESPHome native API  ──►  Home Assistant
                             (or MQTT discovery)
```

Full-image capture + 8-digit inference runs in well under a second on the S3
(~1.3 s with the default model, dominated by the CNN; faster models available).

## Hardware

| Item | Notes |
|------|-------|
| Seeed Studio XIAO ESP32-S3 **Sense** | 8 MB flash **+ 8 MB PSRAM** (the Sense expansion carries the camera connector) |
| OV5640 camera for XIAO ESP32-S3 Sense | 120° wide-angle, autofocus. Same FPC socket as the OV2640 |
| A stable mount + lighting | See [docs/HARDWARE.md](docs/HARDWARE.md) |

## Boards

Two boards are supported, each with its own entrypoint (the AI stack is shared):

| Board | Entrypoint | Camera | Flash |
|-------|-----------|--------|-------|
| Seeed **XIAO ESP32-S3 Sense** | [`config.yaml`](config.yaml) | OV5640 (external clock, no power-down) | SK6812 NeoPixel on GPIO9, or GPIO4 MOSFET LED |
| Adafruit **MEMENTO** (PyCamera) | [`config_memento.yaml`](config_memento.yaml) | OV5640 autofocus (power-down wired) | onboard 8-LED RGBW NeoPixel ring (A1/GPIO18) |

In the ESPHome Builder, use the matching wrapper in [`dashboard/`](dashboard/)
(`xiao-meter-ocr.yaml` or `memento-meter-ocr.yaml`) — it just points the package
at the right entrypoint. Adding a third board = a new `boards/board_*.yaml` (that
defines the camera pins and a flash light with id `${id_prefix}_flash_neopixel`)
plus a thin entrypoint that includes it and `ai_stack.yaml`.

## Quick start

### Option A — ESPHome Builder in Home Assistant (easiest, no local tools)

The [ESPHome Builder add-on](https://esphome.io/guides/getting_started_hassio)
compiles the firmware **inside Home Assistant** and keeps secrets in its own
editor — nothing to install, no `secrets.yaml` in the repo.

1. Open the **ESPHome Builder** add-on → **New device**.
2. Give it a name, skip the Wi-Fi prompt, then **Edit** the device YAML and
   replace it with the contents of
   [`dashboard/xiao-meter-ocr.yaml`](dashboard/xiao-meter-ocr.yaml). That file
   pulls this whole repo in as a package:
   ```yaml
   packages:
     meter_ocr:
       url: https://github.com/tyeth/XiaoS3sense_Ai-On-The-Edge
       ref: main
       files: [config.yaml]
       refresh: 0s
   ```
   The digit model is fetched by URL at build time (via the forked component's
   `external_files` support), so nothing needs copying into the config dir.
3. Secrets: only the standard `wifi_ssid` / `wifi_password` are required, and
   the Builder already has them. MQTT broker/credentials default via
   substitutions in the wrapper — edit them there for your broker.
4. **Install** → *Plug into this computer* (first flash over USB) or *Wirelessly*
   (OTA thereafter). Hold **BOOT** 2-3 s for the first serial flash.

Because the device declares [`dashboard_import`](config.yaml), the Builder also
recognises it as a shareable project and can adopt it directly.

### Option B — ESPHome CLI

```bash
git clone https://github.com/tyeth/XiaoS3sense_Ai-On-The-Edge.git
cd XiaoS3sense_Ai-On-The-Edge

cp secrets.yaml.example secrets.yaml
$EDITOR secrets.yaml          # Wi-Fi + MQTT broker (homeassistant.gdenu.fi)
$EDITOR config.yaml           # meter_unit / device_class (electricity vs gas)

esphome compile config.yaml   # validate
esphome run config.yaml       # flash (hold BOOT 2-3 s first)
```

Then, either way, **calibrate the digit crop zones** — the one step that always
needs tuning per meter. See **[docs/CALIBRATION.md](docs/CALIBRATION.md)**.

## Configuration map

| File | Purpose |
|------|---------|
| [`config.yaml`](config.yaml) | XIAO ESP32-S3 Sense entrypoint (substitutions + includes) |
| [`config_memento.yaml`](config_memento.yaml) | Adafruit MEMENTO entrypoint |
| [`ai_stack.yaml`](ai_stack.yaml) | Shared, board-agnostic AI + capture stack |
| [`dashboard/`](dashboard/) | Paste-in wrappers for the ESPHome Builder (Option A) |
| [`secrets.yaml.example`](secrets.yaml.example) | Wi-Fi + MQTT credentials template (Option B) |
| [`mqtt.yaml`](mqtt.yaml) | MQTT discovery — optional alternative to the native API |
| [`web_server.yaml`](web_server.yaml) | Local web UI for the calibration preview + runtime tweaks |
| [`boards/`](boards/) | XIAO ESP32-S3 Sense board: pins, PSRAM, flash |
| [`esp32_camera.yaml`](esp32_camera.yaml) / [`camera_options.yaml`](camera_options.yaml) | Camera + live tuning entities |
| [`controls/`](controls/) | HA runtime controls + the meter/validator entities |
| [`models/`](models/) | The embedded `.tflite` digit model |
| [`docs/`](docs/) | Hardware, calibration, Home Assistant setup |

## Electricity **and** gas

One device reads one meter. To read both, run **two** ESPHome nodes: copy the
repo (or just use two config files), and in each set a unique `name` /
`id_prefix` plus the right `meter_unit` / `meter_device_class`
(`energy`/`kWh` vs `gas`/`m³`). See [docs/HOME_ASSISTANT.md](docs/HOME_ASSISTANT.md).

## Status

The full config **validates cleanly** (`esphome config` → "Configuration is
valid!") when pulled as a remote package in the ESPHome Builder, including the
URL-fetched model. It has **not yet been hardware-flashed from this exact repo**,
so expect to spend time on crop-zone calibration and camera focus/lighting
before readings are trustworthy. CI runs a full `esphome compile` on every push.
Issues & PRs welcome.

## License

MIT — see [LICENSE](LICENSE) and [ATTRIBUTION.md](ATTRIBUTION.md).
