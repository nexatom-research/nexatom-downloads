## Callback Types

The NexatomTT SDK uses asynchronous callbacks to push data and events from the background C++ `ProcessingThread` to the host application.

> **Python Implementation Note:** While the underlying C API requires a function pointer and an opaque `void* user_data` context pointer, the Python wrapper abstracts this away. You can simply pass standard Python functions or class methods. Python closures and object states eliminate the need for manual `user_data` management.

### [Data callbacks](5_4_callback_types.md#callback-types)

Data callbacks are triggered periodically based on the integration time of the respective hardware measurement engine. All data payloads are dispatched by value (see Section 6.5).

| Registration Method | Expected Python Signature | Trigger Condition |
| :--- | :--- | :--- |
| `set_time_histogram_callback` | `def on_tihi(data: NexatomTihiData) -> None:` | Fires when a TIHI measurement completes its integration window. |
| `set_multifold_coincidence_callback` | `def on_mfco(data: NexatomMfcoData) -> None:` | Fires when an MFCO measurement completes its integration window. |
| `set_count_rate_callback` | `def on_cps(data: NexatomCpsData) -> None:` | Fires continuously based on the `cps_period_selector` interval (typically 100ms or 1000ms). |
| `set_multi_tau_correlation_callback` | `def on_corm(data: NexatomCormData) -> None:` | Fires when a CORM measurement completes its integration window. |
| `set_linear_correlation_callback` | `def on_corl(data: NexatomCorlData) -> None:` | Fires when a CORL measurement completes its integration window. |
| `set_telemetry_callback` | `def on_telem(data: NexatomTelemetryData) -> None:` | Fires upon explicit request via `request_telemetry()` or continuously if auto-telemetry is enabled. |
| `set_config_dump_callback` | `def on_config(data: NexatomConfigDumpData) -> None:` | Fires upon explicit request via `request_config_dump()`. |

### [Event callbacks](5_4_callback_types.md#event-callbacks)

Event callbacks are triggered asynchronously based on system state changes, rather than hardware integration periods.

| Registration Method | Expected Python Signature | Trigger Condition |
| :--- | :--- | :--- |
| `set_connection_status_callback` | `def on_status(connected: bool, ready: bool) -> None:` | Fires when the USB physical layer drops (`connected=False`) or when the device successfully reaches the Idle state (`ready=True`). |
| `set_log_callback` *(Global Library Method)* | `def on_log(record: NexatomLogRecord) -> None:` | Fires whenever an internal SDK module emits a log message that passes the configured severity thresholds. |
| *(Passed as argument to firmware loader)* | `def on_progress(progress: NexatomFieldUpdateProgress) -> None:` | Fires continuously during a firmware flash to report state machine phases and completion percentages. |
