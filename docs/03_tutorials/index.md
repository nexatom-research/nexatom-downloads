# 3. Tutorials

Run commands from the extracted SDK root. Use the complete example source in your package; snippets here explain a stage and do not replace its validation/cleanup.

| Tutorial | Packaged Python example |
| --- | --- |
| [3.1 Library load](3_1_dll_load_and_version_check.md) | `python/examples/version_info.py` |
| [3.2 Runtime startup](3_2_booting_into_runtime_from_bootloader.md) | `python/examples/boot_runtime.py` |
| [3.3 Live callbacks](3_3_realtime_cps_and_telemetry_streaming.md) | `python/examples/ftdi_realtime_smoke.py` |
| [3.4 Raw capture and decode](3_4_raw_time_tag_capture_and_offline_csv_export.md) | `python/examples/file_save_and_offline_decode.py` |
| [3.5 Plotting](3_5_live_tihi_and_mfco_plotting.md) | `python/examples/tihi_mfco_matplotlib.py` |
| [3.6 Service round trip](3_6_runtime_bootloader_handoff_validation.md) | `python/examples/runtime_bootloader_handoff.py` |
| [3.7 Firmware update](3_7_end_to_end_firmware_field_update.md) | `python/examples/field_update_e2e.py` |

## Processed and raw template anatomy

Start with `python/examples/processed_acquisition.py` for CPS/TIHI/MFCO CSV and `file_save_and_offline_decode.py` for raw NXTT/CSV. The [quick start](../01_getting_started/1_2_quick_start.md) gives working Python commands.

For C/C++, the extracted `examples/sdk/` directory contains `hardware.c` (target `hardware_c`) and `hardware.cpp` (target `hardware_cpp`), both supporting processed and raw workflows. They share `acquisition.c` and `acquisition.h`; the C++ entry point adds RAII while using the same public C ABI. Follow that directory's README for exact build/run options and platform libraries.

The primary templates follow this sequence, explained by the expanded inline comments shipped in preview.8. Their API behaviour remains consistent with preview.7:

1. **Select and connect:** retain the enumerated device identity; let native establish runtime/profile readiness.
2. **Validate a complete plan:** check authorized channels, output/feature support, threshold, delay and profile-derived pulse timing before writing settings.
3. **Prepare while quiet:** configure inputs, callbacks, measurement engines and file sinks without starting acquisition accidentally.
4. **Start explicitly:** enable the system/output path and the intended engines; enable internal pulses only when requested.
5. **Collect meaningful evidence:** processed callbacks retain owned snapshots while native saves CSV; raw capture saves NXTT for subsequent decoding.
6. **Stop and finalize:** observe expected terminal results, quiet output, stop pulses, close sinks and check disconnect. Attempt all cleanup steps even if one fails.
7. **Read the result:** reject absent/empty data, report status/quality and retain the summary alongside the files.

For processed data, REPLACE snapshots are separate results. CPS is already Hz; TIHI reports bin counts/peak location; MFCO distinguishes exact patterns from inclusive channel coincidences without inventing a live-time denominator. For raw data, check the whole file set, channel counts and interval statistics; preserve errors instead of reporting a partial decode as success.

[Manual contents](../index.md)
