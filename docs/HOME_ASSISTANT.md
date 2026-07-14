# Home Assistant integration

Two transports are supported. **Pick one** — running both makes HA show every
entity twice.

## Default: ESPHome native API (recommended for setup)

The device runs the ESPHome native API ([`config.yaml`](../config.yaml) `api:`
block). Home Assistant discovers it automatically:

1. **Settings → Devices & Services** → you'll see a discovered **ESPHome** device
   (`xiao-meter-ocr`). Click **Configure / Add**.
2. All entities appear under one device: *Meter Reading*, *Meter Reading
   Confidence*, the **camera image**, timing diagnostics, and the config
   controls (update interval, validator thresholds, camera tuning, *Setup Mode
   (Preview)*, *Set Crop Zones*, pause/reload, restart).

Why the API is nice for bring-up:
- The **camera entity** shows the live/preview image right in HA — no need to
  open the web UI to frame the meter.
- The component's **services** are callable from **Developer Tools → Actions**:
  `esphome.xiao_meter_ocr_set_crop_zones`, `..._trigger_inference`,
  `..._start_flash_calibration`. (Crop zones are also settable via the *Set Crop
  Zones* entity — see [CALIBRATION.md](CALIBRATION.md).)

Optional: add an `api: encryption: key:` for an encrypted connection; without it
the API still works on a trusted LAN.

## Alternative: MQTT discovery

Prefer MQTT (e.g. to decouple from the ESPHome integration)? You have the
built-in Mosquitto broker, so:

1. In [`config.yaml`](../config.yaml) uncomment `- !include mqtt.yaml` in the
   packages list (and comment the reliance on the API if you want MQTT only).
2. Set the broker creds — **Mosquitto refuses anonymous logins** (you'll see MQTT
   return code `0x5`), so provide `mqtt_username` / `mqtt_password` (substitutions
   or `!secret`). `mqtt_broker` defaults to `homeassistant.gdenu.fi`, port 1883.
3. The device auto-registers via MQTT discovery under **Settings → Devices &
   Services → MQTT**.

> The broker is not the HA web UI. `https://homeassistant.gdenu.fi:8123` (self-
> signed cert) is the UI; MQTT uses the broker on port 1883. The invalid 8123
> certificate is irrelevant to MQTT.

> ⚠️ Don't run the API integration **and** MQTT for the same device — you'll get
> duplicate entities. Add it via one integration only.

## Energy dashboard

The *Meter Reading* sensor already carries the right classes, driven by the
[`config.yaml`](../config.yaml) substitutions:

```yaml
# electricity
meter_unit: kWh
meter_device_class: energy
meter_state_class: total_increasing
```

```yaml
# gas (second device)
meter_unit: m³
meter_device_class: gas
meter_state_class: total_increasing
```

Then **Settings → Dashboards → Energy** → add the electricity sensor under the
grid and the gas sensor under gas consumption.

To shape the value in HA instead of touching the device, wrap it in a template
sensor:

```yaml
template:
  - sensor:
      - name: "Electricity Meter Reading"
        unit_of_measurement: "kWh"
        device_class: energy
        state_class: total_increasing
        state: "{{ states('sensor.xiao_meter_ocr_meter_reading') }}"
```

## Two meters

Run one node per meter, each with a unique `name` / `id_prefix` so entity ids
don't collide. Point each camera at its meter and calibrate crop zones
independently.
