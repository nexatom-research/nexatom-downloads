## Callback Thread Safety and Data Lifetime

Bridging native hardware events into Python or a C/C++ application requires clear rules for data lifetime and handler ownership. The SDK provides owned Python result copies and defined C callback layouts; the application remains responsible for its queues, shared state and shutdown order.

### [Pass-by-value callback semantics](6_5_callback_thread_safety_and_data_lifetime.md#pass-by-value-callback-semantics)

Most fixed C data callbacks use **pass-by-value** records. The versioned configuration-dump callback is a pointer-view exception described below.

For a fixed record such as `nexatom_tihi_callback_data_t`, the C handler receives the histogram and fit fields by value. That argument is still a local C value: retaining its address after return is invalid. The public Python binding copies the native record before calling the Python handler, so the handler can retain the resulting Python object.

*   **Data ownership:** Copy fixed C records into application storage before returning if they are needed later. Python records from the public wrapper are already owned copies.
*   **Deferred processing:** Owned records can be passed to a synchronized application queue for plotting or analysis. Queue bounds and overload reporting remain application decisions.
*   **Callback cost:** Return promptly. Copying a result provides a clear lifetime boundary, but expensive plotting, blocking I/O and unbounded analysis in a handler can delay native processing.

```c
/* Fragment: user_data points to application-owned, synchronized storage. */
static void on_cps(nexatom_cps_data_t data, void *user_data) {
    nexatom_cps_data_t *snapshot = (nexatom_cps_data_t *)user_data;
    *snapshot = data; /* Copy the value. Never keep &data after this returns. */
}
```

If another thread reads `snapshot`, protect the transfer with application synchronization; the fragment alone is not a thread-safe queue.

**Configuration-view exception:** The C `nexatom_tt_set_config_dump_view_callback_v1` handler receives a borrowed view; it is the one callback that receives a pointer, and every other callback receives its data by value. Both the view and its `records` pointer expire when the callback returns. Copy metadata and deep-copy the valid records; copying only the view preserves a dangling pointer. The public Python wrapper performs this deep copy and retains the copied records with the returned view.

### [Python callback reference management](6_5_callback_thread_safety_and_data_lifetime.md#python-callback-reference-management)

When passing a Python function to a C API via `ctypes.CFUNCTYPE`, the Python Garbage Collector (GC) is unaware that the native C library holds a reference to the function pointer. If the Python function object falls out of scope and is garbage collected, subsequent executions of the callback by the C++ background thread will instantly trigger a segmentation fault.

The `NexatomDevice` wrapper class safely encapsulates this lifecycle:
*   **`_callbacks` dictionary:** Retains a strong reference to the active `CFUNCTYPE` object, preventing premature garbage collection.
*   **`_retired_callbacks` list:** Replaced or cleared ctypes handlers remain referenced until native device destruction completes. A worker can already have selected the old handler when registration changes, so the next Python GC cycle is not a safe retirement boundary.

### [Callback registration and clearing](6_5_callback_thread_safety_and_data_lifetime.md#callback-registration-and-clearing)

The C API exposes registration functions for measurement, diagnostic and event endpoints, including:
*   `nexatom_tt_set_time_histogram_callback()`
*   `nexatom_tt_set_multifold_coincidence_callback()`
*   `nexatom_tt_set_count_rate_callback()`
*   `nexatom_tt_set_multi_tau_correlation_callback()`
*   `nexatom_tt_set_linear_correlation_callback()`
*   `nexatom_tt_set_telemetry_callback()`
*   `nexatom_tt_set_config_dump_callback()`
*   `nexatom_tt_set_config_dump_view_callback_v1()`
*   `nexatom_tt_set_telemetry_view_callback_v1()`
*   `nexatom_tt_set_fast_tihi_result_callback()`
*   `nexatom_tt_set_connection_status_callback()`
*   `nexatom_tt_set_log_callback()` *(Global scope, not per-device)*

The Python wrapper exposes the corresponding public measurement and diagnostic callbacks, including CORL/CORM and configuration views. Logging registration is a method of `NexatomLibrary`, because its scope is process-global. See [Python callback signatures](../05_api_reference/5_4_callback_types.md) and the C callback reference.

`nexatom_tt_clear_callbacks()` clears device registrations; it is **not** a quiescence fence for an invocation already selected by native. C/FFI callers must keep handler code and `user_data` alive until native device destruction completes. Python's retired-reference handling provides that protection for the public wrapper; it does not make arbitrary application-owned state safe to destroy early.

### Shutdown, logging and reentrancy

Keep an application control thread responsible for stopping measurements, finalizing sinks, disconnecting and destroying the handle. Do not close/destroy a device, replace callbacks or wait for the same callback from inside a native callback. Unsupported reentrant operations can return BUSY or other errors. Report cleanup failures instead of assuming the device is closed.

Connection callback registration can invoke a handler inline; field-update progress also runs within its calling operation. UI changes must therefore be marshalled to the UI thread regardless of callback type.

Logging has a separate quiescent unregister operation, `unregister_log_callback(timeout_ms=...)`. A successful unregister permits releasing logger handler state. On timeout/BUSY/failure, retain those resources. Device callback clearing does not unregister logging, and ordinary callback setters do not inherit logging's stronger quiescence contract.
