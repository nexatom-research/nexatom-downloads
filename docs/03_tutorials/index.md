# 3 Tutorials

The NexatomTT SDK includes a suite of Python example scripts located in the `python/examples` directory. These scripts are designed to provide reference implementations for common operational workflows, ranging from basic DLL initialization to advanced high-bandwidth data acquisition and firmware management.

The following tutorials provide step-by-step walkthroughs of these reference scripts, detailing the SDK classes used, the expected hardware state transitions, and the data processing pipelines.

## Chapter contents

| Topic | Reference Scripts |
|---|---|
| [DLL Load and Version Check](3_1_dll_load_and_version_check.md) | `load_smoke.py`, `version_info.py` |
| [Booting into Runtime from Bootloader](3_2_booting_into_runtime_from_bootloader.md) | `boot_runtime.py` |
| [Realtime CPS and Telemetry Streaming](3_3_realtime_cps_and_telemetry_streaming.md) | `ftdi_realtime_smoke.py` |
| [Raw Time-Tag Capture and Offline CSV Export](3_4_raw_time_tag_capture_and_offline_csv_export.md) | `file_save_and_offline_decode.py` |
| [Live TIHI and MFCO Plotting](3_5_live_tihi_and_mfco_plotting.md) | `tihi_mfco_matplotlib.py` |
| [Runtime ↔ Bootloader Handoff Validation](3_6_runtime_bootloader_handoff_validation.md) | `runtime_bootloader_handoff.py` |
| [End-to-End Firmware Field Update](3_7_end_to_end_firmware_field_update.md) | `field_update_e2e.py` |

More scripts in `python/examples` that have no tutorial page of their own:

| Script | What it shows | See |
|---|---|---|
| `processed_acquisition.py` | Processed CPS, TIHI and MFCO recording to CSV (template; CPS only on the UTT160810) | [Quick start](../01_getting_started/1_2_quick_start.md) |
| `channel_setup.py` | Editable per-input settings and board selection by `connection_id`, shared by the templates | [Input channels](../02_hardware/2_2_input_channels.md) |
| `export_time_tags.py` | A saved `.nxtt` file or series turned into one CSV or TAB file; no board needed | [Time-tag export](../06_in_depth_guides/6_2_offline_time_tag_binary_decoder.md#time-tag-export) |
| `fast_tihi_click_table.py` | A Fast TIHI run recorded to `.nxti` click files and exported in all three table layouts (UTT160810) | [Fast TIHI click files](../06_in_depth_guides/6_2_offline_time_tag_binary_decoder.md#fast-tihi-index-files) |
| `utt_16_8_fast_dtc.py` | Profile-gated Fast TIHI and pulse-output example (UTT160810); diagnostic only unless its explicit write flag is given | [Pulse outputs](../02_hardware/2_13_dtc_pulse_outputs.md) |
| `saved_data_review.py` | Short recordings in every saved-file format for a readability review, or `--inspect-only` to recheck saved files without hardware | [File saving](../06_in_depth_guides/6_1_file_saving_and_data_export.md) |

## Processed and raw template anatomy

Start with `python/examples/processed_acquisition.py` for CPS/TIHI/MFCO CSV, or `file_save_and_offline_decode.py` for NXTT recording and CSV export. The [quick start](../01_getting_started/1_2_quick_start.md#configure-channels-and-record-processed-results) explains the settings and resulting files.

The C/C++ project in `examples/sdk/` provides `hardware.c` and `hardware.cpp`, both sharing `acquisition.c`. C++ adds resource ownership around the public C ABI. The [language-integration chapter](../01_getting_started/1_3_programming.md#c-cpp-examples) gives build and run commands.

| Stage | What happens | Where to adapt it |
|---|---|---|
| Enter runtime | Discover the instrument by `connection_id`; native attaches or boots a valid runtime | Runtime timeout and intended device selection |
| Validate | Check the full plan against the native profile | Selected channels, supported modes and features |
| Configure | Quiet the system, configure inputs and measurement engines | Threshold, edge, delay, hysteresis, TIHI span and coincidence window |
| Prepare recording | Register callbacks and open native file sinks | Destination, format and rotation limits |
| Acquire | Start explicitly; callbacks retain owned snapshots while native records | Observation duration and optional test pulses |
| Finish | Stop, collect terminal results where appropriate, quiet sources, finalize files and check disconnect | Preserve cleanup on every error path |
| Analyze | Read new files and retain counts, units, completion and quality | Small analysis functions and output summaries |

The example sources include comments at these stages. A recorded histogram is not automatically an independent increment: the templates use REPLACE snapshots, so their summaries describe the latest result. Raw timestamps remain integer picoseconds through subtraction. These tutorials explain how those choices affect interpretation.
