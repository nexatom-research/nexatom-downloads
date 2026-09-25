# Telemetry and Register Diagnostics

To ensure long-term stability in continuous-monitoring environments, the NexatomTT hardware continually monitors its own health (uptime, sync clock lock status, FPGA thermals, FIFO buffer underruns).

Use callbacks or the supported request/getter API to observe available health fields. Configuration dumps are bounded versioned responses; inspect their reported/copied counts instead of assuming they contain every hardware register.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_telemetry` | `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms or disarms the periodic generation of telemetry packets from the FPGA. |
| `nexatom_tt_set_telemetry_mode` | `[In] nexatom_tt_handle device`<br>`[In] uint8_t mode` | `nexatom_error_code_t` | `1` = Manual on-request only. `2` = Periodic automatic transmission. |
| `nexatom_tt_request_telemetry` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Manually triggers the FPGA to emit a single telemetry payload. |
| `nexatom_tt_get_telemetry` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_telemetry_data_t* telemetry` | `nexatom_error_code_t` | Legacy request/wait getter returning the fixed telemetry record. |
| `nexatom_tt_get_telemetry_view_v1` | `[In] nexatom_tt_handle device`<br>`[In/Out] nexatom_tt_telemetry_view_v1_t* telemetry` | `nexatom_error_code_t` | Reads the latest cached versioned view; initialize size/version and inspect field availability. Does not itself request a new sample. |
| `nexatom_tt_request_config_dump`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests the runtime's defined configuration-register dump. The legacy callback copies at most 128 records; use the versioned view for the full supported response and check its record count. |

### Data Structures

#### `nexatom_telemetry_data_t`
The telemetry struct tracks critical hardware lifecycle statuses. It is populated either via polling (`nexatom_tt_get_telemetry`) or the async callback registered in Section 7.8.

| Field | Type | Description |
| :--- | :--- | :--- |
| `device_serial` | `uint32_t` | Numeric instrument serial from telemetry. It is unrelated to the FT601 USB serial and is what automatic reconnection compares; boards are selected by `connection_id`. |
| `firmware_version` | `uint16_t` | Running firmware version. |
| `hardware_revision` | `uint16_t` | PCB hardware revision identifier. |
| `sequence_number` | `uint16_t` | TELM sequence number; wraps and must be interpreted with freshness/session identity. |
| `telemetry_version` | `uint8_t` | TELM packet version identifier. |
| `active_channels_mask` | `uint8_t` | Legacy activity/status mask, not a substitute for the profile's public-channel authority. |
| `uptime_seconds` | `uint32_t` | Number of seconds the FPGA has been powered on. |
| `system_status_word` | `uint32_t` | Legacy status prefix, not the full common-runtime status payload or output-mode readback. |
| `mode_status_word` | `uint32_t` | Full 32-bit mode status register (bits [23:16] carry system error copy). |
| `histogram_errors_word` | `uint32_t` | Flags for dropped packets, saturated FIFOs, or histogram engine errors. |
| `temperature_raw` | `uint16_t` | Raw FPGA die temperature sensor readout. |
| `calibration_metadata` | `uint8_t` | Calibration status bits from the FPGA. |
| `temp_status_flags` | `uint8_t` | Temperature status flags (independent of calibration control). |
| `active_channels` | `uint64_t` | Extended container for decoded activity bits; its width does not authorize 64 channels. |
| `current_mode` | `uint8_t` | Raw current-mode byte from system status. |
| `sync_clock_requested` | `bool` | True if the host asked the PLL to lock to the 10MHz reference. |
| `sync_clock_active` | `bool` | True if the external sync path is actually selected by the FPGA. |
| `sync_clock_locked` | `bool` | **Critical Flag**: True if the external 10MHz reference is valid and successfully phase-locked. |
| `error_flags` | `uint8_t` | 8-bit aggregated error status. |
| `system_status` | `uint8_t` | 8-bit system state summary. |
| `_padding_sync` | `uint8_t[2]` | Alignment padding. |
| `temperature_celsius` | `float` | Device temperature in degrees Celsius (converted from `temperature_raw`). |

#### `nexatom_config_dump_data_t`
Used purely for debug, this struct is delivered exclusively via the configuration dump callback (Section 7.8). It contains raw 32-bit register addresses and their currently active values.

| Field | Type | Description |
| :--- | :--- | :--- |
| `version` | `uint32_t` | Config dump protocol version. |
| `register_count` | `uint32_t` | Hardware-reported total register pair count. |
| `copied_register_count` | `uint32_t` | The number of valid registers actually present in the `registers` array. |
| `reserved` | `uint32_t` | Reserved for future use. |
| `registers` | `struct[128]` | Array of `{ uint32_t address; uint32_t value; }` pairs detailing the raw FPGA configuration memory. Max 128 entries (`NEXATOM_CONFIG_DUMP_MAX_REGISTERS`). |

### C Example: Synchronous Health Check

```c
nexatom_telemetry_data_t health_status;

