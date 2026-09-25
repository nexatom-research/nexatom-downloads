## Callback Types

The NexatomTT SDK uses callbacks to push data and events into the host application. Measurement callbacks generally run on native workers; connection registration and field-update progress can invoke handlers inline during an API call. No callback should assume that it runs on a UI thread.

> **Python Implementation Note:** While the underlying C API requires a function pointer and an opaque `void* user_data` context pointer, the Python wrapper abstracts this away. You can simply pass standard Python functions or class methods. Python closures and object states eliminate the need for manual `user_data` management.

<a id="callback-types"></a>

### [Data callbacks](5_4_callback_types.md#callback-types)

Data callbacks deliver results according to the processor's result span and run end; `result_status` says why each one was published. The public Python wrappers supply owned records. At the C boundary every callback record is passed by value, including the Fast TIHI result with its bins, so the receiver owns its copy. The one exception is the versioned configuration-dump view, a pointer valid only during the call (see [callback lifetime](../06_in_depth_guides/6_5_callback_thread_safety_and_data_lifetime.md)).

| Registration Method | Expected Python Signature | Trigger Condition |
| :--- | :--- | :--- |
| `set_time_histogram_callback` | `def on_tihi(data: NexatomTihiData) -> None:` | Fires for each published TIHI result: about once a second in `WHOLE_RUN`, once per block in `BLOCK`, and a final `RUN_COMPLETE` or `STOPPED` result. |
| `set_multifold_coincidence_callback` | `def on_mfco(data: NexatomMfcoData) -> None:` | Fires for each published MFCO result, as for TIHI. |
| `set_count_rate_callback` | `def on_cps(data: NexatomCpsData) -> None:` | Fires continuously based on the `cps_period_selector` interval (typically 100ms or 1000ms). |
| `set_multi_tau_correlation_callback` | `def on_corm(data: NexatomCormData) -> None:` | Fires for each published CORM result, as for TIHI. |
| `set_linear_correlation_callback` | `def on_corl(data: NexatomCorlData) -> None:` | Fires for each published CORL result, as for TIHI. |
| `set_telemetry_callback` | `def on_telem(data: NexatomTelemetryData) -> None:` | Delivers the legacy telemetry representation. Request explicitly; periodic behavior depends on the runtime. |
| `set_config_dump_callback` | `def on_config(data: NexatomConfigDumpData) -> None:` | Fires upon explicit request via `request_config_dump()`. |
| `set_telemetry_view_callback` | `def on_telem(data: NexatomTelemetryViewV1) -> None:` | Supplies a versioned telemetry view; check availability flags. |
| `set_config_dump_view_callback` | `def on_config(data: NexatomConfigDumpViewV1) -> None:` | Supplies an owned Python copy of versioned configuration records. |
| `set_fast_tihi_result_callback` | `def on_fast(data: NexatomFastTihiResult) -> None:` | Supplies one Fast TIHI context's result (four back to back per result) when the profile supports this engine. The C record is passed by value, bins included. |

### [Event callbacks](5_4_callback_types.md#event-callbacks)

Event callbacks report state changes or progress rather than histogram data. Registration can immediately report the current connection state.

| Registration Method | Expected Python Signature | Trigger Condition |
| :--- | :--- | :--- |
| `set_connection_status_callback` | `def on_status(connected: bool, ready: bool) -> None:` | Fires when the link drops (`connected=False`) or the device becomes ready (`ready=True`), including after an automatic reconnect. Registering only observes the connection; it does not open one. |
| `set_log_callback` *(Global Library Method)* | `def on_log(record: NexatomLogRecord) -> None:` | Fires whenever an internal SDK module emits a log message that passes the configured severity thresholds. |
| *(Passed as argument to firmware loader)* | `def on_progress(progress: NexatomFieldUpdateProgress) -> None:` | Fires continuously during a firmware flash to report state machine phases and completion percentages. |

### Queueing a result for application processing

```python
from queue import Queue, Full

# A bounded queue makes overload visible instead of consuming unlimited RAM.
results = Queue(maxsize=32)
application_drops = 0

def on_tihi(data):
    global application_drops
    try:
        results.put_nowait(data)  # Public Python wrapper supplied an owned copy.
    except Full:
        application_drops += 1  # Report this alongside the measurement outcome.

device.set_time_histogram_callback(on_tihi)
# Start acquisition using the complete tutorial's sequence. A separate consumer
# reads results, checks result_status and acquisition_done_status and updates
# plots or analysis. After Stop, wait for the one STOPPED result.
```

Do not perform slow plotting or unbounded disk work in the native callback. The application must synchronize shared state and define how it reports dropped queue items. Native file saving is a separate path and does not depend on this example queue.

After stopping acquisition, `clear_callbacks()` clears registrations but does not fence an already-selected native invocation. Python retains retired ctypes handlers until device destruction. C/FFI callers must keep their handler and user-data storage alive through the same boundary. The process-global logger has a separate quiescent `unregister_log_callback(timeout_ms=...)`; clearing device callbacks does not unregister it. See [lifetime and shutdown](../06_in_depth_guides/6_5_callback_thread_safety_and_data_lifetime.md).
