# File Saving

The File Saving module enables direct-to-disk streaming of instrument data. This circumvents the need for the host application to manually serialize data payloads received in the callbacks.

The C++ background thread handles high-speed disk I/O, utilizing asynchronous buffering and automatic file rotation based on limits (e.g., maximum megabytes, duration, or event counts).

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_time_tag_file_saving` | `[In] nexatom_tt_handle device`<br>`[In] const char* filename`<br>`[In] nexatom_time_tag_file_format_t format` | `nexatom_error_code_t` | Enables native time-tag saving; NXTT uses packed decoded records, not aligned 16-byte C structs. |
| `nexatom_tt_disable_time_tag_file_saving`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Finalizes the native time-tag sink and closes its file; check the result. This is not a power-loss durability guarantee. During a recording this is Stop: the saved span ends at the call, the writer saves the backlog it holds for up to 2 s, and the call always succeeds. |
| `nexatom_tt_set_save_end` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_run_end_kind_t kind`<br>`[In] uint64_t value` | `nexatom_error_code_t` | Ends each later recording by itself: `NEXATOM_RUN_END_NONE` (until disabled), `_TIME` (value = ms of board time from the first tag) or `_COUNT` (value = tags). Set it before enabling saving; `NEXATOM_ERROR_DEVICE_BUSY` while any saving is enabled. |
| `nexatom_tt_get_saving_result` | `[In] nexatom_tt_handle device`<br>`[In,Out] nexatom_tt_saving_result_t* result` | `nexatom_error_code_t` | The latched result of the current or last recording: how it ended, what was saved, the files. Cache-only, also after disconnect. |
| `nexatom_tt_get_last_incident` | `[In] nexatom_tt_handle device`<br>`[In,Out] nexatom_tt_incident_info_t* info` | `nexatom_error_code_t` | The last incident raised for the handle (id and short summary). |
| `nexatom_tt_get_board_loss` | `[In] nexatom_tt_handle device`<br>`[In,Out] nexatom_tt_board_loss_t* loss` | `nexatom_error_code_t` | A board loss that ended the current or last recording. |
| `nexatom_tt_get_data_flow_status` | `[In] nexatom_tt_handle device`<br>`[In,Out] nexatom_tt_data_flow_status_t* status` | `nexatom_error_code_t` | How the host keeps up while data flows (`OK`, `WARN`, `STALLED`, per stage). |
| `nexatom_tt_check_start_rate` | `[In] nexatom_tt_handle device`<br>`[In] uint64_t expected_input_rate_hz`<br>`[In,Out] nexatom_tt_start_check_t* check` | `nexatom_error_code_t` | Before a start: whether the expected input rate can pass the USB link and the host. |
| `nexatom_tt_get_board_time_upper_bound` | `[In] nexatom_tt_handle device`<br>`[Out] uint64_t* board_time_ps` | `nexatom_error_code_t` | An upper bound of the board time now, in ps; `NEXATOM_ERROR_NOT_AVAILABLE` when there is none. |
| `nexatom_tt_get_file_saving_statistics` | `[In] nexatom_tt_handle device`<br>`[In,Out] nexatom_tt_file_saving_statistics_t* statistics` | `nexatom_error_code_t` | Reads the file-saving stage counters, so you can prove a capture is complete. Synchronous and cache-only: it never touches the device. Returns `NEXATOM_ERROR_INVALID_PARAMETER` for a null or wrongly sized record and `NEXATOM_ERROR_NOT_CONNECTED` before connect. |
| `nexatom_tt_set_time_tag_file_config` | `[In] nexatom_tt_handle device`<br>`[In] const nexatom_time_tag_file_config_t* config` | `nexatom_error_code_t` | Configures file rotation limits and naming suffix rules for raw binary files. |
| `nexatom_tt_enable_processed_file_saving`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_processed_packet_type_t packet_type`<br>`[In] const char* filename`<br>`[In] nexatom_processed_file_format_t format` | `nexatom_error_code_t` | Arms the background thread to save one processed packet family (CPS, TIHI, MFCO, CORL, CORM, telemetry, Fast TIHI index). Files hold each packet received from the instrument, before host summing, background subtraction, fitting or pattern filtering; the summed result reaches only the callbacks. |
| `nexatom_tt_disable_processed_file_saving`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_processed_packet_type_t packet_type` | `nexatom_error_code_t` | Flushes and closes the target processed stream file. During a recording this is Stop for that type: the records the file stage holds at the call are written for up to 2 s; when they are not written in time the call fails with `NEXATOM_ERROR_TIMEOUT` and the saving goes on. |
| `nexatom_tt_set_processed_file_config` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_processed_packet_type_t packet_type`<br>`[In] const nexatom_processed_file_config_t* config`| `nexatom_error_code_t` | Configures file rotation limits for processed payloads. |

### Data Structures: `nexatom_time_tag_file_config_t`

*(Note: `nexatom_processed_file_config_t` shares this identical memory layout, with `nexatom_processed_file_format_t` for its format field).*

Pass this structure through a const pointer as declared above. Zero disables the corresponding per-file limit. HDF5 uses a session container rather than applying all per-packet rolling limits.

| Field | Type | Description |
| :--- | :--- | :--- |
| `format` | `nexatom_time_tag_file_format_t` | File format specifier (e.g., standard `.nxtt` binary). |
| `max_file_size_mb` | `uint64_t` | Size limit per file in MiB, header included. No file exceeds it: the whole tags (or rows) that fit go into the open file and the rest into the next suffix part; a tag is never split. A limit smaller than the header plus one record is refused with `NEXATOM_ERROR_INVALID_PARAMETER`. |
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

// 4. Safely terminate the stream: Stop the recording while RAW_TAGS is still on,
//    check the statistics (below), then go quiet.
nexatom_tt_disable_time_tag_file_saving(my_device);
nexatom_tt_set_output_type(my_device, NEXATOM_OUTPUT_NO_OUTPUT);
nexatom_tt_enable_system(my_device, false);
```

