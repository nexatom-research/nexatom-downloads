## Data Structures

This section explains the principal fields of the SDK's data payloads. When data is dispatched via callbacks or queried synchronously, the Python wrapper supplies typed `ctypes.Structure` records. Tables omit padding and some auxiliary fields for readability; use the packaged definitions for exact layouts. Fixed arrays are ctypes arrays, not Python lists; use `list(record.bins[:record.num_bins])` when a list is needed.

### [Device information](5_3_data_structures.md#device-information)

These structures are populated during USB discovery and device initialization.

#### `NexatomDeviceInfo`
| Field | Type | Description |
| :--- | :--- | :--- |
| `serial_number` | byte string | FT601 USB serial, for service information only. Several boards can share it; never select a board by it. Decode text with UTF-8. |
| `firmware_version` | byte string | Descriptive firmware string; not authoritative model/image evidence. |
| `hardware_version` | byte string | Descriptive hardware version. |
| `device_name` | byte string | Human-readable discovery name. |
| `connection_type`, `connection_id` | byte strings | Transport description and the board's identity. `connection_id` is `usb:` plus the USB port path, for example `usb:PCIROOT(0)#PCI(0801)#PCI(0004)#USBROOT(0)#USB(4)` on Windows or `usb:2-1.3` on Linux. It selects the board at connect; moving the board to another port changes it. |

#### `NexatomCapabilities`
| Field | Type | Description |
| :--- | :--- | :--- |
| `num_channels` | `int` | Public channel count; use the profile mask when choosing channel IDs. |
| `max_count_rate` | `int` | Reported count-rate capability, not a measured host capture throughput. |
| `time_resolution_ps` | `int` | Base hardware resolution (picoseconds). |
| `max_threshold_mv` | `int` | Maximum valid discriminator voltage. |
| `max_histogram_bins` | `int` | Active device's configurable TIHI limit. |
| `supports_calibration`, `supports_file_saving`, `supports_external_clock`, `supports_gating` | `bool` | Public operation availability; also check the resolved profile. |

#### `NexatomDeviceProfileV1`

The profile carries the authority used by native to admit controls. These are separate from display strings returned during discovery.

| Field | Type | Description |
| --- | --- | --- |
| `profile_flags` | `int` | Product/application resolution, protocol compatibility, service state and control authorization. |
| `product_model_id`, `application_image_id` | `int` | Product model and firmware image identities. |
| `identity_source`, `identity_confidence` | `int` | Source and strength of resolved identity evidence. |
| `physical_tdc_count`, `physical_tdc_mask` | `int` | Physical TDC resources where reported. |
| `effective_public_tdc_mask` | `int` | Channels authorized for public controls. |
| `dtc_output_count` | `int` | DTC output resources; feature authorization is still required. |
| `feature_flags` | `int` property | Combined 64-bit feature mask from `feature_flags_low/high`. |
| `supported_output_mode_mask` | `int` | Authorized output modes. |
| `max_channel_input_delay_ps` | `int` | Maximum delay for this profile, in ps. |
| `usb_link_speed` | `int` | USB generation the board enumerated at: `NEXATOM_TT_USB_LINK_SPEED_SUPER` (3, USB 3), `_HIGH` (2, USB 2), `_UNKNOWN` (0). |
| `fast_tihi_max_input_rate_hz` | uint32 array[4] | Most trigger plus detector events per second each Fast TIHI context can take without dropping pairs; 0 for a context the board does not have. |
| `dtc_output_latency_ps` | `int` | Fixed latency from input to pulse output (UTT160810); 0 when the image has no supported pulse outputs. See [Pulse outputs](../02_hardware/2_13_dtc_pulse_outputs.md). |
| `test_pulse_clock_hz`, `test_pulse_min_period_cycles`, `test_pulse_max_period_cycles` | `int` | Clock and period bounds for the internal test pulse. |

Versioned records include size/version fields. High-level getters initialize and validate these; C callers must follow the corresponding declaration.

