## Data Structures

This section explains the principal fields of the SDK's data payloads. When data is dispatched via callbacks or queried synchronously, the Python wrapper supplies typed `ctypes.Structure` records. Tables omit padding and some auxiliary fields for readability; use the packaged definitions for exact layouts. Fixed arrays are ctypes arrays, not Python lists; use `list(record.bins[:record.num_bins])` when a list is needed.

### [Device information](5_3_data_structures.md#device-information)

These structures are populated during USB discovery and device initialization.

#### `NexatomDeviceInfo`
| Field | Type | Description |
| :--- | :--- | :--- |
| `serial_number` | byte string | USB bridge serial identifier. Decode text with UTF-8; preserve the complete value. |
| `firmware_version` | byte string | Descriptive firmware string; not authoritative model/image evidence. |
| `hardware_version` | byte string | Descriptive hardware version. |
| `device_name` | byte string | Human-readable discovery name. |
| `connection_type`, `connection_id` | byte strings | Transport description and device-selection identifier. |

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
| `test_pulse_clock_hz`, `test_pulse_min_period_cycles`, `test_pulse_max_period_cycles` | `int` | Clock and period bounds for the internal test pulse. |

Versioned records include size/version fields. High-level getters initialize and validate these; C callers must follow the corresponding declaration.

---

### [Time tag data](5_3_data_structures.md#time-tag-data)

Structures utilized for raw event streaming and offline `.nxtt` binary decoding.

#### `NexatomTimeTag`
| Field | Type | Description |
| :--- | :--- | :--- |
| `timestamp_ps` | `int` (uint64) | Decoded hardware timestamp in picoseconds. It is not Unix wall-clock time; do not assume every host start creates a new zero origin. |
| `channel` | `int` (uint8) | Decoded event channel identifier; interpret it using the active image/profile's mapping. |

#### `NexatomTimeTagFileConfig`
| Field | Type | Description |
| :--- | :--- | :--- |
| `format` | `int` | Enum specifying export format (`BINARY`, `CSV`, `TAB`). |
| `max_file_size_mb` | `int` | File rotation threshold. `0` disables rotation by size. |
| `max_duration_minutes` | `int` | File rotation threshold. `0` disables rotation by time. |
| `max_event_count` | `int` | File rotation threshold. `0` disables rotation by count. |
| `rotate_on_acquisition_boundary` | `bool` | Requests supported acquisition-boundary rotation. |
| `include_timestamp` | `bool` | Adds timestamp information to the generated filename. |
| `flush_immediately` | `bool` | Requests file-manager flushing; not a durable-storage guarantee. |
| `custom_suffix` | byte string | Optional filename suffix (fixed 64-byte field). |

`NexatomProcessedFileConfig` has the same option fields but uses `NEXATOM_PD_FILE_*` formats. Time-tag formats use `NEXATOM_TT_FILE_*`; their integer values are not interchangeable. The aligned host `NexatomTimeTagFileHeader` exposes `magic`, `created_timestamp_us` and `output_data_type`; it is not the serialized disk layout.

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

---

<a id="time-travel-histogram-data"></a>

### [Time Interval Histogram (TIHI) data](5_3_data_structures.md#time-travel-histogram-data)

Dispatched by `set_time_histogram_callback`. Contains TCSPC measurement data and curve-fitting results.

#### `NexatomTihiData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `bins` | uint32 array[1024] | Histogram storage; only the first `num_bins` entries are valid. |
| `num_bins` | `int` | Number of active bins in the histogram. |
| `bin_width_ps` | `int` | Temporal width of each bin. |
| `total_counts` | `int` | Sum of all events successfully binned. |
| `peak_bin_index` | `int` | Index of the bin containing the highest count. |
| `fwhm_ps` | `float` | Computed Full-Width at Half-Maximum of the primary peak. |
| `roi` | `NexatomTihiRoiData` | Data regarding the user-defined Region of Interest bounds. |
| `fitting` | `NexatomTihiFittingData`| Nested struct containing Levenberg-Marquardt solver results. |
| `acquisition_done_status` | `int` | Terminal reason code; inspect for overflow or unexpected stop. |
| `packets_accumulated` | `int` | Number of packets represented by this result. |
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
| `pattern_bins` | uint32 array[256] | Index represents the eight-bit channel combination. |
| `coincidence_window_ps` | `int` | The active coincidence grouping window, in ps. |
| `singles` | uint32 array[8] | Single-channel pattern counts. |
| `num_doubles` | `int` | Total count of 2-fold coincidences. |
| `num_triples` | `int` | Total count of 3-fold coincidences. |
| `top_patterns` | `NexatomMfcoTopPattern[10]` | Pattern/count records for the most frequent patterns. |
| `aggregation_mode`, `packets_accumulated` | `int` | How consecutive packets are represented. Do not sum overlapping accumulated snapshots again. |
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
| `aggregation_mode`, `status`, `packets_accumulated`, `measurement_duration_ms` | integers | Aggregation and acquisition context. |

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
| `base_sampling_period_ns`, `tau_groups`, `packets_accumulated`, `measurement_duration_ms` | values/arrays | Lag grouping and accumulation context. |

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

`NexatomConfigDumpViewV1` carries `configuration_version`, `reported_record_count`, `record_count`, `record_stride` and `records` with identity/value pairs. Python callbacks own a deep copy; C callbacks receive a borrowed view valid only during invocation.

### Fast TIHI and DTC records

`NexatomFastTihiConfigV1`/`V2` configure supported fast-histogram contexts; `NexatomFastTihiHistogramV1` reports their results. `NexatomDtcOutputConfigV1`, `NexatomDtcApplyResultV1` and `NexatomDtcStatusV1` describe DTC configuration, the hardware apply decision and current status. A class being present in the package does not establish that the connected model supports it. Check profile features before using these controls.
