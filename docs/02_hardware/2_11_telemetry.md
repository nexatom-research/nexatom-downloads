## Telemetry and Diagnostics

The UTT810 reports operating state, temperature and hardware diagnostics through telemetry. The native API decodes the received layout automatically. Applications consume the common telemetry view and its validity flags rather than asking users to select a telemetry version.

### Telemetry modes

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

### Requesting telemetry on-demand

In a supported runtime telemetry mode, the host can request a packet. This is useful for observing state around a software action, but a request is not an exactly simultaneous hardware snapshot of that action. `request_telemetry()` reports command admission and does not wait for the response.

#### C API
```c
nexatom_tt_request_telemetry(device);
```

#### Python
```python
device.request_telemetry()
```

### Polling latest telemetry

There are two distinct read operations. `get_telemetry()` requests a fresh response and waits (the C/Python facade uses a one-second response budget). `get_telemetry_view()` copies the latest already-published versioned view without requesting hardware traffic. Register a callback or request telemetry separately when using the cache-only view.

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

For a cache-only dashboard or control display:

```python
from nexatomtt import NEXATOM_TT_TELEMETRY_VIEW_HAS_COMMON_STATUS

view = device.get_telemetry_view()
if view is None:
    print("No telemetry has been published in this runtime session yet")
elif int(view.validity_flags) & NEXATOM_TT_TELEMETRY_VIEW_HAS_COMMON_STATUS:
    print(f"Uptime: {view.uptime_seconds} s")
    print(f"Temperature: {view.temperature_celsius:.2f} C")
else:
    print("This frame does not provide the common status fields")
```

The C equivalent is `nexatom_tt_get_telemetry_view_v1()`. Zero-initialize its output record and set `struct_size` and `struct_version` first; `NEXATOM_ERROR_TIMEOUT` means there is no cached view in the current runtime session. A cached view is not necessarily a response to a later command, so use sequence/callback progress when establishing freshness.

### Telemetry data fields

The retained `nexatom_telemetry_data_t` (C) or `NexatomTelemetryData` (Python) record contains the following legacy projection. New cross-model applications should use `nexatom_tt_telemetry_view_v1_t` / `NexatomTelemetryViewV1` for explicit validity and full-width status/identity information.

| Field | Type | Description |
|---|---|---|
| `device_serial` | `uint32_t` | Firmware identity field; not the USB bridge's serial string |
| `firmware_version` | `uint16_t` | Active firmware version |
| `hardware_revision` | `uint16_t` | Active hardware revision |
| `uptime_seconds` | `uint32_t` | Runtime-reported uptime; may restart with a firmware transition |
| `temperature_celsius` | `float` | XADC temperature reading in Celsius |
| `system_status_word` | `uint32_t` | Full hardware status register dump |
| `mode_status_word` | `uint32_t` | Full operating mode status register dump |
| `histogram_errors_word`| `uint32_t` | Bitmask of active histogram errors/overflows |
| `active_channels_mask` | `uint8_t` | Public-channel projection; do not use it as a physical enable mask or control authorization |
| `calibration_metadata` | `uint8_t` | Calibration engine state (see Section 2.4.3) |
| `temp_status_flags` | `uint8_t` | Contains the `NEXATOM_TELM_TEMP_STATUS_FIXED_WARNING` flag indicating thermal drift |
| `sync_clock_*` | `bool` | Booleans indicating external clock state (requested, active, locked) |

#### Versioned view: check validity before reading a group

| Validity flag | Fields it makes meaningful |
|---|---|
| `NEXATOM_TT_TELEMETRY_VIEW_HAS_HEADER` | Framed telemetry header/version information |
| `NEXATOM_TT_TELEMETRY_VIEW_HAS_COMMON_STATUS` | Documented common status, uptime, temperature and sync projection |
| `NEXATOM_TT_TELEMETRY_VIEW_HAS_IDENTITY_MIRROR` | Model/image identity, layout and capability mirror |
| `NEXATOM_TT_TELEMETRY_VIEW_HAS_COMPLETE_STATUS` | Complete system/error words and physical activity diagnostics |
| `NEXATOM_TT_TELEMETRY_VIEW_HAS_DTC_STATUS` | Digital timing output status/rejection fields |
| `NEXATOM_TT_TELEMETRY_VIEW_UNKNOWN_VERSION` | A framed version whose full layout is not known to this SDK |

Fields absent from a frame are zero with their validity bit clear. Thus zero temperature, no error flags or an inactive output mask must not be interpreted as measured facts unless the corresponding field group is valid. Similarly, a physical mask or mirrored feature bit is diagnostic information; `device.get_device_profile()` remains the source of runtime control authority.

For example, a valid complete status view can distinguish “all TDC calibration ready” from a TDC calibration error using the documented `NEXATOM_TT_TELEMETRY_VIEW_SYSTEM_*` and `NEXATOM_TT_TELEMETRY_VIEW_ERROR_*` masks. The complete reference, including DTC fields, is in [Telemetry C API](../07_c_api/7_15_telemetry.md).

The same word carries the sticky [transport-stop record](../07_c_api/7_15_telemetry.md#transport-stop-record): if the instrument's on-board buffer fills or its data transport fails, the instrument stops the measurement and counts the stop there until the next peripheral reset. The SDK reports it and does not restart the measurement.

### Configuration register dump

For deep diagnostic tracing, the host can request the device to dump the contents of all its active configuration registers.

Configuration dumps are available through C and Python callbacks. This is an inspection interface; callers do not need to build register dumps or issue arbitrary register writes for normal measurement setup.

When requested, the hardware emits a specialized telemetry frame that is delivered asynchronously to the `nexatom_config_dump_callback`.

#### C API
```c
nexatom_tt_request_config_dump(device);
```

#### Data structure (`nexatom_config_dump_data_t`)
The retained fixed-size payload contains up to `128` `nexatom_config_register_value_t` pairs, each containing:
*   `address` (`uint32_t`): Register memory address
*   `value` (`uint32_t`): Current 32-bit register value

`register_count` is the hardware-reported count; `copied_register_count` is the number actually copied into this fixed array. Iterate only to `copied_register_count`, not to the larger reported count. Use the versioned configuration-dump view when all records are needed.

#### Python

```python
def on_config_dump(data):
    pairs = [(int(data.registers[i].address), int(data.registers[i].value))
             for i in range(int(data.copied_register_count))]
    print("Reported:", int(data.register_count), "copied:", len(pairs))
    # Queue/copy pairs here if they are needed after this callback returns.

device.set_config_dump_callback(on_config_dump)
device.request_config_dump()
```

At the C boundary, the full configuration-dump view contains a borrowed pointer to register pairs; copy those records before returning if you need them later. Python's `set_config_dump_view_callback()` wrapper supplies an owned copy of the view and its records, which the callback can retain. See [Callback Thread Safety and Data Lifetime](../06_in_depth_guides/6_5_callback_thread_safety_and_data_lifetime.md) for both cases.
