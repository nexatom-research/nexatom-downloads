# Multifold Coincidence (MFCO)

The Multifold Coincidence (MFCO) module is designed for complex quantum optics experiments where multiple channels must be evaluated for concurrent firing within a tight temporal window.

Rather than logging absolute time delays like TIHI, the MFCO module evaluates all 8 physical channels simultaneously and generates a binary pattern map (from `0x00` to `0xFF`). For example, if Channel 0 and Channel 2 fire within 500 picoseconds of each other, the engine records an event in bin `0x05` (`0b00000101`).

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_multifold_coincidence`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms or disarms the MFCO engine in the FPGA. |
| `nexatom_tt_set_multifold_coincidence_channels`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel_mask` | `nexatom_error_code_t` | Selects which of the 8 physical channels are fed into the coincidence gate. |
| `nexatom_tt_set_multifold_coincidence_window`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t window_ps` | `nexatom_error_code_t` | Sets the maximum time difference (in picoseconds) between photons to be considered "coincident". |
| `nexatom_tt_set_multifold_coincidence_pattern_filter`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t required`<br>`[In] uint8_t forbidden` | `nexatom_error_code_t` | Instructs the host to silently discard specific coincidence combinations (e.g., must contain Ch0, must not contain Ch7). |
| `nexatom_tt_disable_multifold_coincidence_pattern_filter`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Disables software filtering, passing all 256 logic bins directly to the callback. |
| `nexatom_tt_start_multifold_coincidence`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Synchronously issues the FPGA "START" command. |
| `nexatom_tt_stop_multifold_coincidence`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Halts the measurement and forces a final data dispatch. |
| `nexatom_tt_set_multifold_coincidence_stop_conditions`| `[In] nexatom_tt_handle device`<br>`[In] const nexatom_stop_conditions_t* cond`| `nexatom_error_code_t` | Configures auto-stop triggers (e.g., stop after 1000 coincidences). |
| `nexatom_tt_set_multifold_coincidence_aggregation_mode`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t mode` | `nexatom_error_code_t` | Sets accumulation behavior (`0` = Reset on Start, `1` = Overwrite on new packets). |
| `nexatom_tt_set_mfco_background_method`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t method_enum` | `nexatom_error_code_t` | Configures background subtraction for accidental dark count coincidence rejection. |
| `nexatom_tt_set_mfco_user_background_value`| `[In] nexatom_tt_handle device`<br>`[In] double value` | `nexatom_error_code_t` | Manual noise floor subtraction. |
| `nexatom_tt_set_mfco_background_bin`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t pattern_bin` | `nexatom_error_code_t` | Sets a specific logic bin to represent the baseline noise level dynamically. |

### Data Structures: `nexatom_mfco_callback_data_t`

MFCO payloads are passed by-value to the callback registered in Section 7.8. The `pattern_bins` array is strictly fixed at 256 indices representing all possible 8-channel boolean firing combinations.

| Field | Type | Description |
| :--- | :--- | :--- |
| `coincidence_window_ps` | `uint32_t` | Echoes the active temporal window configuration. |
| `pattern_filter.enabled` | `uint8_t` | `1` if host-side software rejection was applied. |
| `pattern_filter.required_mask` | `uint8_t` | Echoes the active logic filter requirements. |
| `pattern_bins` | `uint32_t[256]` | **The absolute counts for each pattern.** Index `0x05` contains the counts where exactly Ch0 and Ch2 fired together. |
| `singles` | `uint32_t[8]` | Convenience array tracking counts where *only* that specific channel fired. |
| `total_counts` | `uint64_t` | Sum of all events across all 256 logic combinations. |
| `num_doubles` | `uint32_t` | Count of events with exactly 2 channels firing simultaneously. |
| `num_triples` | `uint32_t` | Count of events with exactly 3 channels firing simultaneously. |
| `num_higher` | `uint32_t` | Count of events with 4 or more channels firing simultaneously. |
| `top_patterns` | `struct[10]` | A sorted leaderboard array of the 10 most frequently occurring logic patterns and their counts, provided automatically by the C++ thread. |