// Fragment: the authorized profile supports legacy telemetry.
// This getter owns its request/wait; do not add a guessed USB sleep.
if (nexatom_tt_get_telemetry(my_device, &health_status) == 0) {
    printf("Device Uptime: %u seconds\n", health_status.uptime_seconds);
    printf("FPGA Temperature: %.1f °C\n", health_status.temperature_celsius);
    
    // Check if our external 10MHz master clock is locked securely
    if (health_status.sync_clock_requested && !health_status.sync_clock_locked) {
        printf("WARNING: External 10MHz Reference is NOT locked!\n");
    }
}
```

For request-based polling call `nexatom_tt_request_telemetry` and observe a later versioned callback/view. Periodic reporting is output-dependent: current common runtime requires processed output, while explicit supported requests can respond in quiet mode. Do not start acquisition merely to obtain telemetry. Absent version-specific fields are unknown; the current-mode/status bytes must not be reinterpreted as output selection or calibration completion.

Use `nexatom_tt_set_config_dump_view_callback_v1` for the versioned configuration view. DTC-capable profiles additionally expose `nexatom_tt_apply_dtc_output_v1`, `nexatom_tt_get_dtc_status_v1`, `nexatom_tt_set_dtc_global_enable_v1` and `nexatom_tt_clear_dtc_status_v1`; initialize the matching records and inspect apply rejection/status rather than treating a request as applied. These functions do not establish DTC support on an otherwise unsupported profile.

<a id="transport-stop-record"></a>

### Transport-stop record

When the instrument's on-board data buffer fills, or its data transport hits a protocol error, the instrument stops the measurement, resets its data path and stays idle. It records each stop in `complete_error_flags_word` of the telemetry view (valid with `NEXATOM_TT_TELEMETRY_VIEW_HAS_COMPLETE_STATUS`):

| Mask | Meaning |
| :--- | :--- |
| `NEXATOM_TT_TELEMETRY_VIEW_ERROR_GLOBAL_EXPORTER_FATAL` (bit 8) | Set while any stop is recorded (ORed with the record). |
| `NEXATOM_TT_TELEMETRY_VIEW_ERROR_TRANSPORT_STOP_COUNT_MASK` (bits 23..16), `_SHIFT` 16 | Number of stops; saturates at 255. |
| `NEXATOM_TT_TELEMETRY_VIEW_ERROR_DDR_PROTOCOL_STOP` (bit 24) | At least one stop was caused by a protocol error rather than a full buffer. |

The record is sticky: it clears only with the peripheral reset (`nexatom_tt_reset_peripherals()`, SYSTEM_CONTROL bit 1). The SDK reports the word and does not restart the measurement; the application decides what to do.

```c
// Fragment: view is a nexatom_tt_telemetry_view_v1_t read with nexatom_tt_get_telemetry_view_v1().
if (view.validity_flags & NEXATOM_TT_TELEMETRY_VIEW_HAS_COMPLETE_STATUS) {
    uint32_t word = view.complete_error_flags_word;
    unsigned stops = (unsigned)((word & NEXATOM_TT_TELEMETRY_VIEW_ERROR_TRANSPORT_STOP_COUNT_MASK)
                                >> NEXATOM_TT_TELEMETRY_VIEW_ERROR_TRANSPORT_STOP_COUNT_SHIFT);
    if (stops > 0) {
        printf("Measurement stopped by the instrument %u time(s)%s\n", stops,
               (word & NEXATOM_TT_TELEMETRY_VIEW_ERROR_DDR_PROTOCOL_STOP) ? " (protocol error)" : "");
    }
}
```
