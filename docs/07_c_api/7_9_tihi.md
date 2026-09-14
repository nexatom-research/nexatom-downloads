# Time Interval Histogram (TIHI)

The Time Interval Histogram (TIHI) module is the primary engine for Time-Correlated Single Photon Counting (TCSPC) and fluorescence lifetime measurements. It records the temporal delay between a "Start" event and subsequent "Stop" events, accumulating them into a high-speed hardware array.

This module supports bidirectional (Start-Stop and Stop-Start) accumulation, multi-stop architectures, and aggressive host-side background subtraction and curve fitting (Exponential, Gaussian, Lorentzian).

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_time_histogram` | `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms or disarms the TIHI measurement engine inside the FPGA. |
| `nexatom_tt_set_time_histogram_channels` | `[In] nexatom_tt_handle device`<br>`[In] uint8_t start_channel`<br>`[In] uint8_t stop_channel` | `nexatom_error_code_t` | Maps the physical inputs for the Start-Stop logic. |
| `nexatom_tt_set_time_histogram_bidirectional`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Enables measuring negative time differences (Stop arriving before Start). |
| `nexatom_tt_set_time_histogram_first_stop_mode`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | If true, only the first Stop photon after a Start is counted (classic TDC). If false, multi-stop is enabled. |
| `nexatom_tt_set_time_histogram_bin_width`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t bin_width_ps` | `nexatom_error_code_t` | Sets the hardware bin width resolution in picoseconds. |
| `nexatom_tt_set_time_histogram_num_bins` | `[In] nexatom_tt_handle device`<br>`[In] uint16_t num_bins` | `nexatom_error_code_t` | Sets the length of the hardware histogram array (max `1024`). |
| `nexatom_tt_start_time_histogram` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Synchronously issues the FPGA "START" command. |
| `nexatom_tt_stop_time_histogram` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Synchronously halts the TIHI module and forces a final callback dispatch. |
| `nexatom_tt_set_time_histogram_stop_conditions`| `[In] nexatom_tt_handle device`<br>`[In] const nexatom_stop_conditions_t* conditions`| `nexatom_error_code_t` | Configures auto-stop limits (e.g., stop after 10 seconds or 1,000,000 counts). |
| `nexatom_tt_set_time_histogram_aggregation_mode`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t mode` | `nexatom_error_code_t` | Controls whether host arrays accumulate `0` or overwrite `1` on new packets. |
| `nexatom_tt_set_tihi_background_method` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t method_enum` | `nexatom_error_code_t` | Sets the algorithm used to subtract baseline noise (`0`=None, `1`=Manual, `2`=Percentile, `3`=Mean). |
| `nexatom_tt_set_tihi_user_background_value` | `[In] nexatom_tt_handle device`<br>`[In] double value` | `nexatom_error_code_t` | Injects a manual background subtraction floor. |
| `nexatom_tt_set_tihi_signal_region` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t start_bin`, `end_bin` | `nexatom_error_code_t` | Defines the Region of Interest (ROI) boundaries for SNR optimization. |
| `nexatom_tt_set_tihi_background_region` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t start_bin`, `end_bin` | `nexatom_error_code_t` | Defines the bin region used by the `AUTO_MEAN` background method. |
| `nexatom_tt_enable_tihi_fitting` | `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Instructs the host background thread to run Levenberg-Marquardt fitting. |
| `nexatom_tt_set_tihi_fitting_parameters` | `[In] nexatom_tt_handle device`<br>`[In] double* initial_guesses`<br>`[In] uint32_t count` | `nexatom_error_code_t` | Seeds the curve fitter with starting values to prevent divergence. |
| `nexatom_tt_set_tihi_fitting_model` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_fitting_model_t model` | `nexatom_error_code_t` | Assigns the math model (`EXPONENTIAL`, `GAUSSIAN`, `LORENTZIAN`, etc.). |

### Data Structures: `nexatom_tihi_callback_data_t`

When the TIHI engine dispatches a frame, it passes this massive structure by value. The arrays are contiguous for cache efficiency.

| Field | Type | Description |
| :--- | :--- | :--- |
| `num_bins` / `bin_width_ps` | `uint32_t` | Active bins limit and temporal resolution per bin. |
| `start_channel` / `stop_channel` | `uint8_t` | The hardware channel maps used to generate this curve. |
| `acquisition_done_status` | `uint32_t` (Enum) | Reason measurement stopped (0 = Active, 1 = Done, 2 = Timeout). |
| `_padding1` | `uint8_t[1]` | **Required 1-byte alignment padding.** |
| `total_counts` | `uint64_t` | Aggregate sum of all counts across the histogram array. |
| `mean_time_ps` / `std_dev_ps` | `double` | Statistical estimates of arrival time and distribution width. |
| `peak_bin_index` / `peak_counts` | `uint32_t` | Bin mode location and height. |
| `fwhm_ps` | `double` | Full Width Half Maximum estimate of the pulse. |
| `fitting.converged` | `uint8_t` | Returns `1` if the C++ math model successfully fit the data. |
| `fitting.model_type` | `char[32]` | String identifying the model (e.g., `"exponential"`). |
| `fitting.chi_squared` | `double` | Statistical goodness-of-fit indicator. |
| `fitting.exp_lifetime_ns` | `double` | Example specific fit output (populated if exponential model chosen). |
| `fitting.fitted_curve` | `float[1024]` | Predicted Y-values matching the histogram bins for GUI overlay plotting. |
| `roi.enabled` | `uint8_t` | `1` if ROI windowing was applied to the statistics above. |
| `background_subtracted` | `uint8_t` | `1` if `bg_method` was executed. |
| `bins` | `uint32_t[1024]` | **The absolute time-binned photon counts. Only `[0]` to `num_bins-1` are valid.** |
