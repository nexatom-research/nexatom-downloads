## Offline Time-Tag Binary Decoder

While the `.nxtt` binary format is optimized for raw disk-write throughput, analyzing the captured data requires a performant decoder. The SDK provides a native C parser that safely handles byte-alignment and file boundaries. The Python wrapper exposes this parser via the `NexatomTimeTagReader` class, allowing users to iterate over billions of events without exhausting system memory.

### [Single-file reader](6_2_offline_time_tag_binary_decoder.md#single-file-reader)

To decode an isolated `.nxtt` file, instantiate a single-file reader. In the C API, this yields an opaque `nexatom_tt_time_tag_reader_t*` handle which must be manually closed. The Python wrapper provides a context manager to guarantee resource cleanup.

**C API:**
```c
nexatom_tt_time_tag_reader_t* reader = NULL;
nexatom_error_code_t err = nexatom_tt_open_time_tag_file_reader(
    "captures/data.nxtt", &reader
);
// ... processing ...
if (err == NEXATOM_SUCCESS) {
    /* Read bounded batches; success with zero returned tags means EOF. */
    nexatom_tt_close_time_tag_reader(reader);
} else {
    /* Report the error; a failed open is not an empty capture. */
}
```

**Python:**
```python
with library.open_time_tag_file_reader("captures/data.nxtt") as reader:
    for tag in reader.iter_tags():
        # Process one decoded event; keep timestamp arithmetic in integer ps.
        timestamp_ps = int(tag.timestamp_ps)
        channel = int(tag.channel)
```

### [Series reader (multi-file sequences)](6_2_offline_time_tag_binary_decoder.md#series-reader-multi-file-sequences)

When the native file saver rotates files based on size or duration limits (see Section 6.1.1), it generates a sequentially numbered series of files. The Series Reader abstraction automatically stitches these segmented files together, presenting them to the host application as a single, contiguous time-tag stream.

**Python:**
```python
# Example for files named my_experiment_TAGS_0001.nxtt onward.
# Use the actual date directory and base[_timestamp]_TAGS[_suffix] series key.
with library.open_time_tag_series_reader(
    directory="captures",
    series_key="my_experiment_TAGS",
    seq_start=1,
    seq_end=0  # All remaining contiguous files in this series.
) as reader:
    for tag in reader.iter_tags():
        # Consume this event before advancing; no whole-capture list is needed.
        channel = int(tag.channel)
```

### [Reading tags and header](6_2_offline_time_tag_binary_decoder.md#reading-tags-and-header)

Once a reader handle is open, the host can inspect the file metadata and extract the time tags.

