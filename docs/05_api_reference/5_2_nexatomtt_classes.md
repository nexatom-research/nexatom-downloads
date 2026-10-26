## NexatomTT Classes

The `nexatomtt` Python package provides a robust, object-oriented abstraction over the native C API. This section serves as the definitive API reference for the core classes you will use to initialize the SDK, control hardware, and parse data.

### [`NexatomLibrary`](5_2_nexatomtt_classes.md#nexatomlibrary)

The `NexatomLibrary` class acts as the root context for the SDK. It loads the packaged Windows DLL or Linux shared library, manages the `ctypes` bindings, and discovers connected hardware.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `__init__` | `home: str` | `NexatomLibrary` | Initializes the library context from the extracted SDK root. Keep the package's library/dependency layout intact. |
| `version` | *None* | `str` | Returns the semantic version of the native C++ library (e.g., `"1.2.0"`). |
| `library_info` | *None* | `str` | Returns a detailed build string, including the compiler version and Git commit hash used to build the native library. |
| `discover_devices` | `max_devices: int = 8` | `list[NexatomDeviceInfo]` | Scans the USB bus and returns selection records. Select one intended instrument for an ordinary application session. |
| `create_device` | `device_info: NexatomDeviceInfo` | `NexatomDevice` | Allocates an owning device handle. Call `connect_runtime()` for measurement readiness, or use `open_runtime_device()`. |
| `open_time_tag_file_reader` | `path: str` | `NexatomTimeTagReader` | Opens a single `.nxtt` binary file for offline decoding. |
| `open_time_tag_series_reader` | `directory: str`, `series_key: str`, `seq_start: int = 1`, `seq_end: int = 0` | `NexatomTimeTagReader` | Opens a sequence of rotated `.nxtt` files (e.g., `_0001` to `_0099`) as a single contiguous stream; `seq_end = 0` reads every following file. |
| `count_boards_not_using_nexatom_driver` | *None* | `int` | Attached boards that cannot be opened because they do not use the Nexatom USB driver; `discover_devices()` lists none of them. Windows only; always 0 on Linux. |
| `export_time_tags` | `output_path`, keyword `input_path` or `series_directory` + `series_key` (+ `seq_start`, `seq_end`), `format: str = "csv"`, `header: bool = True` | `NexatomTimeTagExportResult` | Writes every event of a `.nxtt` file or series as one CSV or TAB file. See [Time-tag export](../06_in_depth_guides/6_2_offline_time_tag_binary_decoder.md#time-tag-export). |
| `start_time_tags_export` | As `export_time_tags` | `NexatomTimeTagExportJob` | Runs the same export as a job with progress and cancel. |
| `open_fast_tihi_index_file_reader` / `open_fast_tihi_index_series_reader` | `path`, or `directory`, `series_key`, `seq_start = 1`, `seq_end = 0` | `NexatomFastTihiIndexReader` | Opens a Fast TIHI click file (`.nxti`) or series. See [Fast TIHI click files](../06_in_depth_guides/6_2_offline_time_tag_binary_decoder.md#fast-tihi-index-files). |
| `export_fast_tihi_index` | `output_path`, keyword input as above, `layout: str = "table"`, `format: str = "csv"` | `NexatomFastTihiIndexExportResult` | Writes the readable click table (`"table"`, `"files"` or `"columns"`). |
| `start_fast_tihi_index_export` | As `export_fast_tihi_index` | `NexatomFastTihiIndexExportJob` | Runs the same export as a job with progress and cancel. |

---

### [`NexatomDevice`](5_2_nexatomtt_classes.md#nexatomdevice)

The `NexatomDevice` represents one selected instrument. It exposes hardware configuration, lifecycle and measurement endpoints through the common native API. Available channels and operations depend on the resolved native profile.

#### Connection & Lifecycle
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `connect` | `timeout_ms: int = 5000` | `None` | Claims the USB interface and establishes communication with the FPGA. |
| `connect_runtime` | `timeout_ms: int = 5000` | `None` | Establishes native measurement readiness, booting an existing valid image if required and waiting for profile authority. Use a larger budget, such as 20000 ms, for cold startup. |
| `disconnect` | *None* | `None` | Safely terminates the active USB session after cleaning up the board (see [Disconnect](../02_hardware/2_1_operating_conditions.md#disconnect)). |
| `close` / `destroy` | *None* | `None` | Destroys the native device handle. A `with` block calls `close()` on exit; explicit `disconnect()` allows its errors to be checked before destruction. |
| `is_connected` | *None* | `bool` | Returns `True` if the USB session is active. |
| `state` | *None* | `int` | Returns the current `nexatom_tt_state_t` enum (e.g., `CONNECTED`, `ACQUIRING`, `ERROR`). |
| `hardware_protocol_mode` | *None* | `int` | Identifies if the device is currently in `RUNTIME` (data acquisition) or `BOOTLOADER` (firmware flashing) mode. |
| `get_device_info` | *None* | `NexatomDeviceInfo` | Returns descriptive device information. |
| `get_device_profile` | *None* | `NexatomDeviceProfileV1` | Returns resolved model/image identity, authority, channel masks, features and delay/pulse limits. |
| `get_capabilities` | *None* | `NexatomCapabilities` | Returns public control limits, including the active histogram-bin maximum. |
| `set_auto_reconnect` | `enable: bool` | `None` | Off by default. Reconnects in the background after a working link drops (see [Pulled cable and re-plug](../02_hardware/2_1_operating_conditions.md#pulled-cable-and-re-plug)); it never opens the first connection. |
| `auto_reconnect_state` | *None* | `int` | `NEXATOM_AUTO_RECONNECT_OFF` (0), `_ON` (1) or `_REFUSED_OTHER_INSTRUMENT` (2). |
| `set_connection_status_callback` | `callback(connected: bool, ready: bool)` or `None` | `None` | Reports connection and readiness changes, including a pulled cable and a re-plug. |

`connect()` alone is useful for a firmware-service client; it does not replace the readiness contract of `connect_runtime()`. Creating a handle, connecting, selecting output, starting an engine and enabling a file saver are separate actions.

#### System Control
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_system` | `enable: bool` | `None` | The master switch. Pass `True` to start the FPGA data shuffler. **Must be `False` before changing output modes.** |
| `set_output_type` | `output_type: int` | `None` | Switches hardware between `NEXATOM_OUTPUT_NO_OUTPUT`, `NEXATOM_OUTPUT_RAW_TAGS` and `NEXATOM_OUTPUT_REALTIME_DATA`. A change during a recording ends it (see [Output type and recordings](../02_hardware/2_3_data_connection.md)). |
| `get_output_type` | *None* | `int` | The output type last set; `NO_OUTPUT` after connect. |
| `reset_peripherals` | `reset: bool` | `None` | Issues a hard reset to the internal hardware measurement counters. |
| `request_global_stop_all_modes` | *None* | `None` | Safely commands the hardware to terminate all active TIHI, MFCO, and Correlation measurements. |

#### Channel Configuration
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `set_channel_threshold` | `channel: int`, `threshold_mv: int` | `None` | Sets the discriminator voltage trigger level for a specific input channel (0-7). |
| `set_channel_input_delay` | `channel: int`, `delay_ps: int` | `None` | Applies delay in ps to align channels. Check `profile.max_channel_input_delay_ps`; current profiles can support up to 256000 ps (256 ns). |
| `set_channel_edge_type` | `channel: int`, `edge_type: int` | `None` | Chooses the discriminator edge using the exported edge constants. |
| `set_channel_hysteresis` | `channel: int`, `hysteresis_mv: int` | `None` | Configures discriminator hysteresis in mV, subject to native validation. |
| `enable_channel_test_pulse`| `channel: int`, `enable: bool` | `None` | Routes the internal FPGA test oscillator to the specified channel to verify software logic without physical signals. |
| `get_channel_input_delay` | `channel: int`, `timeout_ms: int = 1000` | `NexatomChannelInputDelay` | UTT160810: the delay the input runs with, and whether it is confirmed by the board (see [Reading back the delay in use](../02_hardware/2_7_synthetic_input_delay.md)). |
| `set_input_enable_mask` / `get_input_enable_mask` | `mask: int` / *None* | `None` / `int` | Switches inputs on or off (bit k = input k, 1 = on). See [Switching inputs on and off](../02_hardware/2_2_input_channels.md#switching-inputs-on-and-off). |

#### Clocks
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `request_sync_clock` | `enable: bool` | `None` | Requests the external 10 MHz reference on REF IN. |
| `set_reference_auto_relock` | `enabled: bool` | `None` | Whether the board locks to REF IN again by itself after a loss (default on). |
| `get_reference_clock_status` | *None* | `NexatomReferenceClockStatus` | `OFF`, `WAITING`, `LOCKED`, `LOST` or `LOCK_FAILED`, with a fault code. |
| `enable_reference_clock_output` | `enable: bool` | `None` | 10 MHz on REF OUT. |

See [External clock](../02_hardware/2_8_external_clock.md) for the rules on each board.

#### Pulse outputs (UTT160810)
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `apply_dtc_output` | `configuration: NexatomDtcOutputConfigV1`, `timeout_ms: int = 1000` | `NexatomDtcApplyResultV1` | Applies one pulse output; raises `NexatomDtcApplyRejected` when the board refuses it. |
| `set_dtc_global_enable` | `enable: bool` | `None` | Master switch of all pulse outputs. |
| `get_dtc_status` | `timeout_ms: int = 1000` | `NexatomDtcStatusV1` | Calibration and output status. |
| `get_dtc_output_limits` | `output: int` | `NexatomDtcOutputLimits` | Smallest width and largest delay + width an output accepts. |
| `get_dtc_reapply_status` | *None* | `NexatomDtcReapplyStatus` | What the SDK remembers and applies again by itself. |
| `restart_dtc_calibration` | *None* | `None` | Recalibrates the pulse-output delay lines; apply the outputs again afterwards. |
| `set_dtc_pulse_generator_period` / `get_dtc_pulse_generator_period` | `output: int`, `period_ps: int` / `output: int` | `None` / `int` | Pulse generator period, 8000 ps to 2^40 - 1 ps. |
| `set_dtc_pulse_generator_enable_mask` / `get_dtc_pulse_generator_enable_mask` | `mask: int` / *None* | `None` / `int` | Which pulse generators run. |

See [Pulse outputs](../02_hardware/2_13_dtc_pulse_outputs.md) for the fixed output latency, the limits and the rule for pulses that come too close.

#### Measurement Modules
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_time_histogram` | `enable: bool` | `None` | Powers on the TIHI hardware processor. |
| `set_time_histogram_channels`| `start_ch: int`, `stop_ch: int` | `None` | Assigns the start and stop trigger channels for the TCSPC histogram. |
| `set_time_histogram_bin_width`| `bin_width_ps: int` | `None` | Sets the temporal resolution of a single histogram bin. |
| `set_time_histogram_num_bins`| `num_bins: int` | `None` | Sets active bins, bounded by `get_capabilities().max_histogram_bins`. |
| `start_time_histogram` | *None* | `None` | Arms the TIHI engine. |
| `enable_multifold_coincidence`| `enable: bool` | `None` | Powers on the MFCO hardware processor. |
| `set_multifold_coincidence_channels` | `channels: iterable[int]` | `None` | Supplies channel IDs for up to eight MFCO slots. `0xff` disables a slot; omitted slots are padded with `0xff`. This is not a channel bitmask. |
| `set_multifold_coincidence_window` | `window_ps: int` | `None` | Sets the temporal window (in picoseconds) for grouping simultaneous events. |
| `get_mfco_counting_rule` | *None* | `int` | How the connected UTT810 counts MFCO patterns: `NEXATOM_MFCO_COUNTING_RULE_PER_HIT` (firmware below `0x0103`), `_PER_EVENT` (from `0x0103`) or `_UNKNOWN` (before telemetry, on the UTT160810, or an unlisted version). Sends nothing to the board. |

#### Callbacks & File Saving
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `set_time_histogram_callback`| `func: Callable` | `None` | Registers a Python function to receive `NexatomTihiData` structs asynchronously. |
| `set_multifold_coincidence_callback` | `func: Callable` | `None` | Registers a Python function to receive `NexatomMfcoData` structs asynchronously. |
| `clear_callbacks` | *None* | `None` | Clears registrations. It does not fence an invocation already selected by native; the wrapper retains retired ctypes handlers through native destruction. |
| `set_time_tag_file_config` | `config: NexatomTimeTagFileConfig` | `None` | Sets rotation limits (size, duration, event count) for the native binary saver. |
| `enable_time_tag_file_saving`| `base_filename: str`, `format: int`| `None` | Starts routing `RAW_TAGS` directly to disk, bypassing Python. |
| `disable_time_tag_file_saving` | *None* | `None` | Finalizes the time-tag saver. Call it while the `RAW_TAGS` output is still on: the recording then ends as a user stop. |
| `set_processed_file_config` | `packet_type: int`, `config: NexatomProcessedFileConfig` | `None` | Sets per-packet processed file options. |
| `enable_processed_file_saving` | `packet_type: int`, `base_filename: str`, `format: int` | `None` | Enables a supported processed packet sink (CSV, TAB or HDF5); does not start acquisition. |
| `disable_processed_file_saving` | `packet_type: int` | `None` | Finalizes the selected processed sink. |
| `set_save_end` | `kind: int`, `value: int = 0` | `None` | Ends each recording by itself after a time or a count; set it before enabling saving. |
| `get_file_saving_statistics` | *None* | `NexatomFileSavingStatistics` | The file stage's counters and the progress toward the save end; `complete` says whether every received tag was written. |
| `get_saving_result` | *None* | `NexatomSavingResult` | Why the current or last recording ended, and whether its span is whole. |
| `get_last_incident` | *None* | `NexatomIncidentInfo` | The last incident raised for the handle. |
| `get_board_loss` | *None* | `NexatomBoardLoss` | Where and why the board dropped tags, when that ended the recording. |
| `get_data_flow_status` | *None* | `NexatomDataFlowStatus` | How the host keeps up with the saving recording. |
| `check_start_rate` | `expected_input_rate_hz: int`, `planned_file_saving_mask: int = 0` | `NexatomStartCheck` | Before start: whether the expected rate can pass the USB link. |
| `get_board_time_upper_bound` | *None* | `int` | An upper bound of the board time now, in ps on the raw-save time base. |
| `get_measurement_abort` | *None* | `NexatomMeasurementAbort` | UTT160810: a measurement the board aborted by itself (see [Measurement abort](../02_hardware/2_11_telemetry.md#measurement-abort)). |
| `get_calibration_hold_state` | *None* | `NexatomCalibrationHoldState` | UTT810: whether automatic calibration is held during a file save. |

The [file-saving guide](../06_in_depth_guides/6_1_file_saving_and_data_export.md) explains naming, rotation, format limitations and [how a recording ends](../06_in_depth_guides/6_1_file_saving_and_data_export.md#ending-a-recording). The [callback reference](5_4_callback_types.md) includes correlation, versioned telemetry/configuration and Fast TIHI callbacks.

---

### [`NexatomTimeTagReader`](5_2_nexatomtt_classes.md#nexatomtimetagreader)

This class provides memory-safe iteration over offline `.nxtt` binary files. It implements the Python Context Manager protocol (`with ... as reader:`), ensuring file handles are closed properly.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `header` | *None* | `NexatomTimeTagFileHeader` | Returns the metadata block containing the creation timestamp. |
| `read_batch` | `max_tags: int = 4096` | `list[NexatomTimeTag]` | Reads a bounded batch. An empty batch means EOF; an exception is a read error. |
| `iter_tags` | `batch_size: int = 4096`| `Generator` | Yields decoded tags sequentially without loading the whole file into RAM. |
| `export_csv` | `output_path: str`, `format: str = "csv"`, `header: bool = True` | `int` | Writes the whole file or series as CSV or TAB through the library's time-tag export, however far the reader has been read. Returns the rows written. |

---

### `NexatomFastTihiIndexReader`

Reads a Fast TIHI click file (`.nxti`) or series; a context manager like `NexatomTimeTagReader`.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `header` / `status` / `file_facts` | *None* | records | The file header, the reader status and the facts of the file |
| `input_switches` / `input_settings` | *None* | `list` | Input on/off changes and per-input settings recorded in a format 2 file |
| `read_batch` | `max_clicks` | `list` | Reads a bounded batch of click records |
| `iter_clicks` | `batch_size` | `Generator` | Yields click records sequentially |
| `export_csv` | `output_path`, `layout: str = "table"`, `format: str = "csv"` | `int` | Writes the whole click table; returns the clicks written |

### Export jobs (`NexatomTimeTagExportJob`, `NexatomFastTihiIndexExportJob`)

Returned by `start_time_tags_export()` and `start_fast_tihi_index_export()`; context managers that cancel and finish the job on the way out.

| Method | Description |
| :--- | :--- |
| `progress()` | Phase, files, bytes and tags (or clicks) done so far |
| `cancel()` | Asks the job to stop; returns at once, from any thread |
| `finished` | `True` once the job has ended |
| `finish(timeout_ms=None)` | Waits for the job and returns its result; with a timeout, returns `None` while it still runs. Raises `NexatomError` (`NEXATOM_ERROR_CANCELLED` after a cancel) |

---

### [`NexatomError`](5_2_nexatomtt_classes.md#nexatomerror)

The SDK uses exceptions to handle hardware and configuration failures. If a C API function returns a negative error code, the Python wrapper intercepts it and raises a `NexatomError`.

| Property | Type | Description |
| :--- | :--- | :--- |
| `code` | `int` | The native `nexatom_error_code_t` enum value (e.g., `-5` for Timeout). |
| `native_message` | `str` | The highly detailed, internal C++ string explaining exactly why the failure occurred. |

`NexatomDtcApplyRejected` is a `NexatomError` raised by `apply_dtc_output()` when the board completed the request and refused it; its `result` attribute holds the board's `NexatomDtcApplyResultV1`.

---

<a id="runtime-boot-classes"></a>

### [Runtime Boot Classes (`nexatomtt.runtime_boot`)](5_2_nexatomtt_classes.md#runtime-boot-classes)

These classes support the `open_runtime_device(library, device_info, options=...)` context manager. Normal startup delegates to native `connect_runtime()` and retains one device handle throughout boot and readiness; an already-running or supported legacy runtime uses the same entry point.

#### `RuntimeBootOptions`
| Property | Type | Description |
| :--- | :--- | :--- |
| `timeout_ms` | `int` | Budget passed to native runtime startup; default 5000 ms, commonly increased to 20000 ms for cold boot. |
| `mode_timeout_sec` | `float` | Compatibility option for protocol detection in the explicit preferred-slot workflow. |
| `poll_sec` | `float` | Poll interval for that explicit protocol detection; normal native startup does not use a Python discovery loop. |
| `preferred_slot` | `int` \| `None` | Optional service-mode boot target. Uses the reported slot inventory; does not write an image or change the persistent default. |

#### `DeviceIdentity`
Retained identity utility for describing a discovery record. Normal runtime startup does not require clients to close, re-enumerate and recreate a device using it.

| Property | Type | Description |
| :--- | :--- | :--- |
| `connection_id` | `str` | The board's identity: `usb:` plus the USB port path (Windows: PnP location path; Linux: sysfs device name). It selects the board and changes if the board moves to another USB port. |
| `serial_number` | `str` | FT601 USB serial, for service information only. Boards can share it, so it never selects a board. Model and image identity are separate native profile fields. |

```python
from nexatomtt import NexatomLibrary, RuntimeBootOptions, open_runtime_device

library = NexatomLibrary(home="/path/to/extracted-sdk")
devices = library.discover_devices()
if len(devices) != 1:
    raise RuntimeError("Select one intended instrument before starting")
with open_runtime_device(library, devices[0],
                         options=RuntimeBootOptions(timeout_ms=20000)) as device:
    # Native startup is complete; inspect authority before choosing controls.
    profile = device.get_device_profile()
    print(f"Model: 0x{profile.product_model_id:08x}")
    # Configure and acquire using a complete tutorial's start/stop sequence.
    device.disconnect()  # Check shutdown errors explicitly before destruction.
```
