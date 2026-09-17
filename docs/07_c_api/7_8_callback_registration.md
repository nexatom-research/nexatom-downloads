# Callback Registration

The NexatomTT C API dispatches decoded data asynchronously to registered callbacks. Applications use the public API rather than polling USB buffers or depending on internal thread classes.

**Callback ownership:** Fixed data records such as TIHI/MFCO/CPS are passed by value; copy them before retaining data beyond the callback. The versioned configuration-dump callback instead receives a borrowed view pointer and borrowed records: deep-copy both metadata and records before return. Neither a pointer to a local by-value argument nor a shallow copy of a pointer-based view establishes lasting ownership.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_set_time_histogram_callback` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_time_histogram_callback callback`<br>`[In] void* user_data` | `nexatom_error_code_t` | Binds a function pointer to receive `nexatom_tihi_callback_data_t` payloads containing decay curves. |
| `nexatom_tt_set_multifold_coincidence_callback` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_multifold_coincidence_callback callback`<br>`[In] void* user_data` | `nexatom_error_code_t` | Binds a function pointer to receive `nexatom_mfco_callback_data_t` containing boolean logic patterns. |
| `nexatom_tt_set_count_rate_callback` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_count_rate_callback callback`<br>`[In] void* user_data` | `nexatom_error_code_t` | Binds a function pointer to receive `nexatom_cps_data_t` containing periodic Count Per Second metrics. |
| `nexatom_tt_set_multi_tau_correlation_callback` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_multi_tau_correlation_callback callback`<br>`[In] void* user_data` | `nexatom_error_code_t` | Binds a function pointer to receive `nexatom_corm_callback_data_t` (logarithmic correlation and DLS/FCS physics). |
| `nexatom_tt_set_linear_correlation_callback` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_linear_correlation_callback callback`<br>`[In] void* user_data` | `nexatom_error_code_t` | Binds a function pointer to receive `nexatom_corl_callback_data_t` (linear correlation). |
| `nexatom_tt_set_telemetry_callback` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_telemetry_callback callback`<br>`[In] void* user_data` | `nexatom_error_code_t` | Binds a function pointer to receive periodic `nexatom_telemetry_data_t` hardware health updates. |
| `nexatom_tt_set_config_dump_callback` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_config_dump_callback callback`<br>`[In] void* user_data` | `nexatom_error_code_t` | Binds a function pointer to receive register states requested via manual diagnostic dumps. |
| `nexatom_tt_set_connection_status_callback` | `[In] nexatom_tt_handle device`<br>`[In] nexatom_connection_status_callback callback`<br>`[In] void* user_data` | `nexatom_error_code_t` | Binds a function pointer that triggers asynchronously if the physical USB connection drops or re-establishes. |
| `nexatom_tt_clear_callbacks` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Clears stored registrations; an already-selected invocation can remain. Retain callback code/user data until native destruction completes. |

### Expected Function Signatures

When writing your callback functions in C, you must strictly adhere to the following `typedef` signatures. Every callback includes an opaque `void* user_data` pointer, allowing you to pass class instances (`this` in C++) or application state contexts safely into the background thread.

```c
typedef void (*nexatom_time_histogram_callback)(nexatom_tihi_callback_data_t data, void* user_data);
typedef void (*nexatom_multifold_coincidence_callback)(nexatom_mfco_callback_data_t data, void* user_data);
typedef void (*nexatom_count_rate_callback)(nexatom_cps_data_t data, void* user_data);
typedef void (*nexatom_multi_tau_correlation_callback)(nexatom_corm_callback_data_t data, void* user_data);
typedef void (*nexatom_linear_correlation_callback)(nexatom_corl_callback_data_t data, void* user_data);
typedef void (*nexatom_telemetry_callback)(nexatom_telemetry_data_t data, void* user_data);
typedef void (*nexatom_config_dump_callback)(nexatom_config_dump_data_t data, void* user_data);
typedef void (*nexatom_tt_config_dump_view_callback_v1)(
    const nexatom_tt_config_dump_view_v1_t* data, void* user_data);
typedef void (*nexatom_connection_status_callback)(int connected, int ready, void* user_data);
```

> **Note:** The `nexatom_tt_set_log_callback` function is also available for registering a logging callback. It is documented in Section 7.2 (Logging Configuration) since it is typically configured alongside log levels and module filters rather than data stream callbacks.

### C Example: Safe Registration and Teardown

```c
#include <stdio.h>
#include "nexatomtt_c_api.h"

// 1. Define the callback
void on_count_rate_received(nexatom_cps_data_t data, void* user_data) {
    // Cast the opaque pointer back to our application context
    int* total_records = (int*)user_data;
    (*total_records)++;

    printf("CPS on Ch0: %u\n", data.counts[0]);
}

// 2. The acquisition owner initializes its counter to zero and keeps that
//    storage alive throughout registration, measurement, and final teardown.
static nexatom_error_code_t register_counter(
    nexatom_tt_handle my_device, int* record_counter) {
    return nexatom_tt_set_count_rate_callback(
        my_device, on_count_rate_received, record_counter);
}

// 3. Call after stopping acquisition and finalizing any active file sinks.
//    This helper consumes the handle: the caller must not destroy it again.
static int close_device_and_report(
    nexatom_tt_handle my_device, int* record_counter) {
    // Clearing is not a fence: a previously selected callback may still run.
    nexatom_error_code_t clear_rc = nexatom_tt_clear_callbacks(my_device);
    if (clear_rc != NEXATOM_SUCCESS)
        fprintf(stderr, "Callback clearing failed: %s\n", nexatom_tt_get_last_error_message());
    nexatom_error_code_t close_rc = nexatom_tt_disconnect(my_device);
    if (close_rc != NEXATOM_SUCCESS)
        fprintf(stderr, "Disconnect failed: %s\n", nexatom_tt_get_last_error_message());

    // Keep callback code and counter storage alive until destruction returns.
    nexatom_tt_destroy(my_device);
    printf("Total records processed: %d\n", *record_counter);
    return clear_rc == NEXATOM_SUCCESS && close_rc == NEXATOM_SUCCESS ? 0 : 1;
}
```

These are helpers for the acquisition owner, not a complete measurement program. Check registration before starting the engine. The counter is read only after native destruction; protect it if another application thread reads it during acquisition. The complete acquisition template additionally owns configuration, starts/stops, and file finalization. Do not invoke disconnect or destruction from inside a native callback; request shutdown on the owning application thread instead.

Preview.7 also exposes `nexatom_tt_set_telemetry_view_callback_v1`, `nexatom_tt_set_config_dump_view_callback_v1` and `nexatom_tt_set_fast_tihi_histogram_callback_v1`. Use the exact versioned typedefs and available fields from the header. Registering a callback does not grant hardware capability. Ordinary callback clearing has no logging-style quiescence guarantee; see [lifetime details](../06_in_depth_guides/6_5_callback_thread_safety_and_data_lifetime.md).
