# Intensity Correlation (CORL/CORM)

The Intensity Correlation module provides two simultaneous real-time hardware correlators:
1. **Linear Correlator (CORL):** Evaluates $g^{(2)}(\tau)$ over 80 linearly spaced lag points. Ideal for short, deterministic delays.
2. **Multi-Tau Correlator (CORM):** Evaluates $g^{(2)}(\tau)$ over 80 logarithmically spaced lag points. Capable of evaluating 6+ decades of time (nanoseconds to seconds) with extremely low computational overhead. This is the foundation for DLS and FCS physics analysis.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_intensity_correlation`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Global switch for the intensity correlation engine. Must be true to use either CORL or CORM. |
| `nexatom_tt_enable_linear_correlator`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms the internal Linear Correlator (CORL) hardware. |
| `nexatom_tt_enable_multi_tau_correlator`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms the internal Multi-Tau Correlator (CORM) hardware. |
| `nexatom_tt_set_intensity_correlation_channel_a`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel` | `nexatom_error_code_t` | Sets the first input channel for the correlation (0-7). |
| `nexatom_tt_set_intensity_correlation_channel_b`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel` | `nexatom_error_code_t` | Sets the second input channel. If $A = B$, it acts as auto-correlation. If $A \neq B$, it acts as cross-correlation. |
| `nexatom_tt_set_intensity_correlation_bin_width`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t bin_width_ps` | `nexatom_error_code_t` | Sets the base sampling period $T$ in picoseconds. For CORL, $\tau_i = i \times T$. |
| `nexatom_tt_set_intensity_correlation_num_bins`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t num_bins` | `nexatom_error_code_t` | Configures the number of computed lags (currently locked to `80` by the FPGA RTL). |
| `nexatom_tt_start_intensity_correlation`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Synchronously issues the "START" command to the armed correlator modules. |
| `nexatom_tt_stop_intensity_correlation`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Halts correlation and forces a final data dispatch to the host. |
| `nexatom_tt_set_intensity_stop_conditions`| `[In] nexatom_tt_handle device`<br>`[In] const nexatom_stop_conditions_t* cond` | `nexatom_error_code_t` | Configures auto-stop limits based on time or aggregate counts. |
| `nexatom_tt_set_linear_correlator_aggregation_mode`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t mode` | `nexatom_error_code_t` | Sets accumulation behavior for CORL (`0` = Reset, `1` = Accumulate over packets). |
| `nexatom_tt_set_multi_tau_correlator_aggregation_mode`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t mode` | `nexatom_error_code_t` | Sets accumulation behavior for CORM (`0` = Reset, `1` = Accumulate over packets). |

### Data Structures: Callback Payloads

When the correlator dispatches a completed cycle, it emits either a `nexatom_corl_callback_data_t` or a `nexatom_corm_callback_data_t` via the callback registered in Section 7.8.

Both structs share a very similar 80-bin array layout, but **CORM** contains additional nested structures for C++ background curve fitting and embedded DLS/FCS models.

| Field | Type | Description |
| :--- | :--- | :--- |
| `g2_values` | `float[80]` | The normalized $g^{(2)}(\tau)$ correlation function values. Expected baseline $\approx 1.0$. |
| `lag_times_ns` | `uint64_t[80]` | The actual $\tau$ lag time for each bin in nanoseconds. (Linear for CORL, quasi-logarithmic for CORM). |
| `baseline_level` | `float` | $g^{(2)}(\infty)$ - Evaluated correlation at infinite lag. |
| `contrast` | `float` | $g^{(2)}(0) - g^{(2)}(\infty)$ - The amplitude of the correlation peak. |
| `mean_intensity_a` / `b` | `float` | Normalized count rates per period. Required for host-side normalization checks. |
| `fit_result` *(CORM Only)* | `struct` | Output from the background C++ Levenberg-Marquardt engine (e.g., `chi_squared`, `correlation_time_ns`). |
| `analysis_result` *(CORM Only)* | `struct` | Contains nested physics data if DLS, FCS, or DCS analysis modules are activated (See sections 7.12 - 7.14). |
