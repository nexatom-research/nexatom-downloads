# File Saving

The File Saving module enables direct-to-disk streaming of instrument data. This circumvents the need for the host application to manually serialize data payloads received in the callbacks.

The C++ background thread handles high-speed disk I/O, utilizing asynchronous buffering and automatic file rotation based on limits (e.g., maximum megabytes, duration, or event counts).

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_time_tag_file_saving` | `[In] nexatom_tt_handle device`<br>`[In] const char* filename`<br>`[In] nexatom_time_tag_file_format_t format` | `nexatom_error_code_t` | Enables native time-tag saving; NXTT uses packed decoded records, not aligned 16-byte C structs. |
| `nexatom_tt_disable_time_tag_file_saving`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Finalizes the native time-tag sink and closes its file; check the result. This is not a power-loss durability guarantee. |
| `nexatom_tt_set_time_tag_file_config` | `[In] nexatom_tt_handle device`<br>`[In] const nexatom_time_tag_file_config_t* config` | `nexatom_error_code_t` | Configures file rotation limits and naming suffix rules for raw binary files. |
| `nexatom_tt_enable_processed_file_saving`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_processed_packet_type_t packet_type`<br>`[In] const char* filename`<br>`[In] nexatom_processed_file_format_t format` | `nexatom_error_code_t` | Arms the background thread to serialize aggregated histogram or correlation curves (TIHI, MFCO, CORM) to disk. |
| `nexatom_tt_disable_processed_file_saving`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_processed_packet_type_t packet_type` | `nexatom_error_code_t` | Flushes and closes the target processed stream file. |
| `nexatom_tt_set_processed_file_config` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_processed_packet_type_t packet_type`<br>`[In] const nexatom_processed_file_config_t* config`| `nexatom_error_code_t` | Configures file rotation limits for processed payloads. |

### Data Structures: `nexatom_time_tag_file_config_t`

*(Note: `nexatom_processed_file_config_t` shares this identical memory layout, with `nexatom_processed_file_format_t` for its format field).*

Pass this structure through a const pointer as declared above. Zero disables the corresponding per-file limit. HDF5 uses a session container rather than applying all per-packet rolling limits.

| Field | Type | Description |
| :--- | :--- | :--- |
| `format` | `nexatom_time_tag_file_format_t` | File format specifier (e.g., standard `.nxtt` binary). |
| `max_file_size_mb` | `uint64_t` | Closes the current file and creates a new suffix part when this size (in MB) is exceeded. |
| `max_duration_minutes` | `uint32_t` | Rotates the file after X minutes of acquisition time. |
| `max_event_count` | `uint64_t` | Rotates the file after X absolute photon events. |
| `rotate_on_acquisition_boundary`| `bool` | True to cleanly rotate the file every time the hardware is stopped and started again. |
| `include_timestamp` | `bool` | Includes the run timestamp in the filename, in compact `YYYYMMDD_HHMMSS_mmm` form. |
| `flush_immediately` | `bool` | Selects immediate file-manager flushing; not a promise of per-write `fsync` or completed acquisition. |
| `custom_suffix` | `char[64]` | Optional string appended before the file extension. |
| `_padding` | `uint8_t[5]` | FFI alignment padding. |

### C Example: Streaming Raw Binaries

```c
nexatom_time_tag_file_config_t config = {0};
config.format = NEXATOM_TT_FILE_BINARY;
config.max_file_size_mb = 1024; // Rotate every 1 GB
config.rotate_on_acquisition_boundary = true;
config.include_timestamp = true;

// Fragment: profile permits raw output; complete code checks every result.
// Configure while quiet, then prepare the sink before enabling acquisition.
nexatom_tt_enable_system(my_device, false);
nexatom_tt_set_output_type(my_device, NEXATOM_OUTPUT_NO_OUTPUT);
nexatom_tt_set_time_tag_file_config(my_device, &config);

// 2. Enable the sink; actual file creation may wait for data.
nexatom_tt_enable_time_tag_file_saving(
    my_device, "/data/my_experiment_run", NEXATOM_TT_FILE_BINARY
);

// 3. Start the hardware
nexatom_tt_set_output_type(my_device, NEXATOM_OUTPUT_RAW_TAGS);
nexatom_tt_enable_system(my_device, true);

// Observe a bounded capture using an appropriate input source.

// 4. Safely terminate the stream
nexatom_tt_enable_system(my_device, false);
nexatom_tt_set_output_type(my_device, NEXATOM_OUTPUT_NO_OUTPUT);
nexatom_tt_disable_time_tag_file_saving(my_device);
```

Use the complete raw template for drain/finalization and checked disconnect. Enabling raw saving disables processed saving and vice versa; active processed CSV/TAB cannot be mixed with HDF5. Formats, CSV metadata parsing, sequence naming and rotation are described in [file saving and export](../06_in_depth_guides/6_1_file_saving_and_data_export.md).

### Choosing a readable format

Time-tag saving supports `NEXATOM_TT_FILE_BINARY`, `NEXATOM_TT_FILE_CSV` and `NEXATOM_TT_FILE_TAB`. The text formats contain individual `timestamp_ps,channel` events (or the same fields separated by a tab). Keep the timestamp as an integer when importing it; converting a large timestamp to a floating-point spreadsheet cell can lose precision. Use the [binary reader](7_18_binary_decoder.md) for efficient batched processing of NXTT captures.

Processed saving supports CSV, TAB and HDF5. Select the packet family separately from its file format; enabling a sink does not start the associated measurement engine. HDF5 is a structured binary container for scientific tools, rather than a plain-text document.

**Preview.8 known limitation:** the native CORL and CORM CSV writers emit JSON metadata without the CSV quoting needed for its commas and quotation marks. Standard CSV readers can reject these rows or split them incorrectly. Use TAB or HDF5 for correlation data in this release; do not interpret a nonempty CSV file as evidence of a valid import. This limitation does not change the two-column time-tag CSV format.
