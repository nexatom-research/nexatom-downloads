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
nexatom_tt_close_time_tag_reader(reader);
```

**Python:**
```python
with library.open_time_tag_file_reader("captures/data.nxtt") as reader:
    pass # process data
```

### [Series reader (multi-file sequences)](6_2_offline_time_tag_binary_decoder.md#series-reader-multi-file-sequences)

When the native file saver rotates files based on size or duration limits (see Section 6.1.1), it generates a sequentially numbered series of files. The Series Reader abstraction automatically stitches these segmented files together, presenting them to the host application as a single, contiguous time-tag stream.

**Python:**
```python
# Opens all files matching: captures/my_experiment_TAGS_000.nxtt to ..._099.nxtt
with library.open_time_tag_series_reader(
    directory="captures", 
    series_key="my_experiment", 
    seq_start=0, 
    seq_end=99
) as reader:
    pass
```

### [Reading tags and header](6_2_offline_time_tag_binary_decoder.md#reading-tags-and-header)

Once a reader handle is open, the host can inspect the file metadata and extract the time tags.

**Header Inspection:**
*   `NexatomTimeTagReader.header()` returns a `NexatomTimeTagFileHeader` object containing the creation timestamp and internal data type verification.

**Data Extraction:**
The SDK provides two paradigms for pulling the 16-byte records from disk into memory:
1.  **Batch Loading:** `reader.read_batch(max_tags)` loads a fixed number of `NexatomTimeTag` objects into memory at once.
2.  **Lazy Iteration:** `reader.iter_tags(batch_size)` provides a Python generator that yields tags sequentially. This is the strictly recommended approach for datasets exceeding available host RAM.

> **Performance Note.** Instantiating millions of Python objects per second carries significant garbage-collection overhead. For high-density statistical analysis, users should utilize the native CSV export (Section 6.2.4) and process the resulting data with optimized libraries like `pandas` or `polars`.

### [CSV export](6_2_offline_time_tag_binary_decoder.md#csv-export)

To convert raw binary data into a universally readable text format, the reader exposes a highly optimized `export_csv(output_path, batch_size)` method.

Because the CSV conversion executes entirely within the native C++ library, it circumvents the Python Global Interpreter Lock (GIL) and prevents the instantiation of intermediate Python objects. The resulting file is formatted with two zero-padded columns: `timestamp_ps,channel`.

**Python:**
```python
with library.open_time_tag_file_reader("captures/data.nxtt") as reader:
    row_count = reader.export_csv("captures/data.csv", batch_size=100000)
    print(f"Successfully exported {row_count} tags.")
```
