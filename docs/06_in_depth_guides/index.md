# In-Depth Guides

While the tutorials in Chapter 3 demonstrate the baseline code required to operate the UTT810, advanced integration often requires a deeper understanding of the underlying SDK mechanics and mathematical models.

This chapter explores the theoretical and architectural concepts driving the NexatomTT SDK. It provides comprehensive specifications for the proprietary binary file formats, mathematical definitions for the on-board curve fitting algorithms, and detailed state-machine rules for orchestrating complex hardware transitions.

## Chapter contents

| Topic | Description |
|---|---|
| [File Saving and Data Export](6_1_file_saving_and_data_export.md) | Native file rotation, export formats, and the `.nxtt` binary byte layout. |
| [Offline Time-Tag Binary Decoder](6_2_offline_time_tag_binary_decoder.md) | Advanced usage of the `NexatomTimeTagReader` for decoding single files and sequential series. |
| [Logging System](6_3_logging_system.md) | Two-tier filtering architecture, log modules, and integration with host telemetry. |
| [Curve Fitting and Analysis Pipelines](6_4_curve_fitting_and_analysis_pipelines.md) | Mathematical models for TIHI lifetime fitting, CORM correlation, DLS, FCS, and DCS. |
| [Callback Thread Safety and Data Lifetime](6_5_callback_thread_safety_and_data_lifetime.md) | Memory ownership, pass-by-value FFI boundaries, and Python garbage collection safety. |
| [Bootloader-First Device Startup](6_6_bootloader_first_device_startup.md) | Firmware slot prioritization, USB re-enumeration, and the `open_runtime_device()` state machine. |