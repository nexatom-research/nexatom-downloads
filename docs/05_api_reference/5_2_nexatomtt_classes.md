## NexatomTT Classes

The `nexatomtt` Python package provides a robust, object-oriented abstraction over the native C API. This section serves as the definitive API reference for the core classes you will use to initialize the SDK, control hardware, and parse data.

### [`NexatomLibrary`](5_2_nexatomtt_classes.md#nexatomlibrary)

The `NexatomLibrary` class acts as the root context for the SDK. It is responsible for loading the underlying `nexatomTT.dll` into memory, managing the `ctypes` bindings, and discovering connected hardware.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `__init__` | `home: str` | `NexatomLibrary` | Initializes the library context. The `home` parameter must point to the directory containing `nexatomTT.dll` and its dependencies. |
| `version` | *None* | `str` | Returns the semantic version of the native C++ library (e.g., `"1.2.0"`). |
| `library_info` | *None* | `str` | Returns a detailed build string, including the compiler version and Git commit hash used to build the native library. |
| `discover_devices` | `max_devices: int = 16` | `list[NexatomDeviceInfo]` | Scans the USB bus and returns a list of data structures representing all connected NexatomTT instruments. |
| `create_device` | `device_info: NexatomDeviceInfo` | `NexatomDevice` | Allocates a new device session handle based on the provided discovery info. *(Note: You must call `connect()` on the returned device object to open the USB session).* |
| `open_time_tag_file_reader` | `path: str` | `NexatomTimeTagReader` | Opens a single `.nxtt` binary file for offline decoding. |
| `open_time_tag_series_reader` | `directory: str`, `series_key: str`, `seq_start: int`, `seq_end: int` | `NexatomTimeTagReader` | Opens a sequence of rotated `.nxtt` files (e.g., `_001` to `_099`) as a single contiguous stream. |

---

### [`NexatomDevice`](5_2_nexatomtt_classes.md#nexatomdevice)

The `NexatomDevice` is the primary god-object representing a physical UTT810 instrument. It exposes all hardware configuration, lifecycle, and measurement endpoints.

#### Connection & Lifecycle
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `connect` | `timeout_ms: int = 5000` | `None` | Claims the USB interface and establishes communication with the FPGA. |
| `disconnect` | *None* | `None` | Safely terminates the active USB session. |
| `close` / `destroy` | *None* | `None` | Frees the native memory allocated for this device handle. **Must be called before script exit.** |
| `is_connected` | *None* | `bool` | Returns `True` if the USB session is active. |
| `state` | *None* | `int` | Returns the current `nexatom_tt_state_t` enum (e.g., `CONNECTED`, `ACQUIRING`, `ERROR`). |
| `hardware_protocol_mode` | *None* | `int` | Identifies if the device is currently in `RUNTIME` (data acquisition) or `BOOTLOADER` (firmware flashing) mode. |

#### System Control
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_system` | `enable: bool` | `None` | The master switch. Pass `True` to start the FPGA data shuffler. **Must be `False` before changing output modes.** |
| `set_output_type` | `output_type: int` | `None` | Switches hardware between `NEXATOM_OUTPUT_RAW_TAGS` and `NEXATOM_OUTPUT_REALTIME_DATA`. |
| `reset_peripherals` | `reset: bool` | `None` | Issues a hard reset to the internal hardware measurement counters. |
| `request_global_stop_all_modes` | *None* | `None` | Safely commands the hardware to terminate all active TIHI, MFCO, and Correlation measurements. |

#### Channel Configuration
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `set_channel_threshold` | `channel: int`, `threshold_mv: int` | `None` | Sets the discriminator voltage trigger level for a specific input channel (0-7). |
| `set_channel_input_delay` | `channel: int`, `delay_ps: int` | `None` | Applies a synthetic delay (0-4000 ps) to align differing external cable lengths. |
| `enable_channel_test_pulse`| `channel: int`, `enable: bool` | `None` | Routes the internal FPGA test oscillator to the specified channel to verify software logic without physical signals. |

#### Measurement Modules
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_time_histogram` | `enable: bool` | `None` | Powers on the TIHI hardware processor. |
| `set_time_histogram_channels`| `start_ch: int`, `stop_ch: int` | `None` | Assigns the start and stop trigger channels for the TCSPC histogram. |
| `set_time_histogram_bin_width`| `bin_width_ps: int` | `None` | Sets the temporal resolution of a single histogram bin. |
| `set_time_histogram_num_bins`| `num_bins: int` | `None` | Sets total bins. (Max: 1023). |
| `start_time_histogram` | *None* | `None` | Arms the TIHI engine. |
| `enable_multifold_coincidence`| `enable: bool` | `None` | Powers on the MFCO hardware processor. |
| `set_multifold_coincidence_channels` | `channel_mask: int` | `None` | Applies a bitmask to select which channels participate in the coincidence window. |
| `set_multifold_coincidence_window` | `window_ps: int` | `None` | Sets the temporal window (in picoseconds) for grouping simultaneous events. |

