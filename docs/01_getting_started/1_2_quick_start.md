# 1.2 Quick start workflow

Connect one intended instrument for the first run. The primary templates print their selected device and validate requested settings against the native profile.

## Device lifecycle

Discover → create one handle → request native runtime readiness → inspect profile → prepare the measurement → explicitly enable/start it → collect and validate results → stop, finalize files and disconnect.

Python's `open_runtime_device(library, device_info, options=...)` and C's `nexatom_tt_connect_runtime` own runtime discovery and booting an existing valid slot when necessary. A runtime already running may retain stopped settings; connecting is not a request to begin measurement. Do not add model-specific boot polling or assume a USB description authorizes controls.

## Realtime CPS and telemetry streaming

```sh
# From the extracted SDK root: discover, connect and observe live callbacks.
python python/examples/ftdi_realtime_smoke.py --home . --duration-sec 10
```

See [telemetry streaming](../03_tutorials/3_3_realtime_cps_and_telemetry_streaming.md) for request-based telemetry and callback ownership.

## Processed acquisition in C, C++ and Python

```sh
# Internal pulses exercise acquisition and saving without an external source.
python python/examples/processed_acquisition.py --home . --channels 0,1 --threshold-mv 500 --edge rising --delay-ps 0 --hysteresis-mv 10 --duration-sec 5 --internal-test
```

This saves native CPS/TIHI/MFCO CSV results and `processed_summary.json` in a new run directory. Verify populated files, terminal acquisition status and quality fields; process success alone is not a scientific result. The example uses REPLACE aggregation, so its latest summary must not be interpreted as a sum of all saved snapshots.

For C/C++, use the matching processed templates and build commands in the extracted package's `examples/sdk/README.md`. The [template anatomy](../03_tutorials/index.md#processed-and-raw-template-anatomy) explains the same stages in all three languages.

## Raw time-tag capture and CSV export

```sh
# Capture NXTT files, stop acquisition, then decode every captured file.
python python/examples/file_save_and_offline_decode.py --home . --duration-sec 5 --internal-test --decode-csv
```

Raw capture and processed acquisition use different output modes. The raw example checks decoded event counts and intervals and writes a summary. Use [the raw tutorial](../03_tutorials/3_4_raw_time_tag_capture_and_offline_csv_export.md) before changing rotation or shutdown.

## Live TIHI and MFCO plotting

Use `python/examples/tihi_mfco_matplotlib.py` and its `--help` for optional plots. See [the plotting tutorial](../03_tutorials/3_5_live_tihi_and_mfco_plotting.md). Start with headless capture so file and numerical results are visible independently of a GUI.

Omit `--internal-test` for external inputs and choose thresholds/edges appropriate to that source. Internal pulses do not validate analog threshold, hysteresis, timing accuracy or physical edge response. Channel selection in these templates is not a promise that all other physical inputs are disabled.

[Getting started](index.md) · [Language integration](1_3_programming.md)
