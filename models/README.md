# Models

The `.tflite` file here is **embedded into the firmware at build time** — no SD
card, no network fetch. At runtime it's loaded into a TensorFlow Lite Micro
interpreter whose tensor arena lives in the XIAO's 8 MB PSRAM.

## Committed model

`digit_recognizer_v40_quantized_integer_quant_uint8.tflite`
- 11 output classes: digits `0-9` plus `N` (not-a-number / mid-rotation).
- Quantised (uint8) for a small footprint and fast integer inference.
- ~99.06 % accuracy, ~1300 ms inference, ~106 KB footprint (per upstream).

`config.yaml` points at it via:

```yaml
meter_reader_tflite:
  model: "models/digit_recognizer_v40_quantized_integer_quant_uint8.tflite"
```

## Other models

The upstream project ships several trade-offs. Drop another `.tflite` here and
change the `model:` path to switch:

| Model | Acc. | Inference | Footprint | Notes |
|-------|------|-----------|-----------|-------|
| v40 | 99.06% | ~1300 ms | ~106 KB | **default** — best balance |
| v3  | 97.95% | ~1500 ms | ~154 KB | |
| v24 | 98.86% | ~2600 ms | ~135 KB | |
| v23 | 97.87% | ~2300 ms | ~262 KB | robust on very poor images |
| v16 | 99.46% | ~4150 ms | ~264 KB | highest accuracy, slowest |

Browse / download: https://github.com/nliaudat/esphome_ai_component/tree/main/models
Model source & training: https://github.com/nliaudat/digit_recognizer

## Training your own

If your meter's font/style isn't read well, retrain on your own captures:
1. Collect labelled digit crops (the `data_collector` component / the upstream
   data-extractor server can help gather low-confidence images).
2. Train the CNN (see `nliaudat/digit_recognizer`).
3. Quantise to int8 `.tflite`, drop it here, update the `model:` path.