---

### [Time tag data](5_3_data_structures.md#time-tag-data)

Structures utilized for raw event streaming and offline `.nxtt` binary decoding.

#### `NexatomTimeTag`
| Field | Type | Description |
| :--- | :--- | :--- |
| `timestamp_ps` | `int` (uint64) | Decoded hardware timestamp in picoseconds. It is not Unix wall-clock time; do not assume every host start creates a new zero origin. In raw mode the SDK extends the board's 59-bit wire time; on the UTT160810 it runs on through the board counter wrap and across output-type changes within a runtime session. |
| `channel` | `int` (uint8) | Decoded event channel identifier; interpret it using the active image/profile's mapping. |

#### `NexatomTimeTagFileConfig`
| Field | Type | Description |
| :--- | :--- | :--- |
| `format` | `int` | Enum specifying export format (`BINARY`, `CSV`, `TAB`). |
| `max_file_size_mb` | `int` | File rotation threshold in MiB, header included; no file exceeds it. `0` disables rotation by size. |
| `max_duration_minutes` | `int` | File rotation threshold. `0` disables rotation by time. |
| `max_event_count` | `int` | File rotation threshold. `0` disables rotation by count. |
| `rotate_on_acquisition_boundary` | `bool` | Requests supported acquisition-boundary rotation. |
| `include_timestamp` | `bool` | Adds timestamp information to the generated filename. |
| `flush_immediately` | `bool` | Requests file-manager flushing; not a durable-storage guarantee. |
| `custom_suffix` | byte string | Optional filename suffix (fixed 64-byte field). |

`NexatomProcessedFileConfig` has the same option fields but uses `NEXATOM_PD_FILE_*` formats. Time-tag formats use `NEXATOM_TT_FILE_*`; their integer values are not interchangeable. The aligned host `NexatomTimeTagFileHeader` exposes `magic`, `created_timestamp_us`, `output_data_type` and `format_version` (1 or 2); it is not the serialized disk layout.

---

<a id="count-rate-data"></a>

### [Count rate (CPS) data](5_3_data_structures.md#count-rate-data)

Dispatched by `set_count_rate_callback`. Provides high-frequency throughput metrics.

#### `NexatomCpsData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `total_count` | `int` | Sum of the channel rate values supplied by the decoded CPS result. |
| `measurement_period_ms`| `int` | The integration time slice for this packet. |
| `num_channels` | `int` | Valid channel prefix in the fixed array. |
| `counts` | uint32 array[8] | Per-channel rates, already in Hz. Do not divide by `measurement_period_ms` again. |
| `gap_flags`, `gap_ms` | `int` | UTT160810: marks a window in which counting paused (calibration, input-delay change). See [Gaps in a CPS window](../02_hardware/2_10_cps_configuration.md). |

---

<a id="time-travel-histogram-data"></a>

### [Time Interval Histogram (TIHI) data](5_3_data_structures.md#time-travel-histogram-data)

Dispatched by `set_time_histogram_callback`. Contains TCSPC measurement data and curve-fitting results.

