# 2.4 Calibration

## Manual calibration trigger

Use `trigger_calibration()` only when the native capability permits it. Successful return means the command was accepted; it does not mean calibration completed synchronously or within a fixed 100 ms delay.

## Auto-calibration configuration

Python exposes `configure_auto_calibration(temp_celsius, time_minutes)`, `enable_auto_calibration(enable_temp, enable_time)` and the combined `set_calibration_settings(...)`. The C functions have the same units and corresponding names. Choose trigger settings for the actual instrument and experiment, not a generic temperature recommendation.

## Calibration status via telemetry

Request telemetry and examine the versioned view's available fields. Calibration metadata differs across telemetry layouts; absent fields must not be interpreted as successful/failed calibration. Preserve timestamps/sequence and profile identity with diagnostic observations.

## Calibration data structure

`NexatomCalibrationData` / `nexatom_calibration_data_t` describes a public record; its existence alone does not imply that every model exposes a calibration result callback or getter. Use the actual supported API and telemetry view. Numeric control units do not establish timing accuracy after calibration.

[Device operation](index.md) · [C calibration API](../07_c_api/7_16_calibration.md)
