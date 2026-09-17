# 5.4 Callback types

Register Python callables through the device/library methods, not raw ctypes trampolines. The public binding retains callback references and copies native records for Python ownership.

| Registration | Data |
| --- | --- |
| `set_connection_status_callback` | `callback(connected: bool, ready: bool)`; no error argument, and registration can invoke it inline |
| `set_count_rate_callback` | `NexatomCpsData` |
| `set_time_histogram_callback` | `NexatomTihiData` |
| `set_multifold_coincidence_callback` | `NexatomMfcoData` |
| `set_linear_correlation_callback` | `NexatomCorlData` |
| `set_multi_tau_correlation_callback` | `NexatomCormData` |
| `set_telemetry_callback` | Legacy `NexatomTelemetryData` |
| `set_telemetry_view_callback` | `NexatomTelemetryViewV1` |
| `set_config_dump_callback` | Legacy configuration dump |
| `set_config_dump_view_callback` | Versioned configuration view |
| `set_fast_tihi_histogram_callback` | Fast TIHI result where supported |
| Library `set_log_callback` | Owned `NexatomLogRecord`; process-global logger |

```python
# Keep native callback work small; a separate consumer can process this snapshot.
latest = {}
def on_histogram(data):
    latest["tihi"] = data  # The public Python wrapper supplies an owned record.
device.set_time_histogram_callback(on_histogram)
# Start/observe/stop the measurement using the complete acquisition template.
# Clear callbacks from the owning control thread after acquisition is retired.
```

`clear_callbacks()` clears device registrations but is not a quiescence fence for an already-selected invocation. Python retains retired callback references until native destruction; C/FFI consumers must preserve equivalent ownership. Library logging separately offers `unregister_log_callback(timeout_ms=...)` with explicit quiescence. On failure retain resources. Do not close a device or block waiting for yourself inside a callback. See [lifetime rules](../06_in_depth_guides/6_5_callback_thread_safety_and_data_lifetime.md).

[Python reference](index.md)
