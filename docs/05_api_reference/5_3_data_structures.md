# 5.3 Data structures

Use the structures exported by the packaged `nexatomtt` module. They preserve the C ABI's natural alignment and reserved fields; do not recreate layouts from abbreviated tables below. Versioned records require their documented size/version and availability checks, handled by the high-level methods where applicable.

| Record | Fields and interpretation |
| --- | --- |
| `NexatomDeviceInfo` | Full `serial_number`, firmware/hardware strings, device name, connection type/ID; discovery strings are not authoritative model evidence |
| `NexatomCapabilities` | `num_channels`, `max_count_rate`, `time_resolution_ps`, `max_threshold_mv`, `max_histogram_bins`, supported-operation booleans; use with the resolved profile |
| `NexatomDeviceProfileV1` | Authority/service flags, identity source/model/image, effective public mask, features, supported output mask, delay limit and pulse timing |
| `NexatomCpsData` | `total_count`, `measurement_period_ms`, `num_channels`, `counts`; channel values already Hz |
| `NexatomTihiData` | Channels, `num_bins`, `bin_width_ps`, status, statistics, fitting/ROI records, aggregation and valid prefix of `bins` |
| `NexatomMfcoData` | `coincidence_window_ps`, 256 `pattern_bins`, singles/order counts, filters/top patterns, aggregation, completion and versioned quality metadata |
| `NexatomCorlData` | 80 `g2_values`, `lag_times_ns`, sampling period, channels, aggregation/status and normalization statistics |
| `NexatomCormData` | 80 points, lag groups, fitting/analysis results, accumulation and normalization statistics |
| `NexatomTelemetryData` | Legacy fixed record including `uptime_seconds`, `temperature_celsius`, `system_status_word` |
| `NexatomTelemetryViewV1` | Versioned cross-runtime view with explicit availability; absent fields are unknown |
| `NexatomConfigDumpData` | `register_count`, `copied_register_count` and `registers` entries with address/value |
| `NexatomConfigDumpViewV1` | Versioned configuration view; inspect supplied record count and identity |
| `NexatomTimeTag` | `timestamp_ps`, `channel`; aligned host structure, not a serialized disk record |
| `NexatomTimeTagFileHeader` | `magic`, `created_timestamp_us`, `output_data_type`; host representation of the disk header |
| `NexatomTimeTagFileConfig`, `NexatomProcessedFileConfig` | Size/duration/event rotation limits, acquisition-boundary rotation, timestamp/flush flags and suffix |

Field-update records use integer identifiers, not inferred display strings: `NexatomFieldUpdateSlotInfo` contains `slot_index`, `image_name_ascii32`, `image_version`, `slot_state` and `is_default`; progress contains integer `phase`, `percent`, byte counts and error/slot fields. Use the reported slot count.

Fast TIHI uses `NexatomFastTihiConfigV1`/`V2` and `NexatomFastTihiHistogramV1`. DTC uses `NexatomDtcOutputConfigV1`, `NexatomDtcApplyResultV1` and `NexatomDtcStatusV1`. Their presence in Python does not imply active-device support.

For correlation, check `normalization_valid` before calling data normalized g². Use the main fit record's validity; nested reserved quality fields can remain zero, and unavailable derived quantities can be NaN. See [analysis](../06_in_depth_guides/6_4_curve_fitting_and_analysis_pipelines.md).

[Python reference](index.md)
