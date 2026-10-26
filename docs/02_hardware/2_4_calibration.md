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

- **What it calibrates.** The call fires one calibration of the input timing (TDC). The pulse outputs of the UTT160810 are calibrated separately (see [Pulse outputs](2_13_dtc_pulse_outputs.md#dtc-status)).
- **Automatic calibration stays as configured.** On images with automatic calibration, the trigger writes the automatic-calibration settings in force together with the calibrate-now request, so a manual trigger does not switch automatic calibration off.
- **Duration.** On the UTT810 every input is dark for about 0.63 s while a calibration runs (measured with firmware `0x0103`); all inputs go dark together and come back together. On the UTT160810 the board holds its data (raw tags, count rates and Fast TIHI) for the whole calibration; its duration has not been measured and is estimated at about 1.2 s. Pulse-output settings survive a calibration, but triggers that arrive during it give no pulse.
- **During a file save.** While a file save of any kind runs, `trigger_calibration()` returns `NEXATOM_ERROR_DEVICE_BUSY` and writes nothing. A run, a live histogram or pulse outputs without a save do not refuse it.
- **Log.** Each request is logged as `LIVE CALIBRATION_STARTED cause=manual src=sdk` (log category `LIVE_CHANGE`).

### Auto-calibration configuration

Images with the established automatic-recalibration register contract can trigger it from elapsed time or temperature drift. The detailed settings below apply to those images. The resolved `UTT_16_8_V1` application contract rejects `configure_auto_calibration`, `enable_auto_calibration` and `set_calibration_settings` with `NEXATOM_ERROR_NOT_SUPPORTED` ("Automatic calibration is not supported by the resolved modern device profile") and issues no register writes. This restriction must not be generalized to every bootloader-based Zynq or Kintex image. A startup-calibration feature alone does not imply automatic recalibration support.

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

The threshold the board uses is `trunc(|ΔT| × 8.14) / 8.14` °C: 1 °C becomes 0.98 °C, 5 °C becomes 4.91 °C, 20 °C becomes 19.90 °C. A threshold of 0 °C fires on any temperature change.

Each call changes only its own part of the policy:

| Call | Changes | Keeps |
|---|---|---|
| `configure_auto_calibration()` | Temperature threshold and interval | The temperature and time enables in force |
| `enable_auto_calibration()` | The temperature and time enables | The threshold and interval in force |
| `set_calibration_settings()` | All four settings | — (it does not fire a calibration) |

<a id="policy-after-connect"></a>

#### Send the policy after every connect

Connect switches automatic calibration off on every image that has it, before control is granted, so a policy that an earlier session left on the board does not survive. Send your automatic-calibration policy after every connect, including after a re-plug on the same handle.

<a id="calibration-during-a-file-save"></a>

#### Calibration during a file save (UTT810)

A calibration darkens the inputs, so automatic calibration waits while a file save of any kind runs and is caught up when the save ends:

- When the first file saving is enabled, the SDK holds automatic calibration: it writes the policy with the temperature and time enables clear; threshold and interval stay.
- Policy calls during a save return `NEXATOM_SUCCESS`; the SDK stores the policy and writes it when the save ends.
- When the last save has ended and its result is final, the SDK writes the policy back, and fires a calibration at once when one fell due during the save (by time or by temperature).
- A session that ends without a disconnect writes nothing; the next connect switches automatic calibration off, and your policy after connect restores it.

`get_calibration_hold_state()` (C: `nexatom_tt_get_calibration_hold_state()`, 64-byte record) reads the SDK's own state and never touches the device:

| Field | Meaning |
|---|---|
| `held` | 1 = the enables are held clear on the board |
| `due`, `due_reason` | 1 = a calibration is due; `NEXATOM_TT_CALIBRATION_DUE_TIME` (1) or `_TEMPERATURE` (2) |
| `busy_reasons` | `NEXATOM_TT_CALIBRATION_BUSY_RECORDING` while a file save runs; "calibrate now" is refused exactly while this bit is set |
| `flags` | `NEXATOM_TT_CALIBRATION_HOLD_FLAG_FAULT` (the calibration stayed active too long), `_BASELINE_VALID` |
| `policy_word` | The policy last set in this session; 0 if none |
| `held_for_ms`, `since_last_start_ms` | How long the hold has lasted; time since the last calibration start (or connect) |
| `baseline_temperature_code`, `current_temperature_code` | Temperature codes at the last start and now |
| `incident_id` | The incident raised for a calibration fault; 0 without a fault |

The UTT160810 has no automatic calibration, so the call returns `NEXATOM_ERROR_NOT_SUPPORTED` there.

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
| `0x08u` | `NEXATOM_TELM_CALIB_META_PENDING_LEVEL` | Legacy telemetry layouts only: a calibration request is pending. Reserved and zero in current telemetry (TELM v10) |
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

The public SDK retains the following calibration record (`NexatomCalibrationData` in Python). The SDK still does not expose a public callback/getter that fills it after `trigger_calibration()`. An application must not allocate this record and treat its default values as a calibration result; use supported telemetry/status evidence instead.

#### `nexatom_calibration_data_t`

| Field | Type | Description |
|---|---|---|
| `calibration_time_ms` | `uint64_t` | Milliseconds since the Unix epoch |
| `is_valid` | `bool` | True if calibration succeeded |
| `validation_message` | `char[256]` | Diagnostic message |
