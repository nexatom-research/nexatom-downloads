## Telemetry and Diagnostics

The UTT810 continuously monitors its internal state, temperature, and hardware errors. This diagnostic information is delivered to the host via telemetry packets.

### [Telemetry modes](2_11_telemetry.md#telemetry-modes)

The telemetry subsystem operates in one of three modes, which determines when the FPGA emits telemetry packets over the USB data path.

| Mode Value | Description |
|---|---|
| `0` | **Disabled:** No telemetry packets are emitted. |
| `1` | **On-Request:** Telemetry is only emitted when explicitly requested by the host. |
| `2` | **Periodic:** Telemetry is emitted automatically at approximately 1 Hz. |

*(Note: Mode 3 is reserved and will be rejected).*

#### C API
```c
/* Set the active telemetry mode */
nexatom_tt_set_telemetry_mode(device, 2);

/* Legacy enable toggle (maps to periodic if true, disabled if false) */
nexatom_tt_enable_telemetry(device, true);
```

#### Python
```python
device.set_telemetry_mode(mode=2)
device.enable_telemetry(enable=True)
```

### [Requesting telemetry on-demand](2_11_telemetry.md#requesting-telemetry-on-demand)

When the telemetry mode is set to On-Request (`1`) or Periodic (`2`), the host can force the immediate emission of a telemetry packet. This is useful for capturing the exact device state synchronized with a specific software event.

#### C API
```c
nexatom_tt_request_telemetry(device);
```

#### Python
```python
device.request_telemetry()
```

### [Polling latest telemetry](2_11_telemetry.md#polling-latest-telemetry)

While telemetry packets invoke the `nexatom_telemetry_callback` (if registered), the SDK also caches the most recently received telemetry frame. The host can poll this cached state synchronously at any time.

#### C API
```c
nexatom_telemetry_data_t telemetry;
nexatom_tt_get_telemetry(device, &telemetry);
```

#### Python
```python
telemetry = device.get_telemetry()
print(f"Uptime: {telemetry.uptime_seconds} s")
```

### [Telemetry data fields](2_11_telemetry.md#telemetry-data-fields)

The `nexatom_telemetry_data_t` (C) or `NexatomTelemetryData` (Python) structure exposes comprehensive diagnostic fields:

| Field | Type | Description |
|---|---|---|
| `device_serial` | `uint32_t` | Numeric portion of the device serial number |
| `firmware_version` | `uint16_t` | Active firmware version |
| `hardware_revision` | `uint16_t` | Active hardware revision |
| `uptime_seconds` | `uint32_t` | Total elapsed seconds since power-on |
| `temperature_celsius` | `float` | XADC temperature reading in Celsius |
| `system_status_word` | `uint32_t` | Full hardware status register dump |
| `mode_status_word` | `uint32_t` | Full operating mode status register dump |
| `histogram_errors_word`| `uint32_t` | Bitmask of active histogram errors/overflows |
| `active_channels_mask` | `uint8_t` | Bitmask of currently enabled physical channels |
| `calibration_metadata` | `uint8_t` | Calibration engine state (see Section 2.4.3) |
| `temp_status_flags` | `uint8_t` | Contains the `NEXATOM_TELM_TEMP_STATUS_FIXED_WARNING` flag indicating thermal drift |
| `sync_clock_*` | `bool` | Booleans indicating external clock state (requested, active, locked) |

### [Configuration register dump](2_11_telemetry.md#configuration-register-dump)

For deep diagnostic tracing, the host can request the device to dump the contents of all its active configuration registers.

> **Python Wrapper Support.** Configuration register dumping is currently only exposed in the native C API.

When requested, the hardware emits a specialized telemetry frame that is delivered asynchronously to the `nexatom_config_dump_callback`.

#### C API
```c
nexatom_tt_request_config_dump(device);
```

#### Data structure (`nexatom_config_dump_data_t`)
The payload contains an array of `nexatom_config_register_value_t` structs (up to `128` pairs), each containing:
*   `address` (`uint32_t`): Register memory address
*   `value` (`uint32_t`): Current 32-bit register value
