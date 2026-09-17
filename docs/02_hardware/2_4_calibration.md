## Calibration

The UTT810 relies on timing circuits whose response can change with temperature and operating conditions. Calibration support belongs to the resolved hardware contract. Startup/manual calibration and automatic temperature/time-trigger calibration are separate capabilities; a bootloader by itself does not establish either policy. Do not infer support merely from the presence of a C function or Python method.

Calibration calls are exposed in both the C API and the Python `NexatomDevice` class. The native library returns `NEXATOM_ERROR_NOT_SUPPORTED` (a `NexatomError` in Python) when the resolved image does not support an operation.

### Manual calibration trigger

Calibration can be requested on demand through the native API. Perform it between acquisitions and inspect the device's telemetry for its progress/completion. A successful trigger call means the command was accepted, not that calibration has finished; the SDK does not establish a universal completion time or a guarantee of uninterrupted measurement during calibration.

#### C API

```c
nexatom_error_code_t nexatom_tt_trigger_calibration(nexatom_tt_handle device);
```

#### Python

```python
device.enable_system(False)  # Stop the measurement before requesting calibration.
device.trigger_calibration()
# Observe the device's telemetry before starting the next measurement.
```

### Auto-calibration configuration

Images with the established automatic-recalibration register contract can trigger it from elapsed time or temperature drift. The detailed settings below apply to those images. In preview.8 the resolved `UTT_16_8_V1` application contract rejects `configure_auto_calibration`, `enable_auto_calibration` and `set_calibration_settings` without issuing incompatible register writes. This restriction must not be generalized to every bootloader-based Zynq or Kintex image. A startup-calibration feature alone does not imply automatic recalibration support.

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

Alternatively, apply all settings in a single call on a supporting image:
```c
nexatom_tt_set_calibration_settings(
    device,
    5.0f,     /* temp_celsius */
    120,      /* time_minutes */
    true,     /* enable_temp */
    false     /* enable_time */
);
```

#### Python

```python
# For an image supporting automatic recalibration; not part of normal startup.
device.set_calibration_settings(
    temp_celsius=5.0,
    time_minutes=120,
    enable_temp=True,
    enable_time=False,
)
```

The legacy temperature conversion uses the magnitude of a finite Celsius value, truncates to ADC codes and saturates at the 12-bit limit. The interval saturates at 4095 minutes; zero disables that threshold. Use reasonable experiment-specific values rather than relying on saturation. Configure thresholds before separately enabling triggers, or use the combined setter to apply them together.

### Calibration status via telemetry

The legacy telemetry projection exposes `calibration_metadata` and `temp_status_flags`. Their bit definitions below are useful when inspecting a legacy image; modern applications should prefer the versioned telemetry view and its validity fields so an unavailable field is not confused with a false/zero state. See [Telemetry and Diagnostics](2_11_telemetry.md).

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

On a legacy image, this warning can indicate that a new calibration should be considered between measurements. It is a firmware diagnostic threshold, not a thermal operating limit or a guarantee of restored precision. Modern images may report calibration/temperature differently through their versioned telemetry view.

### Calibration data structure

The public SDK retains the following calibration record (`NexatomCalibrationData` in Python). Preview.8 does not expose a public callback/getter that fills it after `trigger_calibration()`. An application must not allocate this record and treat its default values as a calibration result; use supported telemetry/status evidence instead.

#### `nexatom_calibration_data_t`

| Field | Type | Description |
|---|---|---|
| `calibration_time_ms` | `uint64_t` | Milliseconds since the Unix epoch |
| `is_valid` | `bool` | True if calibration succeeded |
| `validation_message` | `char[256]` | Diagnostic message |
