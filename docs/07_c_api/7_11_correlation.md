# Intensity Correlation (CORL/CORM)

The Intensity Correlation module provides two simultaneous real-time hardware correlators:
1. **Linear Correlator (CORL):** Evaluates g²(τ) over 80 linearly spaced lag points. Ideal for short, deterministic delays.
2. **Multi-Tau Correlator (CORM):** Evaluates g²(τ) over 80 logarithmically spaced lag points. Capable of evaluating 6+ decades of time (nanoseconds to seconds) with extremely low computational overhead. This is the foundation for DLS and FCS physics analysis.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_intensity_correlation`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Global switch for the intensity correlation engine. Must be true to use either CORL or CORM. |
| `nexatom_tt_enable_linear_correlator`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms the internal Linear Correlator (CORL) hardware. |
| `nexatom_tt_enable_multi_tau_correlator`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms the internal Multi-Tau Correlator (CORM) hardware. |
| `nexatom_tt_set_intensity_correlation_channel_a`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel` | `nexatom_error_code_t` | Sets the first input channel for the correlation (0-7). |
| `nexatom_tt_set_intensity_correlation_channel_b`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel` | `nexatom_error_code_t` | Sets the second input channel. If A = B, it acts as auto-correlation. If A ≠ B, it acts as cross-correlation. |
| `nexatom_tt_set_intensity_correlation_bin_width`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t bin_width_in_8ns_units` | `nexatom_error_code_t` | Sets the base sampling period T in units of 8 nanoseconds. For example, a value of `1` gives T = 8 ns, a value of `125` gives T = 1 μs. For CORL, τᵢ = i × T. |
| `nexatom_tt_set_intensity_correlation_num_bins`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t num_bins` | `nexatom_error_code_t` | Configures integration sample depth, with native profile-specific validation; not the 80 returned lag points. |
| `nexatom_tt_start_intensity_correlation`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests START for enabled correlators; observe measurement results separately. |
| `nexatom_tt_stop_intensity_correlation`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests stopping correlation; command success is separate from observing the final result. |
| `nexatom_tt_set_intensity_stop_conditions`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t stop_count`<br>`[In] uint32_t stop_duration_ms`<br>`[In] bool use_duration` | `nexatom_error_code_t` | Configures auto-stop limits. Set `use_duration = true` for time-based stops, or `use_duration = false` for count-based stops. |
| `nexatom_tt_set_linear_correlator_aggregation_mode`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_aggregation_mode_t mode` | `nexatom_error_code_t` | Sets accumulation behavior for CORL (`NEXATOM_AGGREGATION_ACCUMULATE` = Add, `NEXATOM_AGGREGATION_REPLACE` = Overwrite). |
| `nexatom_tt_set_multi_tau_correlator_aggregation_mode`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_aggregation_mode_t mode` | `nexatom_error_code_t` | Sets accumulation behavior for CORM (`NEXATOM_AGGREGATION_ACCUMULATE` = Add, `NEXATOM_AGGREGATION_REPLACE` = Overwrite). |

### Data Structures: Callback Payloads

When the correlator dispatches a completed cycle, it emits either a `nexatom_corl_callback_data_t` or a `nexatom_corm_callback_data_t` via the callback registered in Section 7.8.

Both structs share a very similar 80-bin array layout, but **CORM** contains additional nested structures for C++ background curve fitting and embedded DLS/FCS models.

| Field | Type | Description |
| :--- | :--- | :--- |
| `g2_values` | `float[80]` | The normalized g²(τ) correlation function values. Expected baseline ≈ 1.0. |
| `lag_times_ns` | `uint64_t[80]` | The actual τ lag time for each bin in nanoseconds. (Linear for CORL, quasi-logarithmic for CORM). |
| `baseline_level` | `float` | g²(∞) - Evaluated correlation at infinite lag. |
| `contrast` | `float` | g²(0) − g²(∞) - The amplitude of the correlation peak. |
| `mean_intensity_a` / `b` | `float` | Mean counts per integration sample, sum of counts divided by sample count; not rates in Hz. |
| `fit_result` *(CORM Only)* | `struct` | Output from the background C++ Levenberg-Marquardt engine (e.g., `chi_squared`, `correlation_time_ns`). |
| `analysis_result` *(CORM Only)* | `struct` | Contains nested physics data if DLS, FCS, or DCS analysis modules are activated (See sections 7.12 - 7.14). |

### Integration and result validity

`nexatom_tt_set_intensity_correlation_num_bins` configures **integration sample depth**, not the number of returned lags (the callback arrays have 80 points). Native validates the applicable depth range: OG uses 1000–65535 and UTT requires at least 4096. The bin-width API retains 8 ns units; native translates for the active profile.

Inspect `normalization_valid`, `sum_a_counts`, `sum_b_counts` and `sample_count`. A value in the historically named `g2_values` array is not necessarily normalized g² when normalization is invalid. Retain the supplied lag axis and aggregation metadata.

The exported CORM analysis type selects one result, with DLS then FCS then DCS priority when multiple results are available. Enabling an analysis does not guarantee a successful fit or selected output; see [analysis validity](../06_in_depth_guides/6_4_curve_fitting_and_analysis_pipelines.md).
