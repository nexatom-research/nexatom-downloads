# In-Depth Guides

While the tutorials in Chapter 3 demonstrate the baseline code required to operate the UTT810, advanced integration often requires a deeper understanding of the underlying SDK mechanics and mathematical models.

This chapter explores the theoretical and architectural concepts driving the NexatomTT SDK. It explains the binary file layout, the native host curve-fitting models and the lifecycle rules for firmware/runtime transitions. The original guide structure is retained while the API details describe 0.1.0-preview.18.2.

## Chapter contents

| Topic | Description |
|---|---|
| [File Saving and Data Export](6_1_file_saving_and_data_export.md) | Native file rotation, export formats, and the `.nxtt` binary byte layout. |
| [Offline Time-Tag Binary Decoder](6_2_offline_time_tag_binary_decoder.md) | Advanced usage of the `NexatomTimeTagReader` for decoding single files and sequential series. |
| [Logging System](6_3_logging_system.md) | Filtering, log modules, bounded delivery, statistics and clean unregister. |
| [Curve Fitting and Analysis Pipelines](6_4_curve_fitting_and_analysis_pipelines.md) | Mathematical models for TIHI lifetime fitting, CORM correlation, DLS, FCS, and DCS. |
| [Callback Thread Safety and Data Lifetime](6_5_callback_thread_safety_and_data_lifetime.md) | Memory ownership, pass-by-value FFI boundaries, and Python garbage collection safety. |
| [Bootloader-First Device Startup](6_6_bootloader_first_device_startup.md) | Slot selection and native runtime readiness on the same device handle. |