#### Callbacks & File Saving
| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `set_time_histogram_callback`| `func: Callable` | `None` | Registers a Python function to receive `NexatomTihiData` structs asynchronously. |
| `set_multifold_coincidence_callback` | `func: Callable` | `None` | Registers a Python function to receive `NexatomMfcoData` structs asynchronously. |
| `clear_callbacks` | *None* | `None` | Synchronously blocks and removes all registered callbacks. **Prevents GC segmentation faults during teardown.** |
| `set_time_tag_file_config` | `config: NexatomTimeTagFileConfig` | `None` | Sets rotation limits (size, duration, event count) for the native binary saver. |
| `enable_time_tag_file_saving`| `base_filename: str`, `format: int`| `None` | Starts routing `RAW_TAGS` directly to disk, bypassing Python. |

---

### [`NexatomTimeTagReader`](5_2_nexatomtt_classes.md#nexatomtimetagreader)

This class provides memory-safe iteration over offline `.nxtt` binary files. It implements the Python Context Manager protocol (`with ... as reader:`), ensuring file handles are closed properly.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `header` | *None* | `NexatomTimeTagFileHeader` | Returns the metadata block containing the creation timestamp. |
| `read_batch` | `max_tags: int` | `list[NexatomTimeTag]` | Loads a fixed chunk of time tags into a Python list. Best for small files. |
| `iter_tags` | `batch_size: int = 100000`| `Generator` | **Recommended:** Returns a Python generator that yields tags sequentially. Allows processing of files larger than system RAM. |
| `export_csv` | `output_path: str`, `batch_size: int = 100000` | `int` | Utilizes the C++ core to convert the binary file to a CSV at maximum speed, bypassing the Python GIL. Returns total rows written. |

---

### [`NexatomError`](5_2_nexatomtt_classes.md#nexatomerror)

The SDK uses exceptions to handle hardware and configuration failures. If a C API function returns a negative error code, the Python wrapper intercepts it and raises a `NexatomError`.

| Property | Type | Description |
| :--- | :--- | :--- |
| `code` | `int` | The native `nexatom_error_code_t` enum value (e.g., `-5` for Timeout). |
| `native_message` | `str` | The highly detailed, internal C++ string explaining exactly why the failure occurred. |

---

### [Runtime Boot Classes (`nexatomtt.runtime_boot`)](5_2_nexatomtt_classes.md#runtime-boot-classes)

These dataclasses are required to orchestrate the transition from the safety `BOOTLOADER` mode into `RUNTIME` mode via the `open_runtime_device()` context manager.

#### `RuntimeBootOptions`
| Property | Type | Description |
| :--- | :--- | :--- |
| `timeout_ms` | `int` | Total maximum time (in milliseconds) allowed for the boot and USB reconnect process. |
| `mode_timeout_sec` | `float` | Maximum time to wait for the device to settle into a new protocol mode. |
| `poll_sec` | `float` | Sleep interval between USB discovery attempts while the device is offline. |
| `preferred_slot` | `int` \| `None` | The specific firmware flash slot (0-2) to boot. If `None`, the default slot is used. |

#### `DeviceIdentity`
Used internally by the orchestrator to ensure that the device that reconnects to the USB bus is the exact same physical unit that was commanded to boot.

| Property | Type | Description |
| :--- | :--- | :--- |
| `connection_id` | `str` | The physical USB port topological path. (Primary matching criteria). |
| `serial_number` | `str` | The unique hardware serial string. (Fallback matching criteria). |
