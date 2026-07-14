# Home Assistant integration

Readings are delivered over **MQTT auto-discovery** (see [`mqtt.yaml`](../mqtt.yaml)).

## Prerequisites
- The **MQTT integration** is set up in Home Assistant, pointing at a broker
  (usually Mosquitto on the HA host).
- `secrets.yaml` has `mqtt_broker` / `mqtt_username` / `mqtt_password`.

> The broker is **not** the HA web UI. `https://homeassistant.gdenu.fi:8123`
> (with its self-signed cert) is the UI; MQTT talks to the broker on port 1883.
> The invalid 8123 certificate does not affect MQTT. To use TLS MQTT (8883)
> instead, provide the broker CA in `mqtt.yaml` — see the comments there.

## Discovery
On boot the device publishes discovery configs under the `homeassistant/`
prefix. A **device** named *XIAO Meter OCR* appears under
**Settings → Devices & Services → MQTT**, with entities including:

- **Meter Reading** (`sensor`) — the OCR value, with your configured
  `device_class` / `state_class` / unit.
- **Meter Reading Confidence** (`%`)
- **Inference Time**, **Capture to Publish Time** (diagnostics)
- Config controls: update interval, validator thresholds, flash timing,
  camera tuning, pause/preview switches, restart, unload/reload.

> ⚠️ **Do not also add this device via the ESPHome/API integration.** It would
> create a duplicate set of entities. Use MQTT only.

## Energy dashboard
For the Energy dashboard, the *Meter Reading* sensor needs
`state_class: total_increasing` and the right `device_class`. These are already
driven by the `config.yaml` substitutions:

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
grid, and the gas sensor under gas consumption.

If you prefer to keep the raw OCR entity untouched and shape it in HA instead,
you can wrap it in a template sensor:

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
Run one node per meter, each with a unique `name` / `id_prefix`, so their MQTT
topics and entity ids don't collide. Point each camera at its meter and
calibrate crop zones independently.
