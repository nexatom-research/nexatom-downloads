## Data Structures

This section details the memory layouts and field types for all data payload structures utilized by the SDK. When data is dispatched via callbacks (or queried synchronously), the Python wrapper translates the native C structs into strictly typed Python objects (or `ctypes.Structure` subclasses).

### [Device information](5_3_data_structures.md#device-information)

These structures are populated during USB discovery and device initialization.

#### `NexatomDeviceInfo`
| Field | Type | Description |
| :--- | :--- | :--- |
| `serial_number` | `str` | Unique factory identifier (e.g., `"UTT810-A1B2"`). |
| `firmware_version` | `str` | Active firmware semantic version. |
| `hardware_version` | `str` | Hardware PCB revision string. |
| `device_name` | `str` | Human-readable product name. |
| `connection_id` | `str` | Topological USB path used for OS-level re-enumeration tracking. |

#### `NexatomCapabilities`
| Field | Type | Description |
| :--- | :--- | :--- |
| `channel_count` | `int` | Number of physical inputs (typically 8). |
| `max_count_rate_cps` | `int` | The absolute maximum hardware throughput limit. |
| `time_resolution_ps` | `int` | Base hardware resolution (picoseconds). |
| `max_threshold_mv` | `int` | Maximum valid discriminator voltage. |
| `feature_flags` | `int` | Bitmask indicating active optional features (e.g., CORM, DLS support). |

---

### [Time tag data](5_3_data_structures.md#time-tag-data)

Structures utilized for raw event streaming and offline `.nxtt` binary decoding.

#### `NexatomTimeTag`
| Field | Type | Description |
| :--- | :--- | :--- |
| `timestamp_ps` | `int` (uint64) | Absolute hardware timestamp in picoseconds since acquisition start. |
| `channel` | `int` (uint8) | The physical input channel (0-7) that generated the event. |

#### `NexatomTimeTagFileConfig`
| Field | Type | Description |
| :--- | :--- | :--- |
| `format` | `int` | Enum specifying export format (`BINARY`, `CSV`, `TAB`). |
| `max_file_size_mb` | `int` | File rotation threshold. `0` disables rotation by size. |
| `max_duration_minutes` | `int` | File rotation threshold. `0` disables rotation by time. |
| `max_event_count` | `int` | File rotation threshold. `0` disables rotation by count. |

---

### [Count rate (CPS) data](5_3_data_structures.md#count-rate-data)

Dispatched by `set_count_rate_callback`. Provides high-frequency throughput metrics.

#### `NexatomCpsData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `total_count` | `int` | Sum of all events detected across all channels during this period. |
| `measurement_period_ms`| `int` | The integration time slice for this packet. |
| `channel_counts` | `list[int]` | Array of exactly 8 integers containing per-channel counts. |

---

### [Time Interval Histogram (TIHI) data](5_3_data_structures.md#time-travel-histogram-data)

Dispatched by `set_time_histogram_callback`. Contains TCSPC measurement data and curve-fitting results.

#### `NexatomTihiData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `bins` | `list[int]` | Histogram array of size `num_bins` (max 1024). |
| `num_bins` | `int` | Number of active bins in the histogram. |
| `bin_width_ps` | `int` | Temporal width of each bin. |
| `total_counts` | `int` | Sum of all events successfully binned. |
| `peak_bin_index` | `int` | Index of the bin containing the highest count. |
| `fwhm_ps` | `float` | Computed Full-Width at Half-Maximum of the primary peak. |
| `roi` | `NexatomTihiRoiData` | Data regarding the user-defined Region of Interest bounds. |
| `fitting` | `NexatomTihiFittingData`| Nested struct containing Levenberg-Marquardt solver results. |

