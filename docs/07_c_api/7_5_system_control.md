# System Control

The System Control module provides logical controls for the system data path, external-clock request, peripheral reset and output selection. Connection, system enable, measurement start and file saving are separate operations.

The complete templates prepare settings while output/system are quiet. Native owns output-transition barriers; a client must not reproduce firmware sequences or assume that mode selection automatically starts its measurement.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_system` | `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | The master hardware switch. Pass `true` to arm the FPGA data shuffler and permit data flow to the host. |
| `nexatom_tt_reset_peripherals` | `[In] nexatom_tt_handle device`<br>`[In] bool reset` | `nexatom_error_code_t` | Issues a hard reset command to the internal measurement counters and FIFO buffers. |
| `nexatom_tt_request_sync_clock` | `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Requests the FPGA to phase-lock to a 10MHz reference signal on the external Sync-Clock input port. |
| `nexatom_tt_request_global_stop_all_modes` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Commands the hardware to gracefully terminate all active TIHI, MFCO, and Correlation measurement processors. |
| `nexatom_tt_set_performance_monitoring` | `[In] nexatom_tt_handle device`<br>`[In] bool enabled` | `nexatom_error_code_t` | Compatibility control; does not expose or enable host throughput statistics. |
| `nexatom_tt_set_output_type` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_output_data_type_t mode` | `nexatom_error_code_t` | Routes hardware logic between processed histogram streams and raw event buffering. |
| `nexatom_tt_get_output_type` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_output_data_type_t* mode_out` | `nexatom_error_code_t` | Returns the API's tracked output mode; not independent hardware readback. |
| `nexatom_tt_set_cps_period_selector` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t selector` | `nexatom_error_code_t` | Sets the hardware integration window for Count Per Second (CPS) packets (`0` = 1000ms, `1` = 100ms). |
| `nexatom_tt_get_cps_period_selector` | `[In] nexatom_tt_handle device`<br>`[Out] uint32_t* selector_out` | `nexatom_error_code_t` | Queries the current CPS integration period. |

### Data Structures & Enums

#### `nexatom_output_data_type_t` (Enum)
Defines the structure of the data stream emitted by the instrument over the USB bulk endpoints.

| Value | Identifier | Description |
| :--- | :--- | :--- |
| `0` | `NEXATOM_OUTPUT_NO_OUTPUT` | Quiet acquisition output; supported explicit identity/telemetry requests can still respond. |
| `1` | `NEXATOM_OUTPUT_ENCODED_TAGS` | (Legacy) Compact time-tag streaming. |
| `2` | `NEXATOM_OUTPUT_RAW_TAGS` | Raw acquisition; NXTT saving uses its own packed file format. |
| `3` | `NEXATOM_OUTPUT_REALTIME_DATA` | Processed acquisition where supported; system and intended engines still need enabling/start. |

### C Example: Safe Output Mode Switching

```c
// Fragment: profile permits raw output; the complete owner handles cleanup.
nexatom_error_code_t rc = nexatom_tt_enable_system(my_device, false);
if (rc == NEXATOM_SUCCESS)
    rc = nexatom_tt_set_output_type(my_device, NEXATOM_OUTPUT_RAW_TAGS);
// Prepare the file sink before explicitly starting this measurement.
// A deliberate reset needs true (assert) followed by false (release); it is
// not a substitute for native output-transition handling.
if (rc != NEXATOM_SUCCESS)
    fprintf(stderr, "Setup failed: %s\n", nexatom_tt_get_last_error_message());
```
