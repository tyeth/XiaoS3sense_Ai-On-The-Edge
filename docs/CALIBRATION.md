# Calibration — telling the model where the digits are

The model classifies **one digit tile at a time**. You must tell it where each
digit sits in the frame, as a list of `[x1, y1, x2, y2]` boxes (top-left to
bottom-right, in pixels of the captured frame), left-to-right.

These are the **crop zones**, stored in the `${id_prefix}_crop_zones` global in
[`config.yaml`](../config.yaml).

Everything below is done **at runtime — no recompilation.** This build ships a
`web_server`, the camera-utils drawing subsystem, and a writable *Set Crop Zones*
entity, so you calibrate on the running device.

## 1. See what the camera sees
- Flip the **"Setup Mode (Preview)"** switch ON (in Home Assistant, or the
  device web UI at `http://<device-ip>/`). This turns on the annotated preview.
- Open `http://<device-ip>/` to view the image. Adjust framing, focus, and
  `vertical_flip` / `horizontal_mirror` / brightness / flash timing using the
  entities from [`camera_options.yaml`](../camera_options.yaml) — all live.
- Get the digits sharp, level, and filling the frame.

## 2. Get the coordinates
The frame is `640×480`. Read the pixel box of each digit off the preview, or use
the upstream **`draw_regions.py`** tool against a snapshot to draw boxes and
export a `regions.json` array:
https://github.com/nliaudat/esphome_ai_component (see `tools/`).

The result is a JSON array of `[x1, y1, x2, y2]` boxes, one per digit,
left-to-right. Example for an 8-digit meter:

```json
[[9,10,49,74],[64,10,104,74],[118,9,158,81],[173,9,213,81],[230,9,270,81],[286,10,326,82],[339,11,379,83],[394,10,434,82]]
```

## 3. Apply (runtime)
Paste that array into the **"Set Crop Zones"** text entity (Home Assistant →
the device's config entities, or the web UI). It applies **immediately** and is
**persisted** (the backing global has restore), so it survives reboots. With
*Show Crop Areas* on, the preview redraws the boxes so you can nudge them.

Then flip **"Setup Mode (Preview)"** back OFF for normal low-CPU operation.

> Prefer compile-time instead? You can still bake a default into the
> `${id_prefix}_crop_zones` global's `initial_value` in
> [`config.yaml`](../config.yaml) — but you shouldn't need to.

## 4. Tune confidence
Watch the *Meter Reading Confidence* and per-digit behaviour. Adjust the
`value_validator` thresholds (also exposed as HA entities):
- `per_digit_confidence_threshold`, `high_confidence_threshold`
- `max_absolute_diff` / `max_rate_change` reject impossible jumps between reads
- `allow_negative_rates: false` suits monotonic meters

If a specific digit style reads poorly, consider a different model or retraining
(see [`models/README.md`](../models/README.md)).
