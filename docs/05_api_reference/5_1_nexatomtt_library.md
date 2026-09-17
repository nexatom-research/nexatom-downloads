# 5.1 NexatomTT library and constants

`NexatomLibrary(home=...)` locates and loads the extracted SDK. `version()` returns the native version string; `library_info()` returns build information. Neither should be confused with the SDK preview release label.

Use `error_message(code)` for a stable code description and copy `last_error_message()` promptly for current native diagnostics. `NexatomError` carries the failed code and message; do not suppress cleanup failures after a capture.

| Constant family | Meaning |
| --- | --- |
| `NEXATOM_OUTPUT_*` | Output selection, checked against the profile's output-mode mask |
| `NEXATOM_AGGREGATION_*` | REPLACE, ACCUMULATE or AVERAGE result handling |
| `NEXATOM_PD_PACKET_*`, `NEXATOM_PD_FILE_*` | Processed packet and file formats |
| `NEXATOM_TT_FILE_*` | Time-tag file formats |
| `NEXATOM_TT_PROFILE_*`, `NEXATOM_TT_FEATURE_*` | Native profile authority and available feature families |
| `NEXATOM_HARDWARE_PROTOCOL_MODE_*` | Unknown, service/bootloader or runtime protocol |

Use exported symbolic constants instead of reproducing enum numbers. Acquisition completion is not a generic sequential success code: retain the actual result status and consult the result contract. The legacy array maxima describe ABI storage, not permission to use every physical channel or the maximum for every model.

The binding also exposes library-level structured logging through `set_log_callback`, `unregister_log_callback`, logging capabilities/configuration/statistics and module controls. See [logging](../06_in_depth_guides/6_3_logging_system.md).

[Python reference](index.md)
