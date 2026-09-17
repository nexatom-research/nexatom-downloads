# Multifold Coincidence (MFCO)

The Multifold Coincidence (MFCO) module is designed for complex quantum optics experiments where multiple channels must be evaluated for concurrent firing within a tight temporal window.

Rather than recording a START-to-STOP histogram like TIHI, MFCO reports an eight-bit pattern map (`0x00` to `0xFF`). Pattern `0x05` (`0b00000101`) represents bits 0 and 2 within the configured coincidence window. This pattern representation is distinct from a product's physical lane count and public channel authority.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_multifold_coincidence`| `[In] nexatom_tt_handle device`<br>`[In] bool enable` | `nexatom_error_code_t` | Arms or disarms the MFCO engine in the FPGA. |
| `nexatom_tt_set_multifold_coincidence_channels`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t ch0`<br>`[In] uint8_t ch1`<br>`[In] uint8_t ch2`<br>`[In] uint8_t ch3`<br>`[In] uint8_t ch4`<br>`[In] uint8_t ch5`<br>`[In] uint8_t ch6`<br>`[In] uint8_t ch7` | `nexatom_error_code_t` | Eight channel-ID slots; `0xff` disables a slot. These are not enable booleans. Current returned pattern bins are normally filtered in software; mask metadata is not physical-input gating proof. |
| `nexatom_tt_set_multifold_coincidence_window`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t window_ps` | `nexatom_error_code_t` | Sets the maximum time difference (in picoseconds) between photons to be considered "coincident". |
| `nexatom_tt_set_multifold_coincidence_pattern_filter`| `[In] nexatom_tt_handle device`<br>`[In] const uint8_t requirements[8]` | `nexatom_error_code_t` | Applies per-channel requirement constraints. Each element of the 8-byte array specifies the filter rule for the corresponding channel. |
| `nexatom_tt_disable_multifold_coincidence_pattern_filter`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Disables software filtering, passing all 256 logic bins directly to the callback. |
| `nexatom_tt_start_multifold_coincidence`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests START for the enabled MFCO engine; observe measurement results separately. |
| `nexatom_tt_stop_multifold_coincidence`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests stop; observe the expected terminal result separately before finalizing sinks. |
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
| `pattern_filter.patterns_after_filter` | `uint32_t` | Number of retained patterns, which can include zero-count bins. |
| `pattern_bins` | `uint32_t[256]` | **The absolute counts for each pattern.** Index `0x05` contains the counts where exactly Ch0 and Ch2 fired together. |
| `singles` | `uint32_t[8]` | Convenience array tracking counts where *only* that specific channel fired. |
| `total_counts` | `uint64_t` | Sum of retained corrected bins, including pattern zero and exact singles. |
| `num_doubles` | `uint32_t` | Count of events with exactly 2 channels firing simultaneously. |
| `num_triples` | `uint32_t` | Count of events with exactly 3 channels firing simultaneously. |
| `num_higher` | `uint32_t` | Count of events with 4 or more channels firing simultaneously. |
| `top_patterns` | `struct[10]` | A sorted leaderboard array of the 10 most frequently occurring logic patterns and their counts. |
| `packets_accumulated` | `uint32_t` | Number of raw data packets aggregated. |
| `measurement_duration_ms` | `uint64_t` | Host wall-clock duration of the aggregation emission epoch; not a guaranteed MFCO live-time denominator. |
| `result_metadata_version` | `uint8_t` | Zero means unavailable; version 1 or newer establishes the associated metadata contract. |
| `done_status_error_flags_raw` | `uint8_t` | Preserved hardware error/status flags; inspect even if counts are nonzero. |
| `host_quality_flags` | `uint16_t` | Host quality flags; retain with completion and aggregation information. |

An exact pattern count differs from an inclusive coincidence count over all supersets. See [pattern analysis](../05_api_reference/5_6_mfco_pattern_analysis_helpers.md). REPLACE, ACCUMULATE and AVERAGE results have different aggregation meaning; do not sum overlapping snapshots or infer Hz from host elapsed time without a defined live-time measurement.
