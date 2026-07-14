# Attribution

This project stands on the shoulders of excellent prior work.

## nliaudat/esphome_ai_component (MIT)
- https://github.com/nliaudat/esphome_ai_component

The on-device inference engine — the `meter_reader_tflite`, `tflite_micro_helper`,
`esp32_camera_utils`, `flash_light_controller`, and `value_validator` ESPHome
components — comes from this project. We pull them via `external_components` in
[`config.yaml`](config.yaml) from a light fork,
[tyeth/esphome_ai_component](https://github.com/tyeth/esphome_ai_component)
(`url-model-support` branch). The fork carries a single change on top of
upstream: `meter_reader_tflite` accepts a **URL** for its `model:` option and
downloads it at build time via ESPHome's `external_files` helper (the same
mechanism `micro_wake_word` uses). This lets the model be referenced by URL so
it doesn't need to be copied into the ESPHome config directory when this repo is
consumed as a remote package. Intended to be offered back upstream.

The YAML include files in this repository (the board definition, camera config,
per-component control entities, logger/time/wifi/globals, etc.) are **vendored
and adapted** from that project's configuration files. Each vendored file carries
a header noting its origin. The adaptations here target the **Seeed Studio XIAO
ESP32-S3 Sense with the OV5640 camera**, and swap the Home Assistant transport to
**MQTT auto-discovery**.

## nliaudat/digit_recognizer (MIT)
- https://github.com/nliaudat/digit_recognizer

The committed model in [`models/`](models/) is a quantised digit-recognition CNN
from this project.

## jomjol/AI-on-the-edge-device (MIT-family)
- https://github.com/jomjol/AI-on-the-edge-device

The original "AI on the edge" concept for reading analog/mechanical utility
meters, and the digit-classification dataset lineage.

All upstream projects above are MIT-licensed. This repository is likewise MIT
(see [LICENSE](LICENSE)), and preserves the upstream copyright notices.
