## Raw Time-Tag Capture and Offline CSV Export

The UTT810 is capable of streaming unaggregated, raw time tags directly to disk at maximum USB 3.0 bandwidth. This tutorial demonstrates how to configure the native file writer to save binary `.nxtt` files, the strict teardown sequence required to prevent file corruption, and how to decode the binary data to CSV offline.

**Relevant script:**
*   `file_save_and_offline_decode.py`

### Workflow

1.  **Boot into Runtime:** The script leverages the `open_runtime_device()` orchestrator to acquire a valid `NexatomDevice` handle in runtime mode.
2.  **Suspend Processing:** To prevent partial packet delivery or state corruption, the primary FPGA data shuffler must be explicitly disabled (`device.enable_system(False)`) before altering output modes.
3.  **Configure File Saving:** A `NexatomTimeTagFileConfig` struct is populated to define file rotation thresholds (`max_file_size_mb`, `max_duration_minutes`, `max_event_count`) and timestamp inclusion. This is applied via `device.set_time_tag_file_config()`.
4.  **Hardware Output Mode:** The hardware output mode is explicitly switched to `RAW_TAGS` (`device.set_output_type(NEXATOM_OUTPUT_RAW_TAGS)`).
    > **Callback suspension.** In `RAW_TAGS` mode, realtime callback dispatch (CPS, TIHI, MFCO) is suspended to dedicate maximum bus bandwidth to the raw binary stream.
5.  **Start File Writer:** The native file saving thread is engaged by calling `device.enable_time_tag_file_saving(base_prefix, NEXATOM_TT_FILE_BINARY)`.
6.  **Enable System:** The FPGA data shuffler is enabled (`device.enable_system(True)`) to commence the raw data flow over the USB interface.
7.  **Acquisition & Safe Teardown:** The main thread sleeps for the duration of the capture. To prevent file corruption or trailing incomplete packets, the script executes a strict shutdown sequence:
    *   Command the hardware to halt the data stream (`device.set_output_type(NEXATOM_OUTPUT_NO_OUTPUT)`).
    *   Instruct the SDK to flush buffers and close the file handle (`device.disable_time_tag_file_saving()`).
    *   Disable the system (`device.enable_system(False)`).
8.  **Offline Decoding:** Once hardware operations conclude, the script leverages the `NexatomTimeTagReader` class to safely parse the binary file. Calling `reader.export_csv(csv_path)` writes a human-readable CSV representation of the exact time tags to disk.

### Execution

To run the capture and automatically trigger the offline CSV export, use the `--decode-csv` flag:

```powershell
python python\examples\file_save_and_offline_decode.py --duration-sec 5.0 --decode-csv
```

**Expected Output:**

```text
Output directory: captures; base prefix: captures\nexatomtt_capture; duration: 5s; decode CSV: yes.
Discovering NexatomTT devices.
Selected device: name=UTT810, serial=NTT-00000001, connection=FTDI:1.
Runtime firmware ready; suspending system for configuration.
Capture config: binary .nxtt, max_file_size_mb=256, max_duration_minutes=1...
Starting RAW_TAGS output.
Enabling binary .nxtt file saving.
Binary .nxtt file saving enabled with prefix captures\nexatomtt_capture.
Enabling system to start data flow.
Capturing for 5.0 seconds.
Capture duration complete.
Cleanup: set output NO_OUTPUT.
Cleanup: disable time-tag file saving.
Cleanup: disable system.
Searching for new or updated .nxtt files.
Newest NXTT file: captures\nexatomtt_capture_000.nxtt (8388608 bytes)
Starting offline decode from captures\nexatomtt_capture_000.nxtt to captures\nexatomtt_capture_000.csv.
offline decode exported 1048576 tags to captures\nexatomtt_capture_000.csv
Capture workflow complete.
```
