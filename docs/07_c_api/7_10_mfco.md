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
| `nexatom_tt_stop_multifold_coincidence`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests stop. Exactly one `STOPPED` result follows within 2 s; wait for it before finalizing sinks. |
| `nexatom_tt_set_result_span`, `nexatom_tt_set_run_end`, `nexatom_tt_clear_result` | `processor = NEXATOM_RESULT_PROCESSOR_MULTIFOLD_COINCIDENCE` | `nexatom_error_code_t` | What a result covers, when the run ends and restarting the result. See the [result model](7_9_tihi.md#result-model). |
| `nexatom_tt_set_mfco_background_method`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_mfco_background_method_t method` | `nexatom_error_code_t` | Configures background subtraction for accidental dark count coincidence rejection (`NEXATOM_MFCO_BG_NONE` = 0, `NEXATOM_MFCO_BG_USER_CONSTANT` = 1, `NEXATOM_MFCO_BG_USER_SELECTED_PATTERN_BIN` = 2). |
| `nexatom_tt_set_mfco_user_background_value`| `[In] nexatom_tt_handle device`<br>`[In] uint16_t value` | `nexatom_error_code_t` | Manual noise floor subtraction (range 0–65535). Only effective when method is `USER_CONSTANT`. |
| `nexatom_tt_set_mfco_background_bin`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t pattern_bin` | `nexatom_error_code_t` | Sets a specific logic bin (0–255) to represent the baseline noise level dynamically. Only effective when method is `USER_SELECTED_PATTERN_BIN`. |

### Data Structures: `nexatom_mfco_callback_data_t`

MFCO payloads are passed by-value to the callback registered in Section 7.8. The `pattern_bins` array is strictly fixed at 256 indices representing all possible 8-channel boolean firing combinations.

| Field | Type | Description |
| :--- | :--- | :--- |
| `channel_mask` | `uint8_t` | MFCO channel mask metadata/readback. |
| `acquisition_done_status` | `nexatom_acquisition_done_status_t` | Hardware status: manual stop (`0x2`), stop-count reached (`0x4`), or duration reached (`0x5`) are defined terminal reasons. Zero can mean Running; do not infer success from the legacy `NORMAL_COMPLETION` alias. |
| `background_subtracted` | `uint8_t` | `1` if background was subtracted from pattern counts. |
| `coincidence_window_ps` | `uint32_t` | Echoes the active temporal window configuration. |
| `background_level_per_pattern` | `float` | Background counts per pattern that were subtracted (if applied). |
| `pattern_filter.enabled` | `uint8_t` | `1` if host-side software rejection was applied. |
| `pattern_filter.required_mask` | `uint8_t` | Echoes the active logic filter requirements. |
| `pattern_filter.forbidden_mask` | `uint8_t` | Channels that must not be present. |
| `pattern_filter.patterns_after_filter` | `uint32_t` | Number of retained patterns, which can include zero-count bins. |
| `pattern_bins` | `uint64_t[256]` | **The absolute counts for each pattern.** Index `0x05` contains the counts where exactly Ch0 and Ch2 fired together. |
| `singles` | `uint64_t[8]` | Convenience array tracking counts where *only* that specific channel fired. |
| `total_counts` | `uint64_t` | Sum of retained corrected bins, including pattern zero and exact singles. |
| `num_doubles` | `uint64_t` | Count of events with exactly 2 channels firing simultaneously. |
| `num_triples` | `uint64_t` | Count of events with exactly 3 channels firing simultaneously. |
| `num_higher` | `uint64_t` | Count of events with 4 or more channels firing simultaneously. |
| `top_patterns` | `struct[10]` | A sorted leaderboard array of the 10 most frequently occurring logic patterns and their counts. |
| `packets_accumulated` | `uint32_t` | Hardware batches in this result. |
| `measurement_duration_ms` | `uint64_t` | Host time from the start of this result to its last batch; for rates use `live_time_ms`. |
| `result_status` | `nexatom_result_status_t` | Why the result was published: `RUNNING`, `BLOCK_COMPLETE`, `RUN_COMPLETE` or `STOPPED`. See [result model](7_9_tihi.md#result-model). |
| `result_span` | `nexatom_result_span_t` | What it covers: `WHOLE_RUN` or `BLOCK`. |
| `block_index` | `uint64_t` | Zero-based block number in `BLOCK`; 0 in `WHOLE_RUN`. |
| `live_time_ms` | `double` | Measurement time in the result, excluding the dead time between batches. |
| `live_time_exact` | `uint8_t` | 1 when every batch had a known length; 0 when Stop ended a batch. |
| `result_metadata_version` | `uint8_t` | Zero means unavailable; version 1 or newer establishes the associated metadata contract. |
| `done_status_error_flags_raw` | `uint8_t` | Preserved hardware error/status flags; inspect even if counts are nonzero. |
| `host_quality_flags` | `uint16_t` | Host quality flags; retain with completion and result information. |

An exact pattern count differs from an inclusive coincidence count over all supersets. See [pattern analysis](../05_api_reference/5_6_mfco_pattern_analysis_helpers.md). A `WHOLE_RUN` result is a running total and a `BLOCK` result covers one block; do not add overlapping results together. Use `live_time_ms` as the rate denominator rather than host elapsed time.
