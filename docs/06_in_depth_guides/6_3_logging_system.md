## Logging System

The NexatomTT SDK features a highly granular, asynchronous logging architecture. To prevent the host application from being overwhelmed by high-frequency diagnostic output from the C++ core, the SDK implements a strict two-tier filtering mechanism.

### [Log architecture (two-tier filtering)](6_3_logging_system.md#log-architecture)

Log messages are evaluated against two distinct gates before being dispatched to the host application's callback:
1.  **Tier 1 (Module Enable):** The log message's originating module must be globally enabled.
2.  **Tier 2 (Severity Threshold):** The message's severity level must be greater than or equal to the minimum threshold configured for that specific module.

By default, the SDK only dispatches messages of `INFO` level or higher from the `GLOBAL` and `DEVICE` modules.

### [Log modules](6_3_logging_system.md#log-modules)

The SDK codebase is partitioned into 11 distinct logging modules (`nexatom_log_module_t`), allowing users to isolate diagnostics to specific subsystems:

| Module | Subsystem Monitored |
|---|---|
| `GLOBAL` | SDK-wide lifecycle and unhandled exceptions |
| `TRANSPORT` | FTDI D3XX kernel driver I/O and USB bulk transfers |
| `DECODER` | Proprietary binary framing protocol parsing |
| `PROCESSING` | Data routing and hardware processor aggregation |
| `FILE_SAVING` | Native binary and HDF5/CSV export engines |
| `HARDWARE_CMD` | Register-level FPGA read/write commands |
| `DEVICE` | High-level device state machine (Connect, Disconnect) |
| `CALLBACKS` | Asynchronous host dispatch timings |
| `CONFIG` | Threshold, routing, and synthetic delay application |
| `READER` | Offline `.nxtt` binary decoding |
| `FITTING` | On-board math routines (TIHI, DLS, FCS, DCS) |

### [Log levels](6_3_logging_system.md#log-levels)

Severity thresholds are defined by `nexatom_log_level_t`:

1.  `TRACE`: High-frequency, verbose diagnostic data (e.g., individual packet byte dumps).
2.  `DEBUG`: Step-by-step logic tracing.
3.  `INFO`: Standard operational milestones.
4.  `WARNING`: Non-fatal issues (e.g., USB dropped packets successfully recovered).
5.  `ERROR`: Fatal operational failures requiring host intervention.
6.  `CRITICAL`: Severe subsystem crashes.

### [Log record structure](6_3_logging_system.md#log-record-structure)

When a log passes the internal filters, it is dispatched to the host via a `nexatom_log_record_t` struct (pass-by-value). This struct contains:
*   **`timestamp`**: ISO-8601 UTC string.
*   **`module`**: The `nexatom_log_module_t` enum.
*   **`level`**: The `nexatom_log_level_t` enum.
*   **`category`**: A `nexatom_log_category_t` bitmask (`STATE_CHANGE`, `PACKET_SUMMARY`, `REGISTER_WRITE`, `ERROR_CONDITION`, `PERFORMANCE`, `INITIALIZATION`). This allows host applications to filter logs semantically without parsing the raw message string.
*   **`sequence_id`**: A monotonically increasing integer.
*   **`repeat_count`**: Number of times this exact message was suppressed to prevent log flooding.
*   **`message`**: A null-terminated character buffer (up to 1024 bytes).

### [Configuration functions](6_3_logging_system.md#configuration-functions)

The logging subsystem is configured independently of any specific device handle, as logs may be emitted during device discovery.

**C API:**
```c
// 1. Register the callback
nexatom_tt_set_log_callback(my_log_handler, user_data);

// 2. Enable/disable specific modules
nexatom_tt_set_module_enabled(MODULE_TRANSPORT, true);

// 3. Set module-specific severity thresholds
nexatom_tt_set_module_log_level(MODULE_TRANSPORT, LEVEL_DEBUG);
```

### [Usage patterns](6_3_logging_system.md#usage-patterns)

For optimal integration, host applications should tailor the logging configuration to their current operational context.

*   **Global Error Monitoring (Production):** Set all modules to `ERROR`. This guarantees zero performance overhead during normal operation, but ensures hardware disconnects or memory allocation failures are immediately reported.
*   **Hardware Debugging (Integration):** Enable the `TRANSPORT` and `HARDWARE_CMD` modules at `DEBUG` level. This reveals the exact hex values being written to the FPGA registers, isolating whether a bug resides in the SDK wrapper or the physical hardware.
*   **Algorithm Verification (Scientific):** Enable the `PROCESSING` and `FITTING` modules at `TRACE` level. This exposes the iterative convergence matrices of the Levenberg-Marquardt fitting algorithms for DLS/FCS models, aiding in mathematical troubleshooting.
