## Raw Time-Tag Capture and Offline CSV Export

Raw output records individual timestamped events rather than histograms or rates. This tutorial configures the native file writer to save binary `.nxtt` files, finalizes recording, and reads the events back for CSV export and interval analysis. Achievable throughput depends on the instrument, host and storage; this example is not a peak-bandwidth benchmark.

**Relevant script:**
*   `file_save_and_offline_decode.py`

### Workflow

1.  **Boot into Runtime:** The script leverages the `open_runtime_device()` orchestrator to acquire a valid `NexatomDevice` handle in runtime mode.
2.  **Prepare Inputs:** The script reads the native profile, validates threshold/edge/delay/hysteresis settings and applies them while output and the system are quiet. `--internal-test` selects generated pulses; omit it for external signals.
3.  **Configure File Saving:** A `NexatomTimeTagFileConfig` struct is populated to define file rotation thresholds (`max_file_size_mb`, `max_duration_minutes`, `max_event_count`) and timestamp inclusion. This is applied via `device.set_time_tag_file_config()`.
4.  **Start File Writer:** Call `device.enable_time_tag_file_saving(base_prefix, NEXATOM_TT_FILE_BINARY)` before enabling producers. A fresh `raw-<id>` directory prevents older captures from being mistaken for this run's data.
5.  **Hardware Output Mode:** Select `RAW_TAGS` with `device.set_output_type(NEXATOM_OUTPUT_RAW_TAGS)`. This workflow records events; it does not depend on receiving CPS/TIHI/MFCO processed callbacks during raw output.
6.  **Enable System:** The FPGA data shuffler is enabled (`device.enable_system(True)`) to commence the raw data flow over the USB interface.
7.  **Acquisition & Safe Teardown:** The main thread sleeps for the duration of the capture. To prevent file corruption or trailing incomplete packets, the script executes a strict shutdown sequence and attempts every step even after an earlier one failed:
    *   Disable any test pulses enabled by this run.
    *   Finalize saving with `device.disable_time_tag_file_saving()` while the output is still `RAW_TAGS`. This is the user's stop, and the file records the end as "stopped by user". Switching to `NO_OUTPUT` first is also a normal end, but the saving then ends at the switch and records "the measurement type changed".
    *   Read `device.get_file_saving_statistics()`. Its `complete` flag says whether every raw tag the file stage received was written, with nothing dropped and no write failure; the script fails the run when it is false.
    *   Quiet output (`device.set_output_type(NEXATOM_OUTPUT_NO_OUTPUT)`) and disable the system.
    *   Check `device.disconnect()`; report cleanup errors even when capture produced data.
8.  **Offline Decoding:** Once files are finalized, the script opens every new `.nxtt` file with `NexatomTimeTagReader`. It counts events and computes within-channel intervals; `--decode-csv` additionally writes a CSV for each file with `reader.export_csv()` and checks that its exported row count matches the independent reader count. The export always writes the whole file, however far the reader has been read.

### Execution

To run the capture and automatically trigger the offline CSV export, use the `--decode-csv` flag:

```powershell
python python/examples/file_save_and_offline_decode.py --home . --channels 0,1 --threshold-mv 500 --edge rising --delay-ps 0 --hysteresis-mv 10 --internal-test --duration-sec 5 --decode-csv
```

The final console line has this form; the actual count and number of rotations depend on the signal and duration:

```text
Read <event-count> time tags from <file-count> new .nxtt files.
```

### Read individual events

The exported CSV starts with three lines that begin with `#`: what the file is, with the recording time and the source file; what each column means; and the tag count with the inputs that recorded them and how the saving ended (for example `# 3 tags on Inputs 2, 4, confirmed; ended: stopped by user`). Then come the heading row `Event time (ps),Input` and one row per event in recorded order. `Event time (ps)` is an integer instrument clock value in picoseconds, not Unix time; read it as a 64-bit integer, never as a float. `Input` counts from 1, as the application does: Input N is channel N-1. The reader, the `.nxtt` file and the C API keep the 0-based channel. The following fragment reads a finalized file without connecting hardware:

```python
from nexatomtt import NexatomLibrary

library = NexatomLibrary(home=".")
previous = {}
with library.open_time_tag_file_reader("capture.nxtt") as reader:
    for tag in reader.iter_tags():
        channel = int(tag.channel)
        timestamp = int(tag.timestamp_ps)
        if channel in previous and timestamp >= previous[channel]:
            interval_ps = timestamp - previous[channel]  # Integer subtraction first.
            print(channel, timestamp, interval_ps / 1_000_000, "us")
        previous[channel] = timestamp
```

Use an actual finalized filename in place of `capture.nxtt`. The complete example's `raw_summary.json` stores event counts and interval statistics per channel and per file. It excludes intervals across file boundaries and reports backwards timestamps. Approximately 100 kHz internal pulses give roughly 10 us within-channel separation; cross-channel phase is a separate measurement.

### Files to inspect

| Output | Contents |
|---|---|
| `.nxtt` | Native binary event recording, including file metadata |
| `.csv` with `--decode-csv` | Three `#` header lines, the heading row `Event time (ps),Input`, then one row per event |
| `raw_summary.json` | Requested settings, counts, interval statistics, the file-saving statistics and failure/cleanup details |

### Exporting a recording later

`python/examples/export_time_tags.py` turns a saved recording into a CSV or TAB file without a board:

```powershell
python python/examples/export_time_tags.py --home . captures/raw-1/capture_TAGS_0001.nxtt
python python/examples/export_time_tags.py --home . captures/raw-1 --series-key capture_TAGS --format tab
```

The first form exports one file. The second exports a whole series (the rotated files `capture_TAGS_0001.nxtt`, `_0002`, ... of one recording) as one file; `--seq-start` and `--seq-end` pick part of it. `--no-header` writes the rows only, for tools that read plain columns, and `--output` names the output file (its folder must exist). By default the output is written beside the input; an existing file is replaced, and a failed export leaves no file. The library call behind it is `NexatomLibrary.export_time_tags()` (C: `nexatom_tt_export_time_tags()`); see [Time-tag export](../06_in_depth_guides/6_2_offline_time_tag_binary_decoder.md#time-tag-export).

The native live time-tag saver also supports CSV and TAB; binary followed by offline export is the workflow shown here. Processed histograms use the separate processed-file API. See [file saving](../06_in_depth_guides/6_1_file_saving_and_data_export.md) for the format choices.
