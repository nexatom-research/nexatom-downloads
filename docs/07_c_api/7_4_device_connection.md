# Device Connection and State

Once a `nexatom_tt_handle` has been allocated via `nexatom_tt_create` (see Section 7.3), these functions manage the physical USB session, execute the initial handshakes with the FPGA, and expose hardware capabilities.

This module also provides deterministic checks for the FPGA bootloader state (`nexatom_tt_get_hardware_protocol_mode`) and internal state machine queries (`nexatom_tt_get_state`), which are critical for robust error recovery.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_connect` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t timeout_ms` | `nexatom_error_code_t` | Claims the USB interface, initializes kernel endpoints, and handshakes with the FPGA. Will block up to `timeout_ms`. |
| `nexatom_tt_disconnect` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Gracefully closes USB endpoints and terminates the physical session without freeing host memory. |
| `nexatom_tt_get_device_info` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_tt_info_t* info` | `nexatom_error_code_t` | Retrieves the static string metadata (Serial Number, firmware version) for the currently connected handle. |
| `nexatom_tt_get_capabilities` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_tt_capabilities_t* capabilities` | `nexatom_error_code_t` | Extracts hardware limits (like `max_histogram_bins` and `time_resolution_ps`) dynamically from the FPGA core. |
| `nexatom_tt_is_connected` | `[In] nexatom_tt_handle device`<br>`[Out] bool* is_connected` | `nexatom_error_code_t` | Queries if the physical USB session is currently established. |
| `nexatom_tt_get_state` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_tt_state_t* state` | `nexatom_error_code_t` | Retrieves the active software driver state (e.g., `CONNECTED`, `ACQUIRING`, `ERROR`). |
| `nexatom_tt_get_hardware_protocol_mode` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_hardware_protocol_mode_t* mode` | `nexatom_error_code_t` | Crucial diagnostic query. Returns whether the hardware is running the `RUNTIME` (data) or `BOOTLOADER` (flashing) image. |

### Data Structures & Enums

#### `nexatom_tt_state_t` (Enum)
Represents the internal state machine of the C++ driver backing the `nexatom_tt_handle`.
*   `0`: `DISCONNECTED`
*   `1`: `CONNECTING`
*   `2`: `CONNECTED` (Idle)
*   `3`: `CONFIGURING`
*   `4`: `ACQUIRING` (Data flowing)
*   `5`: `STOPPING`
*   `6`: `ERROR` (Requires teardown)

#### `nexatom_hardware_protocol_mode_t` (Enum)
Directly maps to the FPGA bitstream identity currently executing on the hardware.
*   `0`: `UNKNOWN` (Detecting or unpowered)
*   `1`: `RUNTIME` (Standard measurement mode)
*   `2`: `BOOTLOADER` (Recovery and flashing mode)

#### `nexatom_tt_capabilities_t` (Struct)
Passed by-value from the hardware core. Defines the physical limits of the connected NexatomTT model.
| Field | Type | Description |
| :--- | :--- | :--- |
| `num_channels` | `uint8_t` | Total available physical SMA inputs. |
| `max_count_rate` | `uint32_t` | Maximum throughput in Hz before FIFO saturation. |
| `time_resolution_ps` | `uint32_t` | Base bin resolution limit in picoseconds. |
| `max_threshold_mv` | `uint16_t` | Absolute maximum analog comparator voltage. |
| `max_histogram_bins` | `uint32_t` | Maximum hardware bins for TIHI (usually 1024). |
| `supports_calibration` | `bool` | True if the unit supports auto-thermal calibration. |
| `supports_file_saving` | `bool` | True if direct-to-disk binary saving is enabled. |
| `supports_external_clock` | `bool` | True if a 10MHz Reference In is available. |
| `supports_gating` | `bool` | True if hardware TTL gating is supported. |

### C Example: Connecting and Verifying Hardware

```c
nexatom_tt_state_t state;
nexatom_hardware_protocol_mode_t mode;
nexatom_tt_capabilities_t caps;

// 1. Establish the connection (5000ms timeout)
if (nexatom_tt_connect(my_device, 5000) == 0) {
    
    // 2. Ensure we are in RUNTIME mode before measuring
    nexatom_tt_get_hardware_protocol_mode(my_device, &mode);
    if (mode == NEXATOM_HARDWARE_PROTOCOL_MODE_BOOTLOADER) {
        printf("ERROR: Device is trapped in the bootloader!\n");
        nexatom_tt_disconnect(my_device);
        return -1;
    }

    // 3. Extract the device limits
    nexatom_tt_get_capabilities(my_device, &caps);
    printf("Connected! Native Resolution: %u ps\n", caps.time_resolution_ps);
    printf("Supported Channels: %u\n", caps.num_channels);
    
    // 4. Verify state machine
    nexatom_tt_get_state(my_device, &state);
    if (state == NEXATOM_STATE_CONNECTED) {
        printf("Device is idle and ready for configuration.\n");
    }
}
```