The rotation limits apply at once, also to a file that is already open when `nexatom_tt_set_time_tag_file_config()` is called during a save. The files of a series together hold exactly the tags written, in order.

<a id="file-saving-statistics"></a>

### Proving a capture is complete: `nexatom_tt_get_file_saving_statistics`

A run of successful return codes does not prove that every time tag reached the file. The file-saving stage counts each raw tag at three points (received, turned into a write task, written) and reports the counts in a caller-owned, versioned record:

| Field | Type | Meaning |
| :--- | :--- | :--- |
| `struct_size` | `uint32_t` | In: `sizeof` your record. Out: bytes written. |
| `struct_version` | `uint32_t` | In/out: `NEXATOM_TT_FILE_SAVING_STATISTICS_VERSION` (2); only version 2 is accepted. |
| `buffers_received` | `uint64_t` | Buffers that reached the file stage. |
| `buffers_processed` | `uint64_t` | Buffers the file stage finished handling. |
| `buffers_dropped` | `uint64_t` | Buffers discarded unwritten: the input queue was full, or they were still queued at a `NO_OUTPUT` barrier or at disable. |
| `raw_tags_received` | `uint64_t` | Raw tags the time-tag recording took in, from its start to the Stop's cut. |
| `raw_tags_enqueued` | `uint64_t` | Raw tags turned into write tasks. |
| `raw_tags_written` | `uint64_t` | Raw tags the writer put in a file. |
| `bytes_written` | `uint64_t` | Bytes the writer put on disk, headers included; for time-tag files and processed text files this equals their size on disk. HDF5 counts the data appended, not the file size. |
| `write_errors` | `uint64_t` | Failed file operations. |
| `files_created` | `uint64_t` | First file of each recording and data type; files on disk = `files_created + rotation_count`. A file opens with its first tag (or row), so a save that ends before it creates no file. |
| `rotation_count` | `uint64_t` | Files a size, time or count rotation opened. |
| `file_saving_enabled` | `uint32_t` | 1 while saving is enabled or its result is not final yet, else 0. Once 0, `nexatom_tt_get_saving_result()` is final, except while its `CLOSE_PENDING` flag is set. |
| `raw_save_end_kind`, `raw_save_end_target`, `raw_save_end_done` | `uint32_t`, `uint64_t` | The save end of the current or last recording (`nexatom_tt_set_save_end`): its kind, target and progress (ms of board time since the first tag, or tags; 0 for `NONE`). A progress bar is done / target. |
| `raw_first_board_time_ps`, `raw_last_board_time_ps` | `uint64_t` | Board time of the first tag written in this recording and of the last one so far. |
| `tidx_records_written`, `tidx_chunks_written` | `uint64_t` | Fast TIHI click records and chunks written (UTT160810), cumulative. |
| `tidx_windows_complete`, `tidx_window_mismatches` | `uint64_t` | Windows whose click records equal the window's histogram sum, and windows whose records did not. |
| `tidx_chunk_gaps`, `tidx_duplicate_chunks` | `uint64_t` | Gaps in the click chunk numbers, and duplicate or regressed chunks, cumulative. |
| `tidx_save_end_target`, `tidx_save_end_done`, `tidx_save_end_kind` | `uint64_t`, `uint32_t` | The save end for a click recording, in ms or clicks. |
| `span_complete` | `uint32_t` | 1: the last recording that ended has a whole saved span. Set after every normal end (save end count or time, measurement end reached, Stop, disconnect); clear only on a real defect. 0 while a recording runs and before the first one ends. |

