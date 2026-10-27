## Logging System

The NexatomTT SDK provides structured, asynchronous logging for diagnostics. Filtering selects the modules, severities and categories delivered to the application, while a bounded queue separates log producers from the callback. Logging is process-global for the loaded native module, not per device.

<a id="log-architecture"></a>

### [Log architecture (two-tier filtering)](6_3_logging_system.md#log-architecture)

Two principal filters determine which messages are eligible for delivery:
1.  **Tier 1 (Module Enable):** The log message's originating module must be globally enabled.
2.  **Tier 2 (Severity Threshold):** The message's severity level must be greater than or equal to the minimum threshold configured for that specific module.

Category filtering, compiled build policy and suppression also apply. Query `get_logging_capabilities()` and `get_logging_configuration()` for the effective settings rather than assuming that every diagnostic level is available in every package. Even an eligible message can be dropped if the bounded queue fills; logging is not a lossless acquisition channel.

### [Log modules](6_3_logging_system.md#log-modules)

The SDK codebase is partitioned into 11 distinct logging modules (`nexatom_log_module_t`), allowing users to isolate diagnostics to specific subsystems:

| Module | Subsystem Monitored |
|---|---|
| `GLOBAL` | SDK-wide lifecycle and unhandled exceptions |
| `TRANSPORT` | FTDI runtime I/O and USB bulk transfers |
| `DECODER` | Proprietary binary framing protocol parsing |
| `PROCESSING` | Data routing and host summing of processor results |
| `FILE_SAVING` | Native binary and HDF5/CSV export engines |
| `HARDWARE_CMD` | Register-level FPGA read/write commands |
| `DEVICE` | High-level device state machine (Connect, Disconnect) |
| `CALLBACKS` | Asynchronous host dispatch timings |
| `CONFIG` | Threshold, routing, and synthetic delay application |
| `READER` | Hardware reader operations |
| `FITTING` | Host mathematical fitting and analysis |

### [Log levels](6_3_logging_system.md#log-levels)

Severity thresholds are defined by `nexatom_log_level_t`:

1.  `TRACE`: Fine-grained diagnostics where compiled and emitted by the module.
2.  `DEBUG`: Step-by-step logic tracing.
3.  `INFO`: Standard operational milestones.
4.  `WARNING`: Conditions requiring attention; a warning does not establish that dropped data was recovered.
5.  `ERROR`: Fatal operational failures requiring host intervention.
6.  `CRITICAL`: Severe subsystem crashes.

### [Log record structure](6_3_logging_system.md#log-record-structure)

When a log passes the internal filters, it is dispatched to the host via a `nexatom_log_record_t` struct (pass-by-value). This struct contains:
*   **`timestamp`**: ISO-8601 UTC string.
*   **`module`**: The `nexatom_log_module_t` enum.
*   **`level`**: The `nexatom_log_level_t` enum.
*   **`category`**: A `nexatom_log_category_t` bitmask (`STATE_CHANGE`, `PACKET_SUMMARY`, `REGISTER_WRITE`, `ERROR_CONDITION`, `PERFORMANCE`, `INITIALIZATION`, `LIVE_CHANGE`). This allows host applications to filter logs semantically without parsing the raw message string.
*   **`sequence_id`**: A monotonically increasing integer.
*   **`repeat_count`**: Number of times this exact message was suppressed to prevent log flooding.
*   **`message_length`**: Length of the copied UTF-8 message.
*   **`message`**: A null-terminated character buffer (up to 1024 bytes).

<a id="live-change-lines"></a>

### Live-change lines

The category `NEXATOM_LOG_CATEGORY_LIVE_CHANGE` (`0x40`) gives one INFO line (module `DEVICE`) for each live change and its effect, so that a session can be traced from its log alone. It is on in the default configuration (supported and default category masks `0x7F`), and its lines are never collapsed by repetition suppression. A configuration that leaves `0x40` out of the enabled category mask, or out of a module's category mask, switches it off.

Every line reads `LIVE <EVENT> board=<model> <serial> <subject> <details> src=sdk|board`. The subject is `input=<n> (id <id>)` with n = id + 1, the application's input number. `src=board` marks what the board itself reported (telemetry); `src=sdk` marks the SDK's own action.

