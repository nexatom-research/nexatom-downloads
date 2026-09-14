# C API Complete Function Reference

This chapter serves as the exhaustive, low-level dictionary for the native C Application Programming Interface defined in `nexatomtt_c_api.h`.

While Chapter 5 focuses on the object-oriented Python abstractions, this chapter documents the raw hardware control endpoints, memory management rules, and pointer semantics required when integrating the SDK into C, C++, Rust, LabVIEW, or other Foreign Function Interface (FFI) environments.

### Architectural Rules
Before utilizing the C API, developers must adhere to the following architectural constraints:
1.  **Return Codes:** Every function returns a signed 32-bit `nexatom_error_code_t`. A return value of `0` (`NEXATOM_SUCCESS`) indicates success. Any negative value indicates a failure.
2.  **String Encodings:** All string buffers (e.g., serial numbers, error messages) expect and return strictly **UTF-8** encoded null-terminated character arrays.
3.  **Pass-by-Value:** To guarantee thread safety, all asynchronous data payloads (TIHI, MFCO, CPS) are dispatched strictly by-value. The host application unconditionally owns the memory provided in the callback.

## Chapter Contents

*Note: Due to the massive scale of the C API (130 functions), this reference is split into modular pages based on functional subsystems.*

| Module                                   | Function Count | Description |
| :--- |:-----------------------------------------| :--- |
| [**Library Version and Error Handling**](7_1_library_version.md) | 4 | Endpoints for checking SDK versions and extracting error strings. |
| [**Logging Configuration**](7_2_logging_config.md)           | 4 | Host-side log levels, file rotation, and stdout rules. |
| [**Device Discovery and Lifecycle**](7_3_device_discovery.md)       | 3 | Scanning the USB bus and allocating hardware handles. |
| [**Device Connection and State**](7_4_device_connection.md)          | 7 | Establishing USB sessions and querying system states. |
| [**System Control**](7_5_system_control.md)                       | 9 | Master routing, system resets, and hardware mode selection. |
| [**Field Update / Bootloader**](7_6_field_update.md)            | 7 | Flash memory mapping and firmware flashing state machines. |
| [**Channel Configuration**](7_7_channel_config.md)                | 9 | Input thresholds, synthetic delays, and test pulse routing. |
| [**Callback Registration**](7_8_callback_registration.md)                | 9 | Binding host C functions to background hardware event loops. |
| [**Time Interval Histogram (TIHI)**](7_9_tihi.md)       | 17 | High-speed Start/Stop decay profiling and curve fitting. |
| [**Multifold Coincidence (MFCO)**](7_10_mfco.md)         | 12 | 8-channel temporal correlation and pattern filtering logic. |
| [**Intensity Correlation (CORL/CORM)**](7_11_correlation.md)    | 12 | Linear (CORL) and Logarithmic Multi-Tau (CORM) correlators. |
| [**DLS Analysis**](7_12_dls.md)                         | 6 | Dynamic Light Scattering models (Hydrodynamic radius, PDI). |
| [**FCS Analysis**](7_13_fcs.md)                         | 5 | Fluorescence Correlation Spectroscopy (Diffusion, Concentration). |
| [**DCS Analysis**](7_14_dcs.md)                         | 5 | Diffuse Correlation Spectroscopy (Blood flow, tissue perfusion). |
| [**Telemetry and Register Diagnostics**](7_15_telemetry.md)   | 5 | FPGA thermals, voltages, and low-level register probing. |
| [**Calibration**](7_16_calibration.md)                          | 4 | Hardware thermal-drift compensation and delay line sweeps. |
| [**File Saving**](7_17_file_saving.md)                          | 6 | Direct-to-disk streaming of raw binary time tags and processed arrays. |
| [**Time Tag Binary Decoder**](7_18_binary_decoder.md)              | 5 | Offline `.nxtt` file parsing and CSV conversion logic. |
