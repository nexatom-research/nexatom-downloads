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
*   `NexatomTimeTagReader.header()` returns a `NexatomTimeTagFileHeader` object containing the creation timestamp and internal data type verification.

**Data Extraction:**
The native reader decodes the packed disk records into aligned host records (see the [NXTT layout](6_1_file_saving_and_data_export.md#nxtt-binary-file-format)). Python offers two ways to consume them:
1.  **Batch Loading:** `reader.read_batch(max_tags)` loads a fixed number of `NexatomTimeTag` objects into memory at once.
2.  **Lazy Iteration:** `reader.iter_tags(batch_size)` provides a Python generator that yields tags sequentially. This is the strictly recommended approach for datasets exceeding available host RAM.

> **Performance note.** Creating Python objects for large event sets has overhead. Batching bounds peak memory, while CSV export makes the data usable by spreadsheet and table-analysis tools. The Python exporter itself is not a zero-copy or GIL-free operation. Choose a batch size that fits the intended host, and avoid loading an entire long capture into a list.

### [CSV export](6_2_offline_time_tag_binary_decoder.md#csv-export)

To convert decoded events into readable text, the reader exposes `export_csv(output_path, batch_size=4096)`.

Native code performs binary decoding; the Python wrapper writes the decoded batches using Python's standard CSV module. The output has two columns, `timestamp_ps,channel`, containing decimal integers. Export begins at the reader's current position, so reopen the file if a previous read has already consumed tags.

**Python:**
```python
with library.open_time_tag_file_reader("captures/data.nxtt") as reader:
    row_count = reader.export_csv("captures/data.csv", batch_size=100000)
    print(f"Successfully exported {row_count} tags.")
```

An empty successful batch is EOF. A native read failure raises an exception; do not catch it and report EOF. For rotated captures, use the exact series key and required sequence range; do not silently omit a missing required segment.

### Decoding while a device is connected

Finish raw acquisition and finalize savers before opening an offline reader. Native rejects decoding while tag output or file saving remains active. When decoding is otherwise allowed with a connected device, the reader can establish quiet output and restore the previous mode when it releases ownership. EOF releases this gate automatically, but the application should still close the reader. A mode-restoration failure remains an error.

For post-processing, subtract timestamps as integers before converting the difference to ns or seconds. Hardware timestamps are not wall-clock time and do not by themselves establish synchronization between instruments.
