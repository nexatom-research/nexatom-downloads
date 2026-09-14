# System Control

The System Control module provides the master logical switches for the NexatomTT hardware. These 9 endpoints are responsible for toggling the global data shuffler, routing external clock signals, resetting peripheral state machines, and asserting the physical output formatting for the USB stream.

*Note: The hardware output mode (`NEXATOM_OUTPUT_REALTIME_DATA` vs `RAW_TAGS`) must be configured while the system is disabled (`enable = false`) to prevent pipeline corruption.*

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_system` | `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | The master hardware switch. Pass `true` to arm the FPGA data shuffler and permit data flow to the host. |
| `nexatom_tt_reset_peripherals` | `[In] nexatom_tt_handle device`<br>`[In] bool reset` | `nexatom_error_code_t` | Issues a hard reset command to the internal measurement counters and FIFO buffers. |
| `nexatom_tt_request_sync_clock` | `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Requests the FPGA to phase-lock to a 10MHz reference signal on the external Sync-Clock input port. |
| `nexatom_tt_request_global_stop_all_modes` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Commands the hardware to gracefully terminate all active TIHI, MFCO, and Correlation measurement processors. |
| `nexatom_tt_set_performance_monitoring` | `[In] nexatom_tt_handle device`<br>`[In] bool enabled` | `nexatom_error_code_t` | Enables aggressive host-side USB throughput metrics and buffer underrun tracking. |
| `nexatom_tt_set_output_type` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_output_data_type_t mode` | `nexatom_error_code_t` | Routes hardware logic between processed histogram streams and raw event buffering. |
| `nexatom_tt_get_output_type` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_output_data_type_t* mode_out` | `nexatom_error_code_t` | Queries the currently active hardware routing mode. |
| `nexatom_tt_set_cps_period_selector` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t selector` | `nexatom_error_code_t` | Sets the hardware integration window for Count Per Second (CPS) packets (`0` = 1000ms, `1` = 100ms). |
| `nexatom_tt_get_cps_period_selector` | `[In] nexatom_tt_handle device`<br>`[Out] uint32_t* selector_out` | `nexatom_error_code_t` | Queries the current CPS integration period. |

### Data Structures & Enums

#### `nexatom_output_data_type_t` (Enum)
Defines the structure of the data stream emitted by the instrument over the USB bulk endpoints.

| Value | Identifier | Description |
| :--- | :--- | :--- |
| `0` | `NEXATOM_OUTPUT_NO_OUTPUT` | Complete hardware silence. Safest state for configuration changes. |
| `1` | `NEXATOM_OUTPUT_ENCODED_TAGS` | (Legacy) Compact time-tag streaming. |
| `2` | `NEXATOM_OUTPUT_RAW_TAGS` | High-bandwidth 16-byte absolute time-tag streaming for offline disk saving. |
| `3` | `NEXATOM_OUTPUT_REALTIME_DATA` | (Default) FPGA processes TIHI/MFCO models internally and dispatches small summary payloads. |

### C Example: Safe Output Mode Switching

```c
// 1. Suspend the active data stream to prevent FIFO corruption
nexatom_tt_enable_system(my_device, false);

// 2. Safely switch the output logic
nexatom_tt_set_output_type(my_device, NEXATOM_OUTPUT_RAW_TAGS);

// 3. Clear any residual packets buffered in the kernel or hardware
nexatom_tt_reset_peripherals(my_device, true);

// 4. Resume acquisition
nexatom_tt_enable_system(my_device, true);
```
