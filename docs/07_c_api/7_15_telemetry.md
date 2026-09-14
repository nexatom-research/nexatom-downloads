# Telemetry and Register Diagnostics

To ensure long-term stability in continuous-monitoring environments, the NexatomTT hardware continually monitors its own health (uptime, sync clock lock status, FPGA thermals, FIFO buffer underruns).

Developers can extract these hardware health metrics periodically via the telemetry callback, or poll them synchronously. Additionally, for deep troubleshooting, developers can request a `config_dump`—a direct readback of all active 32-bit registers mapped inside the FPGA's configuration space.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_telemetry` | `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms or disarms the periodic generation of telemetry packets from the FPGA. |
| `nexatom_tt_set_telemetry_mode` | `[In] nexatom_tt_handle device`<br>`[In] uint8_t mode` | `nexatom_error_code_t` | `1` = Manual on-request only. `2` = Periodic automatic transmission. |
| `nexatom_tt_request_telemetry` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Manually triggers the FPGA to emit a single telemetry payload. |
| `nexatom_tt_get_telemetry` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_telemetry_data_t* tel` | `nexatom_error_code_t` | Synchronously copies the most recent telemetry packet into the provided struct pointer. |
| `nexatom_tt_request_config_dump`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Commands the hardware to serialize its entire memory-mapped configuration space and send it back to the host via the `config_dump` callback. |

### Data Structures

#### `nexatom_telemetry_data_t`
The telemetry struct tracks critical hardware lifecycle statuses. It is populated either via polling (`nexatom_tt_get_telemetry`) or the async callback registered in Section 7.8.

| Field | Type | Description |
| :--- | :--- | :--- |
| `uptime_seconds` | `uint32_t` | Number of seconds the FPGA has been powered on. |
| `temperature_raw` | `uint16_t` | Internal FPGA die thermals. |
| `active_channels_mask` | `uint8_t` | Bitmask representing physically enabled SMA input ports. |
| `sync_clock_requested` | `bool` | True if the host asked the PLL to lock to the 10MHz reference. |
| `sync_clock_locked` | `bool` | **Critical Flag**: True if the external 10MHz reference is valid and successfully phase-locked. |
| `histogram_errors_word` | `uint32_t` | Flags for dropped packets or saturated measurement FIFOs. |

#### `nexatom_config_dump_data_t`
Used purely for debug, this struct is delivered exclusively via the configuration dump callback (Section 7.8). It contains raw 32-bit register addresses and their currently active values.

| Field | Type | Description |
| :--- | :--- | :--- |
| `copied_register_count` | `uint32_t` | The number of valid registers present in the `registers` array. |
| `registers` | `struct[128]` | Array of `{ uint32_t address; uint32_t value; }` pairs detailing the raw FPGA configuration memory. |

### C Example: Synchronous Health Check

```c
nexatom_telemetry_data_t health_status;

// 1. Ask the FPGA to immediately push its current state
nexatom_tt_request_telemetry(my_device);

// Wait briefly for USB transfer
Sleep(50); // Or use the callback for pure async

// 2. Extract the latest known values
if (nexatom_tt_get_telemetry(my_device, &health_status) == 0) {
    printf("Device Uptime: %u seconds\n", health_status.uptime_seconds);
    
    // Check if our external 10MHz master clock is locked securely
    if (health_status.sync_clock_requested && !health_status.sync_clock_locked) {
        printf("WARNING: External 10MHz Reference is NOT locked!\n");
    }
}
```
