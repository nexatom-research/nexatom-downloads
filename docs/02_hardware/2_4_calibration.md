## Calibration

The UTT810 relies on precise timing circuits that can drift marginally with significant temperature changes over extended periods. The hardware includes an internal calibration engine to compensate for these drifts.

> **Python Wrapper Support.** Calibration configuration is currently only exposed in the native C API. It is not yet wrapped in the `NexatomDevice` Python class.

### [Manual calibration trigger](2_4_calibration.md#manual-calibration-trigger)

Calibration can be triggered on demand. When triggered, the FPGA briefly suspends data acquisition to perform the calibration sequence.

#### C API

```c
nexatom_error_code_t nexatom_tt_trigger_calibration(nexatom_tt_handle device);
```

### [Auto-calibration configuration](2_4_calibration.md#auto-calibration-configuration)

To maintain uninterrupted precision during long experiments, the device supports automatic calibration triggers based on elapsed time or ambient temperature drift.

#### Trigger conditions

- **Temperature drift:** Triggers calibration when the internal XADC senses a temperature change exceeding the defined threshold (`temp_celsius`).
- **Time interval:** Triggers calibration at a fixed interval (`time_minutes`).

#### C API

Configure the thresholds:
```c
nexatom_tt_configure_auto_calibration(
    device,
    5.0f,     /* 5.0 °C delta trigger */
    120       /* 120 minutes interval trigger */
);
```

Enable the subsystems:
```c
nexatom_tt_enable_auto_calibration(
    device,
    true,     /* enable temperature trigger */
    false     /* disable time trigger */
);
```

Alternatively, apply all settings in a single call:
```c
nexatom_tt_set_calibration_settings(
    device,
    5.0f,     /* temp_celsius */
    120,      /* time_minutes */
    true,     /* enable_temp */
    false     /* enable_time */
);
```

### [Calibration status via telemetry](2_4_calibration.md#calibration-status-via-telemetry)

Calibration state and history are broadcast within the realtime telemetry stream. When polling telemetry data (`nexatom_tt_get_telemetry()`), the `calibration_metadata` and `temp_status_flags` bytes provide insight into the calibration engine's state.

#### Calibration metadata (`calibration_metadata`)

The `calibration_metadata` byte contains 8 boolean flags mapped to the following constants:

| Bitmask | Constant | Description |
|---------|---|---|
| `0x80u` | `NEXATOM_TELM_CALIB_META_START_SEEN` | Calibration start sequence detected |
| `0x40u` | `NEXATOM_TELM_CALIB_META_DONE_SEEN` | Calibration completion detected |
| `0x20u` | `NEXATOM_TELM_CALIB_META_TEMP_TRIGGER_SEEN` | Last calibration was triggered by temperature |
| `0x10u` | `NEXATOM_TELM_CALIB_META_TIME_TRIGGER_SEEN` | Last calibration was triggered by timer |
| `0x08u` | `NEXATOM_TELM_CALIB_META_PENDING_LEVEL` | A calibration request is currently pending |
| `0x04u` | `NEXATOM_TELM_CALIB_META_ACTIVE_LEVEL` | The calibration engine is currently active |
| `0x02u` | `NEXATOM_TELM_CALIB_META_AUTO_ENABLED_LEVEL` | Auto-calibration is enabled in hardware |
| `0x01u` | `NEXATOM_TELM_CALIB_META_LAST_CALIB_AUTO` | The most recent calibration was automatic |

#### Thermal warning flag (`temp_status_flags`)

The `temp_status_flags` byte in the telemetry struct includes a fixed thermal warning flag:

| Bitmask | Constant | Description |
|---------|---|---|
| `0x02u` | `NEXATOM_TELM_TEMP_STATUS_FIXED_WARNING` | Fixed thermal warning. Indicates the temperature has drifted by roughly ≥5°C from the baseline established at the last calibration. |

If this bit is asserted and auto-calibration is disabled, the user should manually trigger a calibration to restore optimal timing precision.

### [Calibration data structure](2_4_calibration.md#calibration-data-structure)

The SDK provides a standard structure for communicating calibration timestamp and validation status.

#### `nexatom_calibration_data_t`

| Field | Type | Description |
|---|---|---|
| `calibration_time_ms` | `uint64_t` | Milliseconds since the Unix epoch |
| `is_valid` | `bool` | True if calibration succeeded |
| `validation_message` | `char[256]` | Diagnostic message |
