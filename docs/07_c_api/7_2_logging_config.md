# Logging Configuration

The NexatomTT C API includes a high-performance, asynchronous logging pipeline that runs entirely in the background C++ layer. It utilizes a two-tier filtering system:
1. **Module Masking:** Disabling/enabling specific sub-components (e.g., `TRANSPORT`, `HARDWARE_CMD`).
2. **Level Filtering:** Setting minimum severity thresholds (e.g., `DEBUG`, `INFO`) per module.

By default, logs are printed directly to `stdout`. To integrate with your own application's logging infrastructure (like `spdlog` or Windows Event Viewer), you must bind a custom C function pointer using `nexatom_tt_set_log_callback`.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_set_log_callback` | `[In] nexatom_log_callback callback`<br>`[In] void* user_data` | `void` | Overrides the default `stdout` behavior, routing all C++ core logs to a user-provided function pointer. |
| `nexatom_tt_set_module_enabled` | `[In] nexatom_log_module_t module`<br>`[In] bool enable` | `void` | Enables or disables an individual logging module. Disabling a module suppresses all its logs regardless of severity level. |
| `nexatom_tt_set_module_log_level` | `[In] nexatom_log_module_t module`<br>`[In] nexatom_log_level_t level` | `void` | Sets the minimum severity threshold (e.g., `NEXATOM_LOG_WARNING`) for a specific module. |
| `nexatom_tt_set_enabled_modules` | `[In] uint32_t module_mask` | `void` | Convenience function to enable multiple logging modules simultaneously using a bitmask (e.g., `(1 << NEXATOM_LOG_MODULE_TRANSPORT)`). |

### Data Structures: `nexatom_log_record_t`

When a custom callback is registered, it receives a fully-copied `nexatom_log_record_t` passed **by-value**. The host application does not need to free this memory; the struct is valid for the duration of the callback execution.

| Field | Type | Description |
| :--- | :--- | :--- |
| `timestamp` | `char[32]` | Null-terminated ISO-8601 timestamp string (UTC). |
| `module` | `nexatom_log_module_t` | Enum identifying the source component (e.g., `TRANSPORT` = 1, `DECODER` = 2). |
| `level` | `nexatom_log_level_t` | Enum severity (`DEBUG` = 1, `INFO` = 2, `WARNING` = 3, `ERROR` = 4). |
| `category` | `nexatom_log_category_t` | Enum classifying the log event (e.g., `REGISTER_WRITE`, `ERROR_CONDITION`). |
| `sequence_id` | `uint64_t` | Process-global delivery ordering ID. |
| `repeat_count` | `uint32_t` | Identifies duplicate messages suppressed by the backend (0 if none). |
| `message_length` | `uint32_t` | The string length of the message payload (excluding the null terminator). |
| `message` | `char[1024]` | The actual UTF-8 formatted log message, guaranteed to be null-terminated. |

### C Example: Capturing Logs

```c
#include <stdio.h>
#include "nexatomtt_c_api.h"

// 1. Define the callback function matching the signature
void my_custom_logger(nexatom_log_record_t record, void* user_data) {
    if (record.level >= NEXATOM_LOG_ERROR) {
        fprintf(stderr, "[FATAL] %s: %s\n", record.timestamp, record.message);
    } else {
        printf("[INFO] %s\n", record.message);
    }
}

int main() {
    // 2. Register the callback early in process execution
    nexatom_tt_set_log_callback(my_custom_logger, NULL);

    // 3. Configure the verbosity matrix
    // Enable hardware command diagnostics; use queried capabilities for availability.
    for (int i = 0; i < NEXATOM_LOG_MODULE_COUNT; i++) {
        nexatom_tt_set_module_enabled((nexatom_log_module_t)i, false);
    }
    
    nexatom_tt_set_module_enabled(NEXATOM_LOG_MODULE_HARDWARE_CMD, true);
    nexatom_tt_set_module_log_level(NEXATOM_LOG_MODULE_HARDWARE_CMD, NEXATOM_LOG_DEBUG);
    
    // Retire the process-global callback before its state can be released.
    return nexatom_tt_unregister_log_callback(5000) == NEXATOM_SUCCESS ? 0 : 1;
}
```

### Versioned logging controls

Preview.7 also exposes `nexatom_tt_get_logging_capabilities`, `nexatom_tt_get_logging_configuration`, `nexatom_tt_configure_logging` and `nexatom_tt_get_logging_statistics`. Initialize their record `struct_size` and `version` as specified in the shipped header. Query available modules/levels/categories rather than assuming every build includes DEBUG output.

`nexatom_tt_unregister_log_callback(timeout_ms)` waits for quiescence. Logging is process-global and separate from device callback clearing. Retain handler resources on failure; do not unregister from a native callback. Its bounded queue can drop diagnostic records when a consumer is slow; see [logging behaviour](../06_in_depth_guides/6_3_logging_system.md).
