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

## OV5640 focus
The OV5640 has a voice-coil autofocus lens. There are two practical options:

1. **Fixed focus (simplest).** For a meter at a fixed distance, the lens often
   sits at a usable focus out of the box, or you can gently set it once. This is
   the default here — nothing extra to configure.
2. **Software autofocus.** Full autofocus requires uploading the sensor's AF
   firmware over SCCB at boot and triggering a focus command. ESPHome's
   `esp32_camera` doesn't do this natively. If your working distance needs it,
   see the Seeed forum thread on OV5640 autofocus for the firmware blob and the
   register sequence, and add it via an `on_boot` lambda:
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
