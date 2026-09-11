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
