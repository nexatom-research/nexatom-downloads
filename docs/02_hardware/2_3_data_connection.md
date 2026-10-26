## Data Connection

The UTT810 manages a high-bandwidth data connection over USB, offering distinct output modes tailored for realtime analysis or bulk offline storage.

### FTDI USB 3.0 transport layer

Data is streamed from the FPGA to the host via an FTDI FT60x SuperSpeed USB 3.0 controller. The native SDK owns the D3XX transport, asynchronous read buffers, decoding and shutdown. Its buffers absorb short scheduling interruptions; they cannot guarantee loss-free capture when the sustained source rate exceeds transport, processing or storage capacity.

Sustained bandwidth depends on the host USB controller and SSD write speeds (if file saving is enabled). High-count-rate applications should use direct USB 3.0 ports on the host motherboard.

### Output data type modes

The hardware data shuffler can be configured into one of four output modes, defined by `nexatom_output_data_type_t`:

| Value | Constant | Description |
|---|---|---|
| `0` | `NEXATOM_OUTPUT_NO_OUTPUT` | Quiet measurement output; use this before changing saver configuration or leaving raw capture. |
| `1` | `NEXATOM_OUTPUT_ENCODED_TAGS` | Encoded time-tag output where advertised by the resolved profile. No shipping image offers it: `set_output_type()` returns `NEXATOM_ERROR_NOT_SUPPORTED`. |
| `2` | `NEXATOM_OUTPUT_RAW_TAGS` | Time-tag stream for the native time-tag saver: BINARY (`.nxtt`), CSV or TAB. Processed measurement callbacks are not the raw event delivery interface. |
| `3` | `NEXATOM_OUTPUT_REALTIME_DATA` | Processed stream: supported CPS, telemetry, TIHI, MFCO and correlation packets are decoded and routed to registered callbacks/savers. |

An enum value being defined does not mean every model implements it. Check `supported_output_mode_mask` in the connected profile and the relevant feature flag before enabling a mode or measurement engine. A mode the profile does not list returns `NEXATOM_ERROR_NOT_SUPPORTED` before anything is written.

After connect the output is `NO_OUTPUT` and the board sends nothing until the application selects an output type (see [What connect leaves on the board](2_1_operating_conditions.md#what-connect-leaves-on-the-board)).

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

> **State transition rule.** Changing the output mode should generally be done while the system is disabled (`enable_system(False)`) to prevent partial packet delivery.

> **Output type and recordings.** During a file save, switching the output on from `NO_OUTPUT` never ends the recording. Any other change of the output type ends it at the switch, with the end reason `OUTPUT_TYPE_CHANGED`; that is a normal end and the file holds the whole span up to the switch. When `set_output_type()` returns, the earlier recording is fully closed (the call waits up to 5 s; if closing takes longer, the saving result shows `CLOSE_PENDING` until it finishes). To end a `RAW_TAGS` recording as a user stop, call `disable_time_tag_file_saving()` while the raw output is still selected, then select `NO_OUTPUT` (see [File saving and data export](../06_in_depth_guides/6_1_file_saving_and_data_export.md)).

### Performance monitoring

`set_performance_monitoring` is retained for compatibility and still has no effect in this SDK. It accepts a flag and may emit a diagnostic log, but it does **not** activate host throughput statistics or change acquisition, communication or telemetry behavior.

#### Enabling monitoring

Existing applications may continue calling it through C or Python:

```c
nexatom_tt_set_performance_monitoring(device, true);
```

```python
device.set_performance_monitoring(True)
```

#### Acquisition status structure

The public headers retain `nexatom_acquisition_status_t` and Python exposes the matching `NexatomAcquisitionStatus` record. There is no public getter/callback that supplies populated host statistics. To prove that a capture reached its file, use the [file-saving statistics](../07_c_api/7_17_file_saving.md#file-saving-statistics). The fields below explain the retained record layout, not measurements made available by enabling this flag:

| Metric category | Fields | Description |
|---|---|---|
| **Timing** | `start_time_ms`, `elapsed_time_sec` | Session duration. |
| **Event Statistics** | `total_events`, `processed_events`, `dropped_events` | Retained event-counter fields; not populated through a public monitoring interface. |
| **Channel Statistics** | `channel_stats[8]` | Retained `event_count`, `count_rate`, `last_timestamp` fields. |
| **Performance** | `data_rate_mbps`, `processing_rate_meps` | Retained throughput fields; no live values are exposed. |
| **File I/O** | `file_saving_active`, `bytes_written`, `current_filename` | Retained writer fields; inspect actual saver results/files instead. |

The record has no `is_keeping_up` field. To see whether a recording keeps up with the data, use the data-flow status (`get_data_flow_status()`, see [Keeping up with the data](../06_in_depth_guides/6_1_file_saving_and_data_export.md#keeping-up-with-the-data)).

For a real capture, check returned errors, packet quality/error flags, saved-file contents and the instrument's telemetry. Those are separate evidence sources: hardware CPS can remain nonzero even when a host capture is incomplete. See [File Saving and Data Export](../06_in_depth_guides/6_1_file_saving_and_data_export.md) for recording and checking individual time tags and processed packets.