| Event | Logged when |
|---|---|
| `DELAY_COMMITTED` | An input-delay write was sent to the board (`old=`, `new=`); a same-value write the SDK skips on the UTT810 reads `skipped (same value)` |
| `EDGE_CHANGED` | An input edge changed (`old=`, `new=`, `recalibration=started` or `unknown`) |
| `CALIBRATION_STARTED` | A calibration started: `cause=manual`, `edge`, `time`, `temperature` or `unknown` |
| `FENCE_START` / `FENCE_END` | The board held / released its data for a calibration or a delay change (`cause=calibration` or `delay`; `FENCE_END` carries `duration_ms`) |
| `TRANSPORT_LOST` | A pulled USB cable ended the session (`connection_id=`, `after=` the time since connect) |
| `SESSION_RESTORED` | The same handle has a new session after the re-plug (`after=` the time since the loss) |

<a id="incident-blocks"></a>

### Incident blocks

When something goes wrong that support needs to see (for example a recording that ends through the SDK's or the board's fault, or a calibration that stays active too long), the SDK raises an incident and writes a block of log lines that all start with `INCIDENT <id> `: what happened, the board, the host, the state, the latest telemetry and the last board writes, and for a save end its result and files. Incident lines are ERROR level, ERROR category, and reach the log callback whatever the module, level or category filters say. A rate limit per process and minute keeps a burst of incidents from flooding the log. `get_last_incident()` returns the id and a short summary; the saving result carries the id of the incident that ended a recording. Keep the block with a support request.

### [Configuration functions](6_3_logging_system.md#configuration-functions)

The logging subsystem is configured independently of any specific device handle, as logs may be emitted during device discovery.

**C API:**
```c
// 1. Register the callback
nexatom_tt_set_log_callback(my_log_handler, user_data);

// 2. Enable/disable specific modules
nexatom_tt_set_module_enabled(NEXATOM_LOG_MODULE_TRANSPORT, true);

// 3. Set module-specific severity thresholds
nexatom_tt_set_module_log_level(NEXATOM_LOG_MODULE_TRANSPORT, NEXATOM_LOG_DEBUG);
```

### [Usage patterns](6_3_logging_system.md#usage-patterns)

For optimal integration, host applications should tailor the logging configuration to their current operational context.

*   **Error monitoring:** Enable the required modules and set their threshold to `ERROR` to reduce routine output. Also check API return values and result-status fields; a missing log is not proof of success.
*   **Hardware integration:** Enable `TRANSPORT` and `HARDWARE_CMD` at an available diagnostic level for a bounded run. Correlate commands, native results and hardware responses. Do not infer zero received USB bytes solely from absent decoder logs.
*   **Analysis investigation:** Enable `PROCESSING` and `FITTING` diagnostics where available, and retain the returned fit parameters, ROI and quality flags. A TRACE setting does not promise that a specific solver matrix is emitted.

### Python logging and clean unregister

```python
from nexatomtt import NEXATOM_LOG_MODULE_TRANSPORT, NEXATOM_LOG_DEBUG

# library is already loaded from the matching SDK.
records = []  # Suitable for a short diagnostic; use bounded storage for long runs.
library.set_log_callback(lambda record: records.append(record))
try:
    capabilities = library.get_logging_capabilities()
    library.set_module_enabled(NEXATOM_LOG_MODULE_TRANSPORT, True)
    library.set_module_log_level(NEXATOM_LOG_MODULE_TRANSPORT, NEXATOM_LOG_DEBUG)
    # Perform the bounded operation you are diagnosing here.
    statistics = library.get_logging_statistics()
    print("Dropped log records:", statistics.dropped)
finally:
    # Called from the control thread. Success establishes logging quiescence.
    library.unregister_log_callback(timeout_ms=5000)
```

The binding copies a `NexatomLogRecord` for Python ownership. Keep the handler short: slow consumers can cause queue loss. Statistics expose submitted, emitted, suppressed and dropped records, callback failures, queue occupancy and high-water mark. `get_logging_configuration()` returns a record suitable for editing and passing to `configure_logging()`; query the effective settings after applying a configuration.

On unregister timeout/BUSY/failure, retain the callback's resources and handle the error. Do not unregister or wait for logging quiescence from the log callback itself. `device.clear_callbacks()` only clears device registrations and does not unregister the process-global logger.

`set_performance_monitoring()` is a retained compatibility control, and in this SDK still only that: it stores a flag and logs the setting. It does not add host CPU/throughput statistics to public telemetry. Logging statistics describe the logger itself, not acquisition performance monitoring.
