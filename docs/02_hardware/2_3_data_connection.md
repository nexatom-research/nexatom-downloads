# 2.3 Data connection

## FTDI USB 3.0 transport layer

The native library owns FTDI transfers, decoding and shutdown. Use the matched package libraries and normal API cleanup. An application should not manipulate driver transfers or infer compatibility from a successful DLL load.

## Output data type modes

| Value | Public mode | Meaning |
| --- | --- | --- |
| 0 | `NEXATOM_OUTPUT_NO_OUTPUT` | Quiet acquisition output; explicit supported identity/telemetry requests can still respond |
| 1 | `NEXATOM_OUTPUT_ENCODED_TAGS` | Encoded tags, only if authorized by the profile |
| 2 | `NEXATOM_OUTPUT_RAW_TAGS` | Raw acquisition workflow |
| 3 | `NEXATOM_OUTPUT_REALTIME_DATA` | Processed packet workflow |

Test the profile's `supported_output_mode_mask` before selecting a mode. Native owns mode-transition barriers; clients should use the documented templates instead of reproducing the protocol. Selecting mode 3 does not by itself enable the system or start TIHI/MFCO. Switching to raw mode is not a promise that every unrelated hardware engine has been stopped.

Saving raw files and saving processed files are separate configurations. Follow [file saving](../06_in_depth_guides/6_1_file_saving_and_data_export.md), especially quieting acquisition before finalization/decoding.

## Performance monitoring

The compatibility `set_performance_monitoring` control does not expose a throughput or losslessness guarantee. Assess an actual capture using file contents, result metadata and controlled input rates. Do not infer acquisition success from process exit or a connection state alone.

[Device operation](index.md)
