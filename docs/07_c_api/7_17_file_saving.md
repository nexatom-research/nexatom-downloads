# File Saving

The File Saving module enables direct-to-disk streaming of instrument data. This circumvents the need for the host application to manually serialize data payloads received in the callbacks.

The C++ background thread handles high-speed disk I/O, utilizing asynchronous buffering and automatic file rotation based on limits (e.g., maximum megabytes, duration, or event counts).

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_enable_time_tag_file_saving` | `[In] nexatom_tt_handle device`<br>`[In] const char* base_filename` | `nexatom_error_code_t` | Arms the background thread to stream absolute 16-byte raw photon time tags directly to disk as `.nxtt` binaries. |
| `nexatom_tt_disable_time_tag_file_saving`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Flushes the kernel buffers and gracefully closes the raw binary file handle. |
| `nexatom_tt_set_time_tag_file_config` | `[In] nexatom_tt_handle device`<br>`[In] const nexatom_time_tag_file_config_t* config` | `nexatom_error_code_t` | Configures file rotation limits and naming suffix rules for raw binary files. |
| `nexatom_tt_enable_processed_file_saving`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_processed_packet_type_t packet_type`<br>`[In] const char* filename` | `nexatom_error_code_t` | Arms the background thread to serialize aggregated histogram or correlation curves (TIHI, MFCO, CORM) to disk. |
| `nexatom_tt_disable_processed_file_saving`| `[In] nexatom_tt_handle device`<br>`[In] nexatom_processed_packet_type_t packet_type` | `nexatom_error_code_t` | Flushes and closes the target processed stream file. |
| `nexatom_tt_set_processed_file_config` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_processed_packet_type_t packet_type`<br>`[In] const nexatom_processed_file_config_t* config`| `nexatom_error_code_t` | Configures file rotation limits for processed payloads. |

### Data Structures: `nexatom_time_tag_file_config_t`

*(Note: `nexatom_processed_file_config_t` shares this identical memory layout).*

When configuring the auto-rotator, developers pass this struct by-value. Setting a limit to `0` acts as "infinity" (no limit).

| Field | Type | Description |
| :--- | :--- | :--- |
| `format` | `uint32_t` (Enum) | File format specifier (e.g., standard `.nxtt` binary). |
| `max_file_size_mb` | `uint64_t` | Closes the current file and creates a new suffix part when this size (in MB) is exceeded. |
| `max_duration_minutes` | `uint32_t` | Rotates the file after X minutes of acquisition time. |
| `max_event_count` | `uint64_t` | Rotates the file after X absolute photon events. |
| `rotate_on_acquisition_boundary`| `bool` | True to cleanly rotate the file every time the hardware is stopped and started again. |
| `include_timestamp` | `bool` | True to automatically inject ISO-8601 strings into the output filename. |
| `flush_immediately` | `bool` | True to disable block-buffering and call `fsync` after every write (severely impacts performance). |
| `custom_suffix` | `char[64]` | Optional string appended before the file extension. |

### C Example: Streaming Raw Binaries

```c
nexatom_time_tag_file_config_t config = {0};
config.format = NEXATOM_TIME_TAG_FILE_FORMAT_NXTT;
config.max_file_size_mb = 1024; // Rotate every 1 GB
config.rotate_on_acquisition_boundary = true;
config.include_timestamp = true;

// 1. Configure the rotation rules
nexatom_tt_set_time_tag_file_config(my_device, &config);

// 2. Open the file handle (will auto-create the file)
nexatom_tt_enable_time_tag_file_saving(my_device, "/data/my_experiment_run");

// 3. Start the hardware
nexatom_tt_enable_system(my_device, true);

// ... Let it run overnight ...

// 4. Safely terminate the stream
nexatom_tt_enable_system(my_device, false);
nexatom_tt_disable_time_tag_file_saving(my_device);
```
