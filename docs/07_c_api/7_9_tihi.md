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
| `nexatom_tt_set_time_histogram_num_bins` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t num_bins` | `nexatom_error_code_t` | Sets the length of the hardware histogram array (max `1024`). |
| `nexatom_tt_start_time_histogram` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests START for the enabled TIHI engine; observe measurement results separately. |
| `nexatom_tt_stop_time_histogram` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests stopping TIHI. Observe the expected terminal result separately; successful return does not promise its callback already ran. |
| `nexatom_tt_set_time_histogram_stop_conditions`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t stop_count`<br>`[In] uint32_t stop_duration_ms`<br>`[In] bool use_duration`| `nexatom_error_code_t` | Configures auto-stop limits. Set `use_duration = true` and a non-zero `stop_duration_ms` for time-based stops, or `use_duration = false` and a non-zero `stop_count` for count-based stops. |
| `nexatom_tt_set_time_histogram_aggregation_mode`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_aggregation_mode_t mode` | `nexatom_error_code_t` | Controls whether host arrays accumulate (`NEXATOM_AGGREGATION_ACCUMULATE`) or overwrite (`NEXATOM_AGGREGATION_REPLACE`) on new packets. |
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
| `packets_accumulated` | `uint32_t` | Number of raw data packets aggregated into this histogram. |
| `bins` | `uint32_t[1024]` | **The absolute time-binned photon counts. Only `[0]` to `num_bins-1` are valid.** |

Validate the requested histogram length against both `NEXATOM_MAX_HISTOGRAM_BINS` and the active native capability. The public array extent is not a guarantee of every firmware's supported configuration. Preserve aggregation mode: summing overlapping ACCUMULATE snapshots double-counts data.

### Fast TIHI

Fast TIHI is a separate profile-dependent feature, exposed by `nexatom_tt_start_fast_tihi_v1`, `nexatom_tt_start_fast_tihi_v2`, `nexatom_tt_stop_fast_tihi` and its versioned histogram callback. V1/V2 configuration sizes, context count, supported window mode and duration fields are defined in the header. Use the matching versioned record rather than reinterpreting a normal TIHI result. The package's advanced example illustrates capability checks; registration or a model name alone does not establish support.
