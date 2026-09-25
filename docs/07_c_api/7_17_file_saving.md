# File Saving

The File Saving module enables direct-to-disk streaming of instrument data. This circumvents the need for the host application to manually serialize data payloads received in the callbacks.

The C++ background thread handles high-speed disk I/O, utilizing asynchronous buffering and automatic file rotation based on limits (e.g., maximum megabytes, duration, or event counts).

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_time_tag_file_saving` | `[In] nexatom_tt_handle device`<br>`[In] const char* filename`<br>`[In] nexatom_time_tag_file_format_t format` | `nexatom_error_code_t` | Enables native time-tag saving; NXTT uses packed decoded records, not aligned 16-byte C structs. |
| `nexatom_tt_disable_time_tag_file_saving`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Finalizes the native time-tag sink and closes its file; check the result. This is not a power-loss durability guarantee. |
| `nexatom_tt_get_file_saving_statistics` | `[In] nexatom_tt_handle device`<br>`[In,Out] nexatom_tt_file_saving_statistics_v1_t* statistics` | `nexatom_error_code_t` | Reads the file-saving stage counters, so you can prove a capture is complete. Synchronous and cache-only: it never touches the device. Returns `NEXATOM_ERROR_INVALID_PARAMETER` for a null or wrongly sized record and `NEXATOM_ERROR_NOT_CONNECTED` before connect. |
| `nexatom_tt_set_time_tag_file_config` | `[In] nexatom_tt_handle device`<br>`[In] const nexatom_time_tag_file_config_t* config` | `nexatom_error_code_t` | Configures file rotation limits and naming suffix rules for raw binary files. |
| `nexatom_tt_enable_processed_file_saving`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_processed_packet_type_t packet_type`<br>`[In] const char* filename`<br>`[In] nexatom_processed_file_format_t format` | `nexatom_error_code_t` | Arms the background thread to save one processed packet family (CPS, TIHI, MFCO, CORL, CORM, telemetry, Fast TIHI index). Files hold each packet received from the instrument, before host summing, background subtraction, fitting or pattern filtering; the summed result reaches only the callbacks. |
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

<a id="file-saving-statistics"></a>

### Proving a capture is complete: `nexatom_tt_get_file_saving_statistics`

A run of successful return codes does not prove that every time tag reached the file. The file-saving stage counts each raw tag at three points (received, turned into a write task, written) and reports the counts in a caller-owned, versioned record:

| Field | Type | Meaning |
| :--- | :--- | :--- |
| `struct_size` | `uint32_t` | In: `sizeof` your record. Out: bytes written. |
| `struct_version` | `uint32_t` | In/out: `NEXATOM_TT_FILE_SAVING_STATISTICS_V1_VERSION` (1). |
| `buffers_received` | `uint64_t` | Buffers that reached the file stage. |
| `buffers_processed` | `uint64_t` | Buffers the file stage finished handling. |
| `buffers_dropped` | `uint64_t` | Buffers discarded unwritten: the input queue was full, or they were still queued at a `NO_OUTPUT` barrier or at disable. |
| `raw_tags_received` | `uint64_t` | Raw tags in the buffers received. |
| `raw_tags_enqueued` | `uint64_t` | Raw tags turned into write tasks. |
| `raw_tags_written` | `uint64_t` | Raw tags the writer put in a file. |
| `bytes_written` | `uint64_t` | Bytes written by the writer thread. |
| `write_errors` | `uint64_t` | Failed file operations. |
| `files_created` | `uint64_t` | Data files opened. |
| `rotation_count` | `uint64_t` | Size, time or count rotations performed. |
| `file_saving_enabled` | `uint32_t` | 1 while saving is enabled, else 0. |
| `reserved[3]` | `uint32_t` | Zero. |

The record is 104 bytes (`NEXATOM_TT_FILE_SAVING_STATISTICS_V1_SIZE`). The counters are cumulative for the handle and never reset, so take a snapshot before enabling saving and compare differences. After `nexatom_tt_disable_time_tag_file_saving()` returns, the file stage kept everything it was given when `raw_tags_received == raw_tags_written`, `buffers_dropped == 0` and `write_errors == 0`.