**Header Inspection:**
*   `NexatomTimeTagReader.header()` returns a `NexatomTimeTagFileHeader` object containing the creation timestamp, the output data type (`2` raw tags, `1` encoded tags) and `format_version` (1 or 2; the SDK writes format 2, whose closed files end with an end record, see the [NXTT layout](6_1_file_saving_and_data_export.md#nxtt-binary-file-format)).

**Format 2 files:** the reader returns the tags of a closed file and stops at its end record. A closed file whose blocks do not run exactly to the end record is damaged: the read returns `NEXATOM_ERROR_FILE_IO` (Python raises `NexatomError`) with the file name and byte position. A file that was not closed (a crash, a power loss, or a file still being written) reads every whole tag up to its torn end. Raw (`2`) and encoded (`1`) captures store the same 9-byte record, so a reader treats them alike.

**Data Extraction:**
The native reader decodes the packed disk records into aligned host records (see the [NXTT layout](6_1_file_saving_and_data_export.md#nxtt-binary-file-format)). Python offers two ways to consume them:
1.  **Batch Loading:** `reader.read_batch(max_tags)` loads a fixed number of `NexatomTimeTag` objects into memory at once.
2.  **Lazy Iteration:** `reader.iter_tags(batch_size)` provides a Python generator that yields tags sequentially. This is the strictly recommended approach for datasets exceeding available host RAM.

> **Performance note.** Creating Python objects for large event sets has overhead. Batching bounds peak memory, while CSV export makes the data usable by spreadsheet and table-analysis tools. The Python exporter itself is not a zero-copy or GIL-free operation. Choose a batch size that fits the intended host, and avoid loading an entire long capture into a list.

### [CSV export](6_2_offline_time_tag_binary_decoder.md#csv-export)

To convert decoded events into readable text, the reader exposes `export_csv(output_path, format="csv", header=True)` and returns the number of rows written.

The library writes the file (it calls the time-tag export below for the reader's file or series): the same header and heading row as the export, then one row per event in recorded order. `format` is `"csv"` or `"tab"`; `header=False` writes the rows only. The export reads the input from its path, not through the reader: it always covers the whole file or series, however far the reader has been read, and leaves the reader where it is.

**Python:**
```python
with library.open_time_tag_file_reader("captures/data.nxtt") as reader:
    row_count = reader.export_csv("captures/data.csv")
    print(f"Successfully exported {row_count} tags.")
```

<a id="time-tag-export"></a>

### Time-tag export

`NexatomLibrary.export_time_tags()` (C: `nexatom_tt_export_time_tags()`) turns a `.nxtt` file or a whole series into one CSV or TAB file. It reads files only and never touches a board, so it also runs during a recording, during raw output and with several boards connected.

```python
result = library.export_time_tags(
    "captures/run.csv",
    series_directory="captures/raw-1",
    series_key="capture_TAGS",   # or input_path="captures/raw-1/capture_TAGS_0001.nxtt"
    format="csv",                # or "tab"
    header=True,                 # False: rows only
)
print(result.files_read, result.tags_written)
```

- **Input.** Give `input_path` for one file, or `series_directory` and `series_key` (with `seq_start`, `seq_end`; 0 = every following file) for a series. A series becomes one file under one header.
- **Output.** Three `#` lines: what the file is, with its creation time and source files (`# Time tags · <created> · from <files>`); what each column means; and the check, which gives the tag count with the inputs and how the saving ended. Then the heading row `Event time (ps),Input` and one row per event in recorded order. Times are exact whole picoseconds; read them as 64-bit integers, never as floats. `Input` counts from 1 (Input N is channel N-1). In C, `NEXATOM_TT_TIME_TAG_EXPORT_NO_HEADER` writes the rows only.
- **The check.** For a format 2 file the export reads the end records and says, for example, `confirmed; ended: stopped by user`, `saving stopped: <reason>`, `saving not confirmed: the file was not closed` or `saving not confirmed: the end record does not match the data`. Part of a series (a `seq_start` past the first file, or a `seq_end` before the last) never reads confirmed. A format 1 file, written by an earlier SDK, carries no saving record: its check says `saving not confirmed: no saving record`; whether that saving lost nothing is told by the capture's file-saving statistics and saving result.
- **Result.** `files_read`, `tags_written`, and from the first file's header `output_data_type` and `created_timestamp_us`.
- **Files.** The directory of the output must exist. An existing output file is replaced; a failed export leaves no output file. An input that this process's live time-tag saving is still writing returns `NEXATOM_ERROR_FILE_IN_USE` (for a series, when any of its files is still open); export it after the saving has closed it.

The example `python/examples/export_time_tags.py` wraps this call for the command line (see [the raw-capture tutorial](../03_tutorials/3_4_raw_time_tag_capture_and_offline_csv_export.md)).

#### Export with progress and cancel

For a progress bar and a Cancel button, run the same export as a job: `NexatomLibrary.start_time_tags_export()` (C: `nexatom_tt_start_time_tags_export()`) takes the same arguments and writes the same file, byte for byte, on an SDK thread. The options and the input are checked before it returns; an error there starts no job.

- The job writes into `<output_path>.partial`, created new, and renames it to `output_path` only when it is complete. It never replaces or removes a file it did not create: an existing `output_path` or `<output_path>.partial` is refused with `NEXATOM_ERROR_INVALID_PARAMETER`, and a failed or cancelled job leaves no file under `output_path`.
- `job.progress()` (C: `nexatom_tt_get_time_tags_export_progress()`) gives the phase (`CHECKING`, `WRITING`, `FINISHING`, then `DONE`, `CANCELLED` or `FAILED`), `file_index` / `file_count`, `bytes_done` / `bytes_total` and `tags_written` / `tags_total` (`tags_total` is 0 until the first pass ends). It does no file work and never waits.
- `job.cancel()` (C: `nexatom_tt_cancel_time_tags_export()`) returns at once and may be called from any thread.
- Every job must be finished: `job.finish(timeout_ms)` (C: `nexatom_tt_finish_time_tags_export()`) waits for it; with a timeout it returns `None` (C: `NEXATOM_ERROR_TIMEOUT`) while the job still runs. Once the job has ended it returns the result, which adds `check_text` (the check line) and `flags` (the `NEXATOM_TT_FAST_TIHI_INDEX_EXPORT_*` bits: `INTEGRITY_FOUND` when every input file has its end record, `COMPLETE` and `SAVING_CONFIRMED` when the check says `confirmed`; 0 for a format 1 file), or raises `NexatomError` (`NEXATOM_ERROR_CANCELLED` after a cancel). Use the job as a context manager to cancel and finish it on the way out.
- Several jobs may run at once, each with its own output. Unloading the library cancels running jobs and waits for their threads.

<a id="fast-tihi-index-files"></a>

### Fast TIHI click files (`.nxti`)

On the UTT160810, a Fast TIHI run can save every detector click with the trigger pulse it followed (trigger index) and its time after that pulse, as Fast TIHI click records. The SDK saves them in binary as `.nxti` files (format 2, self-describing: a settings record after the file header, a CRC-32 on every record and an end record), or as CSV, TAB or HDF5.

- `NexatomLibrary.open_fast_tihi_index_file_reader(path)` and `open_fast_tihi_index_series_reader(directory, series_key, seq_start=1, seq_end=0)` return a `NexatomFastTihiIndexReader` (C: `nexatom_tt_open_fast_tihi_index_file_reader()` / `..._series_reader()`, closed with `nexatom_tt_close_fast_tihi_index_reader()`). It offers `header()`, `status()`, `file_facts()`, `input_switches()`, `input_settings()`, `read_batch(max_clicks)` and `iter_clicks(batch_size)`. The reader's click records count from 0.
- `NexatomLibrary.export_fast_tihi_index(output_path, input_path=... | series_directory=..., series_key=..., layout="table", format="csv")` (C: `nexatom_tt_export_fast_tihi_index()`) writes the readable click table. `layout="table"` writes one file with the detectors merged by trigger index inside each window; `"files"` writes one file per detector (suffix `_detector<N>`, N = the application's Input N); `"columns"` writes one file with a column pair per detector. Every row names its Measurement (one Start to its Stop) and its Window (the 1 s window in that measurement), counted from 1. Each output file is written as `<name>.partial` and renamed into place only when the whole export is complete, so a failed export leaves an existing file as it was. The result gives the clicks and files written, the completeness flags and the check line. `reader.export_csv(output_path, layout="table", format="csv")` does the same for a reader's file or series.
- `start_fast_tihi_index_export()` (C: `nexatom_tt_start_fast_tihi_index_export()`) runs the same export as a job with progress, cancel and finish, as for time tags above. A click-table job call refuses a time-tag job, and the other way round.

The example `python/examples/fast_tihi_click_table.py` records a Fast TIHI run to `.nxti` and exports the table in all three layouts.

An empty successful batch is EOF. A native read failure raises an exception; do not catch it and report EOF. For rotated captures, use the exact series key and required sequence range; do not silently omit a missing required segment.

### Decoding while a device is connected

Finish raw acquisition and finalize savers before opening an offline reader. Native rejects decoding while tag output or file saving remains active. When decoding is otherwise allowed with a connected device, the reader can establish quiet output and restore the previous mode when it releases ownership. EOF releases this gate automatically, but the application should still close the reader. A mode-restoration failure remains an error.

The exports above (`export_time_tags()`, `export_fast_tihi_index()` and their jobs) do not switch the output and are not refused by a running capture; only a file that the live saving is still writing is refused.

For post-processing, subtract timestamps as integers before converting the difference to ns or seconds. Hardware timestamps are not wall-clock time and do not by themselves establish synchronization between instruments.
