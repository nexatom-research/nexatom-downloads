## Data Connection

The UTT810 manages a high-bandwidth data connection over USB, offering distinct output modes tailored for realtime analysis or bulk offline storage.

### [FTDI USB 3.0 transport layer](2_3_data_connection.md#ftdi-usb-3-transport-layer)

Data is streamed from the FPGA to the host PC via an FTDI FT60x SuperSpeed USB 3.0 controller. The host SDK consumes this stream via the FTDI D3XX kernel driver, managing internal circular buffers to prevent data loss during operating system scheduling interruptions.

Sustained bandwidth depends on the host USB controller and SSD write speeds (if file saving is enabled). High-count-rate applications should use direct USB 3.0 ports on the host motherboard.

### [Output data type modes](2_3_data_connection.md#output-data-type-modes)

The hardware data shuffler can be configured into one of four output modes, defined by `nexatom_output_data_type_t`:

| Value | Constant | Description |
|---|---|---|
| `0` | `NEXATOM_OUTPUT_NO_OUTPUT` | Data path is disabled. No events are sent to the host. Must be set prior to enabling or disabling file saving. |
| `1` | `NEXATOM_OUTPUT_ENCODED_TAGS` | Hardware-encoded time tags. (Intermediate format for specialized processing). |
| `2` | `NEXATOM_OUTPUT_RAW_TAGS` | High-bandwidth raw time tag stream. Used exclusively for binary file capture (`.nxtt`). Callbacks are not invoked in this mode. |
| `3` | `NEXATOM_OUTPUT_REALTIME_DATA` | Processed realtime stream. FPGA emits CPS, telemetry, TIHI, and MFCO packets. SDK routes these to registered callbacks. |

#### C API

```c
/* Set the output mode */
nexatom_tt_set_output_type(device, NEXATOM_OUTPUT_REALTIME_DATA);

/* Query the current mode */
nexatom_output_data_type_t current_mode;
nexatom_tt_get_output_type(device, &current_mode);
```

#### Python

```python
from nexatomtt import NEXATOM_OUTPUT_REALTIME_DATA

device.set_output_type(NEXATOM_OUTPUT_REALTIME_DATA)
current_mode = device.get_output_type()
```

> **State transition rule.** Changing the output mode should generally be done while the system is disabled (`enable_system(False)`) to prevent partial packet delivery. When switching out of `RAW_TAGS` file-saving mode, you must set `NO_OUTPUT` before disabling the file saving subsystem.

### [Performance monitoring](2_3_data_connection.md#performance-monitoring)

The SDK includes a performance monitoring subsystem to track data throughput, processing bottlenecks, and event drops.

#### Enabling monitoring

Performance monitoring can be toggled via the C API. When enabled, the native library tracks processing rates and buffer health.

```c
nexatom_tt_set_performance_monitoring(device, true);
```

#### Acquisition status structure

When monitoring is enabled, the SDK maintains a `nexatom_acquisition_status_t` structure containing comprehensive health metrics.

| Metric category | Fields | Description |
|---|---|---|
| **Timing** | `start_time_ms`, `elapsed_time_sec` | Session duration. |
| **Event Statistics** | `total_events`, `processed_events`, `dropped_events` | Global event counters. Non-zero `dropped_events` indicates host processing or SSD I/O cannot keep up with the FPGA stream. |
| **Channel Statistics** | `channel_stats[8]` | Array of `nexatom_channel_stats_t` containing exact `event_count`, `count_rate` (Hz), and `last_timestamp` (ps) per channel. |
| **Performance** | `data_rate_mbps`, `processing_rate_meps` | Realtime throughput in Megabytes/sec (MB/s) and Millions of Events Per Second (MEPS). |
| **Health** | `is_keeping_up` | Boolean flag. `false` if the SDK circular buffers are nearing overflow. |
| **File I/O** | `file_saving_active`, `bytes_written`, `current_filename` | Status of the active `.nxtt` file writer. |
