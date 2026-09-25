# Time Interval Histogram (TIHI)

The Time Interval Histogram (TIHI) module is the primary engine for Time-Correlated Single Photon Counting (TCSPC) and fluorescence lifetime measurements. It records the temporal delay between a "Start" event and subsequent "Stop" events, accumulating them into a high-speed hardware array.

This module supports bidirectional (Start-Stop and Stop-Start) accumulation, multi-stop architectures, and aggressive host-side background subtraction and curve fitting (Exponential, Gaussian, Lorentzian).

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_time_histogram` | `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms or disarms the TIHI measurement engine inside the FPGA. |
| `nexatom_tt_set_time_histogram_channels` | `[In] nexatom_tt_handle device`<br>`[In] uint8_t start_channel`<br>`[In] uint8_t stop_channel` | `nexatom_error_code_t` | Maps the physical inputs for the Start-Stop logic. |
| `nexatom_tt_set_time_histogram_bidirectional`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Enables measuring negative time differences (Stop arriving before Start). |
| `nexatom_tt_set_time_histogram_first_stop_mode`| `[In] nexatom_tt_handle device`<br>`[In] bool first_stop` | `nexatom_error_code_t` | If true, only the first Stop photon after a Start is counted (classic TDC). If false, multi-stop is enabled. |
| `nexatom_tt_set_time_histogram_bin_width`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t bin_width_ps` | `nexatom_error_code_t` | Sets the hardware bin width resolution in picoseconds. |
| `nexatom_tt_set_time_histogram_num_bins` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t num_bins` | `nexatom_error_code_t` | Sets the histogram length up to the queried `max_histogram_bins` capability and the public array limit. |
| `nexatom_tt_start_time_histogram` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests START for the enabled TIHI engine; observe measurement results separately. |
| `nexatom_tt_stop_time_histogram` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests stopping TIHI. Exactly one `STOPPED` result follows within 2 s; successful return does not promise its callback already ran. |
| `nexatom_tt_set_result_span` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_result_processor_t processor`<br>`[In] nexatom_result_span_t span`<br>`[In] uint32_t block_ms` | `nexatom_error_code_t` | Chooses what a published result covers. See [result model](#result-model). |
| `nexatom_tt_set_run_end` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_result_processor_t processor`<br>`[In] nexatom_run_end_kind_t kind`<br>`[In] uint64_t value` | `nexatom_error_code_t` | Ends the run by itself after a measurement time or an event count. |
| `nexatom_tt_clear_result` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_result_processor_t processor` | `nexatom_error_code_t` | Discards what the host has summed; the hardware keeps running. |
| `nexatom_tt_set_tihi_background_method` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_tihi_background_method_t method` | `nexatom_error_code_t` | Sets the algorithm used to subtract baseline noise (`NEXATOM_TIHI_BG_NONE` = 0, `NEXATOM_TIHI_BG_USER_CONSTANT` = 1, `NEXATOM_TIHI_BG_USER_REGION` = 2). |
| `nexatom_tt_set_tihi_user_background_value` | `[In] nexatom_tt_handle device`<br>`[In] uint16_t value` | `nexatom_error_code_t` | Injects a manual background subtraction floor (range 0–65535). Only effective when background method is `NEXATOM_TIHI_BG_USER_CONSTANT`. |
| `nexatom_tt_set_tihi_signal_region` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t start_bin`<br>`[In] uint32_t end_bin` | `nexatom_error_code_t` | Defines the Region of Interest (ROI) boundaries for SNR optimization. |
| `nexatom_tt_set_tihi_background_region` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t start_bin`<br>`[In] uint32_t end_bin` | `nexatom_error_code_t` | Defines the bin region used by the `USER_REGION` background method. |
| `nexatom_tt_enable_tihi_fitting` | `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Instructs the host background thread to run Levenberg-Marquardt fitting. |
| `nexatom_tt_set_tihi_fitting_parameters` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t min_counts`<br>`[In] double convergence_threshold`<br>`[In] uint32_t max_iterations` | `nexatom_error_code_t` | Configures the curve fitter: `min_counts` sets the minimum photon count threshold before a fit is attempted, `convergence_threshold` sets the chi-squared tolerance, and `max_iterations` caps the solver loop. |
| `nexatom_tt_set_tihi_fitting_model` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_fitting_model_t model` | `nexatom_error_code_t` | Assigns the math model (`NEXATOM_FIT_EXPONENTIAL`, `NEXATOM_FIT_GAUSSIAN`, `NEXATOM_FIT_LORENTZIAN`, etc.). |

### Data Structures: `nexatom_tihi_callback_data_t`

When the TIHI engine dispatches a frame, it passes this massive structure by value. The arrays are contiguous for cache efficiency.

| Field | Type | Description |
| :--- | :--- | :--- |
| `num_bins` / `bin_width_ps` | `uint32_t` | Active bins limit and temporal resolution per bin. |
| `start_channel` / `stop_channel` | `uint8_t` | The hardware channel maps used to generate this curve. |
| `acquisition_done_status` | `nexatom_acquisition_done_status_t` | Completion/status byte. Do not treat the historical zero-valued normal alias as terminal success: zero can mean Running. Validate the documented manual/count/duration terminal statuses. |
| `_padding1` | `uint8_t[1]` | **Required 1-byte alignment padding.** |
| `total_counts` | `uint64_t` | Aggregate sum of all counts across the histogram array. |
| `mean_time_ps` / `std_dev_ps` | `double` | Statistical estimates of arrival time and distribution width. |
| `peak_bin_index` / `peak_counts` | `uint32_t` | Bin mode location and height. |
| `fwhm_ps` | `double` | Full Width Half Maximum estimate of the pulse. |
| `fitting.attempted` | `uint8_t` | `1` if fitting was attempted on this data. |
| `fitting.converged` | `uint8_t` | Returns `1` if the C++ math model successfully fit the data. |
| `fitting.fitted_curve_available` | `uint8_t` | `1` if `fitted_curve` array contains valid fit results. |
| `fitting.model_type` | `char[32]` | String identifying the model (e.g., `"exponential"`). |
| `fitting.chi_squared` | `double` | Chi-squared goodness-of-fit statistic. |
| `fitting.r_squared` | `double` | R² coefficient of determination. |
| `fitting.reduced_chi_squared` | `double` | Chi²/DOF normalized quality metric. |
| `fitting.degrees_of_freedom` | `uint32_t` | n_data − n_parameters. |
| `fitting.iterations` | `uint32_t` | Number of Levenberg-Marquardt iterations needed. |
| `fitting.exp_lifetime_ns` | `double` | Exponential model: decay lifetime in nanoseconds. |
| `fitting.gauss_center_ns` / `gauss_width_ns` | `double` | Gaussian model: peak center and sigma in nanoseconds. |
| `fitting.fitted_curve_size` | `uint32_t` | Number of valid points in the fitted curve. |
| `fitting.fitted_curve` | `float[1024]` | Predicted Y-values matching the histogram bins for GUI overlay plotting. |
| `roi.enabled` | `uint8_t` | `1` if ROI windowing was applied to the statistics above. |
| `roi.start_bin` / `roi.end_bin` | `uint32_t` | ROI boundaries (start inclusive, end exclusive). |
| `roi.roi_counts` | `uint64_t` | Total photon counts within the ROI window. |
| `background_subtracted` | `uint8_t` | `1` if `bg_method` was executed. |
| `bg_method` | `uint8_t` | Background estimation method used. |
| `background_level_per_bin` | `float` | Counts/bin that were subtracted (0 if no subtraction). |
| `packets_accumulated` | `uint32_t` | Hardware batches in this result. |
| `bins` | `uint64_t[1024]` | **The time-binned photon counts. Only `[0]` to `num_bins-1` are valid.** |
| `result_status` | `nexatom_result_status_t` | Why the result was published: `RUNNING`, `BLOCK_COMPLETE`, `RUN_COMPLETE` or `STOPPED`. See [result model](7_9_tihi.md#result-model). |
| `result_span` | `nexatom_result_span_t` | What it covers: `WHOLE_RUN` or `BLOCK`. |
| `block_index` | `uint64_t` | Zero-based block number in `BLOCK`; 0 in `WHOLE_RUN`. |
| `live_time_ms` | `double` | Measurement time in the result, excluding the dead time between batches. |
| `live_time_exact` | `uint8_t` | 1 when every batch had a known length; 0 when Stop ended a batch. |

Validate the requested histogram length against both `NEXATOM_MAX_HISTOGRAM_BINS` and the active native capability. The public array extent is not a guarantee of every firmware's supported configuration. A `WHOLE_RUN` result is a running total: adding successive results together double-counts data.

<a id="result-model"></a>

### Result model (TIHI, MFCO, correlators, Fast TIHI)

The hardware measures in batches that the SDK programs and re-arms: 1 s for TIHI, MFCO and Fast TIHI, `num_bins × T` for the correlators. The host sums whole batches into published results; a result never contains part of a batch. The same three calls serve every processor: `NEXATOM_RESULT_PROCESSOR_TIME_HISTOGRAM`, `NEXATOM_RESULT_PROCESSOR_MULTIFOLD_COINCIDENCE`, `NEXATOM_RESULT_PROCESSOR_CORRELATION` (CORL and CORM) and `NEXATOM_RESULT_PROCESSOR_FAST_TIME_HISTOGRAM`. A processor that the running image lacks returns `NEXATOM_ERROR_NOT_SUPPORTED`. These calls replace the earlier aggregation modes and stop conditions.

**Result span** (`nexatom_tt_set_result_span`):

- `NEXATOM_RESULT_SPAN_WHOLE_RUN` (default): the total since Start, `nexatom_tt_clear_result()` or a change that restarts the result, republished about once a second as `RUNNING`. `block_ms` is ignored.
- `NEXATOM_RESULT_SPAN_BLOCK`: tumbling blocks. A block closes at the first batch edge at or after `block_ms`, is published once as `BLOCK_COMPLETE`, and the next block starts empty. `block_ms` is rounded up to that batch edge, so `block_ms = 100` gives 1 s blocks for TIHI; `live_time_ms` reports the real length. `block_ms = 0` returns `NEXATOM_ERROR_INVALID_PARAMETER`.

**Run end** (`nexatom_tt_set_run_end`): `NEXATOM_RUN_END_NONE` runs until Stop. `NEXATOM_RUN_END_TIME` is measurement time in ms. `NEXATOM_RUN_END_COUNT` counts detected events: the sum of all histogram bins (TIHI), of all received pattern bins before the software pattern filter (MFCO), ΣA + ΣB photons (correlation; ΣA alone for an autocorrelation), or the bins of all four contexts (Fast TIHI). At the first batch edge after the condition is met, the SDK publishes the result as `RUN_COMPLETE`, writes Stop and drops the trailing batch. `TIME` and `COUNT` need a value above 0.

**Stop**: after the processor's Stop call, exactly one `STOPPED` result arrives per processor, within 2 s. Wait for it, not for an `acquisition_done_status` code, before disabling file saving; otherwise the files end before the last partial block. A `STOPPED` result that carries a `run_id` (Fast TIHI) reports the window that Stop ended, which can be newer than the last `run_id` seen. `run_id` is a 32-bit hardware counter: compare it wrap-safely and never match on an exact value.

**Restarts**: changing the span or run end restarts the result, as does changing the TIHI channels, bidirectional or first-stop mode. Writing the current value does not.

```c
// Fragment: 10 s of TIHI in 1 s blocks; the SDK stops the run by itself.
nexatom_tt_set_result_span(my_device, NEXATOM_RESULT_PROCESSOR_TIME_HISTOGRAM,
                           NEXATOM_RESULT_SPAN_BLOCK, 1000);
