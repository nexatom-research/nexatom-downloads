# Device Connection and State

Once a `nexatom_tt_handle` has been allocated via `nexatom_tt_create` (see Section 7.3), these functions manage the physical USB session, execute the initial handshakes with the FPGA, and expose hardware capabilities.

This module also provides deterministic checks for the FPGA bootloader state (`nexatom_tt_get_hardware_protocol_mode`) and internal state machine queries (`nexatom_tt_get_state`), which are critical for robust error recovery.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_connect` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t timeout_ms` | `nexatom_error_code_t` | Opens/detects the session; use runtime connect for measurement readiness. |
| `nexatom_tt_connect_runtime` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t timeout_ms` | `nexatom_error_code_t` | Native discovers runtime/service, boots an existing valid slot when needed and waits for authorized runtime readiness on this handle. |
| `nexatom_tt_disconnect` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Gracefully closes USB endpoints and terminates the physical session without freeing host memory. |
| `nexatom_tt_get_device_info` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_tt_info_t* info` | `nexatom_error_code_t` | Retrieves the static string metadata (Serial Number, firmware version) for the currently connected handle. |
| `nexatom_tt_get_capabilities` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_tt_capabilities_t* capabilities` | `nexatom_error_code_t` | Returns native capability values; combine these with the authorized device profile rather than treating them as measured physical specifications. |
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
Filled through the caller-provided output pointer. These are API capability values; the profile's effective public mask, features and limits establish current control authority.
| Field | Type | Description |
| :--- | :--- | :--- |
| `num_channels` | `uint8_t` | Public channel extent; not the physical lane count or authority mask. |
| `max_count_rate` | `uint32_t` | Capability rate value; not a measured no-loss throughput guarantee. |
| `time_resolution_ps` | `uint32_t` | API timing capability in ps, not a measurement of absolute accuracy. |
| `max_threshold_mv` | `uint16_t` | Maximum supported threshold setting, not an electrical absolute maximum. |
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

// Fragment: my_device is created and its owner handles cleanup on every exit.
if (nexatom_tt_connect_runtime(my_device, 20000) == NEXATOM_SUCCESS) {
    
    // 2. Ensure we are in RUNTIME mode before measuring
    if (nexatom_tt_get_hardware_protocol_mode(my_device, &mode) != NEXATOM_SUCCESS)
        return -1;
    if (mode != NEXATOM_HARDWARE_PROTOCOL_MODE_RUNTIME) {
        printf("ERROR: Runtime is not ready.\n");
        nexatom_tt_disconnect(my_device);
        return -1;
    }

    // 3. Extract the device limits
    if (nexatom_tt_get_capabilities(my_device, &caps) != NEXATOM_SUCCESS)
        return -1;
    printf("Connected! Native Resolution: %u ps\n", caps.time_resolution_ps);
    printf("Supported Channels: %u\n", caps.num_channels);
    
    // 4. Verify state machine
    if (nexatom_tt_get_state(my_device, &state) != NEXATOM_SUCCESS)
        return -1;
    if (state == NEXATOM_STATE_CONNECTED) {
        printf("Device is idle and ready for configuration.\n");
    }
}
```

### Profile and application contract

Before measurement use `nexatom_tt_get_device_profile_v1(device, &profile)`. Initialize `struct_size` and `struct_version` with the matching header constants. Inspect control/service flags, `effective_public_tdc_mask`, supported output mask, feature flags, `max_channel_input_delay_ps` and test-pulse clock/bounds. A state enum or physical channel count is not a substitute for authority.

`nexatom_tt_get_application_contract` exposes native's resolved application contract. `nexatom_tt_apply_inventory_profile_v1`, `nexatom_tt_apply_manual_profile_v1`, `nexatom_tt_clear_profile_evidence_v1`, `nexatom_tt_configure_runtime_compatibility` and `nexatom_tt_set_bootloader_probe_enabled` are controlled compatibility tools, not ordinary startup prerequisites. Use their exact declarations from the header; do not override an unknown model by guessing capabilities.

Use the complete C/C++ acquisition templates for checked cleanup on every failure path. Connection success does not itself enable the application's measurement.