```c
// Fragment: my_device is connected; the capture runs as in the example above.
nexatom_tt_file_saving_statistics_v1_t before = {0};
before.struct_size = sizeof(before);
before.struct_version = NEXATOM_TT_FILE_SAVING_STATISTICS_V1_VERSION;
nexatom_tt_get_file_saving_statistics(my_device, &before);

/* ... enable saving, capture, NO_OUTPUT, disable saving ... */

nexatom_tt_file_saving_statistics_v1_t after = before;  // keeps size and version
nexatom_tt_get_file_saving_statistics(my_device, &after);
uint64_t received = after.raw_tags_received - before.raw_tags_received;
uint64_t written  = after.raw_tags_written  - before.raw_tags_written;
int complete = received == written &&
               after.buffers_dropped == before.buffers_dropped &&
               after.write_errors == before.write_errors;
printf("%llu of %llu tags written: %s\n", (unsigned long long)written,
       (unsigned long long)received, complete ? "complete" : "INCOMPLETE");
```

In Python, `device.get_file_saving_statistics()` returns the same record with a `complete` property.

The counters see only the file stage. Switching to `NO_OUTPUT` runs the pipeline barrier, which discards what the decoder and processing stages still hold; those tags never reach the counters. At rates the host keeps up with, that backlog is empty.

**File runs.** A file run is bounded only by `nexatom_tt_enable_time_tag_file_saving()` and `nexatom_tt_disable_time_tag_file_saving()`. Switching the output type to `NO_OUTPUT` while saving is enabled flushes the open file and keeps it open; it does not close, reopen or truncate it. A data file path is never opened over an existing file: if the path exists, the sequence number advances (`_0001_`, `_0002_`, …).

**Disconnect drops in-flight data.** `nexatom_tt_disconnect()` is bounded and does not drain the pipeline. Disable saving and check the statistics before disconnecting.

**Processed results after Stop.** For processed saving, each processor publishes exactly one `STOPPED` result within 2 s of its Stop call. Wait for it before disabling processed saving, or the files end before the last partial block.

Use the complete raw template for drain/finalization and checked disconnect. Enabling raw saving disables processed saving and vice versa; active processed CSV/TAB cannot be mixed with HDF5. Formats, CSV metadata parsing, sequence naming and rotation are described in [file saving and export](../06_in_depth_guides/6_1_file_saving_and_data_export.md).

### Choosing a readable format

Time-tag saving supports `NEXATOM_TT_FILE_BINARY`, `NEXATOM_TT_FILE_CSV` and `NEXATOM_TT_FILE_TAB`. The text formats contain individual `timestamp_ps,channel` events (or the same fields separated by a tab). Keep the timestamp as an integer when importing it; converting a large timestamp to a floating-point spreadsheet cell can lose precision. Use the [binary reader](7_18_binary_decoder.md) for efficient batched processing of NXTT captures.

Processed saving supports CSV, TAB and HDF5. Select the packet family separately from its file format; enabling a sink does not start the associated measurement engine. HDF5 is a structured binary container for scientific tools, rather than a plain-text document.

Processed CSV and TAB files (`*_csv_v2`, `*_tab_v2`) carry the same plain-language `#` notes, heading rows and values for every packet family, separated by commas or tabs; standard CSV readers can read them after skipping the `#` lines. HDF5 files use schema 4, one table per measurement type with the CSV headings as member names. See [file saving and export](../06_in_depth_guides/6_1_file_saving_and_data_export.md#processed-data-file-saving).

**Files from earlier SDKs:** the preview.8 CORL and CORM CSV writers emitted JSON metadata without CSV quoting, so standard CSV readers can reject those rows or split them incorrectly. Read such files with a format-specific parser, or use their TAB or HDF5 counterparts.
