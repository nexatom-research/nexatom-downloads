# Calibration

To maintain sub-picosecond timing accuracy over long measurement campaigns, the NexatomTT hardware must periodically calibrate its internal delay lines to compensate for ambient thermal drift and internal die heating.

This module allows developers to trigger this calibration manually, or instruct the background C++ thread to execute it automatically based on elapsed time or measured temperature deltas (monitored via the telemetry engine).

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_trigger_calibration` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Synchronously halts hardware acquisition, runs the internal delay-line calibration sweep (~100ms), and resumes operation. |
| `nexatom_tt_configure_auto_calibration` | `[In] nexatom_tt_handle device`<br>`[In] float temp_celsius`<br>`[In] uint16_t time_minutes` | `nexatom_error_code_t` | Sets the delta thresholds for automatic calibration. e.g., trigger every `2.5` degrees of temperature change, or every `60` minutes. |
| `nexatom_tt_enable_auto_calibration` | `[In] nexatom_tt_handle device`<br>`[In] bool enable_temp`<br>`[In] bool enable_time` | `nexatom_error_code_t` | Toggles the active status of the automated background triggers independently. |
| `nexatom_tt_set_calibration_settings` | `[In] nexatom_tt_handle device`<br>`[In] float temp_celsius`<br>`[In] uint16_t time_minutes`<br>`[In] bool enable_temp`<br>`[In] bool enable_time` | `nexatom_error_code_t` | Convenience function that atomically sets both the thresholds and the enable flags in a single C API call. |

### C Example: Configuring Long-Term Stability

```c
// 1. Manually trigger a baseline calibration before the experiment starts
nexatom_tt_trigger_calibration(my_device);

// 2. We want to auto-calibrate if the FPGA die temp shifts by 2.0°C, 
//    or if 45 minutes have elapsed—whichever comes first.
float trigger_temp = 2.0f;
uint16_t trigger_time_mins = 45;

nexatom_tt_set_calibration_settings(
    my_device, 
    trigger_temp, 
    trigger_time_mins, 
    true,   // enable_temp
    true    // enable_time
);

printf("Auto-calibration armed. Device is ready for 24-hour measurement.\n");
```