#### `NexatomTihiFittingData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `model_type` | `int` | The `nexatom_fitting_model_t` enum (e.g., `EXPONENTIAL`). |
| `convergence_status` | `int` | Non-zero if the solver failed to converge. |
| `chi_squared` | `float` | Primary goodness-of-fit metric ($\chi^2$). |
| `r_squared` | `float` | Coefficient of determination ($R^2$). |
| `fitted_curve` | `list[float]` | The evaluated fit equation array, mapping 1:1 with the `bins` array. |
| `parameters` | `list[float]` | The extracted variables ($A, \tau, \beta, \mu, \sigma$, etc.) specific to the model type. |

---

### [Multi-Fold Coincidence (MFCO) data](5_3_data_structures.md#multi-fold-coincidence-data)

Dispatched by `set_multifold_coincidence_callback`.

#### `NexatomMfcoData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `pattern_bins` | `list[int]` | Array of 256 integers. Index represents the 8-bit channel combination. |
| `window_ps` | `int` | The active coincidence grouping window. |
| `singles` | `list[int]` | Array of 8 integers tracking non-coincident events per channel. |
| `num_doubles` | `int` | Total count of 2-fold coincidences. |
| `num_triples` | `int` | Total count of 3-fold coincidences. |
| `top_patterns` | `list[TopPattern]` | Array of the 10 most frequent patterns, sorted descending by count. |

---

### [Linear Correlation (CORL) data](5_3_data_structures.md#linear-correlation-data)

#### `NexatomCorlData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `g2_values` | `list[float]` | The computed correlation array (80 elements). |
| `lag_times_ns` | `list[float]` | The corresponding linear lag times in nanoseconds (80 elements). |
| `baseline` | `float` | Calculated baseline asymptote value. |
| `contrast` | `float` | Correlation contrast ratio. |

---

### [Multi-Tau Correlation (CORM) data](5_3_data_structures.md#multi-tau-correlation-data)

#### `NexatomCormData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `g2_values` | `list[float]` | The computed correlation array (80 elements). |
| `lag_times_ns` | `list[float]` | The quasi-logarithmic lag times (80 elements). |
| `correlation_time` | `float` | Extracted $\tau_c$ fit parameter. |
| `beta` | `float` | Extracted coherence factor fit parameter. |
| `analysis_type` | `int` | Enum identifying if `analysis_result` contains DLS, FCS, or DCS data. |

---

### [Telemetry data](5_3_data_structures.md#telemetry-data)

Dispatched by `set_telemetry_callback` for health monitoring.

#### `NexatomTelemetryData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `uptime_sec` | `int` | Device uptime since last hard reboot. |
| `temperature_c` | `float` | Internal FPGA core temperature (Celsius). |
| `system_word` | `int` | Bitmask of active system faults or warnings. |
| `sync_clock_active` | `bool` | True if successfully locked to an external 10MHz reference. |
| `calibration_mask` | `int` | Bitmask indicating which channel delay lines are actively calibrated. |

---

### [Field update structures](5_3_data_structures.md#field-update-structures)

Used during firmware flashes in `BOOTLOADER` mode.

#### `NexatomFieldUpdateSlotInfo`
| Field | Type | Description |
| :--- | :--- | :--- |
| `slot_index` | `int` | Physical flash memory slot (0, 1, or 2). |
| `is_default` | `bool` | True if this slot is the designated auto-boot target. |
| `is_valid` | `bool` | True if the image passed CRC/signature verification. |
| `version` | `str` | Semantic version of the image residing in this slot. |

#### `NexatomFieldUpdateProgress`
| Field | Type | Description |
| :--- | :--- | :--- |
| `phase` | `int` | The `nexatom_field_update_phase_t` enum (e.g., `ERASING`, `WRITING`). |
| `percent_complete` | `float` | Range 0.0 to 100.0 tracking the current phase. |

---

### [Configuration dump](5_3_data_structures.md#configuration-dump)

Dispatched by `request_config_dump()` for deep hardware debugging.

#### `NexatomConfigDumpData`
| Field | Type | Description |
| :--- | :--- | :--- |
| `register_count` | `int` | Number of populated registers (Max 128). |
| `addresses` | `list[int]` | Raw FPGA register addresses. |
| `values` | `list[int]` | Hexadecimal values currently stored in the corresponding addresses. |
