## Callback Thread Safety and Data Lifetime

Bridging asynchronous, multi-threaded C++ hardware events into managed languages like Python introduces severe memory lifetime and concurrency risks. The NexatomTT SDK employs strict memory boundaries to guarantee thread safety across the Foreign Function Interface (FFI).

### [Pass-by-value callback semantics](6_5_callback_thread_safety_and_data_lifetime.md#pass-by-value-callback-semantics)

Unlike traditional C APIs that dispatch pointers to internal memory buffers, the NexatomTT SDK strictly utilizes **pass-by-value** semantics for all data callbacks.

When the background `ProcessingThread` dispatches a payload (e.g., `nexatom_tihi_callback_data_t`), the entire struct—including all histogram bins and mathematical fit results—is copied directly into the host language's memory space.

*   **Zero Dangling Pointers:** If the hardware connection drops, or if the user issues a `request_global_stop_all_modes()` command mid-callback, the internal hardware buffers are safely torn down without risking access violations in the host application.
*   **Async Dispatch Safety:** The host application unconditionally owns the data packet. It can safely push the struct into thread-safe queues (e.g., Python's `queue.Queue`) for delayed processing by a GUI event loop.
*   **Performance Overhead:** Because aggregated hardware data (TIHI, MFCO, CPS) is typically emitted at display rates (1–10 Hz), the microsecond-level CPU overhead of copying a few kilobytes of struct data is mathematically negligible compared to the architectural safety it provides.

### [Python callback reference management](6_5_callback_thread_safety_and_data_lifetime.md#python-callback-reference-management)

When passing a Python function to a C API via `ctypes.CFUNCTYPE`, the Python Garbage Collector (GC) is unaware that the native C library holds a reference to the function pointer. If the Python function object falls out of scope and is garbage collected, subsequent executions of the callback by the C++ background thread will instantly trigger a segmentation fault.

The `NexatomDevice` wrapper class safely encapsulates this lifecycle:
*   **`_callbacks` dictionary:** Retains a strong reference to the active `CFUNCTYPE` object, preventing premature garbage collection.
*   **`_retired_callbacks` list:** When hot-swapping a callback (assigning a new callback while the hardware is actively acquiring), the old callback is moved to a retired list. This prevents a race condition where the Python GC destroys the old callback while the C++ thread is mid-dispatch. The retired list is safely flushed on the next GC cycle.

### [Callback registration and clearing](6_5_callback_thread_safety_and_data_lifetime.md#callback-registration-and-clearing)

The SDK exposes 9 distinct registration functions to assign function pointers for specific data endpoints:
*   `nexatom_tt_set_time_histogram_callback()`
*   `nexatom_tt_set_multifold_coincidence_callback()`
*   `nexatom_tt_set_count_rate_callback()`
*   `nexatom_tt_set_multi_tau_correlation_callback()`
*   `nexatom_tt_set_linear_correlation_callback()`
*   `nexatom_tt_set_telemetry_callback()`
*   `nexatom_tt_set_config_dump_callback()`
*   `nexatom_tt_set_connection_status_callback()`
*   `nexatom_tt_set_log_callback()` *(Global scope, not per-device)*

To safely detach the host application from the background processing thread—especially prior to destroying a GUI window or shutting down the device—invoke `nexatom_tt_clear_callbacks()`. This synchronously blocks until all active dispatches complete, then zeroes all function pointers, guaranteeing no further code execution in the host environment.