The record is 200 bytes (`NEXATOM_TT_FILE_SAVING_STATISTICS_SIZE`). The counters are cumulative for the handle and never reset, so take a snapshot before enabling saving and compare differences; the `raw_save_end_*` and `raw_*_board_time_ps` fields describe the current or last recording. After `nexatom_tt_disable_time_tag_file_saving()` returns, the file stage kept everything it was given when `raw_tags_received == raw_tags_written`, `buffers_dropped == 0` and `write_errors == 0`.

Whether the last recording's saved span is whole is `span_complete`, not a comparison of the cumulative counters. Tags that reach the file stage after a Stop's cut or a save end's cut are after the end by design: they are counted neither in `raw_tags_received` nor in `buffers_dropped`. A Stop's backlog still queued after its 2 s is also after the end; it is counted in `buffers_dropped` and the span stays complete, because the span ends at the Stop moment.

```c
// Fragment: my_device is connected; the capture runs as in the example above.
nexatom_tt_file_saving_statistics_t before = {0};
before.struct_size = sizeof(before);
before.struct_version = NEXATOM_TT_FILE_SAVING_STATISTICS_VERSION;
nexatom_tt_get_file_saving_statistics(my_device, &before);

/* ... enable saving, capture, NO_OUTPUT, disable saving ... */

nexatom_tt_file_saving_statistics_t after = before;  // keeps size and version
nexatom_tt_get_file_saving_statistics(my_device, &after);
uint64_t received = after.raw_tags_received - before.raw_tags_received;
uint64_t written  = after.raw_tags_written  - before.raw_tags_written;
int complete = after.span_complete != 0 &&
               after.write_errors == before.write_errors;
printf("%llu of %llu tags written: %s\n", (unsigned long long)written,
       (unsigned long long)received, complete ? "complete" : "INCOMPLETE");
```

