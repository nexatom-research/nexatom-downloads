# C API Function Reference

This chapter explains the public C API in the package's `include/nexatomtt_c_api.h`. That shipped header is authoritative for exact declarations, record layouts and newly added functions. These pages cover the preview.7 API families while retaining the earlier manual's reference structure.

While Chapter 5 focuses on the object-oriented Python abstractions, this chapter documents the raw hardware control endpoints, memory management rules, and pointer semantics required when integrating the SDK into C, C++, Rust, LabVIEW, or other Foreign Function Interface (FFI) environments.

### Architectural Rules
Before utilizing the C API, developers must adhere to the following architectural constraints:
1.  **Return Codes:** Check the declared return type. For `nexatom_error_code_t`, `NEXATOM_SUCCESS` is success and a nonzero result must be handled. Some functions return strings or `void` instead.
2.  **String Encodings:** All string buffers (e.g., serial numbers, error messages) expect and return strictly **UTF-8** encoded null-terminated character arrays.
3.  **Callback ownership:** Fixed by-value records can be copied into application storage. The versioned configuration-dump callback is a borrowed pointer/view exception and requires a deep copy of its records. Follow the exact typedef and [callback ownership rules](../06_in_depth_guides/6_5_callback_thread_safety_and_data_lifetime.md).

## Chapter Contents

The reference is grouped by function. For complete processed/raw applications use `examples/sdk/hardware.c` or `hardware.cpp` and their shared acquisition code; see [template anatomy](../03_tutorials/index.md#processed-and-raw-template-anatomy).

| Module | Description |
| --- | --- |
| [Library version and errors](7_1_library_version.md) | Native identity and diagnostics |
| [Logging](7_2_logging_config.md) | Structured logs, configuration, statistics and quiescent unregister |
| [Discovery and lifecycle](7_3_device_discovery.md) | Device selection and handle ownership |
| [Connection and profile](7_4_device_connection.md) | Runtime readiness, authority and capabilities |
| [System control](7_5_system_control.md) | Enable, reset, clock request and output selection |
| [Field update](7_6_field_update.md) | Existing image slots, guarded loading and same-handle boot |
| [Channel configuration](7_7_channel_config.md) | Threshold, edge, delay, hysteresis and test pulses |
| [Callbacks](7_8_callback_registration.md) | Data, telemetry/configuration views and callback lifetime |
| [TIHI](7_9_tihi.md) | Normal/Fast TIHI and fitting |
| [MFCO](7_10_mfco.md) | Patterns, aggregation and result quality |
| [CORL/CORM](7_11_correlation.md) | Intensity correlation and normalization |
| [DLS](7_12_dls.md) | Host light-scattering analysis controls |
| [FCS](7_13_fcs.md) | Host fluorescence-correlation analysis controls |
| [DCS](7_14_dcs.md) | Host diffuse-correlation analysis controls |
| [Telemetry and configuration](7_15_telemetry.md) | Requests, available views and DTC status/apply controls |
| [Calibration](7_16_calibration.md) | Supported requests and automatic triggers |
| [File saving](7_17_file_saving.md) | Native raw/processed file sinks |
| [Binary reader](7_18_binary_decoder.md) | Offline NXTT batches and series |
