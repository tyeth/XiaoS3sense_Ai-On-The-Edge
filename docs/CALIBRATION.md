# Calibration — telling the model where the digits are

The model classifies **one digit tile at a time**. You must tell it where each
digit sits in the frame, as a list of `[x1, y1, x2, y2]` boxes (top-left to
bottom-right, in pixels of the captured frame), left-to-right.

These are the **crop zones**, stored in the `${id_prefix}_crop_zones` global in
[`config.yaml`](../config.yaml).

## 1. See what the camera sees
Enable the live preview so you can read pixel coordinates off the image:

- In `config.yaml` set `meter_reader_tflite: generate_preview: true`, add a
  `web_server:` (port 80) and the drawing subsystem in `esp32_camera_utils`
  (`enable_drawing: true`), then reflash and open `http://<device-ip>/`.
- Adjust framing, focus, `vertical_flip` / `horizontal_mirror`
  (see the entities from [`camera_options.yaml`](../camera_options.yaml)),
  brightness and the flash timing until the digits are sharp and level.

## 2. Get the coordinates
Two ways:

- **`draw_regions.py`** (upstream tool): run it against a captured frame to draw
  boxes interactively and export `regions.json` — copy the array in.
  https://github.com/nliaudat/esphome_ai_component (see `tools/`)
- **By hand**: read the pixel coordinates of each digit box off the preview.

The result is a JSON array, one box per digit. Example for an 8-digit meter:

```yaml
globals:
  - id: ${id_prefix}_crop_zones
    type: std::string
    max_restore_data_length: 254
    initial_value: '"[[9, 10, 49, 74], [64, 10, 104, 74], [118, 9, 158, 81], [173, 9, 213, 81], [230, 9, 270, 81], [286, 10, 326, 82], [339, 11, 379, 83], [394, 10, 434, 82]]"'
```

Note the quoting: the value is a **string containing JSON**, so the inner double
quotes are escaped — outer single quotes wrap `"...[...]..."`.

## 3. Apply
- **Compile-time**: paste the array as `initial_value` (as above) and reflash.
- **Runtime** (only if you added the device to HA via the ESPHome/API
  integration): call the `esphome.<name>_set_crop_zones` service with the JSON
  string. With **MQTT-only** setups this service isn't exposed in HA, so use the
  compile-time route.

## 4. Tune confidence
Watch the *Meter Reading Confidence* and per-digit behaviour. Adjust the
`value_validator` thresholds (also exposed as HA entities):
- `per_digit_confidence_threshold`, `high_confidence_threshold`
- `max_absolute_diff` / `max_rate_change` reject impossible jumps between reads
- `allow_negative_rates: false` suits monotonic meters

If a specific digit style reads poorly, consider a different model or retraining
(see [`models/README.md`](../models/README.md)).