In Python, `device.get_file_saving_statistics()` returns the same record with a `complete` property (the same fact as `span_complete`). The saving result (`nexatom_tt_get_saving_result()`) then says how the recording ended and lists its files. A whole span is not a whole run: when the SDK ends a save early, for example `NEXATOM_TT_SAVE_END_REASON_HOST_STALLED` (the computer could not keep up), the file holds every tag up to its last one and `span_complete` can stay 1, but the rest of the run is not saved, so check `end_reason` too; see [ending a recording and checking it](../06_in_depth_guides/6_1_file_saving_and_data_export.md#ending-a-recording).

The counters see only the file stage. Switching to `NO_OUTPUT` runs the pipeline barrier, which discards what the decoder and processing stages still hold; those tags never reach the counters. That backlog is whatever the host had not yet processed at the `NO_OUTPUT` moment, so it grows when the data rate exceeds what the host decodes and saves. Stop the recording with `nexatom_tt_disable_time_tag_file_saving()` (or `nexatom_tt_enable_system(false)` during a raw recording) before switching to `NO_OUTPUT`: a Stop saves the backlog it holds.

**File runs.** A recording starts when the first file saving is enabled while none is, and ends when the last one closes: at a Stop, at the save end, at a disconnect, or when the output type changes. Starting output from `NO_OUTPUT` keeps a save running; any other change of output type ends the saved span with the end reason `OUTPUT_TYPE_CHANGED`, a normal end. Setting the current type changes nothing. A data file path is never opened over an existing file: if the path exists, the sequence number advances (`_0001_`, `_0002_`, …).

**Disconnect drops in-flight data.** `nexatom_tt_disconnect()` is bounded and does not drain the pipeline. Disable saving and check the statistics before disconnecting.

**Processed results after Stop.** For processed saving, each processor publishes exactly one `STOPPED` result within 2 s of its Stop call. Wait for it before disabling processed saving, or the files end before the last partial block.

Use the complete raw template for drain/finalization and checked disconnect. Enabling raw saving disables processed saving and vice versa; active processed CSV/TAB cannot be mixed with HDF5; a Fast TIHI index saved in binary (`.nxti`) runs beside HDF5 results, enabled in either order (since preview.22.62). Formats, CSV metadata parsing, sequence naming and rotation are described in [file saving and export](../06_in_depth_guides/6_1_file_saving_and_data_export.md).

### Choosing a readable format

Time-tag saving supports `NEXATOM_TT_FILE_BINARY`, `NEXATOM_TT_FILE_CSV` and `NEXATOM_TT_FILE_TAB`. The text formats hold one row per event, an event time in ps and the input it came on, separated by a comma or a tab, under three `#` lines and a heading row, and end with a closing line that gives the count and how the saving ended; the [text file layout](../06_in_depth_guides/6_1_file_saving_and_data_export.md#text-file-layout) describes it. Keep the timestamp as an integer when importing it; converting a large timestamp to a floating-point spreadsheet cell can lose precision. Use the [binary reader](7_18_binary_decoder.md) for efficient batched processing of NXTT captures, and the [time-tag export](7_18_binary_decoder.md#time-tag-export) to turn an NXTT file or series into one CSV or TAB file later.

Processed saving supports CSV, TAB and HDF5. Select the packet family separately from its file format; enabling a sink does not start the associated measurement engine. HDF5 is a structured binary container for scientific tools, rather than a plain-text document.

Processed CSV and TAB files (`*_csv_v2`, `*_tab_v2`) carry the same plain-language `#` notes, heading rows and values for every packet family, separated by commas or tabs; standard CSV readers can read them after skipping the `#` lines. HDF5 files use schema 5, one table per measurement type with the CSV headings as member names; a TIHI, CORL or CORM table continues in a new part when its input pair changes. See [file saving and export](../06_in_depth_guides/6_1_file_saving_and_data_export.md#processed-data-file-saving).

**Files from earlier SDKs:** the preview.8 CORL and CORM CSV writers emitted JSON metadata without CSV quoting, so standard CSV readers can reject those rows or split them incorrectly. Read such files with a format-specific parser, or use their TAB or HDF5 counterparts.
