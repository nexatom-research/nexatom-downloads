# Multifold Coincidence (MFCO)

The Multifold Coincidence (MFCO) module is designed for complex quantum optics experiments where multiple channels must be evaluated for concurrent firing within a tight temporal window.

Rather than logging absolute time delays like TIHI, the MFCO module evaluates all 8 physical channels simultaneously and generates a binary pattern map (from `0x00` to `0xFF`). For example, if Channel 0 and Channel 2 fire within 500 picoseconds of each other, the engine records an event in bin `0x05` (`0b00000101`).

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_multifold_coincidence`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms or disarms the MFCO engine in the FPGA. |
| `nexatom_tt_set_multifold_coincidence_channels`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t ch0`<br>`[In] uint8_t ch1`<br>`[In] uint8_t ch2`<br>`[In] uint8_t ch3`<br>`[In] uint8_t ch4`<br>`[In] uint8_t ch5`<br>`[In] uint8_t ch6`<br>`[In] uint8_t ch7` | `nexatom_error_code_t` | Maps which of the 8 physical channels are fed into the coincidence gate. Each parameter enables (`1`) or disables (`0`) the corresponding channel. |
| `nexatom_tt_set_multifold_coincidence_window`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t window_ps` | `nexatom_error_code_t` | Sets the maximum time difference (in picoseconds) between photons to be considered "coincident". |
| `nexatom_tt_set_multifold_coincidence_pattern_filter`| `[In] nexatom_tt_handle device`<br>`[In] const uint8_t requirements[8]` | `nexatom_error_code_t` | Applies per-channel requirement constraints. Each element of the 8-byte array specifies the filter rule for the corresponding channel. |
| `nexatom_tt_disable_multifold_coincidence_pattern_filter`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Disables software filtering, passing all 256 logic bins directly to the callback. |
| `nexatom_tt_start_multifold_coincidence`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Synchronously issues the FPGA "START" command. |
| `nexatom_tt_stop_multifold_coincidence`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Halts the measurement and forces a final data dispatch. |
| `nexatom_tt_set_multifold_coincidence_stop_conditions`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t stop_count`<br>`[In] uint32_t stop_duration_ms`<br>`[In] bool use_duration` | `nexatom_error_code_t` | Configures auto-stop triggers. Set `use_duration = true` for time-based stops, or `use_duration = false` for count-based stops. |
| `nexatom_tt_set_multifold_coincidence_aggregation_mode`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_aggregation_mode_t mode` | `nexatom_error_code_t` | Sets accumulation behavior (`NEXATOM_AGGREGATION_ACCUMULATE` = Add to existing, `NEXATOM_AGGREGATION_REPLACE` = Overwrite on new packets). |
| `nexatom_tt_set_mfco_background_method`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_mfco_background_method_t method` | `nexatom_error_code_t` | Configures background subtraction for accidental dark count coincidence rejection (`NEXATOM_MFCO_BG_NONE` = 0, `NEXATOM_MFCO_BG_USER_CONSTANT` = 1, `NEXATOM_MFCO_BG_USER_SELECTED_PATTERN_BIN` = 2). |
| `nexatom_tt_set_mfco_user_background_value`| `[In] nexatom_tt_handle device`<br>`[In] uint16_t value` | `nexatom_error_code_t` | Manual noise floor subtraction (range 0–65535). Only effective when method is `USER_CONSTANT`. |
| `nexatom_tt_set_mfco_background_bin`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t pattern_bin` | `nexatom_error_code_t` | Sets a specific logic bin (0–255) to represent the baseline noise level dynamically. Only effective when method is `USER_SELECTED_PATTERN_BIN`. |

### Data Structures: `nexatom_mfco_callback_data_t`

MFCO payloads are passed by-value to the callback registered in Section 7.8. The `pattern_bins` array is strictly fixed at 256 indices representing all possible 8-channel boolean firing combinations.

| Field | Type | Description |
| :--- | :--- | :--- |
| `channel_mask` | `uint8_t` | MFCO channel mask metadata/readback. |
| `acquisition_done_status` | `nexatom_acquisition_done_status_t` | Reason the measurement stopped (normal, timeout, error). |
| `background_subtracted` | `uint8_t` | `1` if background was subtracted from pattern counts. |
| `coincidence_window_ps` | `uint32_t` | Echoes the active temporal window configuration. |
| `background_level_per_pattern` | `float` | Background counts per pattern that were subtracted (if applied). |
| `pattern_filter.enabled` | `uint8_t` | `1` if host-side software rejection was applied. |
| `pattern_filter.required_mask` | `uint8_t` | Echoes the active logic filter requirements. |
| `pattern_filter.forbidden_mask` | `uint8_t` | Channels that must not be present. |
| `pattern_filter.patterns_after_filter` | `uint32_t` | Number of non-zero patterns after filtering. |
| `pattern_bins` | `uint32_t[256]` | **The absolute counts for each pattern.** Index `0x05` contains the counts where exactly Ch0 and Ch2 fired together. |
| `singles` | `uint32_t[8]` | Convenience array tracking counts where *only* that specific channel fired. |
| `total_counts` | `uint64_t` | Sum of all events across all 256 logic combinations. |
| `num_doubles` | `uint32_t` | Count of events with exactly 2 channels firing simultaneously. |
| `num_triples` | `uint32_t` | Count of events with exactly 3 channels firing simultaneously. |
| `num_higher` | `uint32_t` | Count of events with 4 or more channels firing simultaneously. |
| `top_patterns` | `struct[10]` | A sorted leaderboard array of the 10 most frequently occurring logic patterns and their counts. |
| `packets_accumulated` | `uint32_t` | Number of raw data packets aggregated. |
| `measurement_duration_ms` | `uint64_t` | Total measurement time in milliseconds. |