nexatom_tt_set_run_end(my_device, NEXATOM_RESULT_PROCESSOR_TIME_HISTOGRAM,
                       NEXATOM_RUN_END_TIME, 10000);
nexatom_tt_start_time_histogram(my_device);
// The callback sees BLOCK_COMPLETE results, then one RUN_COMPLETE result.
```

```c
// Wrap-safe run_id comparison: true when a is newer than b.
static int run_id_newer(uint32_t a, uint32_t b) {
    return (int32_t)(a - b) > 0;
}
```

### Fast TIHI

Fast TIHI is a separate profile-dependent feature of the UTT160810, exposed by `nexatom_tt_start_fast_tihi` (with `nexatom_tt_fast_tihi_config_t` and four `nexatom_tt_fast_tihi_context_t`), `nexatom_tt_stop_fast_tihi` and `nexatom_tt_set_fast_tihi_result_callback`. It requires feature bit 21 (`NEXATOM_TT_FEATURE_FAST_TIHI_ROLLING_WINDOWS`); without it Start returns `NEXATOM_ERROR_NOT_SUPPORTED` before any write. Results follow the [result model](#result-model) with `NEXATOM_RESULT_PROCESSOR_FAST_TIME_HISTOGRAM`. Each `nexatom_tt_fast_tihi_result_t` is one context's result with `uint64_t` bins, `run_id` and `lost_windows` (windows missing from the result, found from gaps in `run_id`); the callback receives it by value, bins included. Use this record rather than reinterpreting a normal TIHI result. The package's advanced example illustrates capability checks; registration or a model name alone does not establish support.