#### `NexatomTihiData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `bins` | uint64 array[1024] | Histogram storage; only the first `num_bins` entries are valid. |
| `num_bins` | `int` | Number of active bins in the histogram. |
| `bin_width_ps` | `int` | Temporal width of each bin. |
| `total_counts` | `int` | Sum of all events successfully binned. |
| `peak_bin_index` | `int` | Index of the bin containing the highest count. |
| `fwhm_ps` | `float` | Computed Full-Width at Half-Maximum of the primary peak. |
| `roi` | `NexatomTihiRoiData` | Data regarding the user-defined Region of Interest bounds. |
| `fitting` | `NexatomTihiFittingData`| Nested struct containing Levenberg-Marquardt solver results. |
| `acquisition_done_status` | `int` | Terminal reason code; inspect for overflow or unexpected stop. |
| `packets_accumulated` | `int` | Hardware batches in this result. |
| `result_status`, `result_span`, `block_index`, `live_time_ms`, `live_time_exact` | integers/float | Result-model fields; see [result model fields](#result-model-fields). |
| `background_subtracted`, `bg_method`, `background_level_per_bin` | flag/integer/float | Background processing applied to this result. |

#### `NexatomTihiFittingData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `model_type` | byte string | Name of the fitted model. The setter uses an enum; this result field is text. |
| `attempted`, `converged`, `fitted_curve_available` | flags | Whether fitting ran, converged and produced a curve. |
| `chi_squared` | `float` | Primary goodness-of-fit metric ($\chi^2$). |
| `r_squared` | `float` | Coefficient of determination ($R^2$). |
| `fitted_curve_size`, `fitted_curve` | count/float array[1024] | Valid fitted-curve prefix; inspect the availability flag. |
| `exp_amplitude`, `exp_lifetime_ns`, `exp_background` | `float` | Single-exponential parameters ($A, \tau, B$). |
| `gauss_*`, `lorentz_*`, `biexp_*`, `stretch_*` | named floats | Model-specific parameters, including centers/widths/lifetimes in ns and stretched-exponential `stretch_beta`. |
| `reduced_chi_squared`, `degrees_of_freedom`, `iterations` | float/integers | Additional fit quality and solver diagnostics. |

---

<a id="multi-fold-coincidence-data"></a>

### [Multi-Fold Coincidence (MFCO) data](5_3_data_structures.md#multi-fold-coincidence-data)

Dispatched by `set_multifold_coincidence_callback`.

#### `NexatomMfcoData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `pattern_bins` | uint64 array[256] | Index represents the eight-bit channel combination. |
| `coincidence_window_ps` | `int` | The active coincidence grouping window, in ps. |
| `singles` | uint64 array[8] | Single-channel pattern counts (`singles[i] = pattern_bins[1 << i]`); what they count follows `counting_rule`. |
| `counting_rule` | `int` | `NEXATOM_MFCO_COUNTING_RULE_PER_HIT` (1), `_PER_EVENT` (2) or `_UNKNOWN` (0, also when the batches of a result differ); see `get_mfco_counting_rule()`. |
| `num_doubles` | `int` | Total count of 2-fold coincidences. |
| `num_triples` | `int` | Total count of 3-fold coincidences. |
| `top_patterns` | `NexatomMfcoTopPattern[10]` | Pattern/count records for the most frequent patterns. |
| `result_span`, `packets_accumulated` | `int` | What the result covers and how many hardware batches it holds. A `WHOLE_RUN` result is a running total; do not add successive results together. |
| `result_status`, `block_index`, `live_time_ms`, `live_time_exact` | integers/float | Result-model fields; see [result model fields](#result-model-fields). |
| `acquisition_done_status` | `int` | Decoded terminal reason code. |
| `result_metadata_version` | `int` | Version qualifying the following metadata; `result_metadata_available` exposes the check. |
| `done_status_error_flags_raw`, `host_quality_flags` | `int` | Device status/error byte and host saturation/quality flags. |
| `measurement_duration_ms` | `int` | Reported measurement duration where metadata is available. Do not substitute elapsed Python wall time without establishing the correct rate denominator. |

---

<a id="linear-correlation-data"></a>

### [Linear Correlation (CORL) data](5_3_data_structures.md#linear-correlation-data)

#### `NexatomCorlData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `g2_values` | float array[80] | The computed correlation values; check normalization validity. |
| `lag_times_ns` | uint64 array[80] | The corresponding linear lag times in nanoseconds. |
| `baseline_level` | `float` | Calculated baseline asymptote value. |
| `contrast` | `float` | Correlation contrast ratio. |
| `sampling_period_ns`, `channel_a`, `channel_b` | integers | Sampling period and channel selection. |
| `normalization_valid` | flag | Whether `g2_values` can be interpreted as normalized g². |
| `sum_a_counts`, `sum_b_counts`, `sample_count`, `mean_intensity_a`, `mean_intensity_b` | integers/floats | Counts and intensity statistics used for normalization. |
| `decay_1e_index` | `int` | First lag index whose g2 is at or below `baseline_level + contrast / e` (the 1/e point of the decay); 0 when undefined. A lag index: the lag is `decay_1e_index × sampling_period_ns`. For an autocorrelation, lag 0 is left out of this search and of the peak search for `contrast`. |
| `num_bins`, `num_bins_auto`, `count_rate` | integers | The samples per batch (N) in force for the latest batch (0 = not known yet), whether the SDK chooses N (`num_bins` set to 0, the default), and the count rate against N: `NEXATOM_CORR_COUNT_RATE_OK` (0), `_NUM_BINS_LOWERED` (1, the SDK lowered N to keep batches from overflowing; the result is valid) or `_TOO_HIGH` (2, overflowed batches are left out of the result). |
| `result_span`, `status`, `packets_accumulated`, `measurement_duration_ms` | integers | What the result covers, why the latest hardware batch ended, the batches pooled and host duration. |
| `result_status`, `block_index`, `live_time_ms`, `live_time_exact` | integers/float | Result-model fields; see [result model fields](#result-model-fields). |

---

<a id="multi-tau-correlation-data"></a>

### [Multi-Tau Correlation (CORM) data](5_3_data_structures.md#multi-tau-correlation-data)

#### `NexatomCormData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `g2_values` | float array[80] | The computed correlation values; check normalization validity. |
| `lag_times_ns` | uint64 array[80] | The quasi-logarithmic lag times. |
| `fit_result.correlation_time_ns` | `float` | Extracted $\tau_c$ fit parameter, in ns. |
| `fit_result.beta` | `float` | Extracted coherence factor. |
| `fit_result.attempted`, `.converged`, `.fitted_curve_available` | flags | Main fitting validity indicators. |
| `fit_result.fitted_curve`, `.roi_start_index`, `.roi_end_index` | array/indices | Curve values and selected fitting region. |
| `analysis_result.analysis_type` | `int` | Which nested result is exported: NONE, DLS, FCS or DCS. |
| `analysis_result.dls`, `.fcs`, `.dcs` | structures | Technique-specific values; use only the selected result. |
| `normalization_valid`, `sum_a_counts`, `sum_b_counts`, `sample_count` | flag/integers | Normalization evidence, as for CORL. |
| `base_sampling_period_ns`, `tau_groups`, `packets_accumulated`, `measurement_duration_ms` | values/arrays | Lag grouping, batches pooled and host duration. |
| `num_bins`, `num_bins_auto`, `count_rate` | integers | As for CORL. |
| `result_status`, `result_span`, `block_index`, `live_time_ms`, `live_time_exact` | integers/float | Result-model fields; see [result model fields](#result-model-fields). |

<a id="result-model-fields"></a>

### [Result model fields](5_3_data_structures.md#result-model-fields)

TIHI, MFCO, CORL, CORM and Fast TIHI results carry the same result-model fields. The hardware measures in batches that the SDK programs (1 s for TIHI, MFCO and Fast TIHI; `num_bins × T` for the correlators); a result never contains part of a batch.

| Field | Values | Meaning |
| :--- | :--- | :--- |
| `result_status` | `NEXATOM_RESULT_STATUS_RUNNING` (0) | `WHOLE_RUN` progress update; the run continues. |
| | `NEXATOM_RESULT_STATUS_BLOCK_COMPLETE` (1) | A `BLOCK` result reached its length; the next block has started. |
| | `NEXATOM_RESULT_STATUS_RUN_COMPLETE` (2) | The run-end condition set with `set_run_end()` was met; the SDK stopped the processor. Last result of the run. |
| | `NEXATOM_RESULT_STATUS_STOPPED` (3) | Final partial result after Stop. Last result of the run. |
| `result_span` | `NEXATOM_RESULT_SPAN_WHOLE_RUN` (0) | Total since Start, `clear_result()` or a restarting change, republished about once a second (default). |
| | `NEXATOM_RESULT_SPAN_BLOCK` (1) | One tumbling block; the next block starts empty. |
| `block_index` | integer | Zero-based block number in `BLOCK`; 0 in `WHOLE_RUN`. |
| `live_time_ms` | float | Measurement time in the result, excluding the dead time between batches. Correlators: Σ N × T. |
| `live_time_exact` | 0/1 | 1 when every batch had a known length; 0 when Stop ended a batch (its length is host time). |

**After Stop**, exactly one `STOPPED` result arrives per processor, within 2 s. Wait for it before disabling file saving, or the saved files end before the last partial block. For Fast TIHI, the `STOPPED` result reports the window that Stop ended, so its `run_id` can be newer than the last one you saw. `run_id` is a 32-bit hardware counter: compare it wrap-safely (for example `((new - old) & 0xFFFFFFFF) < 0x80000000`) and never match on an exact value.

Use the main `fit_result` quality fields. Reserved quality fields in nested analysis records can remain zero; uncomputed derived quantities can be NaN or zero. Enabling an analysis does not prove a valid scientific fit. See [analysis interpretation](../06_in_depth_guides/6_4_curve_fitting_and_analysis_pipelines.md).

---

### [Telemetry data](5_3_data_structures.md#telemetry-data)

Dispatched by `set_telemetry_callback` for health monitoring.

#### `NexatomTelemetryData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `uptime_seconds` | `int` | Reported device uptime. |
| `temperature_celsius` | `float` | Decoded temperature where supplied by this runtime. |
| `system_status_word`, `mode_status_word`, `histogram_errors_word` | `int` | Runtime status and error words. |
| `sync_clock_requested`, `sync_clock_active`, `sync_clock_locked` | `bool` | Separate requested, active and locked states. |
| `sync_clock_lost_latched`, `sync_clock_lock_failed_latched` | `bool` | The board lost REF IN while running on it / gave up locking; both clear when the request is released. |
| `calibration_metadata`, `temp_status_flags` | `int` | Runtime-specific calibration/temperature status. |

For cross-runtime code prefer `NexatomTelemetryViewV1` from `get_telemetry_view()` or `set_telemetry_view_callback()`. Its availability flags indicate which fields are present; absent telemetry is unknown, not a zero-valued measurement. A request initiates telemetry delivery; do not assume every runtime broadcasts it periodically.

---

### [Field update structures](5_3_data_structures.md#field-update-structures)

Used during firmware flashes in `BOOTLOADER` mode.

#### `NexatomFieldUpdateSlotInfo`
| Field | Type | Description |
| :--- | :--- | :--- |
| `slot_index` | `int` | Slot index within the reported slot inventory. |
| `is_default` | `int` flag | Whether this slot is designated as the default boot target. |
| `slot_state` | `int` | EMPTY, VALID, PENDING or CORRUPT; compare with `NEXATOM_FIELD_UPDATE_SLOT_STATE_*`. |
| `image_name_ascii32` | `int` | Packed image identifier. |
| `image_version` | `int` | Integer image version; not a semantic-version string. |

#### `NexatomFieldUpdateProgress`
| Field | Type | Description |
| :--- | :--- | :--- |
| `phase` | `int` | The `nexatom_field_update_phase_t` enum (e.g., `ERASING`, `WRITING`). |
| `percent` | `int` | Integer progress percentage for the reported operation. |
| `bytes_sent`, `total_bytes` | `int` | Transfer progress. |
| `last_error`, `slot_index`, `slot_state` | `int` | Error and affected slot context. |

---

### [Configuration dump](5_3_data_structures.md#configuration-dump)

Dispatched by `request_config_dump()` for deep hardware debugging.

#### `NexatomConfigDumpData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `register_count` | `int` | Number reported by the device. |
| `copied_register_count` | `int` | Valid prefix copied into fixed storage (maximum 128). |
| `registers` | `NexatomConfigRegisterValue[128]` | Address/value pairs; each entry has `address` and `value`. |

`NexatomConfigDumpViewV1` carries `configuration_version`, `reported_record_count`, `record_count`, `record_stride` and `records` with identity/value pairs. Python callbacks own a deep copy; C callbacks receive a borrowed view valid only during invocation. This view is the one callback argument passed by pointer; every other callback record is passed by value.

### Fast TIHI and DTC records

`NexatomFastTihiConfig` (with four `NexatomFastTihiContext` entries) configures the fast-histogram contexts for `start_fast_tihi()`; `NexatomFastTihiResult` reports one context's result, with `uint64` bins, `run_id`, `lost_windows` and the [result model fields](#result-model-fields). The native callback passes the record by value, bins included. A Fast TIHI click record (`NexatomFastTihiIndexClick`, read from `.nxti` files) holds `trigger_index` (the trigger pulse just before the click, 64 bit), `relative_time_ps` (time from that pulse to the click), `run_id` (the 1 s window), `context` (0–3), `detector_channel`, `trigger_channel` and `flags`. `NexatomDtcOutputConfigV1`, `NexatomDtcApplyResultV1` and `NexatomDtcStatusV1` describe DTC configuration, the hardware apply decision and current status; `NexatomDtcOutputLimits` and `NexatomDtcReapplyStatus` give an output's limits and what the SDK applies again by itself (see [Pulse outputs](../02_hardware/2_13_dtc_pulse_outputs.md)).

### File saving and status records

| Record | Size | Returned by | See |
|---|---|---|---|
| `NexatomFileSavingStatistics` | 200 B | `get_file_saving_statistics()` | [File-saving statistics](../06_in_depth_guides/6_1_file_saving_and_data_export.md#ending-a-recording) |
| `NexatomSavingResult` | 264 B | `get_saving_result()` | [Saving result](../06_in_depth_guides/6_1_file_saving_and_data_export.md#ending-a-recording) |
| `NexatomIncidentInfo` | 168 B | `get_last_incident()` | same |
| `NexatomBoardLoss` | 48 B | `get_board_loss()` | same |
| `NexatomDataFlowStatus` | 56 B | `get_data_flow_status()` | [Keeping up with the data](../06_in_depth_guides/6_1_file_saving_and_data_export.md#keeping-up-with-the-data) |
| `NexatomStartCheck` | 144 B | `check_start_rate()` | same |
| `NexatomReferenceClockStatus` | 24 B | `get_reference_clock_status()` | [External clock](../02_hardware/2_8_external_clock.md#reference-clock-status) |
| `NexatomMeasurementAbort` | 32 B | `get_measurement_abort()` | [Telemetry](../02_hardware/2_11_telemetry.md#measurement-abort) |
| `NexatomCalibrationHoldState` | 64 B | `get_calibration_hold_state()` | [Calibration](../02_hardware/2_4_calibration.md) |
| `NexatomChannelInputDelay` | — | `get_channel_input_delay()` | [Synthetic input delay](../02_hardware/2_7_synthetic_input_delay.md) |

The Python records add read-only properties for their flags and text: `NexatomFileSavingStatistics.complete`; `NexatomSavingResult.running`, `span_complete`, `board_check_unavailable` and `detail_text`; `NexatomIncidentInfo.block_logged` and `summary_text`; `NexatomBoardLoss.valid`, `from_marker`, `from_telemetry` and `inputs`; `NexatomMeasurementAbort.aborted` and `cleanup_done`; `NexatomReferenceClockStatus.locked` and `fault_code_valid`; `NexatomCalibrationHoldState.recording`, `fault` and `baseline_valid`. A class being present in the package does not establish that the connected model supports it. Check profile features before using these controls.
