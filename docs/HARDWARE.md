# Hardware

## Bill of materials
- **Seeed Studio XIAO ESP32-S3 Sense** — the *Sense* variant, which has the
  expansion board with the camera FPC connector, 8 MB flash and **8 MB PSRAM**.
- **OV5640 camera for XIAO ESP32-S3 Sense** — 120° wide-angle, autofocus, up to
  QSXGA (2592×1944). Uses the same FPC socket as the stock OV2640; only the
  sensor differs, and the driver auto-detects it.
- A rigid mount so the camera can't drift relative to the meter, and even,
  glare-free lighting. The on-board LED (GPIO4) can act as a capture "flash".

## Camera pin map (XIAO ESP32-S3 Sense)
Defined in [`boards/board_Seeed_Studio_XIAO_ESP32-S3_sense.yaml`](../boards/board_Seeed_Studio_XIAO_ESP32-S3_sense.yaml):

| Signal | GPIO | | Signal | GPIO |
|--------|------|-|--------|------|
| XCLK | 10 | | VSYNC | 38 |
| SIOD/SDA | 40 | | HREF | 47 |
| SIOC/SCL | 39 | | PCLK | 13 |
| D0..D7 | 15,17,18,16,14,12,11,48 | | PWDN/RESET | not wired |

## Flash / illumination
The XIAO ESP32-S3 Sense has **no usable bright flash LED** (just a dim user LED).
The firmware drives a flash output on **GPIO4** (a free header pin), intended for
**external circuitry** — e.g. a small N-channel MOSFET switching an LED (or LED
ring) from 3V3/5V, gate to GPIO4 through a ~100 Ω resistor, plus the LED's series
resistor. Make it configurable in your device substitutions:

```yaml
substitutions:
  flash_led_pin: GPIO4        # any free GPIO
  flash_led_inverted: "false" # "false" = pin HIGH = flash ON (active-high driver)
```

The capture sequence already does **flash-on → settle → capture → flash-off**:
`Flash Pre-Time` (default 7 s) is the settling window for exposure/white-balance
(and autofocus, below); `Flash Post-Time` (200 ms) holds it briefly after. Both
are runtime number entities — tune them without recompiling.

## Camera entity & bandwidth
The `esp32_camera` **entity** in Home Assistant is only a network stream; it
pushes frames when viewed. **Disabling it saves bandwidth and does not affect
OCR** — inference reads the framebuffer locally in PSRAM. Keep it enabled while
calibrating, then in HA open the entity → settings → **disable**. (Or add
`disabled_by_default: true` under `esp32_camera:` if you want it off from first
boot.)

## OV5640 autofocus
The OV5640 has a voice-coil autofocus lens, but the stock `esp32_camera` driver
**does not load the AF firmware**, so out of the box the lens sits at its default
focus. Options:

1. **Fixed focus (simplest).** For a meter at a fixed distance, set the lens once
   and mount rigidly. Works with no extra config — start here.
2. **Software autofocus (planned option).** Enabling AF requires uploading the
   OV5640 AF firmware blob over SCCB at boot and then either continuous-focus
   mode or a single-focus trigger. The plan for this repo: trigger a single AF
   when the flash turns on, so the existing 7 s pre-flash window covers focus +
   white-balance settling before capture. Tracking this as a fork addition
   (`enable_autofocus`). Reference for the firmware/registers:
   https://forum.seeedstudio.com/t/ov5640-camera-for-xiao-esp32s3-sense-how-to-use-autofocus/272485

## PSRAM
The board config enables octal PSRAM at 80 MHz and routes the camera
framebuffer (`frame_buffer_location: PSRAM`) and the TFLite tensor arena into
it. `psramFound()` must be true — if the camera fails to init, PSRAM is the
first thing to check.

## Storage
No SD card is used. The model is embedded in the app image; captured frames are
held only transiently in PSRAM during inference and never written to storage.
The Sense's SD pins are left documented in the board file but unused.
